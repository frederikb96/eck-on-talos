# Trying it out on Azure

Companion to [README.md](README.md) — provisions 3 Talos VMs on Azure so you can follow the main guide without owning hardware. ~30 min wall-clock, ~€2 total for a 2-hour run. Delete the resource group when done → everything gone.

> ⚠️ **Azure doesn't boot VMs from ISO.** The main README's "Option A — ISO" and "Option B — `dd` from rescue" don't apply here. Instead: upload Talos' official `azure-amd64.vhd` to Azure Storage as a page blob, register it as a custom image, create VMs from that image. All other steps of the main guide work unchanged.

Linux laptop assumed (macOS/WSL users, adjust paths + tool downloads accordingly). Two paths below — pick one:

- **TL;DR script** (next section) — one big `az` CLI block, copy-paste the whole thing or run it chunk by chunk. ~15 min end-to-end, no clicking.
- **Portal walkthrough** (starting at [Step A](#step-a--resource-group)) — same thing, click by click in the browser.

## TL;DR — full `az` CLI script

**Prereqs:** `az login` done (`az account show` prints your subscription), plus `wget`, `xz`, `talosctl`, and an SSH public key at `~/.ssh/id_ed25519.pub`.

Each block below is standalone — run top to bottom, or paste the whole thing at once. Variables persist between blocks in the same shell.

```bash
# ─── Variables (edit SA if it collides globally) ───
RG="eck-on-talos-test"
LOC="westeurope"
SA="eckontalos$RANDOM"
TALOS_VERSION="v1.12.6"
SCHEMATIC="376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba"
SSH_KEY=$(cat ~/.ssh/id_ed25519.pub)

# ─── 1. Register Standard security type (one-time per subscription, ~1–5 min) ───
# Subscriptions default to Trusted Launch for Gen 2 images; Talos needs Standard.
# If already registered, this is a no-op. Poll until state = "Registered".
az feature register --namespace Microsoft.Compute --name UseStandardSecurityType
while [ "$(az feature show --namespace Microsoft.Compute --name UseStandardSecurityType --query properties.state -o tsv)" != "Registered" ]; do
  echo "waiting for feature registration..."; sleep 15
done
az provider register --namespace Microsoft.Compute

# ─── 2. Resource group ───
az group create -n "$RG" -l "$LOC"

# ─── 3. Download + decompress Talos VHD (~2–3 min) ───
mkdir -p /tmp/talos-azure && cd /tmp/talos-azure
wget "https://factory.talos.dev/image/${SCHEMATIC}/${TALOS_VERSION}/azure-amd64.vhd.xz"
xz -d azure-amd64.vhd.xz

# ─── 4. Storage account + container ───
az storage account create -n "$SA" -g "$RG" -l "$LOC" --sku Standard_LRS
KEY=$(az storage account keys list -g "$RG" -n "$SA" --query '[0].value' -o tsv)
az storage container create --account-name "$SA" --account-key "$KEY" -n images

# ─── 5. Upload VHD as page blob (~2–5 min) ───
az storage blob upload \
  --account-name "$SA" --account-key "$KEY" \
  --container-name images \
  --type page \
  --file /tmp/talos-azure/azure-amd64.vhd \
  --name "talos-${TALOS_VERSION}.vhd"

# ─── 6. Register as custom image ───
BLOB_URL="https://${SA}.blob.core.windows.net/images/talos-${TALOS_VERSION}.vhd"
az image create -g "$RG" -n "talos-${TALOS_VERSION}" \
  --source "$BLOB_URL" --os-type Linux --hyper-v-generation V2 --location "$LOC"

# ─── 7. VNet + subnet ───
az network vnet create -g "$RG" -n eck-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name eck-subnet --subnet-prefix 10.0.0.0/24

# ─── 8. NSG + rules + associate to subnet ───
# 0.0.0.0/0 is fine: Talos (50000) + k8s (6443) are mTLS, NodePorts have basic auth.
az network nsg create -g "$RG" -n eck-nsg
az network nsg rule create -g "$RG" --nsg-name eck-nsg -n talos-api        --priority 100 --destination-port-ranges 50000 --protocol Tcp --access Allow
az network nsg rule create -g "$RG" --nsg-name eck-nsg -n k8s-api          --priority 110 --destination-port-ranges 6443  --protocol Tcp --access Allow
az network nsg rule create -g "$RG" --nsg-name eck-nsg -n elastic-nodeport --priority 120 --destination-port-ranges 30920 --protocol Tcp --access Allow
az network nsg rule create -g "$RG" --nsg-name eck-nsg -n kibana-nodeport  --priority 130 --destination-port-ranges 30601 --protocol Tcp --access Allow
az network nsg rule create -g "$RG" --nsg-name eck-nsg -n fleet-nodeport   --priority 140 --destination-port-ranges 30822 --protocol Tcp --access Allow
az network nsg rule create -g "$RG" --nsg-name eck-nsg -n intra-lan        --priority 200 --source-address-prefixes 10.0.0.0/24 --destination-port-ranges '*' --protocol '*' --access Allow
az network vnet subnet update -g "$RG" --vnet-name eck-vnet -n eck-subnet --network-security-group eck-nsg

# ─── 9. Static public IPs for the 3 VMs ───
for i in 1 2 3; do
  az network public-ip create -g "$RG" -n "node$i-pip" --sku Standard --allocation-method Static
done

# ─── 10. Three VMs — Standard security (NOT TrustedLaunch!), parallel create (~2 min) ───
IMAGE_ID=$(az image show -g "$RG" -n "talos-${TALOS_VERSION}" --query id -o tsv)
for i in 1 2 3; do
  az vm create -g "$RG" -n "node$i" \
    --image "$IMAGE_ID" \
    --size Standard_D4s_v5 \
    --admin-username azureuser \
    --ssh-key-values "$SSH_KEY" \
    --vnet-name eck-vnet --subnet eck-subnet \
    --private-ip-address "10.0.0.1$i" \
    --public-ip-address "node$i-pip" \
    --nsg "" \
    --security-type Standard \
    --os-disk-size-gb 64 \
    --storage-sku os=Premium_LRS 0=Premium_LRS \
    --data-disk-sizes-gb 128 \
    --no-wait
done
for i in 1 2 3; do
  timeout 300 az vm wait -g "$RG" -n "node$i" --created \
    || az deployment group list -g "$RG" --query '[?properties.provisioningState==`Failed`].name' -o tsv
done

# ─── 11. Print public IPs + verify Talos maintenance mode ───
az vm list-ip-addresses -g "$RG" -o table
for i in 1 2 3; do
  ip=$(az vm show -g "$RG" -n "node$i" -d --query publicIps -o tsv)
  echo "=== node$i ($ip) ==="
  talosctl -n "$ip" version --insecure
done
# Expected output from each node:
# error getting version: rpc error: code = Unimplemented desc = API is not implemented in maintenance mode
```

Expected final output: each node prints `API is not implemented in maintenance mode`. That's success — jump to the section below on "Continue with the main guide" to proceed with the rest of the steps.

**Cleanup when done:** `az group delete -n "$RG" --yes --no-wait`

## Prerequisites

- Azure subscription + browser signed in to the Portal
- `wget`, `xz`, an SSH public key
- `azcopy` (Microsoft's VHD upload tool — single binary, no install, grabbed below in B.3)

> **Don't use the Portal blob uploader for VHDs.** It drops the final chunk on ~10 GiB page blobs and Azure then rejects the image with `The specified cookie value in VHD footer ... is not a supported VHD`. That's why we use `azcopy`.

## Step A — Resource group

**Portal → Resource groups → + Create**

- Name: `eck-on-talos-test`
- Region: `West Europe` (**every later step must be in this same region**)
- Review + create

## Step B — Talos custom image

### B.1 — Download the VHD

```bash
mkdir -p /tmp/talos-azure && cd /tmp/talos-azure
talos_version="v1.12.6"
# Canonical empty schematic (vanilla Talos, no extensions). Generate your own at
# https://factory.talos.dev/ if you want extras (iscsi-tools, tailscale, ...).
schematic="376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba"
wget "https://factory.talos.dev/image/${schematic}/${talos_version}/azure-amd64.vhd.xz"
xz -d azure-amd64.vhd.xz
```

Result: `azure-amd64.vhd`, sparse 10 GiB raw VHD (`.xz` download is ~200 MiB; extracted file is ~10 GiB on disk but compresses away to almost nothing when uploaded).

### B.2 — Storage account + container

**Portal → Storage accounts → + Create**

- Resource group: `eck-on-talos-test`
- Name: something globally unique like `eckontalos<initials><random>`
- Region: same as RG
- Performance: **Standard**
- Redundancy: **LRS**
- Review → Create

Then **open the storage account → Data storage → Containers → + Container**:

- Name: `images`
- Access level: Private
- Create

### B.3 — Upload with azcopy

Download azcopy (single binary, Linux x64):

```bash
cd /tmp
wget https://aka.ms/downloadazcopy-v10-linux -O azcopy.tgz
tar -xzf azcopy.tgz --strip-components=1 --wildcards '*/azcopy'
./azcopy --version
```

Get a write SAS URL for the container:

**Portal → Storage account → Containers → images → Settings on left panel → Shared access tokens**

- Signing method: **Account key**, Signing key: **key1**
- Permissions: ☑ Read ☑ Add ☑ Create ☑ Write ☑ List
- Expiry: now + 2 hours
- Allowed protocols: HTTPS only
- Generate → copy the **Blob SAS URL**

Upload:

```bash
SAS_URL='<paste the Blob SAS URL — keep the single quotes>'
/tmp/azcopy copy /tmp/talos-azure/azure-amd64.vhd \
  "$SAS_URL" \
  --blob-type PageBlob \
  --put-blob-size-mb 256
```

~2–5 min on home fiber. Look for `Final Job Status: Completed`.

**Portal → Storage account → Containers → images → azure-amd64.vhd** should now exist.

Rename the blob to `talos-v1.12.6.vhd` (blob → ⋯ → Rename) and copy its **Overview → URL** for the next step.

### B.4 — Register as custom image

**Portal → Images → + Create**

- Resource group: `eck-on-talos-test`
- Name: `talos-v1.12.6`
- Region: same as RG
- OS type: **Linux**
- VM generation: **Gen 2**
- Storage blob: paste the blob URL from B.3
- Account type: **Standard SSD LRS**
- Host caching: **Read/write**
- Review + create

Done in ~30 s. The image is now in **Portal → Images**.

## Step C — Network

### C.1 — VNet

**Portal → Virtual networks → + Create**

**Basics tab:**

- Resource group: `eck-on-talos-test`
- Name: `eck-vnet`
- Region: same as RG

**Security tab:** leave everything off, Next.

**IP addresses tab:** this is the confusing one — Azure pre-fills a VNet `10.0.0.0/16` with a default subnet called `default`. You want to keep the VNet range but rename the subnet.

- Leave the IPv4 address space as `10.0.0.0/16`
- Click the pre-existing `default` subnet row → panel opens on the right
- Name: change to `eck-subnet`
- Starting address: `10.0.0.0`, Size: `/24 (256 addresses)`
- Save (the right-hand panel)
- Next → Review + create → Create

### C.2 — NSG

**Portal → Network security groups → + Create** → name `eck-nsg`, same RG/region, Create.

**eck-nsg → Inbound security rules → + Add** (one rule at a time):

| Priority | Name | Source | Dest port | Protocol |
|---|---|---|---|---|
| 100 | talos-api | `0.0.0.0/0` | 50000 | TCP |
| 110 | k8s-api | `0.0.0.0/0` | 6443 | TCP |
| 120 | elastic-nodeport | `0.0.0.0/0` | 30920 | TCP |
| 130 | kibana-nodeport | `0.0.0.0/0` | 30601 | TCP |
| 140 | fleet-nodeport | `0.0.0.0/0` | 30822 | TCP |
| 200 | intra-lan | `10.0.0.0/24` | * | Any |

> **Why `0.0.0.0/0` is fine for a test.** Talos API (50000) and Kubernetes API (6443) are mTLS-protected — you need the client cert from your `talosconfig`/`kubeconfig`, otherwise the TLS handshake fails. Kibana/Elasticsearch/Fleet NodePorts have HTTP basic auth. Same posture as production Talos on bare metal. No SSH rule — Talos has no SSH daemon.

**Associate NSG with subnet:** Virtual networks → eck-vnet → Subnets → eck-subnet → NSG = eck-nsg → Save.

## Step D — 3 VMs

Do this **3 times**: `node1` (10.0.0.11), `node2` (10.0.0.12), `node3` (10.0.0.13).

**Portal → Images → talos-v1.12.6 → + Create VM**

**Basics tab:**

- RG: `eck-on-talos-test`
- Name: `node1` / `node2` / `node3`
- Region: same as RG
- Security type: **Standard** ⚠️ (NOT Trusted Launch — Talos' VHD isn't signed for it, VM will boot-loop)
- Size: **Standard_D4s_v5** (4 vCPU, 16 GiB — matches the main README's sizing)
- Auth: SSH public key, username `azureuser`, paste your `~/.ssh/id_ed25519.pub`
- Public inbound ports: **None**

**Disks tab:**

- OS disk: 64 GiB Premium SSD, delete with VM
- **+ Create and attach a new disk**: `node1-data` / 128 GiB Premium SSD / LUN 0 / delete with VM

**Networking tab:**

- VNet: `eck-vnet`, Subnet: `eck-subnet`
- Public IP: **Create new** → Standard, **Static**, delete with VM
- NIC NSG: **None** (subnet NSG covers it)
- **Private IP assignment: Static** → `10.0.0.11` / `.12` / `.13`

**Review + create → Create.** ~2 min each; you can queue all 3 in parallel tabs.

Note each VM's **public IP** from its Overview page.

## Step E — Verify

```bash
for ip in <node1-public> <node2-public> <node3-public>; do
  talosctl -n "$ip" version --insecure
done
```

Expected: each prints `API is not implemented in maintenance mode`. That's success.

## Continue with the main guide

The main README's **Set your cluster variables** block is designed to handle cloud VMs directly — on Azure you just fill in the two IP sets with different values. When you get to that section, use:

- `node1_lan_ip` / `node2_lan_ip` / `node3_lan_ip` → `10.0.0.11` / `10.0.0.12` / `10.0.0.13` (the private VNet IPs — these match the repo defaults, nothing to change)
- `node1_ip` / `node2_ip` / `node3_ip` → the **three public IPs** printed by the verify loop above

Two tiny Azure-isms that already match the repo defaults (nothing to edit):

- Interface name inside Talos: `eth0` (accelerated networking default)
- Default gateway: `10.0.0.1` (Azure subnet default)

Now jump to **[Step 2 of the main guide](README.md#step-2--locate-the-nodes-and-verify-disks)**.

## Clean-up

```bash
az group delete -n "$RG" --yes --no-wait
```

Everything goes: VMs, disks, VNet, NSG, storage account, image, public IPs. ~€2 for a 2-hour run.

## Troubleshooting

**`talosctl version --insecure` hangs.**

- Serial console (Portal → VM → Support + troubleshooting → Serial console) should show Talos boot messages. If it's stuck at GRUB/BIOS → you picked Trusted Launch by mistake; recreate the VM with Security type = Standard.

**VM fails to deploy: "Allocation failed" / "VM size not available".**

- `Standard_D4s_v5` not available in your region. Try `Standard_D4s_v4` or switch region (everything must move together).

**Step B.4 fails: "cookie value in VHD footer ... is not a supported VHD".**

- The blob is corrupted (its last 512 bytes — the VHD footer — got zeroed out). Almost always caused by Portal browser upload; that's why B.3 uses azcopy. Delete the blob + failed image, re-run B.3 with azcopy, retry B.4. If you already used azcopy and still hit this, the local `.vhd` is damaged — re-run `xz -d` from a fresh `.vhd.xz` download.

**Step B.4 fails: "The disk must be a Page blob".**

- You forgot `--blob-type PageBlob` in the azcopy command. Delete the blob, re-run with the flag.

**azcopy fails: "operation not permitted ... soft-deleted".**

- Storage account → Data protection → disable soft delete for blobs → remove the soft-deleted blob → retry.

**`talosctl -n <public-ip> version` fails with "x509: certificate is valid for …" after Step 4.**

- The laptop-facing (public) IP is not in that node's `machine.certSANs`. Check you set `node1_ip` / `node2_ip` / `node3_ip` to the **public** IPs before running the sed in the main README's Variables block. If not, re-run: `git checkout talos/nodes/`, re-export the variables, re-run the sed, re-apply configs.

**Everything else:** see the main README's [Troubleshooting](README.md#troubleshooting).
