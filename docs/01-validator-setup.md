# Monad Testnet - Validator Node Setup

> Documentation of the initial infrastructure setup for a Monad testnet validator node.  
> Part of Cumulo's validator candidacy process.

---

## Hardware

| Component | Spec |
|---|---|
| CPU | AMD Ryzen 9 7950X3D - 16 cores / 4.2 GHz base (5.7 GHz Turbo) |
| RAM | 128 GB DDR5 |
| Storage | 2× 1.92 TB NVMe (Micron 7450 MTFDKCC1T9TFR) — PCIe Gen4 |
| Network | 1 Gbit/s dedicated - 100 TB/month traffic |
| OS | Ubuntu 24.04 LTS |
| Environment | Bare metal (required by Monad) |

### Hardware vs. Official Requirements

| Requirement | Minimum | Our Server |
|---|---|---|
| CPU cores | 16c / 4.5 GHz+ | 16c / 4.2 GHz (5.7 GHz Turbo) ✅ |
| RAM | 32 GB | 128 GB ✅ |
| TrieDB disk | 2 TB NVMe PCIe 4 | 1.92 TB NVMe PCIe 4 ✅ |
| OS/BFT disk | 500 GB NVMe | 1.92 TB NVMe ✅ |
| Bandwidth | 300 Mbit/s (validator) | 1 Gbit/s ✅ |

> **Note on storage:** Micron 7450 is listed in Monad's official docs as functional but with occasional random slowdowns under heavy load. Samsung 980/990 Pro or PM9A1 are rated higher. Monitoring will be in place to detect any I/O degradation.

---

## Disk Layout

Two physical NVMe drives with dedicated roles, as required by Monad's architecture:

### `nvme0n1` - Operating System + MonadBFT

```
nvme0n1        1.92 TB
├─ nvme0n1p1      1 GB   /boot/efi   (EFI System Partition)
├─ nvme0n1p2      1 GB   /boot
├─ nvme0n1p3     16 GB   [SWAP]
└─ nvme0n1p4    1.7 TB   /           (root filesystem)
```

This disk hosts the OS, MonadBFT ledger, config files, and keystores at `/home/monad/`.

### `nvme1n1` - TrieDB (Execution State)

```
nvme1n1        1.92 TB
└─ nvme1n1p1   1.92 TB   (no filesystem — raw block device)
```

Dedicated exclusively to Monad's TrieDB database. No filesystem is mounted on this partition - Monad writes directly to the block device. Exposed to the system via udev symlink at `/dev/triedb`.

---

## Installation Steps

### 1. Verify disk layout

Before any operation, confirmed which NVMe was the OS disk and which was free:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

Output confirmed `nvme0n1` carries the OS partitions and `nvme1n1` is clean.

### 2. Verify LBA configuration on TrieDB disk

Monad requires 512-byte LBA format on the TrieDB drive:

```bash
nvme id-ns -H /dev/nvme1n1 | grep 'LBA Format' | grep 'in use'
```

```
LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0x2 Good (in use)
```

✅ Already configured correctly - no reformatting needed.

### 3. Partition the TrieDB disk

Created a GPT partition table and a single partition spanning the full drive:

```bash
TRIEDB_DRIVE=/dev/nvme1n1

parted $TRIEDB_DRIVE mklabel gpt
parted $TRIEDB_DRIVE mkpart triedb 0% 100%
```

### 4. Create udev rule and symlink

```bash
PARTUUID=$(lsblk -o PARTUUID $TRIEDB_DRIVE | tail -n 1)

echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" \
  | tee /etc/udev/rules.d/99-triedb.rules

udevadm trigger
udevadm control --reload
udevadm settle
```

Verified symlink:

```bash
ls -l /dev/triedb
# lrwxrwxrwx 1 root root 9 ... /dev/triedb -> nvme1n1p1
```

✅ `/dev/triedb` correctly pointing to `nvme1n1p1`.

### 5. Configure APT repository and install Monad

```bash
cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

curl -fsSL https://pkg.category.xyz/keys/public-key.asc \
  | gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg

apt update
apt install -y monad
```

### 6. Create `monad` user and directory structure

```bash
useradd -m -s /bin/bash monad

mkdir -p /home/monad/monad-bft/config \
         /home/monad/monad-bft/ledger \
         /home/monad/monad-bft/config/forkpoint \
         /home/monad/monad-bft/config/validators
```

### 7. Disable SMT / HyperThreading via BIOS

Monad's official documentation requires SMT to be disabled for optimal node performance.

First, verified SMT was still active after OS install:

```bash
cat /sys/devices/system/cpu/smt/active
lscpu | grep -E "Thread|Core|Socket"
```

```
1
Thread(s) per core: 2
Core(s) per socket: 16
```

SMT was active - accessed BIOS via Supermicro IPMI remote console (iKVM/HTML5), navigated to:

**Advanced → CPU Configuration → SMT Control → Disabled**

After reboot, confirmed from OS:

```bash
cat /sys/devices/system/cpu/smt/active
lscpu | grep -E "Thread|Core|Socket"
```

```
0
Thread(s) per core:                      1
Core(s) per socket:                      16
Socket(s):                               1
```

✅ SMT disabled at hardware level via BIOS. Server now exposes 16 physical cores only.

### 8. Verify kernel version

```bash
uname -r
```

```
6.8.0-110-generic
```

✅ Kernel 6.8.0-110 - well above the minimum required (6.8.0-60).

### 9. Initialize TrieDB with `monad-mpt`

```bash
systemctl start monad-mpt
journalctl -u monad-mpt -n 20 -o cat
```

Output confirmed successful initialization:

```
MPT database on storages:
          Capacity           Used      %  Path
           1.75 Tb      256.03 Mb  0.01%  "/dev/nvme1n1p1"
MPT database internal lists:
     Fast: 1 chunks with capacity 256.00 Mb used 0.00 bytes
     Slow: 1 chunks with capacity 256.00 Mb used 0.00 bytes
     Free: 7134 chunks with capacity 1.74 Tb used 0.00 bytes
...
monad-mpt.service: Deactivated successfully.
```

✅ TrieDB initialized - 1.75 TB available, service completed cleanly.

---

## Status

| Step | Status |
|---|---|
| Hardware provisioned | ✅ |
| LBA 512b verified | ✅ |
| TrieDB partitioned (`nvme1n1p1`) | ✅ |
| `/dev/triedb` symlink active | ✅ |
| `monad` package installed | ✅ |
| `monad` user and directories created | ✅ |
| SMT disabled (BIOS — Supermicro IPMI) | ✅ |
| Kernel ≥ 6.8.0-60 (running 6.8.0-110) | ✅ |
| TrieDB initialized (`monad-mpt`) | ✅ |
| OS hardening (UFW, Fail2ban, users) | 🔄 In progress |
| Node configuration (`node.toml`, keystores) | ⏳ Pending |
| Services started (`monad-bft`, `monad-execution`, `monad-rpc`) | ⏳ Pending |
| Node synced to testnet | ⏳ Pending |

---

## References

- [Monad Node Operations Docs](https://docs.monad.xyz/node-ops)
- [Hardware Requirements](https://docs.monad.xyz/node-ops/hardware-requirements)
- [Full Node Installation](https://docs.monad.xyz/node-ops/full-node-installation)
- [Monad HCL — Community Hardware Compatibility List](https://monadhcl.xyz)
