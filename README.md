# ECK on Talos

[![CI](https://github.com/frederikb96/eck-on-talos/actions/workflows/ci.yaml/badge.svg)](https://github.com/frederikb96/eck-on-talos/actions/workflows/ci.yaml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A hands-on guide to running a **3-node Elastic Stack** on **Talos Linux VMs** using **Elastic Cloud on Kubernetes (ECK)**. The result: Elasticsearch, Kibana, Fleet Server and Elastic Agent — all managed by the Kubernetes operator, running on an immutable OS with essentially zero maintenance burden.

**Why Talos + ECK instead of installing Elasticsearch directly on a Linux VM?**

- **No OS maintenance.** Talos has no SSH, no package manager, no drift. You never patch kernels or run `apt upgrade`. Upgrades are a single declarative command.
- **No hand-tuning Elastic.** The ECK operator reconciles the cluster state from a single YAML file. Version upgrades, JVM settings, TLS, node roles — all declarative. Rolling restarts, certificate rotation and health checks happen automatically.
- **One Git repo is the entire system.** The Talos config, storage layout, CA, and Elastic Stack live in version-controlled files. Nothing lives only on a server.
- **Production patterns out of the box.** Self-monitoring (Stack Monitoring), Elastic Agent with the Kubernetes integration, audit logging, and a proper internal CA are all wired in.

This guide is intentionally opinionated and keeps the moving parts to a minimum. No Terraform, no Flux, no cert-manager, no ingress controller — just `talosctl`, `kubectl` and `helm`.

---

## Table of Contents

- [What you get](#what-you-get)
- [What you don't get](#what-you-dont-get)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1 — Boot Talos on each VM](#step-1--boot-talos-on-each-vm)
- [Step 2 — Locate the nodes and verify disks](#step-2--locate-the-nodes-and-verify-disks)
- [Step 3 — Generate the Talos machine config](#step-3--generate-the-talos-machine-config)
- [Step 4 — Apply the config to each node](#step-4--apply-the-config-to-each-node)
- [Step 5 — Bootstrap the cluster](#step-5--bootstrap-the-cluster)
- [Step 6 — Get kubeconfig and verify](#step-6--get-kubeconfig-and-verify)
- [Step 7 — Create the internal CA](#step-7--create-the-internal-ca)
- [Step 8 — Create namespaces and the CA secret](#step-8--create-namespaces-and-the-ca-secret)
- [Step 9 — Create the StorageClass and PVs](#step-9--create-the-storageclass-and-pvs)
- [Step 10 — Install the ECK operator](#step-10--install-the-eck-operator)
- [Step 11 — Customise values.yaml](#step-11--customise-valuesyaml)
- [Step 12 — Deploy the Elastic stack](#step-12--deploy-the-elastic-stack)
- [Step 13 — Get the elastic user password](#step-13--get-the-elastic-user-password)
- [Step 14 — Access Kibana](#step-14--access-kibana)
- [Step 15 — Trust the CA on clients](#step-15--trust-the-ca-on-clients)
- [Step 16 — Enrolling external Elastic Agents](#step-16--enrolling-external-elastic-agents)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

---

## What you get

- **3 Elasticsearch nodes** with combined master + data + ingest + ml + transform roles
- **2 Kibana replicas** with anti-affinity across nodes
- **2 Fleet Server replicas** (for in-cluster agents and, optionally, external agents via NodePort)
- **Elastic Agent DaemonSet** (one per node) running the Elastic Agent integration plus the Kubernetes integration for full cluster observability
- **Self-monitoring** — Stack Monitoring writes its metrics and logs back into the same cluster, no separate monitoring cluster required
- **Internal CA** you fully control — ECK signs all HTTP certificates with it, so every client only has to trust one certificate
- **Audit logging** enabled on Elasticsearch and Kibana
- **NodePort services** for Elasticsearch (30920), Kibana (30601) and Fleet Server (30822), reachable on any node IP → de-facto HA without a load balancer

## What you don't get

- No searchable snapshot / frozen tier — this guide is a single hot tier
- No ingress controller (Traefik / NGINX / HAProxy) — NodePort is enough for a LAN deployment
- No external load balancer — if you want one, put HAProxy or your cloud LB in front of the three node IPs
- No cert-manager / Let's Encrypt — the internal CA is the source of TLS truth
- No GitOps / Flux / Argo CD — every action in this guide is a manual `kubectl`/`helm` command, and the repo is a reference, not a runtime deployment system
- No multi-tenant Elastic setup — one cluster, one workload

If you need any of the above, the patterns here still work as a baseline — you'll just add tooling on top.

---

## Repository structure

Everything the guide touches lives in one place:

```
eck-on-talos/
├── README.md                          ← THIS file
├── README-template.md                 ← Template for your own private ops repo
├── talos/
│   ├── patches/common.yaml            ← Cluster-wide Talos settings
│   └── nodes/node{1,2,3}.yaml         ← Per-node settings (hostname, IP, disk)
└── kubernetes/
    ├── namespaces.yaml                ← elastic-system + elastic-stack namespaces
    ├── storage/
    │   ├── storageclass.yaml          ← local-storage StorageClass
    │   └── pvs.yaml                   ← 3 local PVs pinned to node1/node2/node3
    ├── eck-operator/values.yaml       ← Helm values for the ECK operator
    └── eck-stack/values.yaml          ← Helm values for ES + Kibana + Fleet + Agent
```

Every one of these files is **small and heavily commented**. Open them as you go — the comments explain what each setting does, why it's set that way, and link to the upstream Elastic documentation. You'll learn more from reading `kubernetes/eck-stack/values.yaml` than from any README.

## Architecture

```
             ┌──────────────────────── LAN ─────────────────────────┐
             │                                                      │
   Client ───┤  https://10.0.0.11:30601                              │
             │  https://10.0.0.12:30601 (Kibana NodePort)            │
             │  https://10.0.0.13:30601                              │
             │                                                      │
             │  https://10.0.0.1X:30920 (Elasticsearch NodePort)     │
             │  https://10.0.0.1X:30822 (Fleet Server NodePort)      │
             └───┬──────────────┬──────────────┬─────────────────────┘
                 │              │              │
              ┌──▼───┐       ┌──▼───┐       ┌──▼───┐
              │node1 │       │node2 │       │node3 │        3× Talos VM
              │      │       │      │       │      │        - CP + worker stacked
              │ ES-1 │       │ ES-2 │       │ ES-3 │        - Flannel (Talos built-in)
              │ Kbn  │       │ Kbn  │       │      │        - local-storage PV
              │ FlSv │       │ FlSv │       │      │
              │ Agt  │       │ Agt  │       │ Agt  │ ◀── DaemonSet
              └──┬───┘       └──┬───┘       └──┬───┘
                 │              │              │
                 └─── /var/mnt/data (dedicated second disk per VM) ──┘
                        ext4, auto-mounted by Talos UserVolumeConfig

  TLS: one internal CA in Secret `eck-ca` (ca.crt + ca.key)
       → ECK signs per-component leaf certs with SANs for node IPs + DNS
       → clients trust one ca.crt, everything works
```

---

## Prerequisites

### Infrastructure

- **3 virtual machines** on the same layer-2 / layer-3 segment. Any hypervisor (KVM / libvirt, Proxmox, VMware, Hyper-V, Nutanix, Azure, GCE, EC2, Hetzner Cloud…) works.
- **Per VM:**
  - ≥ 4 vCPU, ≥ 16 GiB RAM (ideally 32 GiB so Elasticsearch gets a comfortable 8 GiB heap)
  - **Disk 1 (system):** ≥ 32 GiB, Talos OS — you can use the hypervisor's default virtual disk
  - **Disk 2 (data):** ≥ 100 GiB, a second virtual disk dedicated to Elasticsearch data. Talos auto-mounts it at `/var/mnt/data` via `UserVolumeConfig`.
- **Static IPs** on the LAN. This guide assumes `10.0.0.11`, `10.0.0.12`, `10.0.0.13`. Replace them with your own throughout.
- **DNS and default gateway** — Talos needs internet to pull its installer image and the container images the first time. An air-gapped setup is possible but out of scope here.

### Client workstation tools

```bash
talosctl version            # ≥ 1.12
kubectl version --client    # ≥ 1.28
helm version                # ≥ 3.14
openssl version             # 1.1 or 3.x
```

On Debian/Ubuntu:

```bash
curl -sL https://talos.dev/install | sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
  && sudo install kubectl /usr/local/bin/ && rm kubectl
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Clone this repository

```bash
git clone https://github.com/frederikb96/eck-on-talos.git
cd eck-on-talos
```

All commands below assume you're in the repo root.

### Pick your versions

We pin everything the customer has to think about in one place. The defaults are known-good at the time of writing:

```bash
talos_version="v1.12.6"
schematic="ce4c980550d5ca4b12c6951ab2920b663f501547aab525d571e1bfd25111c0be"  # stock, no extensions
```

> Need extra Talos system extensions? Build your own schematic at <https://factory.talos.dev> and swap the ID above.

---

## Step 1 — Boot Talos on each VM

Talos runs directly from a factory-built image. You have two installation paths.

### Option A — ISO boot (recommended, works on any hypervisor that mounts ISOs)

1. Download the Talos ISO for your schematic:
   ```bash
   wget "https://factory.talos.dev/image/${schematic}/${talos_version}/metal-amd64.iso"
   ```
2. Upload the ISO to your hypervisor's image store (Proxmox: "ISO images", VMware: datastore, libvirt: `/var/lib/libvirt/images`, Hetzner Cloud: attach ISO via API/Console).
3. For each VM: set the ISO as the boot device, power on, watch it boot into Talos maintenance mode.
4. Once all three VMs show the Talos dashboard with an IP on their console, unmount the ISO so the next boot uses the installed system disk.

### Option B — `dd` from a rescue system

Use this when your hypervisor doesn't let you attach an ISO but does give you a Linux rescue shell (e.g. Hetzner dedicated, bare-metal boxes).

```bash
cd /tmp
wget "https://factory.talos.dev/image/${schematic}/${talos_version}/metal-amd64.raw.xz"
# Identify the system disk — DOUBLE CHECK THIS, the dd is destructive.
lsblk
# Wipe and write
wipefs -a /dev/sda
xz -dc metal-amd64.raw.xz | dd of=/dev/sda bs=4M status=progress && sync
reboot
```

After reboot the VM comes up in Talos maintenance mode on its primary disk.

### What "maintenance mode" means

Until you apply a machine config, Talos has:

- A DHCP-assigned IP (check your DHCP leases or the hypervisor console)
- An unauthenticated Talos API on port 50000 (that's why every command below uses `--insecure`)
- No Kubernetes yet

---

## Step 2 — Locate the nodes and verify disks

For each VM, find the DHCP IP (call it `$tmp_ip`) and confirm Talos is reachable:

```bash
tmp_ip="192.168.1.50"   # whatever the DHCP server gave the VM
talosctl -n "$tmp_ip" version --insecure
# Expected: "API is not implemented in maintenance mode" → that's the green signal
```

Verify the two disks and the network interface:

```bash
talosctl -n "$tmp_ip" get disks --insecure
talosctl -n "$tmp_ip" get links --insecure
```

**You are looking for:**

- A system disk at `/dev/sda` (or `/dev/vda` on KVM, `/dev/nvme0n1` on NVMe, etc.). Adjust `machine.install.disk` in `talos/nodes/node*.yaml` if yours differs.
- A second, empty data disk (size ≥ 100 GiB). The `UserVolumeConfig` in the same file selects any non-system disk with `!system_disk`, so it does NOT need a fixed path.
- A primary interface named `eth0` on most VMs. If yours is `ens18`, `ens192`, `enp1s0`… update the `interface:` field in `talos/nodes/node*.yaml`.

Repeat for all three VMs. Take notes: which DHCP IP maps to which planned hostname (`node1` → `10.0.0.11`, etc.).

---

## Step 3 — Generate the Talos machine config

From the repo root, run `talosctl gen config` pointing at the first planned IP (not the DHCP IP — Talos embeds this as the cluster endpoint):

```bash
talosctl gen config eck-cluster https://10.0.0.11:6443 \
  --config-patch @talos/patches/common.yaml \
  --output-dir _out
```

This produces:

```
_out/controlplane.yaml    ← the machine config for every node
_out/worker.yaml          ← unused (we stack CP + worker, see common.yaml)
_out/talosconfig          ← client certificate bundle for talosctl
```

Load the talosconfig into your local talosctl client:

```bash
talosctl config merge _out/talosconfig
talosctl config endpoint 10.0.0.11 10.0.0.12 10.0.0.13
talosctl config node 10.0.0.11
```

> The `_out/` directory is gitignored. Keep `_out/talosconfig` safe — it's the root credential for the cluster.

---

## Step 4 — Apply the config to each node

Apply the shared `controlplane.yaml` **plus** the per-node patch to every node. The node's hostname, interface, static IP and data disk mount all come from the patch:

```bash
talosctl apply-config --insecure \
  --nodes <tmp_ip_of_first_vm> \
  --file _out/controlplane.yaml \
  --config-patch @talos/nodes/node1.yaml
```

Talos writes the config to its `STATE` partition, restarts networking, and you'll see the VM's IP switch from its DHCP lease to `10.0.0.11`. Repeat for node2 and node3:

```bash
talosctl apply-config --insecure \
  --nodes <tmp_ip_of_second_vm> \
  --file _out/controlplane.yaml \
  --config-patch @talos/nodes/node2.yaml

talosctl apply-config --insecure \
  --nodes <tmp_ip_of_third_vm> \
  --file _out/controlplane.yaml \
  --config-patch @talos/nodes/node3.yaml
```

Give each node about 60 seconds after apply, then verify it responds on its new static IP:

```bash
talosctl -n 10.0.0.11 version
talosctl -n 10.0.0.12 version
talosctl -n 10.0.0.13 version
```

You should get a normal (non-maintenance) Talos version response.

---

## Step 5 — Bootstrap the cluster

Bootstrap etcd on **node1 only** (running this on more than one node will corrupt etcd):

```bash
talosctl bootstrap --nodes 10.0.0.11 --endpoints 10.0.0.11
```

Wait about a minute for the API server to come up. You can watch progress with:

```bash
talosctl -n 10.0.0.11 dmesg -f
# Ctrl-C when you see "kubernetes bootstrap completed"
```

---

## Step 6 — Get kubeconfig and verify

```bash
talosctl kubeconfig --nodes 10.0.0.11 --endpoints 10.0.0.11 ./kubeconfig
export KUBECONFIG="$PWD/kubeconfig"

kubectl get nodes
# node1   Ready   control-plane   ...
# node2   Ready   control-plane   ...
# node3   Ready   control-plane   ...
```

All three nodes should reach `Ready` within a couple of minutes. If they don't, see [Troubleshooting](#troubleshooting).

> Flannel CNI comes built-in with Talos — you don't install a CNI, don't run `kubectl apply -f cilium.yaml`, don't configure anything. It just works.

---

## Step 7 — Create the internal CA

> **What this step does:** Creates a single Certificate Authority that ECK will use to sign the TLS certificates for Elasticsearch, Kibana and Fleet Server. One CA → three certs → one thing for clients to trust.
>
> **Docs:** [ECK TLS certificates — custom CA](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-tls-certificates.html)

This is the one step that's genuinely different from a vanilla Elastic-on-VMs install: by default ECK generates three separate self-signed CAs (one per component), and your customers would have to trust all three. Instead, we give ECK our own CA, it uses that CA to sign fresh leaf certs for every component, and clients only have to trust the **one** `ca.crt` we give them.

Create a 10-year CA:

```bash
mkdir -p ca && cd ca

openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/CN=ECK Internal CA/O=My Company" \
  -out ca.crt

cd ..
```

> `ca.key` and `ca.crt` are gitignored. Store them somewhere safe — if you lose `ca.key` you can only rotate by regenerating everything.

> Already have an internal company CA? Skip this step. Point `ca.key` / `ca.crt` at the real ones — ECK treats any CA + key pair identically.

---

## Step 8 — Create namespaces and the CA secret

> **What this step does:** Creates the two namespaces the stack lives in (`elastic-system` for the operator, `elastic-stack` for the data plane), then uploads your CA as a Kubernetes Secret so ECK can use it to sign TLS certificates.
>
> **Docs:** [ECK custom certificates (BYO CA)](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-tls-certificates.html) · [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

```bash
kubectl apply -f kubernetes/namespaces.yaml

kubectl -n elastic-stack create secret generic eck-ca \
  --from-file=ca.crt=ca/ca.crt \
  --from-file=ca.key=ca/ca.key
```

**How the BYO CA magic works:** When ECK finds a Secret with BOTH `ca.crt` **and** `ca.key`, it treats it as a signing CA — meaning ECK uses YOUR key to sign fresh leaf certificates for every component (Elasticsearch, Kibana, Fleet Server). Each component gets its own leaf cert, but all three leaves chain back to your single CA. Clients only have to trust **one** `ca.crt` and everything works.

Both namespaces get the `pod-security.kubernetes.io/enforce: privileged` label. This is required because ECK uses a privileged init container to raise `vm.max_map_count`, and the Elastic Agent DaemonSet mounts host paths for container log collection. Without the privileged label, Kubernetes Pod Security Admission rejects those pods. This is a Kubernetes feature, not a Cilium/Talos quirk.

---

## Step 9 — Create the StorageClass and PVs

> **What this step does:** Tells Kubernetes that `/var/mnt/data/elasticsearch` on each node is a storage volume, so Elasticsearch pods can use it for their data.
>
> **Docs:** [Kubernetes Local Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local) · [ECK volumeClaimTemplates](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-volume-claim-templates.html)

We use the Kubernetes native **local volumes** pattern:

| Object | What it does |
|---|---|
| `StorageClass: local-storage` | Says "no dynamic provisioner; volumes are pre-created by hand" and sets `volumeBindingMode: WaitForFirstConsumer` so a PVC only binds once a pod wants to use it. |
| `PersistentVolume: es-data-node1` | A 100 GiB volume pinned to node1 via `nodeAffinity`, pointing at `/var/mnt/data/elasticsearch` on that node. |
| `PersistentVolume: es-data-node2` | Same, pinned to node2. |
| `PersistentVolume: es-data-node3` | Same, pinned to node3. |

```bash
kubectl apply -f kubernetes/storage/storageclass.yaml
kubectl apply -f kubernetes/storage/pvs.yaml

kubectl get storageclass
kubectl get pv
```

Each PV starts in the `Available` phase. When Elasticsearch is deployed (Step 12), the StatefulSet creates three PVCs. Because of `WaitForFirstConsumer`, the PVCs don't bind immediately — they wait until the scheduler picks a node for each pod. The scheduler sees the anti-affinity rules AND the PV nodeAffinities, picks one node per pod, and only then the PVC binds to the matching PV.

**The net effect:** Each Elasticsearch pod is locked to one node forever. Its data lives on that node's local disk. If the pod dies and gets recreated, Kubernetes puts it back on the same node because that's the only place its PVC is bound. Automatic data locality, zero CSI driver, zero Longhorn/Ceph/Mayastor.

> 🛈 **Need bigger disks?** Edit `kubernetes/storage/pvs.yaml` and bump `storage: 100Gi` to whatever your data disks allow. Do this **before** deploying the stack — you cannot grow a local PV after creation. You can always add a second PV per node and run a second ES nodeSet that binds to it.

---

## Step 10 — Install the ECK operator

> **What this step does:** Installs the ECK operator — a single Deployment that watches Elasticsearch/Kibana/Agent/Fleet Server custom resources and reconciles them into working StatefulSets and Services.
>
> **Docs:** [Install ECK using Helm](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-install-helm.html) · [ECK overview](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm upgrade --install eck-operator elastic/eck-operator \
  --version 3.3.1 \
  --namespace elastic-system \
  --values kubernetes/eck-operator/values.yaml

kubectl -n elastic-system rollout status deploy/elastic-operator --timeout=2m
```

The operator pod should reach `1/1 Running` in under a minute. From this point on, everything you want in the cluster is declared as a YAML resource (`kind: Elasticsearch`, `kind: Kibana`, etc.) and the operator does the heavy lifting — no more `kubectl apply` of StatefulSets, no more manually-crafted ConfigMaps.

**What the operator does for you** (so you don't have to):
- Generates all TLS certificates for transport and HTTP layers
- Manages the Elasticsearch cluster state (rolling upgrades, shard rebalancing during node changes)
- Creates the `elastic`, `kibana_system`, and service-account users automatically
- Injects readiness and liveness probes that actually understand Elasticsearch
- Handles version upgrades (drain → restart → wait for green → next node)
- Rotates certificates before they expire (for ECK-generated ones — NOT your BYO CA)

---

## Step 11 — Customise values.yaml

> **What this step does:** Replaces the placeholder IPs and DNS names in `kubernetes/eck-stack/values.yaml` with your real ones before deploying.

The file is heavily commented — every non-trivial setting has an inline explanation and a link to the relevant Elastic docs. Open it, skim through to get a feel for what the stack looks like, then do the placeholder substitution:

| Placeholder | Replace with |
|---|---|
| `10.0.0.11` | Your **node1** IP |
| `10.0.0.12` | Your **node2** IP |
| `10.0.0.13` | Your **node3** IP |
| `elastic.lan` | DNS name you'll use for Elasticsearch, or delete this `dns:` line |
| `kibana.lan` | DNS name you'll use for Kibana, or delete this `dns:` line |
| `fleet.lan` | DNS name you'll use for Fleet Server, or delete this `dns:` line |

> 🛈 **Why the DNS names matter:** They become `subjectAltName` entries on the TLS leaf certificates ECK signs with your CA. When a client connects to `https://elastic.lan:30920`, TLS validation checks whether `elastic.lan` is in the cert's SAN list. If it's not, the client rejects the connection. Adding DNS names here means you (or your customer) can later point a real DNS record at the node IPs and nothing else has to change.

### What's pre-tuned (so you don't have to think about it)

| Setting | Value | Why |
|---|---|---|
| ES JVM heap | `-Xms4g -Xmx4g` | Half of the 8 GiB container memory — the standard Elasticsearch rule of thumb. Raise this if you give ES more memory. |
| ES memory request == limit | `8Gi` == `8Gi` | Elasticsearch is allergic to memory pressure. Request==limit gives it "Guaranteed" QoS and locks the allocation. |
| Elasticsearch audit logging | ON | Records auth attempts, permission denials, config changes. Searchable in Kibana. |
| Kibana self-monitoring | ON | Stack Monitoring data ships back into the same cluster — one UI to rule them all. |
| Fleet Server replicas | 2 | HA, spread across two nodes via anti-affinity. |
| Elastic Agent Kubernetes integration | ON | Cluster-wide observability on every node by default. |
| Pod anti-affinity | Enforced (required) | Kubernetes scheduler refuses to place two pods of the same type on the same node. |

### Sizing for your VMs

The defaults assume **16 GiB per VM**. If your VMs are bigger or smaller, adjust these values:

| VM memory | ES memory (request=limit) | ES `ES_JAVA_OPTS` |
|---|---|---|
| 8 GiB | 4Gi | `-Xms2g -Xmx2g` |
| 16 GiB | 8Gi | `-Xms4g -Xmx4g` ← **default** |
| 32 GiB | 16Gi | `-Xms8g -Xmx8g` |
| 64 GiB | 31Gi | `-Xms30g -Xmx30g` |

> 🛈 **Never give Elasticsearch more than ~31 GiB heap.** Above that threshold the JVM loses "compressed ordinary object pointers" (compressed oops) and each pointer doubles in size — you'd actually have *less* effective heap.

---

## Step 12 — Deploy the Elastic stack

> **What this step does:** Tells Helm to create Elasticsearch, Kibana, Fleet Server and Agent custom resources in the cluster. The ECK operator picks them up and reconciles them into the actual running stack.
>
> **Docs:** [ECK quickstart](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html) · [Fleet quickstart](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-elastic-agent-fleet-quickstart.html) · [Kibana Fleet preconfiguration](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-elastic-agent-fleet-configuration.html)

```bash
helm upgrade --install eck-stack elastic/eck-stack \
  --version 0.18.1 \
  --namespace elastic-stack \
  --values kubernetes/eck-stack/values.yaml \
  --timeout 20m
```

The first deploy takes a few minutes. Elasticsearch pods boot sequentially because they need to form the initial cluster state, then Kibana waits for Elasticsearch to be green, then Fleet Server waits for Kibana's Fleet API to be reachable, then the Elastic Agent DaemonSet waits for Fleet Server. Watch progress in a second terminal:

```bash
watch kubectl -n elastic-stack get pods,pvc,elasticsearch,kibana,agent
```

Expected end state (your pod names will be slightly different):

```
NAME                     READY   STATUS    RESTARTS   AGE
elasticsearch-es-default-0   1/1   Running   0   5m
elasticsearch-es-default-1   1/1   Running   0   4m
elasticsearch-es-default-2   1/1   Running   0   3m
kibana-kb-<hash>             1/1   Running   0   3m
kibana-kb-<hash>             1/1   Running   0   3m
fleet-server-agent-<hash>    1/1   Running   0   2m
fleet-server-agent-<hash>    1/1   Running   0   2m
elastic-agent-agent-<hash>   1/1   Running   0   1m
elastic-agent-agent-<hash>   1/1   Running   0   1m
elastic-agent-agent-<hash>   1/1   Running   0   1m

NAME                  STATUS   VOLUME           CAPACITY   STORAGECLASS
elasticsearch-data-elasticsearch-es-default-0   Bound   es-data-nodeX   100Gi   local-storage
elasticsearch-data-elasticsearch-es-default-1   Bound   es-data-nodeY   100Gi   local-storage
elasticsearch-data-elasticsearch-es-default-2   Bound   es-data-nodeZ   100Gi   local-storage

NAME                HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch       green    3       9.3.2     Ready   5m

NAME       HEALTH   NODES   VERSION   AGE
kibana     green    2       9.3.2     4m
```

---

## Step 13 — Get the elastic user password

ECK auto-creates the `elastic` superuser and stores its password in a secret:

```bash
kubectl -n elastic-stack get secret elasticsearch-es-elastic-user \
  -o go-template='{{.data.elastic | base64decode}}{{"\n"}}'
```

Copy it. You'll need it to log in to Kibana and to call Elasticsearch directly with `curl`.

---

## Step 14 — Access Kibana

### Via NodePort (production access)

Open your browser at any of:

- `https://10.0.0.11:30601`
- `https://10.0.0.12:30601`
- `https://10.0.0.13:30601`

Log in with username `elastic` and the password from Step 13.

The browser will warn about the certificate because your internal CA is not in the system trust store yet — that's Step 15. You can click through the warning for now.

### Via kubectl port-forward (dev / testing)

If your workstation can't reach the node IPs directly (corporate firewalls, VPN quirks):

```bash
kubectl -n elastic-stack port-forward svc/kibana-kb-http 5601:5601
# browse to https://localhost:5601
```

This is convenient for first-time exploration but not a production pattern — it ties Kibana availability to your local `kubectl` session.

---

## Step 15 — Trust the CA on clients

Distribute `ca/ca.crt` to everything that talks to the cluster:

**Linux client (system-wide trust, Debian/Ubuntu):**
```bash
sudo cp ca/ca.crt /usr/local/share/ca-certificates/eck-internal-ca.crt
sudo update-ca-certificates
```

**Linux client (system-wide trust, RHEL/Rocky):**
```bash
sudo cp ca/ca.crt /etc/pki/ca-trust/source/anchors/eck-internal-ca.crt
sudo update-ca-trust
```

**Firefox:** Settings → Privacy & Security → Certificates → View Certificates → Import → select `ca.crt` → check "Trust this CA to identify websites".

**Windows:** Double-click `ca.crt` → Install Certificate → Local Machine → Trusted Root Certification Authorities.

**curl / bash scripts:**
```bash
curl --cacert ca/ca.crt -u "elastic:${elastic_pw}" "https://10.0.0.11:30920/_cluster/health?pretty"
```

**Elasticsearch client libraries** (Python `elasticsearch`, Go `elastic/go-elasticsearch`, etc.): pass the `ca.crt` path as the CA bundle.

After trust is set up, browser warnings disappear and every `curl` works without `--insecure`.

---

## Step 16 — Enrolling external Elastic Agents

Agents running **inside** the cluster are already enrolled via the `eck-agent` policy — no action needed. To enrol an agent **outside** the cluster (a laptop, a server in another subnet):

```bash
# From Kibana → Fleet → Agent policies → "Elastic Agent on ECK policy" → Add agent
#   → copy the enrollment token

sudo elastic-agent install \
  --url=https://10.0.0.11:30822 \
  --enrollment-token=${token} \
  --certificate-authorities=ca/ca.crt
```

The `--certificate-authorities` flag tells the agent to trust your internal CA, so the TLS handshake with Fleet Server succeeds. No `--insecure` needed, no custom SAN work required beyond what ECK already baked into the cert.

---

## Maintenance

### Upgrading Talos

Always one node at a time. Flannel is bundled with Talos — it upgrades with the OS automatically.

```bash
talos_version="v1.12.7"
image="factory.talos.dev/installer/${schematic}:${talos_version}"

# Drain node1, upgrade, uncordon
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data --force --disable-eviction --timeout=180s
talosctl upgrade --nodes 10.0.0.11 --image "$image" --preserve --reboot-mode powercycle --wait=false

# Wait for the node to come back
while ! talosctl -n 10.0.0.11 version 2>&1 | grep -q "v${talos_version#v}"; do sleep 5; done
kubectl wait --for=condition=Ready node/node1 --timeout=5m
kubectl uncordon node1

# Repeat for node2 and node3
```

> **Always use `--reboot-mode powercycle`.** The default kexec path leaves some Intel NICs (notably the I219 series on Hetzner bare metal) in an unusable state. `powercycle` forces a full BIOS reboot and re-initialises the NIC. On hypervisor VMs this is virtually free.

### Upgrading the ECK operator

```bash
helm repo update
helm upgrade eck-operator elastic/eck-operator \
  --version 3.4.0 \
  --namespace elastic-system \
  --values kubernetes/eck-operator/values.yaml
```

The operator does a rolling replace of itself. CRDs are updated automatically. No impact on Elasticsearch/Kibana/Fleet Server during the operator upgrade itself — they only get touched if the operator picks up new reconciliation logic.

### Upgrading the Elastic Stack

Bump every `version:` field in `kubernetes/eck-stack/values.yaml` (there are four: `eck-elasticsearch`, `eck-kibana`, `eck-fleet-server`, `eck-agent`) to the new Elastic version, then:

```bash
helm upgrade eck-stack elastic/eck-stack \
  --version 0.18.1 \
  --namespace elastic-stack \
  --values kubernetes/eck-stack/values.yaml
```

ECK handles the rolling upgrade for you:

- Elasticsearch nodes are restarted one by one respecting shard allocation
- Kibana replicas are rolled in sequence
- Fleet Server replicas are rolled
- Elastic Agent DaemonSet pods are rolled

Total upgrade time for a 3-node cluster: 10–20 minutes depending on data volume.

### Rotating the internal CA

The CA you created in Step 7 is valid for 10 years. **ECK does NOT auto-renew user-provided CAs** — mark the expiry in your calendar and rotate before it expires.

```bash
# Generate a new CA next to the old one
cd ca
openssl genrsa -out ca-new.key 4096
openssl req -x509 -new -nodes -key ca-new.key -sha256 -days 3650 \
  -subj "/CN=ECK Internal CA/O=My Company" \
  -out ca-new.crt
cd ..

# Replace the secret
kubectl -n elastic-stack create secret generic eck-ca \
  --from-file=ca.crt=ca/ca-new.crt \
  --from-file=ca.key=ca/ca-new.key \
  --dry-run=client -o yaml | kubectl apply -f -
```

ECK detects the new CA, re-signs all leaf certs, and triggers a rolling restart of Elasticsearch, Kibana, Fleet Server and Agent (so they pick up the new cert chain). Distribute `ca-new.crt` to all clients before or during the rotation.

---

## Troubleshooting

**Node stays `NotReady`, kubelet logs complain about CNI.**
Talos's built-in Flannel needs cluster networking to come up. Check `talosctl -n <ip> dmesg -f` for errors. Most often this is a wrong interface name in your node patch — Talos tries to bind to the configured interface, can't, and stays stuck.

**PVC stuck in `Pending`, events say "waiting for first consumer".**
This is the *normal* state for `WaitForFirstConsumer`. The PVC binds when the Elasticsearch pod schedules on the matching node. If it's still pending after 5 minutes, check the pod — it's probably failing to schedule for a different reason (see below).

**Elasticsearch pod stuck `Pending` with "no nodes match selector".**
A PV nodeAffinity doesn't match any node's `kubernetes.io/hostname` label. Check:
```bash
kubectl get nodes --show-labels | grep hostname
kubectl get pv -o yaml | grep -A3 nodeAffinity
```

**Elasticsearch pod `CrashLoopBackOff` with `max_map_count` error.**
The init container didn't run. Check that you're using the `values.yaml` shipped with this repo — the `initContainers` block must be present.

**Elasticsearch pod `CrashLoopBackOff` with "permission denied" on the data dir.**
The `fsGroup: 1000` wasn't applied, or `/var/mnt/data/elasticsearch` on the host doesn't exist. Talos's `UserVolumeConfig` auto-creates the mount point; verify with `talosctl -n <ip> ls /var/mnt/data`.

**Kibana pods flap between Ready and NotReady.**
Kibana aggressively health-checks Elasticsearch. Make sure all three ES pods show `elasticsearch green 3`. Kibana stops flapping as soon as the ES cluster goes green.

**Browser says "certificate is not trusted" even after I imported `ca.crt`.**
Check you're connecting to an IP or DNS name that's in `subjectAltNames` in `kubernetes/eck-stack/values.yaml`. If you connect to `https://10.0.0.11:30601` but only `10.0.0.11` was in the SAN list for Kibana (not `kibana.lan`), that's expected. Clear the browser's cert cache after adding new SANs and restart `helm upgrade`.

**`talosctl` says "connection refused" after apply-config.**
Normal during the 60-second window while Talos restarts networking to apply the static IP. Wait it out and target the new IP, not the DHCP one.

**"no such host" when Kibana tries to reach `elasticsearch-es-http.elastic-stack.svc`.**
CoreDNS isn't healthy. `kubectl -n kube-system get pods -l k8s-app=kube-dns` should show 2 running pods. If not, check node readiness — CoreDNS needs at least one worker.

**Stack Monitoring shows no data.**
Look at the Metricbeat sidecar log: `kubectl -n elastic-stack logs <es-pod> -c metricbeat`. Most commonly the CA cert inside the monitoring pipeline is wrong — ECK regenerates this automatically when you rotate the CA, so a rolling restart of ES fixes it.

**External Elastic Agent won't enroll: "x509: certificate signed by unknown authority".**
You forgot `--certificate-authorities=ca/ca.crt` on the `elastic-agent install` command.

---

## FAQ

**Why not Cilium?**
Cilium is great, but it's not "minimal". For a single-cluster, single-workload LAN deployment, Talos's built-in Flannel has zero operational burden and zero configuration. Cilium starts to pay off when you want network policies, observability, or BGP — none of which this guide wants.

**Why hostPath and NodePort instead of "proper" ingress and CSI?**
Same reason. Every "proper" component adds:
- A Helm chart to maintain
- A controller pod to watch
- An upgrade story
- A failure mode

For three nodes running one Elasticsearch cluster on a LAN, NodePort gives you de-facto HA across all three nodes with zero extra components. Local PVs pin data to a node, which is exactly what Elasticsearch's replication model already assumes.

**Can I add a 4th node later?**
Yes. Create `talos/nodes/node4.yaml`, run `talosctl apply-config --insecure --file _out/controlplane.yaml --config-patch @talos/nodes/node4.yaml --nodes <tmp_ip>`. Then add a 4th PV to `kubernetes/storage/pvs.yaml`, bump the Elasticsearch `nodeSet` count to 4, add the 4th IP to every `subjectAltNames` block, and `helm upgrade`. Data will rebalance automatically.

**Can I run the monitoring cluster separately?**
Yes. Remove the `monitoring:` blocks from `eck-elasticsearch` and `eck-kibana` and point them at a separate ECK cluster's `elasticsearchRefs`. Elastic's official recommendation is a dedicated monitoring cluster, but for small deployments self-monitoring into the same cluster is the pragmatic default.

**Does this work on ARM64?**
Yes. Download the `metal-arm64.raw.xz` / `metal-arm64.iso` from `https://factory.talos.dev/image/${schematic}/${talos_version}/metal-arm64.iso`. The Elastic container images are multi-arch.

**Can I use this on Hyperscaler VMs (EC2, GCE, Azure)?**
Yes — each of them lets you attach a second data disk and boot a custom image. The ISO path is usually easier. DNS names from cloud metadata often work in place of static IPs.

**Why not cert-manager + Let's Encrypt?**
For a LAN deployment the nodes don't have publicly resolvable DNS, so ACME HTTP-01 and DNS-01 don't apply. Your internal CA is simpler, faster, and you don't depend on a 90-day renewal cycle.

**What if I already have an internal company CA?**
Skip Step 7 entirely. Drop your existing `ca.crt` and `ca.key` into the secret (Step 8). ECK signs leaf certs with whatever CA you give it.

**Can I turn off audit logging to reduce disk usage?**
Remove `xpack.security.audit.enabled: true` from `eck-elasticsearch.nodeSets[0].config` and `eck-kibana.config`. Audit logs go into Elasticsearch itself and contribute a small but steady index growth — keep them unless you're really tight on disk.

---

## License

MIT — see [LICENSE](LICENSE).
