# <img src="assets/icon.png" alt="" height="56" valign="middle"> ECK on Talos

[![CI](https://github.com/frederikb96/eck-on-talos/actions/workflows/ci.yaml/badge.svg)](https://github.com/frederikb96/eck-on-talos/actions/workflows/ci.yaml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A hands-on guide to running a **production-ready, easy-to-maintain 3-node Elastic Stack** on **Talos Linux VMs** using **Elastic Cloud on Kubernetes (ECK)**. The result: Elasticsearch, Kibana, Fleet Server and Elastic Agent — all managed by the Kubernetes operator, running on an immutable OS with essentially zero maintenance burden.

> 🧪 **Tested end-to-end with 3 Azure VMs.** Every step in this guide was walked through by a fresh user on a new cluster before publishing, specifically to catch the "wait, what do I click here?" moments. A very similar setup has been running in production for multiple years, so the architecture is not experimental — just documented here in its most minimal, most teachable form.

> ☁️ **Want to try it out in the cloud in 30 minutes?** See [README-azure.md](README-azure.md) for a click-through guide that provisions 3 Talos VMs on Azure from the official Talos VHD image. Once your VMs report "maintenance mode", jump to [Step 2](#step-2--locate-the-nodes-and-verify-disks) of this guide and keep going.

**Why Talos + ECK instead of installing Elasticsearch directly on a Linux VM?**

- **No OS maintenance.** Talos has no SSH, no package manager, no drift. You never patch kernels or run `apt upgrade`. Upgrades are a single declarative command.
- **No hand-tuning Elastic.** The ECK operator reconciles the cluster state from a single YAML file. Version upgrades, JVM settings, TLS, node roles — all declarative. Rolling restarts, certificate rotation and health checks happen automatically.
- **One Git repo is the entire system.** The Talos config, storage layout, CA, and Elastic Stack live in version-controlled files. Nothing lives only on a server.
- **Production patterns out of the box.** Self-monitoring (Stack Monitoring), Elastic Agent with the Kubernetes integration, audit logging, and a proper internal CA are all wired in.

This guide is intentionally opinionated and keeps the moving parts to a minimum. No Terraform, no Flux, no cert-manager, no ingress controller — just `talosctl`, `kubectl` and `helm`.

---

## Table of Contents

- [ ECK on Talos](#-eck-on-talos)
  - [Table of Contents](#table-of-contents)
  - [What you get](#what-you-get)
  - [Optional extensions (not part of this guide, but easy to add later)](#optional-extensions-not-part-of-this-guide-but-easy-to-add-later)
  - [Repository structure](#repository-structure)
  - [Architecture](#architecture)
  - [Prerequisites](#prerequisites)
    - [Infrastructure](#infrastructure)
    - [Sizing the VMs (RAM budget per node)](#sizing-the-vms-ram-budget-per-node)
    - [Client workstation tools](#client-workstation-tools)
    - [Clone this repository](#clone-this-repository)
    - [Pick your Talos version](#pick-your-talos-version)
  - [Step 1 — Boot Talos on each VM](#step-1--boot-talos-on-each-vm)
    - [Option A — ISO boot (recommended, works on any hypervisor that mounts ISOs)](#option-a--iso-boot-recommended-works-on-any-hypervisor-that-mounts-isos)
    - [Option B — `dd` from a rescue system](#option-b--dd-from-a-rescue-system)
    - [What "maintenance mode" means](#what-maintenance-mode-means)
  - [Step 2 — Locate the nodes and verify disks](#step-2--locate-the-nodes-and-verify-disks)
    - [🚨 Mandatory — pin the OS disk by a stable identifier](#-mandatory--pin-the-os-disk-by-a-stable-identifier)
    - [Data disk and network interface](#data-disk-and-network-interface)
  - [Step 3 — Generate the Talos machine config](#step-3--generate-the-talos-machine-config)
  - [Step 4 — Apply the config to each node](#step-4--apply-the-config-to-each-node)
  - [Step 5 — Bootstrap the cluster](#step-5--bootstrap-the-cluster)
  - [Step 6 — Get kubeconfig and verify](#step-6--get-kubeconfig-and-verify)
  - [Step 7 — Create the internal CA](#step-7--create-the-internal-ca)
  - [Step 8 — Create namespaces and the CA secret](#step-8--create-namespaces-and-the-ca-secret)
  - [Step 9 — Create the StorageClass and PVs](#step-9--create-the-storageclass-and-pvs)
  - [Step 10 — Install the ECK operator](#step-10--install-the-eck-operator)
  - [Step 11 — Customise values.yaml](#step-11--customise-valuesyaml)
    - [What's pre-tuned (so you don't have to think about it)](#whats-pre-tuned-so-you-dont-have-to-think-about-it)
  - [Step 12 — Deploy the Elastic stack](#step-12--deploy-the-elastic-stack)
  - [Step 13 — Get the elastic user password](#step-13--get-the-elastic-user-password)
  - [Step 14 — Access Kibana](#step-14--access-kibana)
    - [Via NodePort (production access)](#via-nodeport-production-access)
    - [How NodePort routing actually works](#how-nodeport-routing-actually-works)
    - [Via kubectl port-forward (dev / testing)](#via-kubectl-port-forward-dev--testing)
    - [🚨 First thing to do in Kibana — set ILM policies for the observability data](#-first-thing-to-do-in-kibana--set-ilm-policies-for-the-observability-data)
  - [Step 15 — Trust the CA on clients](#step-15--trust-the-ca-on-clients)
  - [Step 16 — Enrolling external Elastic Agents](#step-16--enrolling-external-elastic-agents)
  - [Maintenance](#maintenance)
    - [Upgrading Talos](#upgrading-talos)
      - [What each `kubectl drain` flag does](#what-each-kubectl-drain-flag-does)
      - [`--reboot-mode powercycle`](#--reboot-mode-powercycle)
    - [Upgrading the ECK operator](#upgrading-the-eck-operator)
    - [Upgrading the Elastic Stack](#upgrading-the-elastic-stack)
    - [Changing a configuration value (Kibana / Elasticsearch)](#changing-a-configuration-value-kibana--elasticsearch)
    - [Adding secrets (the Elasticsearch keystore)](#adding-secrets-the-elasticsearch-keystore)
    - [Adding an S3 snapshot repository](#adding-an-s3-snapshot-repository)
    - [Rotating the internal CA](#rotating-the-internal-ca)
  - [Troubleshooting](#troubleshooting)
  - [FAQ](#faq)
  - [License](#license)

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

## Optional extensions (not part of this guide, but easy to add later)

The philosophy of this guide is **start minimal, grow into complexity**. Everything below is something you _could_ add on top of the baseline setup. None of it is required to run a healthy Elastic Stack — and adding all of it upfront would bury the important ideas under tooling noise. Come back to this list once the basic stack is running and you know what you actually need.

- **Searchable snapshot / frozen tier.** This guide deploys a single hot tier (all 3 nodes hold the same kind of data). Adding a frozen tier later is mostly an Elasticsearch-level change: define a snapshot repository (see [Maintenance → Adding an S3 snapshot repository](#adding-an-s3-snapshot-repository) below for a worked example), and optionally define dedicated frozen nodes in a second `nodeSet`. The Talos + ECK foundation doesn't change.
- **External load balancer.** NodePort already gives you LAN-level HA because any client can hit any node IP and reach any replica (see [NodePort vs pod placement](#how-nodeport-routing-actually-works) below). If you want a single DNS name or you're serving traffic from outside the LAN, put HAProxy / your cloud LB in front of the three node IPs on ports 30920 / 30601 / 30822.
- **Ingress controller** (Traefik / NGINX / HAProxy-Ingress). Same logic — for a LAN deployment NodePort is enough. Add an ingress controller only if you're already running one cluster-wide, or you need path-based routing / host-based routing / advanced TLS termination.
- **cert-manager + Let's Encrypt.** Your internal CA is simpler and has no renewal cycle. Add cert-manager only if the cluster will serve publicly-routable DNS names and you actually want public CA-signed certs.
- **GitOps (Flux / Argo CD).** For a single cluster running a single workload, Flux is complete overkill. Its value comes from managing _many_ things across _many_ clusters. This repo stays intentionally Helm-CLI-driven — simple enough that a newcomer can follow it step by step, but still declarative enough that every change is a git commit.

Everything listed here has been deliberately left out to keep the moving parts minimal. The baseline setup is fully production-capable on its own.

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

![3-node ECK on Talos architecture diagram](assets/architecture.png)

Three Talos VMs on the same LAN, each one a stacked Kubernetes control-plane + worker. Clients reach the services via NodePort (30920 Elasticsearch, 30601 Kibana, 30822 Fleet Server) — kube-proxy on any node load-balances to any available pod. Every node runs an Elastic Agent (DaemonSet) for cluster-wide observability; nodes 1 and 2 also run Kibana and Fleet Server (`count: 2` with anti-affinity), while node3 is ES + Agent only. Elasticsearch data lives on each node's second disk (`/var/mnt/data`) bound as a native Kubernetes local Persistent Volume. A single internal CA (stored in the `eck-ca` Secret) is used by ECK to sign the TLS leaf certificates for Elasticsearch, Kibana and Fleet Server — clients trust one `ca.crt` and get valid TLS on every component. Flannel CNI ships built into Talos, so there's no CNI to install or maintain.

---

## Prerequisites

### Infrastructure

- **3 virtual machines** on the same layer-2 / layer-3 segment. Any hypervisor (KVM / libvirt, Proxmox, VMware, Hyper-V, Nutanix, Azure, GCE, EC2, Hetzner Cloud…) works.
- **Per VM:**
  - ≥ 4 vCPU, **≥ 16 GiB RAM** (16 GiB is the recommended minimum — see sizing below)
  - **Disk 1 (system):** ≥ 32 GiB, Talos OS — you can use the hypervisor's default virtual disk
  - **Disk 2 (data):** ≥ 100 GiB, a second virtual disk dedicated to Elasticsearch data. Talos auto-mounts it at `/var/mnt/data` via `UserVolumeConfig`.
- **Static IPs** on the LAN. This guide assumes `10.0.0.11`, `10.0.0.12`, `10.0.0.13`. Replace them with your own throughout.
- **DNS and default gateway** — Talos needs internet to pull its installer image and the container images the first time. An air-gapped setup is possible but out of scope here.

### Sizing the VMs (RAM budget per node)

**16 GiB per VM is the recommended minimum.** Here's the budget for the two busiest nodes (nodes 1 and 2 — they run Kibana and Fleet Server in addition to Elasticsearch):

| Component | Planned memory | Runs on |
|---|---|---|
| Elasticsearch | 8 GiB (locked, request == limit) | every node (1 ES pod per node) |
| Kibana | up to 2 GiB (request 1 GiB, no limit — bursts freely) | 2 of 3 nodes |
| Fleet Server | up to 2 GiB (request 1 GiB, no limit — bursts freely) | 2 of 3 nodes |
| Elastic Agent (DaemonSet) | up to 2 GiB (request 1 GiB, no limit — bursts freely) | every node |
| Talos OS + kubelet | ~2 GiB | every node |
| **Peak footprint on a busy node** | **≈ 16 GiB** | nodes 1 and 2 — node 3 has ~4 GiB headroom |

**Why 16 GiB is the floor:** drop below that and you start eating into Elasticsearch's heap budget or Lucene's off-heap file cache, which hurts search/indexing performance dramatically.

**Want more headroom?** Raise `resources.limits.memory` for Elasticsearch in `kubernetes/eck-stack/values.yaml`:

- **32 GiB VMs** → set ES memory to 16 GiB. Elasticsearch auto-sizes the JVM heap from the container memory limit (no `ES_JAVA_OPTS` to touch), so heap goes to ~8 GiB automatically.
- **64 GiB VMs** → set ES memory to 31 GiB (the highest useful heap — above ~31 GiB the JVM loses compressed ordinary object pointers and gets _less_ efficient, not more).

Going **above 64 GiB per node** for a single-ES-per-node layout like this one is wasted money. If you need more capacity, add nodes, not RAM.

### Client workstation tools

```bash
talosctl version            # ≥ 1.12
kubectl version --client    # ≥ 1.28
helm version                # ≥ 3.14
openssl version             # 1.1 or 3.x
jq --version                # any recent version
```

Install commands that work on any Debian/Ubuntu box. All binaries come from their upstream release pages — no package manager gymnastics needed:

```bash
# talosctl — direct binary from the siderolabs/talos GitHub releases
curl -Lo talosctl https://github.com/siderolabs/talos/releases/latest/download/talosctl-linux-amd64
chmod +x talosctl && sudo install talosctl /usr/local/bin/ && rm talosctl

# kubectl — official upstream release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/ && rm kubectl

# helm — official install script
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# openssl + jq are almost certainly already installed — if not: sudo apt install openssl jq
```

> 🛈 **Homebrew alternative**: if you use Homebrew on Linux, `brew install siderolabs/tap/talosctl kubernetes-cli helm jq` covers everything in one command. See the [talosctl install docs](https://www.talos.dev/latest/talos-guides/install/talosctl/) for macOS / other platforms.

### Clone this repository

```bash
git clone https://github.com/frederikb96/eck-on-talos.git
cd eck-on-talos
```

All commands below assume you're in the repo root.

### Pick your Talos version

```bash
talos_version="v1.12.6"
```

> 🛈 **About Talos schematics** (advanced, optional): Talos lets you add kernel modules or system extensions (ZFS, tailscale, iscsi-tools, …) via a "schematic" ID from [factory.talos.dev](https://factory.talos.dev). **This guide does NOT need any extensions** — the stock Talos image has everything ECK requires. Only set a `schematic` variable if you know you need a specific extension, and then prepend `${schematic}/` to the image URLs below.

---

## Step 1 — Boot Talos on each VM

Talos runs directly from a factory-built image. You have two installation paths.

### Option A — ISO boot (recommended, works on any hypervisor that mounts ISOs)

1. Download the stock Talos ISO directly from the siderolabs GitHub releases:
   ```bash
   wget "https://github.com/siderolabs/talos/releases/download/${talos_version}/metal-amd64.iso"
   ```
   (Or, if you want a custom schematic with extensions: `wget "https://factory.talos.dev/image/<your-schematic-id>/${talos_version}/metal-amd64.iso"`.)
2. Upload the ISO to your hypervisor's image store (Proxmox: "ISO images", VMware: datastore, libvirt: `/var/lib/libvirt/images`, Hetzner Cloud: attach ISO via API/Console).
3. For each VM: set the ISO as the boot device, power on, watch it boot into Talos maintenance mode.
4. Once all three VMs show the Talos dashboard with an IP on their console, unmount the ISO so the next boot uses the installed system disk.

### Option B — `dd` from a rescue system

Use this when your hypervisor doesn't let you attach an ISO but does give you a Linux rescue shell (e.g. Hetzner dedicated, bare-metal boxes).

```bash
cd /tmp
wget "https://github.com/siderolabs/talos/releases/download/${talos_version}/metal-amd64.raw.xz"
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

Find the three DHCP IPs (check your DHCP leases or hypervisor console), then export them — every command in Step 2 and Step 4 reuses these three variables:

```bash
tmp_ip_1="192.168.1.50"   # whatever the DHCP server gave VM1 → will become 10.0.0.11
tmp_ip_2="192.168.1.51"   # VM2 → 10.0.0.12
tmp_ip_3="192.168.1.52"   # VM3 → 10.0.0.13
```

Confirm all three are reachable in maintenance mode:

```bash
for ip in "$tmp_ip_1" "$tmp_ip_2" "$tmp_ip_3"; do
  talosctl -n "$ip" version --insecure
done
# Expected: each prints "API is not implemented in maintenance mode" — that's the green signal
```

Sanity-check what each VM sees — exactly two real disks (ignore `loop0`/`loop1`) and one network interface:

```bash
for ip in "$tmp_ip_1" "$tmp_ip_2" "$tmp_ip_3"; do
  echo "=== $ip ==="
  talosctl -n "$ip" get disks --insecure
  talosctl -n "$ip" get links --insecure
done
```

### 🚨 Mandatory — pin the OS disk by a stable identifier

Short device names (`/dev/sda`, `/dev/vda`, `/dev/nvme0n1`) are **not stable** — the letter depends on platform, kernel, and can change between reboots. Even identical VMs in the same cloud cluster can enumerate differently. If `machine.install` points at the wrong letter, a future `talosctl upgrade` will reinstall Talos onto your **data disk** and wipe it.

**Prefer `serial` on bare metal?** `machine.install.diskSelector` also accepts `serial`, `uuid`, `model`, `busPath`. Physical NVMe/SATA drives usually have a shorter `serial:` — swap `wwid:` for `serial:` in the YAML. Cloud VMs often have empty `serial`, so `wwid` is the safe universal default. Run `talosctl get disks <id> --insecure -o yaml` on any node to see every field Talos surfaces.

Pin the install target by `wwid` (hardware ID that never changes). Run this to print the wwid of each node's current OS disk:

```bash
for ip in "$tmp_ip_1" "$tmp_ip_2" "$tmp_ip_3"; do
  sysdisk=$(talosctl -n "$ip" get systemdisk --insecure -o json | jq -r .spec.diskID)
  wwid=$(talosctl -n "$ip" get disks "$sysdisk" --insecure -o json | jq -r .spec.wwid)
  printf "%-16s  %-10s  %s\n" "$ip" "$sysdisk" "$wwid"
done
```

Example output:

```text
10.0.0.11         sdb         naa.6002248059bddfe77ab4c2f59bca39e4
10.0.0.12         sda         naa.600224800526fac1cb6de3faaa4744bd
10.0.0.13         sda         naa.6002248071cc72055d2a33429c539d4f
```

Now **manually** paste each node's wwid into the matching `talos/nodes/node<N>.yaml`, replacing the placeholder:

```yaml
machine:
  install:
    diskSelector:
      wwid: "naa.6002248059bddfe77ab4c2f59bca39e4"
```

### Data disk and network interface

- **Data disk:** second disk, size ≥ 100 GiB. The `UserVolumeConfig` at the bottom of each node YAML uses `!system_disk` to pick whatever disk is NOT the OS one, so it needs no edit regardless of device name.
- **Network interface:** usually `eth0`. If your VM uses `ens18`, `ens192`, `enp1s0`, etc. (check `get links` output), update the `interface:` field in `talos/nodes/node*.yaml`.

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

Apply the shared `controlplane.yaml` **plus** the per-node patch to every node. The node's hostname, interface, static IP and data disk mount all come from the patch. Reuse the `tmp_ip_1/2/3` variables you set in Step 2:

```bash
talosctl apply-config --insecure \
  --nodes "$tmp_ip_1" \
  --file _out/controlplane.yaml \
  --config-patch @talos/nodes/node1.yaml
```

Talos writes the config to its `STATE` partition, restarts networking, and you'll see the VM's IP switch from its DHCP lease to `10.0.0.11`. Repeat for node2 and node3:

```bash
talosctl apply-config --insecure \
  --nodes "$tmp_ip_2" \
  --file _out/controlplane.yaml \
  --config-patch @talos/nodes/node2.yaml

talosctl apply-config --insecure \
  --nodes "$tmp_ip_3" \
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

> 🛈 **Where the node IPs end up in the config** (they're used in two places):
>
> 1. **TLS SubjectAltNames.** The three IPs become SAN entries on the leaf certificates ECK signs with your CA. When a client connects to `https://10.0.0.11:30920`, TLS validation checks that `10.0.0.11` is in the cert's SAN list. If it's not, the handshake fails.
> 2. **Default Fleet outputs for newly enrolled agents.** The Kibana config declares two Fleet outputs and two Fleet Server hosts. The DEFAULT (`is_default: true`) is an entry listing ALL THREE NODE IPS — `https://10.0.0.11:30920`, `https://10.0.0.12:30920`, `https://10.0.0.13:30920`. Every newly enrolled agent receives all three and fails over between them if one is unreachable, so you get de-facto HA across all three nodes without any external load balancer. Same deal for Fleet Server on port 30822.
>
> 🛈 **Optional DNS names.** `elastic.lan` / `kibana.lan` / `fleet.lan` are placeholders — delete them, replace them with your company's internal DNS, or add them alongside the IPs. Any DNS name you add here MUST also be added to the corresponding `subjectAltNames` block so TLS verification accepts it. If you set up DNS later, clients can move between IPs and DNS interchangeably without re-issuing the certs.

### What's pre-tuned (so you don't have to think about it)

| Setting | Value | Why |
|---|---|---|
| ES memory request == limit | `8Gi` == `8Gi` | Elasticsearch is allergic to memory pressure. Request==limit gives it "Guaranteed" QoS and locks the allocation. ECK auto-sizes the JVM heap to ~50% of this — no `ES_JAVA_OPTS` to fiddle with. |
| Kibana / Fleet Server / Agent memory | request `1Gi`, no limit | Stateless services — they can burst freely under load without risk of OOM-kill. |
| Elasticsearch audit logging | ON | Records auth attempts, permission denials, config changes. Searchable in Kibana. |
| Kibana self-monitoring | ON | Stack Monitoring data ships back into the same cluster — one UI to rule them all. |
| Fleet Server replicas | 2 | HA, spread across two nodes via anti-affinity. |
| Elastic Agent Kubernetes integration | ON | Cluster-wide observability on every node by default. |
| Pod anti-affinity | Enforced (required) | Kubernetes scheduler refuses to place two pods of the same type on the same node. |

If your VMs are bigger than 16 GiB, just raise `resources.limits.memory` on the Elasticsearch section of `values.yaml` — Elasticsearch's auto-heap follows the memory limit automatically (see [Sizing the VMs](#sizing-the-vms-ram-budget-per-node) in Prerequisites for the numbers). Nothing else needs to change.

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

### How NodePort routing actually works

This matters, because the name is misleading. A NodePort on node1 **is not** "the Kibana pod on node1". It's a port that **every Kubernetes node** (node1, node2, node3) listens on. When a request arrives on ANY node's IP at port 30601, kube-proxy intercepts it and forwards it to **any available Kibana pod anywhere in the cluster** — possibly the one running on a different node entirely.

```
Client → https://10.0.0.11:30601
            │
            ▼
         kube-proxy on node1
            │  (sees NodePort 30601 → Kibana Service)
            ▼
         picks a Kibana pod at random (could be on node2 or node3!)
```

That means:

- Any node IP works. You can round-robin across the three in your own LB, or just use a single one — if the node hosting Kibana dies, the OTHER nodes still serve requests.
- It's already a load balancer. kube-proxy distributes connections across all healthy Kibana replicas. You do NOT need a separate LB to get HA on the LAN.
- The node IP you hit is irrelevant to where the work happens. `https://node1-ip:30601` is not "Kibana-on-node1" — it's "the Kibana Service, entered via the node1 door".

You can verify this by looking at which pods actually handle requests. With two Kibana replicas and anti-affinity, one lives on node1 and one on node2; node3 has no Kibana pod — and yet `https://10.0.0.13:30601` still works perfectly because node3 forwards to the other two.

The same is true for Elasticsearch (port 30920) and Fleet Server (port 30822).

### Via kubectl port-forward (dev / testing)

If your workstation can't reach the node IPs directly (corporate firewalls, VPN quirks):

```bash
kubectl -n elastic-stack port-forward svc/kibana-kb-http 5601:5601
# browse to https://localhost:5601
```

This is convenient for first-time exploration but not a production pattern — it ties Kibana availability to your local `kubectl` session.

### 🚨 First thing to do in Kibana — set ILM policies for the observability data

**Do this on day one, before your cluster fills up.**

Our Elastic Agent DaemonSet runs the Kubernetes integration on every node, which pulls metrics every few seconds (pod state, node stats, container stats, network stats, events) AND ships every container log from every pod in the cluster. On a quiet 3-node cluster this easily produces **several GiB of data per day**. Without sensible Index Lifecycle Management (ILM) policies your indices grow forever and eventually fill your disks — regardless of how big they are.

**How ILM actually works in stock Elastic** ([ILM docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)): Elastic ships two catch-all fallback policies, **`logs`** and **`metrics`**. Every data stream created by Fleet integrations (including our Kubernetes integration) is assigned one of those two by default. The built-in defaults have **no delete phase** — data stays forever. That's what we need to change.

**Step 1 — Edit the `logs` policy for a sensible retention curve.**

In Kibana, go to **Stack Management → Index Lifecycle Policies → logs → Edit policy**. Then:

- In the **Hot phase**, expand **Advanced settings** and **turn OFF "Use recommended defaults"**. This unlocks the rollover knobs.
- Set **Maximum age** to `5d`. Rollover cadence should be a fraction of total retention — a 1:6 ratio gives you neat, mostly-sealed segments rather than one giant index that keeps getting rewritten.
- Leave **Maximum primary shard size** at the default `50gb`.
- Scroll down, **enable the Delete phase**, and set it to `30d` after rollover. So: rollover every 5 days, total retention 30 days.
- Save.

**Step 2 — Edit the `metrics` policy similarly, but tighter and with downsampling.**

Metrics are high-cardinality and lose value fast. Same idea:

- In the **Hot phase → Advanced settings**, turn OFF recommended defaults.
- **Maximum age** `1d` (rollover every day).
- **Maximum primary shard size** can stay `50gb`.
- **Enable Downsampling** with an interval of `1h`. This collapses many points per timestamp into one hourly summary — orders of magnitude fewer documents, most of the chart shape preserved. Huge savings for Kubernetes metrics you don't look at with sub-hour resolution anyway.
- **Enable the Delete phase**, set it to `7d`. Metrics worth 7 days of granular + 7 days of downsampled is plenty for operational dashboards.
- Save.

**Step 3 — Force immediate rollover of all existing data streams.**

This is the critical step almost every tutorial skips. When you edit an ILM policy, **existing backing indices keep the OLD policy** until they rollover naturally — which with Elastic's 30-day default rollover could mean nothing happens for a month. Force a lazy rollover across every data stream right now:

```bash
ELASTIC_PW=$(kubectl -n elastic-stack get secret elasticsearch-es-elastic-user \
  -o go-template='{{.data.elastic | base64decode}}')

for stream in $(curl --cacert ca/ca.crt -s -u "elastic:${ELASTIC_PW}" \
  "https://10.0.0.11:30920/_data_stream" | jq -r '.data_streams[].name'); do
  echo "rolling over $stream"
  curl --cacert ca/ca.crt -s -u "elastic:${ELASTIC_PW}" \
    -X POST "https://10.0.0.11:30920/${stream}/_rollover?lazy" >/dev/null
done
```

The `?lazy` flag means "create the new backing index on the NEXT write event" — no interrupted ingest, no duplicate docs, no manual disruption. Every data stream picks up its edited ILM policy immediately after its next write.

**Step 4 — (Optional, only if needed) per-integration policies.**

If a single integration produces an overwhelming amount of data (say, containers that log tens of thousands of lines per second), you can create a dedicated ILM policy and attach it to just that data stream. But for most deployments, tuning `logs` and `metrics` once is enough — everything else inherits sane defaults.

> 🛈 **If you skip this step entirely, the single biggest day-2 failure mode of this stack is disk-full.** The Kubernetes integration alone can eat a 100 GiB PV in a few weeks of normal cluster activity, and once Elasticsearch hits its disk watermarks it starts refusing writes and eventually flipping indices to read-only. A 15-minute tour through the ILM UI now saves you a 2 AM incident later.

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

1. In Kibana → **Fleet → Agent policies** → pick the policy you want the agent to join → **Add agent** → copy the enrollment token.
2. Note what Kibana shows you as the Fleet Server URL — because the default Fleet Server host we pre-configured in `values.yaml` is a list of all three node NodePort URLs, Kibana will hand your new agent a URL like `https://10.0.0.11:30822` automatically. If you run the agent in a different network segment and IP reachability is an issue, pick whichever of the three node IPs is actually reachable from where the agent lives.
3. Install and enroll the agent:
   ```bash
   sudo elastic-agent install \
     --url=https://10.0.0.11:30822 \
     --enrollment-token=${token} \
     --certificate-authorities=ca/ca.crt
   ```
   The `--certificate-authorities` flag tells the agent to trust your internal CA, so the TLS handshake with Fleet Server succeeds. No `--insecure` needed, no custom SAN work required beyond what ECK already baked into the cert.

**Why you didn't have to do anything in the Kibana UI to set this up:** our `values.yaml` pre-declares the default Fleet output and default Fleet Server host as lists of all three node IPs on the NodePort. That means every newly enrolled agent — whether from the Kibana UI or via `elastic-agent install` — automatically receives all three URLs as possible Fleet Server / data-ingest targets, and the agent's own load balancer fails over between them if one becomes unreachable. Effectively you've got HA for agent ingest across all three nodes without touching an external load balancer.

---

## Maintenance

### Upgrading Talos

Always one node at a time. Flannel is bundled with Talos — it upgrades with the OS automatically, you don't install or upgrade a CNI separately.

```bash
talos_version="v1.12.7"
image="ghcr.io/siderolabs/installer:${talos_version}"

# Drain node1, upgrade, uncordon
kubectl drain node1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --disable-eviction \
  --timeout=180s
talosctl upgrade --nodes 10.0.0.11 --image "$image" --preserve --reboot-mode powercycle --wait=false

# Wait for the node to come back
while ! talosctl -n 10.0.0.11 version 2>&1 | grep -q "v${talos_version#v}"; do sleep 5; done
kubectl wait --for=condition=Ready node/node1 --timeout=5m
kubectl uncordon node1

# Repeat for node2 and node3
```

#### What each `kubectl drain` flag does

The drain command has an intimidating flag list. Here's what each one is actually for:

- `--ignore-daemonsets` — the Elastic Agent DaemonSet (and Flannel, kube-proxy, etc.) has one pod per node. You can't drain a DaemonSet pod because the controller would immediately recreate it. Without this flag, drain refuses to start.
- `--delete-emptydir-data` — some system pods mount `emptyDir` volumes (caches, tmp storage). Their contents are lost when the pod is deleted. Drain refuses by default because data loss is scary; passing this flag says "yes, I accept losing the cache data, proceed".
- `--force` — allows deleting orphan pods (pods with no controller — e.g. a pod created with `kubectl run` without a Deployment). Safety net in case something weird is running on the node.
- `--disable-eviction` — this is the important one. Normal drain uses the Eviction API, which respects PodDisruptionBudgets. ECK installs PDBs that can block eviction during a rolling maintenance window (Elasticsearch will refuse to be evicted if the cluster isn't green). `--disable-eviction` sends a direct DELETE instead, which bypasses the PDB. That's usually what you want during a Talos node upgrade: you've already drained one node at a time, you're not going to evict multiple ES pods simultaneously, and you want to actually proceed.
- `--timeout=180s` — a wall-clock budget for the whole drain. If it takes longer than 3 minutes, drain bails out. Useful because a stuck pod would otherwise block the upgrade forever.

#### `--reboot-mode powercycle`

**Always use `--reboot-mode powercycle` on bare metal.** The default `kexec` reboot path leaves some Intel NICs (notably the I219 series used in Hetzner dedicated servers) in an unusable state and the node never comes back. `powercycle` forces a full BIOS reboot and re-initialises the NIC. On hypervisor VMs this is basically free (no BIOS firmware to slow it down) — no downside to using it as the default everywhere.

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

### Changing a configuration value (Kibana / Elasticsearch)

Every non-trivial setting in your stack lives in `kubernetes/eck-stack/values.yaml`. To change one:

1. **Edit the file.** E.g. to enable a new Kibana setting, find the `eck-kibana.config:` block and add your key.
2. **Re-run helm upgrade:**
   ```bash
   helm upgrade eck-stack elastic/eck-stack \
     --version 0.18.1 \
     --namespace elastic-stack \
     --values kubernetes/eck-stack/values.yaml
   ```
3. **ECK picks up the change and restarts the pods.** For Elasticsearch it does a true rolling restart — drains shards, restarts one node at a time, waits for green, moves on. For Kibana it recreates all pods at once (Kibana cannot be upgraded with a rolling restart because the running replicas would see inconsistent migration state — ECK deletes and replaces them, causing a brief UI downtime of a few seconds). Fleet Server's pods are rolled one at a time. All of this is orchestrated by the operator — you don't touch `kubectl delete pod`.
4. **Watch it happen:**
   ```bash
   kubectl -n elastic-stack get pods -w
   ```

That's the entire change loop. No manual SSHing into VMs, no editing `/etc/elasticsearch/elasticsearch.yml`, no `systemctl restart elasticsearch.service`.

### Adding secrets (the Elasticsearch keystore)

Some settings can't live in plaintext config — passwords, API keys, S3 credentials. Elasticsearch has a [keystore](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-settings.html) for these; ECK exposes it via `spec.secureSettings` which references one or more Kubernetes Secrets and injects their keys into the keystore at pod startup.

**The flow:**

1. Create a regular Kubernetes Secret with the values you want in the keystore.
2. Reference the Secret from `eck-elasticsearch.secureSettings` in `values.yaml`, optionally remapping each key to a specific keystore setting name.
3. `helm upgrade` → ECK restarts Elasticsearch → the new keystore entries become active.

There's no `/etc/elasticsearch/elasticsearch.keystore` file for you to touch. The operator reconciles the keystore state automatically on every rolling restart.

### Adding an S3 snapshot repository

Here's a worked example that puts the maintenance flow, the keystore flow and the S3 repository settings together.

**Step 1.** Create a Secret with your S3 access key and secret key:

```bash
kubectl -n elastic-stack create secret generic s3-snapshot \
  --from-literal=access_key='AKIAEXAMPLE' \
  --from-literal=secret_key='wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'
```

**Step 2.** In `kubernetes/eck-stack/values.yaml`, add a `secureSettings` block under `eck-elasticsearch` so ECK loads those values into the Elasticsearch keystore under the expected key names:

```yaml
eck-elasticsearch:
  secureSettings:
    - secretName: s3-snapshot
      entries:
        - key: access_key
          path: s3.client.default.access_key
        - key: secret_key
          path: s3.client.default.secret_key
  nodeSets:
    - name: default
      config:
        # Also tell Elasticsearch where to find the S3 endpoint:
        s3.client.default.endpoint: "https://s3.example-region.amazonaws.com"
        # ... rest of your existing config
```

**Step 3.** `helm upgrade eck-stack ...` as shown above. ECK reloads the keystore and does a rolling restart so every ES pod sees the new credentials.

**Step 4.** Register the snapshot repository in Elasticsearch itself (once, via the REST API — this is cluster state, not node config):

```bash
ELASTIC_PW=$(kubectl -n elastic-stack get secret elasticsearch-es-elastic-user \
  -o go-template='{{.data.elastic | base64decode}}')

curl --cacert ca/ca.crt -u "elastic:${ELASTIC_PW}" \
  -X PUT "https://10.0.0.11:30920/_snapshot/my-backup" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "my-elastic-backups",
      "client": "default"
    }
  }'
```

**Step 5.** Take a snapshot:

```bash
curl --cacert ca/ca.crt -u "elastic:${ELASTIC_PW}" \
  -X PUT "https://10.0.0.11:30920/_snapshot/my-backup/snapshot-1?wait_for_completion=true"
```

Once you have a snapshot repository wired up, **everything else that depends on it is trivial**: Snapshot Lifecycle Management (SLM) policies, index rollover with rollup-and-delete, searchable snapshots, and eventually a frozen tier. All of it is configurable from the Kibana UI once the keystore credentials are in place. This is the usual "add a frozen tier to my cluster" entry point you keep hearing about — and it all starts with this one Secret.

See the [Elasticsearch S3 repository docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/repository-s3.html) for all available client settings (region, max retries, bucket versioning, etc.).

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
Yes. Create `talos/nodes/node4.yaml`, run `talosctl apply-config --insecure --file _out/controlplane.yaml --config-patch @talos/nodes/node4.yaml --nodes "$tmp_ip_4"` (set `tmp_ip_4` to the new VM's DHCP IP first). Then add a 4th PV to `kubernetes/storage/pvs.yaml`, bump the Elasticsearch `nodeSet` count to 4, add the 4th IP to every `subjectAltNames` block, and `helm upgrade`. Data will rebalance automatically.

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
