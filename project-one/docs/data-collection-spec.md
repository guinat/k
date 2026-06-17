# Data Collection Specification: OS-agnostic storage schema (multi-OS)

> **Raw** storage metadata to collect per server, expressed in a **single, unified, OS-agnostic schema** valid across AIX · Linux (all distributions) · Windows Server, all first-class. This is **the data contract**: the authoritative description of what each collection run produces.

This specification covers the data-collection boundary only. The reference standards ("golden images"), the OS-by-OS field-applicability matrix, the field-by-field rationale with the Ansible collection logic, the LangGraph pipeline architecture, and the machine-validatable JSON Schema file are explicit next steps, described elsewhere as upcoming.

---

## 1. Scope & framing

This specification describes **raw collection only**: a first pass over storage metadata, plus an optional second-pass `content_aggregates` enrichment. It carries **no interpretation and no derived/computed values**.

Every computed field (storage class, percentages, growth rates, deviations, severities, confidence scores, etc.) is **excluded** from collection and **produced downstream** by the pipeline from the raw fields. See §7 for the explicit list of derived fields and §6 for how the raw fields cover the five objectives.

The five objectives this data must ultimately serve:

- **O1: Deviation detection vs standards**: spot servers whose partitioning / filesystem layout differs from the model expected for their OS and version.
- **O2: Optimization (useless / obsolete / misplaced)**: reclaim space: obsolete data, over-provisioned volumes, application data sitting on system partitions.
- **O3: Reliability & full-filesystem prevention**: detect ahead of time the filesystems about to fill up (in **bytes AND inodes**) to avoid incidents.
- **O4: Operational efficiency & forecasting**: standardize storage structures and anticipate growth / capacity needs.
- **O5: Risk reduction & compliance**: ensure only appropriate data resides on system partitions; check mount options / ACLs (CIS baselines).

---

## 2. Scope, platforms & delivery

- **One unified, OS-agnostic schema.** A single host schema, structurally identical regardless of platform. Field presence is **conditional / nullable** per OS and filesystem type, expressed within this one schema rather than through divergent schemas. The collection command branches per platform, but it always feeds this one schema.
- **AIX + Linux (all distributions) + Windows are all first-class, delivered together.** Windows has the same depth, the same obligation levels, and the same objective coverage as AIX and Linux. Windows collection (WinRM + PowerShell) and Windows reference standards are delivered in the same release as AIX and Linux.
- **Fleet scope: >= 500 servers**, spanning AIX, the RHEL family (RHEL / Oracle Linux / Rocky / AlmaLinux), Debian / Ubuntu, SUSE / SLES, and Windows Server.
- **Non-sensitive metadata only.** Only capacities, topology, mount options, and a non-sensitive ACL summary (on Windows) are collected. No filenames, no owners, no file contents, no named ACEs/SIDs.
- **Delivery = ONE envelope file per collection run.** The delivery unit is a single envelope object `{ snapshot_id, generated_at, host_count, hosts[] }` where each element of `hosts[]` is one server's document: all servers for the run in a single file, under the one shared schema.
- **JSONL accepted for very large fleets.** For very large fleets the same content may be delivered as JSONL: one host object per line. In that form, each line carries `snapshot_id` and `scan_timestamp` so every line remains attributable to its run.

---

## 3. Data model & collection principles

1. **Exact bytes everywhere.** All sizes (`size_*`, `reserved_bytes`, `extent_size_bytes`, paging sizes, etc.) are integer **bytes**, never blocks, KB, MB, GB, or percentages. Native outputs in KB or blocks (`df -k`, `lsps`, `vgs --units b`) are normalized to bytes at collection time.

2. **Agnostic model: `filesystems[]` + `storage_topology`.**
   - `filesystems[]` is the **universal denominator**: it covers both Unix mount points (`/`, `/var`, …) AND Windows volumes (`C:`, `D:`, and folder mount points). It is the only structure mandatory on every OS.
   - `storage_topology` describes the physical/logical topology in **neutral levels** (`groups` / `volumes` / `disks`), each carrying a `*_kind` discriminator. It is **empty** for plain-partition Linux or a basic-disk Windows host with no pool.

3. **Linkage by keys (`backing_ref` / `group_ref`), not by nesting.** A filesystem references its underlying volume/group by **key** (`backing_ref`, `group_ref`), not by nesting. This weak coupling lets the pipeline **derive `storage_class`** (system vs data) generally: rootvg/datavg on AIX, system/data VGs on Linux, `C:` (system) vs other drives (data) on Windows.

4. **Conditional collection, branched per platform.** The command reads `platform` then runs the appropriate branch (`lsvg`/`lslv`/`lspv`/`df`/`lsps` for AIX; `vgs`/`lvs`/`pvs`/`df`/`tune2fs`/`btrfs` for Linux; PowerShell `Get-*`/`Win32_PageFileUsage`/`Get-Acl` for Windows). A field that is not applicable to an OS/fstype is returned as explicit `null` (**verified N/A**), never silently omitted, so the pipeline can distinguish "N/A" from "not collected". The collection command branches per platform; the schema does not.

5. **UTC time series.** Each host document is a snapshot stamped with `scan_timestamp` (UTC ISO-8601), grouped under a run-level `snapshot_id`. Collection is **repeated over time and append-only; never overwrite**. The successive snapshots per host form the time series that makes growth derivation and saturation forecasting possible downstream.

6. **Non-sensitive metadata only.** Only capacity, topology, mount options, and a **non-sensitive ACL summary** (`acl_summary`, Windows) are collected: counts, inheritance flag, and well-known principals, never named ACEs/SIDs, filenames, owners, or contents.

7. **Per-OS nuances (decisive for byte accuracy):**
   - **AIX**: LVM (rootvg/datavg), JFS2; `lsvg`/`lslv`/`lspv`, `df -k`/`df -v`, `lsps -a`. No ext-style reserved blocks → `reserved_bytes = null`.
   - **RHEL / Oracle Linux / Rocky / AlmaLinux**: **XFS by default** → no reserved blocks (`reserved_bytes = null`); use `df -i` for inodes and `xfs_info`. LVM common (`vgs`/`lvs`/`pvs --units b`). `/boot/efi` is present **only on UEFI x86_64**.
   - **Debian / Ubuntu**: **ext4 by default** → `reserved_bytes` derived via **`tune2fs -l`** (reserved blocks × block size). LVM optional.
   - **SUSE / SLES**: **btrfs by default, with subvolumes + snapshots** → `df` is **misleading**; use **`btrfs filesystem usage`** / qgroups for the real bytes. btrfs has no fixed inode table → `inodes_total` / `inodes_used` are `null`. LVM optional.
   - **Windows Server**: **NTFS / ReFS**, drive letters / folder mount points; collected over **WinRM + PowerShell**. No inodes (`inodes_* = null`). **ACLs replace mount options** (`access_model = ntfs-acl`, `mount_options = null`, `acl_summary` populated). **Storage Spaces**: pool = `group` (`storage-pool`), virtual disk = `volume` (`virtual-disk`). **Basic disk**: partitions exposed as `volumes` (`partition`), `groups` empty. Sizes already in bytes.

---

## 4. The JSON schema

The delivery unit is the **envelope** below. Run-level identifiers (`snapshot_id`, `generated_at`, `host_count`) live on the envelope and are **not** duplicated inside any host. Each `hosts[]` element is one server's document and carries its own `scan_timestamp`, but **no per-host `snapshot_id`**: the run identifier is the envelope's.

**Obligation legend:** `REQ` = required · `REC` = recommended · `COND` = conditional (nullable per OS/fstype) · `REQ*` = required once a group/volume exists (else the key is `null`).

### 4.1 The envelope (one object per collection run)

```jsonc
// ---- THE FILE: one object for the whole fleet, per run ------------------
{
  "snapshot_id":  "string",   // REQ: collection-run identifier (RUN level)
  "generated_at": "string",   // REQ: UTC ISO-8601, file generation time (RUN level)
  "host_count":   number,     // REQ: number of elements in hosts[] (RUN level)
  "hosts": [ /* ONE document per server: host schema below */ ]
}
```

### 4.2 The host element (one per server)

```jsonc
// ---- ONE ELEMENT OF hosts[]: a single server ---------------------------
{
  // ── Identity & collection metadata (UNIVERSAL, all OS) ─────────────────
  "host_id":          "string",   // REQ: stable host identifier
  "hostname":         "string",   // REQ
  "gsma_code":        "string",   // REQ: site/fleet code
  "host_role_env":    "string",   // REQ: prod | preprod | test | dev ...
  "platform":         "string",   // REQ: enum: aix | linux | windows
  "os_family":        "string",   // REQ: AIX | RedHat | Debian | Suse | OracleLinux | Windows
  "os_distribution":  "string",   // REQ: AIX | RHEL | Ubuntu | Debian | SLES | OracleLinux | WindowsServer
  "os_version":       "string",   // REQ
  "architecture":     "string",   // REQ: ppc64 | x86_64 | ...
  "scan_timestamp":   "string",   // REQ: UTC ISO-8601 (this host's scan time); NO per-host snapshot_id
  "collection_method":"string",   // REQ: ssh-shell | winrm-powershell | agent
  "scan_confidence":  number,     // REQ: 0..1 collection reliability

  // ── UNIVERSAL DENOMINATOR: every mountable filesystem / Windows volume ──
  "filesystems": [{
    "mount_point":         "string",       // REQ: "/", "/var" | "C:", "D:" | mount-folder
    "label":               "string|null",  // REC
    "fstype":              "string",        // REQ: xfs|ext4|btrfs|jfs2|ntfs|refs|vfat...
    "size_total_bytes":     number,         // REQ: bytes
    "size_used_bytes":      number,         // REQ: bytes
    "size_available_bytes": number,         // REQ: bytes
    "reserved_bytes":      "number|null",   // COND: ext only; null on xfs/jfs2/ntfs/refs/btrfs
    "inodes_total":        "number|null",   // COND: null on Windows NTFS/ReFS
    "inodes_used":         "number|null",   // COND: null on Windows NTFS/ReFS
    "access_model":         "string",       // REQ: "posix-mount-options" | "ntfs-acl"
    "mount_options":       "string|null",   // COND: Unix flags; null on Windows
    "acl_summary":         "object|null",   // COND: non-sensitive Windows ACL summary; null on Unix
    "backing_kind":         "string",       // REQ: lvm-lv | partition | virtual-disk | network | pseudo
    "backing_ref":         "string|null",   // REQ*: key to the underlying volume/disk
    "group_ref":           "string|null",   // REQ*: key to the group (VG/pool)
    "content_aggregates":  { /* object */ } // REC: non-sensitive aggregates (2nd pass)
  }],

  // ── PLATFORM STORAGE TOPOLOGY: neutral levels + *_kind discriminator ───
  "storage_topology": {
    "groups": [{
      "group_name":       "string",         // REQ* (once a group exists)
      "group_kind":       "string",         // REQ*: lvm-vg | storage-pool
      "size_total_bytes":  number,          // REQ*: bytes
      "size_free_bytes":   number,          // REQ*: bytes
      "member_disk_count": number,          // REQ*
      "state":            "string",         // REQ*: active | online | degraded ...
      "extent_size_bytes":"number|null",    // COND: PE size (LVM) / extent; null if N/A
      "capabilities": {                     // COND: nullable per OS
        "max_lvs":     "number|null",       //   AIX
        "max_pps":     "number|null",       //   AIX
        "vg_type":     "string|null",       //   AIX (big | scalable ...)
        "resiliency":  "string|null"        //   Windows (Mirror | Parity | Simple)
      }
    }],
    "volumes": [{
      "volume_name": "string",              // REQ* (once a volume exists)
      "volume_kind": "string",              // REQ*: lvm-lv | partition | virtual-disk
      "group_ref":   "string|null",         // REQ*: key to parent group; null if standalone
      "size_bytes":   number,               // REQ*: bytes
      "type":        "string|null",         // REC: jfs2 | linear | striped | mirror | gpt-part ...
      "redundancy":  "string|null"          // COND: none | mirror | parity ...
    }],
    "disks": [{
      "disk_name":  "string",               // REQ* (if exposed)
      "size_bytes":  number,                // REQ*: bytes
      "free_bytes": "number|null",          // REC: bytes
      "group_ref":  "string|null",          // REC: key to group
      "media_type": "string|null"           // REC: hdd | ssd | unspecified
    }]
  },

  // ── PAGING / SWAP / PAGEFILE ────────────────────────────────────────────
  "paging": [{
    "name":             "string",           // REQ
    "kind":             "string",           // REQ: aix-paging-lv | linux-swap-partition | linux-swap-lv | linux-swap-file | windows-pagefile
    "size_total_bytes":  number,            // REQ: bytes
    "size_used_bytes":  "number|null"       // REC: bytes
  }]
}
```

`REQ*` means required but **nullable** when the corresponding topology does not exist (plain-partition Linux, basic-disk Windows): the key is then present and set to `null` (or the array is empty), never omitted.

---

## 5. Fields to collect, by level

Columns: **Field** | **Type** | **Source AIX** | **Source Linux** | **Source Windows (PowerShell)** | **Obligation**.

### 5.1 Identity & collection metadata

| Field | Type | Source AIX | Source Linux | Source Windows (PowerShell) | Obligation |
|---|---|---|---|---|---|
| host_id | string | CMDB / `uname -f` | CMDB / `hostnamectl` | CMDB / `(Get-CimInstance Win32_ComputerSystemProduct).UUID` | REQ |
| hostname | string | `hostname` | `hostname` | `$env:COMPUTERNAME` | REQ |
| gsma_code | string | inventory | inventory | inventory | REQ |
| host_role_env | string | inventory | inventory | inventory | REQ |
| platform | enum | `aix` | `linux` | `windows` | REQ |
| os_family | enum | `uname -s` → AIX | `/etc/os-release` ID_LIKE | `Windows` | REQ |
| os_distribution | enum | AIX | `/etc/os-release` ID | `(Get-CimInstance Win32_OperatingSystem).Caption` | REQ |
| os_version | string | `oslevel -s` | `/etc/os-release` VERSION_ID | `[Environment]::OSVersion.Version` | REQ |
| architecture | string | `uname -p` / `prtconf` | `uname -m` | `$env:PROCESSOR_ARCHITECTURE` | REQ |
| scan_timestamp | string (UTC ISO-8601) | `date -u` | `date -u` | `(Get-Date).ToUniversalTime().ToString('o')` | REQ |
| collection_method | string | `ssh-shell` | `ssh-shell` | `winrm-powershell` | REQ |
| scan_confidence | number 0..1 | collector | collector | collector | REQ |

> Note: `snapshot_id`, `generated_at`, and `host_count` are **run-level** fields on the envelope (§4.1) and are deliberately **absent** from the host element.
>
> Note: identity enum fields are **normalized** from raw command output to the canonical values; e.g. `/etc/os-release` `ID` / `ID_LIKE` → the `os_family` enum (`RedHat`, `Debian`, `Suse`, …), `Win32_OperatingSystem.Caption` → `WindowsServer`, and `Get-Volume.DriveLetter` (`C`) → a `mount_point` such as `C:`. `CMDB` = configuration management database.

### 5.2 filesystems[] (universal denominator)

| Field | Type | Source AIX | Source Linux | Source Windows (PowerShell) | Obligation |
|---|---|---|---|---|---|
| mount_point | string | `df -k` / `lsfs` | `df -P` / `findmnt` | `Get-Volume`.DriveLetter / Path | REQ |
| label | string\|null | `lsfs` | `lsblk -o LABEL` / `blkid` | `Get-Volume`.FileSystemLabel | REC |
| fstype | string | `lsfs` (jfs2) | `findmnt -o FSTYPE` / `df -T` | `Get-Volume`.FileSystem (NTFS/ReFS) | REQ |
| size_total_bytes | number | `df -k` ×1024 | `df -PB1` / `df --output=size` ×1024 | `Get-Volume`.Size | REQ |
| size_used_bytes | number | `df -k` ×1024 | `df -PB1` used | `Size − SizeRemaining` | REQ |
| size_available_bytes | number | `df -k` ×1024 | `df -PB1` avail | `Get-Volume`.SizeRemaining | REQ |
| reserved_bytes | number\|null | null (JFS2) | **ext: `tune2fs -l`** (rsv blocks × block size); **xfs/btrfs: null** | null (NTFS/ReFS) | COND |
| inodes_total | number\|null | `df -v` / `istat` | ext/xfs: `df -i`; btrfs: null (dynamic inodes) | null | COND |
| inodes_used | number\|null | `df -v` | ext/xfs: `df -i`; btrfs: null (dynamic inodes) | null | COND |
| access_model | enum | `posix-mount-options` | `posix-mount-options` | `ntfs-acl` | REQ |
| mount_options | string\|null | `lsfs` / `/etc/filesystems` | `findmnt -o OPTIONS` | null | COND |
| acl_summary | object\|null | null | null | `Get-Acl` → non-sensitive summary: `inheritance_enabled`, `ace_count`, `well_known_principals`, `non_well_known_ace_count`, `has_everyone_write` (no owners, no named non-well-known principals) | COND |
| backing_kind | enum | `lvm-lv` (jfs2) | `lvm-lv`\|`partition`\|`network`\|`pseudo` | `virtual-disk`\|`partition`\|`network` | REQ |
| backing_ref | string\|null | `lslv` LV name | `lsblk` LV/part | `Get-Partition`/`Get-VirtualDisk` id | REQ* |
| group_ref | string\|null | `lslv` → VG | `lvs -o vg_name` | `Get-StoragePool` FriendlyName | REQ* |
| content_aggregates | object | 2nd pass (non-sensitive) | 2nd pass (non-sensitive) | 2nd pass (non-sensitive) | REC |

> **btrfs (SLES):** `size_*` come from **`btrfs filesystem usage`** (and qgroups), not from `df`, which is misleading with subvolumes/snapshots.

### 5.3 storage_topology.groups[] (lvm-vg | storage-pool)

| Field | Type | Source AIX | Source Linux | Source Windows (PowerShell) | Obligation |
|---|---|---|---|---|---|
| group_name | string | `lsvg` | `vgs -o vg_name` | `Get-StoragePool`.FriendlyName | REQ* |
| group_kind | enum | `lvm-vg` | `lvm-vg` | `storage-pool` | REQ* |
| size_total_bytes | number | `lsvg` (TOTAL PPs × PP size) | `vgs -o vg_size --units b` | `Get-StoragePool`.Size | REQ* |
| size_free_bytes | number | `lsvg` (FREE PPs × PP size) | `vgs -o vg_free --units b` | `Size − AllocatedSize` | REQ* |
| member_disk_count | number | `lsvg -p` | `pvs -o vg_name` \| `vgs -o pv_count` | `Get-PhysicalDisk -StoragePool` count | REQ* |
| state | string | `lsvg` (VG STATE) | `vgs -o vg_attr` | `Get-StoragePool`.HealthStatus | REQ* |
| extent_size_bytes | number\|null | `lsvg` (PP SIZE) | `vgs -o vg_extent_size --units b` | null | COND |
| capabilities.max_lvs | number\|null | `lsvg` (MAX LVs) | null | null | COND |
| capabilities.max_pps | number\|null | `lsvg` (MAX PPs) | null | null | COND |
| capabilities.vg_type | string\|null | `lsvg` (big/scalable) | null | null | COND |
| capabilities.resiliency | string\|null | null | null | `Get-StoragePool`.ResiliencySettingNameDefault | COND |

### 5.4 storage_topology.volumes[] (lvm-lv | partition | virtual-disk)

| Field | Type | Source AIX | Source Linux | Source Windows (PowerShell) | Obligation |
|---|---|---|---|---|---|
| volume_name | string | `lslv` / `lsvg -l` | `lvs -o lv_name` / `lsblk` (part) | `Get-VirtualDisk`.FriendlyName / `Get-Partition` | REQ* |
| volume_kind | enum | `lvm-lv` | `lvm-lv`\|`partition` | `virtual-disk`\|`partition` | REQ* |
| group_ref | string\|null | `lslv` → VG | `lvs -o vg_name` | `Get-VirtualDisk`→pool; null if basic partition | REQ* |
| size_bytes | number | `lslv` (LPs × PP size) | `lvs -o lv_size --units b` / `lsblk -b SIZE` | `Get-VirtualDisk`.Size / `Get-Partition`.Size | REQ* |
| type | string\|null | `lslv` (TYPE jfs2/…) | `lvs -o segtype` (linear/striped) / `gpt-part` | `Get-VirtualDisk`.ProvisioningType / part type | REC |
| redundancy | string\|null | `lslv` (COPIES → mirror) | `lvs -o lv_attr` (mirror/raid) | `Get-VirtualDisk`.ResiliencySettingName | COND |

### 5.5 storage_topology.disks[]

| Field | Type | Source AIX | Source Linux | Source Windows (PowerShell) | Obligation |
|---|---|---|---|---|---|
| disk_name | string | `lspv` (hdiskN) | `pvs -o pv_name` / `lsblk -d` | `Get-Disk`.Number / `Get-PhysicalDisk`.FriendlyName | REQ* |
| size_bytes | number | `lspv` (TOTAL PPs × PP size) | `pvs -o pv_size --units b` / `lsblk -b -d SIZE` | `Get-Disk`.Size / `Get-PhysicalDisk`.Size | REQ* |
| free_bytes | number\|null | `lspv` (FREE PPs × PP size) | `pvs -o pv_free --units b` | `Get-Disk` unallocated | REC |
| group_ref | string\|null | `lspv` → VG | `pvs -o vg_name` | `Get-PhysicalDisk -StoragePool`.FriendlyName | REC |
| media_type | string\|null | null | `lsblk -o ROTA` → hdd/ssd | `Get-PhysicalDisk`.MediaType | REC |

### 5.6 paging[]

| Field | Type | Source AIX | Source Linux | Source Windows (PowerShell) | Obligation |
|---|---|---|---|---|---|
| name | string | `lsps -a` (Page Space) | `swapon --show=NAME` | `Win32_PageFileUsage`.Name | REQ |
| kind | enum | `aix-paging-lv` | `linux-swap-partition`\|`linux-swap-lv`\|`linux-swap-file` | `windows-pagefile` | REQ |
| size_total_bytes | number | `lsps -a` (MB ×1048576) | `swapon --show=SIZE --bytes` | `Win32_PageFileUsage`.AllocatedBaseSize ×1MB | REQ |
| size_used_bytes | number\|null | `lsps -a` (%Used × size) | `swapon --show=USED --bytes` | `Win32_PageFileUsage`.CurrentUsage ×1MB | REC |

### 5.7 content_aggregates (second pass, non-sensitive): `filesystems[].content_aggregates`

| Field | Type | Source AIX | Source Linux | Source Windows (PowerShell) | Obligation |
|---|---|---|---|---|---|
| file_count | number\|null | `find -type f \| wc -l` | `find -type f \| wc -l` | `(Get-ChildItem -Recurse -File).Count` | REC |
| dir_count | number\|null | `find -type d \| wc -l` | `find -type d \| wc -l` | `(Get-ChildItem -Recurse -Directory).Count` | REC |
| largest_file_bytes | number\|null | `find -type f -printf '%s'` max | `find -printf '%s'` max | `Measure-Object Length -Max` | REC |
| oldest_mtime / newest_mtime | string\|null (UTC) | `find -printf '%T@'` min/max | `find -printf '%T@'` min/max | `LastWriteTimeUtc` min/max | REC |
| ext_size_top (per extension) | object\|null | `find` + aggregate | `find` + aggregate | `Group-Object Extension` / `Sum Length` | REC |
| world_writable_dir_count | number\|null | `find -type d -perm -0002 \| wc -l` | `find -type d -perm -0002 \| wc -l` | `Get-ChildItem -Directory` + ACL check, count only | REC |

> Aggregates are **strictly non-nominative** (counts, sizes, dates only, never filenames or contents). This second pass is optional enrichment; the first pass alone already covers all five objectives.

---

## 6. Objective coverage

The **mandatory set** (Identity + `filesystems[]` + `storage_topology` where applicable + `paging[]`) is sufficient to cover all five objectives on **every OS**.

| Objective | Mandatory fields that carry it | AIX | Linux | Windows |
|---|---|---|---|---|
| **O1: Deviation detection vs standards** (FS + groups) | `mount_point`, `fstype`, `size_total_bytes`, `group_*`, `volume_*` | OK | OK | OK |
| **O2: Optimization** (useless/obsolete/misplaced) | `size_used_bytes`/`size_available_bytes`, `group_ref`/`backing_ref` (→ system/data), `content_aggregates` | OK | OK | OK |
| **O3: Reliability & full-FS prevention** (growth) | `size_total/used/available_bytes`, `inodes_*` (Unix), `scan_timestamp` (series) | OK | OK | OK |
| **O4: Operational efficiency & forecasting** | neutral topology `groups/volumes/disks`, `extent_size_bytes`, `paging[]`, UTC series | OK | OK | OK |
| **O5: Risk reduction & compliance** | `group_ref`/`backing_ref` (→ `storage_class`), `access_model`, `acl_summary`/`mount_options`, `content_aggregates.world_writable_dir_count` (CIS, 2nd pass) | OK | OK | OK |

Two OS-specific notes that keep coverage complete and at the same level on all three platforms:

- **Inodes are `null` on Windows and on btrfs (SUSE/SLES)** (O3): NTFS/ReFS have no inode notion and btrfs allocates inodes dynamically (no fixed table), so byte saturation (`size_*`) carries O3 there; ext4/xfs/jfs2 report inodes.
- **`acl_summary` replaces `mount_options` on Windows** (O5): Unix compliance is checked via mount options; Windows compliance is checked via the non-sensitive ACL summary (`access_model = ntfs-acl`).

---

## 7. Derived fields (EXCLUDED from collection)

The following fields are **not collected**; they are **derived downstream** by the pipeline from the raw fields above. They are listed here only to mark the collection / processing boundary.

| Derived field | Derived from | Semantics |
|---|---|---|
| `storage_class` | `group_ref` / `backing_ref` + OS rules | **system vs data**; generalizes rootvg/datavg (AIX), system/data VGs (Linux), and `C:` = system / others = data (Windows) |
| `mount_category` | `mount_point` + standards | normalized role (`/var`, `/tmp`, `D:` data, …) |
| `used_percent` | `size_used_bytes / size_total_bytes` | byte utilization rate |
| `inode_used_percent` | `inodes_used / inodes_total` | inode saturation (Unix) |
| `growth_rate` | snapshot time series | rate of growth |
| `projected_days_to_full` | `growth_rate` + `size_available_bytes` | saturation forecast |
| `deviation_type` | comparison to standards | type of deviation (FS / group / placement) |
| `severity` | `deviation_type` + thresholds | criticality |
| `confidence_score` | `scan_confidence` + cross-checks | confidence of the analysis |

---

## 8. JSON examples

One **envelope file** for a single collection run. Its `hosts[]` array holds five host documents: AIX (JFS2/LVM rootvg+datavg), RHEL (XFS, no LVM, UEFI), Debian/Ubuntu (ext4, populated `reserved_bytes`), SLES (btrfs subvolumes), and Windows (NTFS `C:` system + ReFS `D:` data + Storage Space + pagefile). The envelope carries `snapshot_id`, `generated_at`, and `host_count`; **no host carries its own `snapshot_id`**, each keeps only its `scan_timestamp`.

```json
{
  "snapshot_id": "run-20260612T020000Z-fleet01",
  "generated_at": "2026-06-12T02:20:00Z",
  "host_count": 5,
  "hosts": [
    {
      "host_id": "AIX-PWR-0421-UUID",
      "hostname": "aixprd0421",
      "gsma_code": "FR-PAR-DC1",
      "host_role_env": "prod",
      "platform": "aix",
      "os_family": "AIX",
      "os_distribution": "AIX",
      "os_version": "7.2.5.4",
      "architecture": "ppc64",
      "scan_timestamp": "2026-06-12T02:00:00Z",
      "collection_method": "ssh-shell",
      "scan_confidence": 0.99,
      "filesystems": [
        {
          "mount_point": "/",
          "label": null,
          "fstype": "jfs2",
          "size_total_bytes": 2147483648,
          "size_used_bytes": 1288490188,
          "size_available_bytes": 858993460,
          "reserved_bytes": null,
          "inodes_total": 524288,
          "inodes_used": 38211,
          "access_model": "posix-mount-options",
          "mount_options": "rw,log=/dev/hd8",
          "acl_summary": null,
          "backing_kind": "lvm-lv",
          "backing_ref": "hd4",
          "group_ref": "rootvg",
          "content_aggregates": {}
        },
        {
          "mount_point": "/data",
          "label": null,
          "fstype": "jfs2",
          "size_total_bytes": 107374182400,
          "size_used_bytes": 64424509440,
          "size_available_bytes": 42949672960,
          "reserved_bytes": null,
          "inodes_total": 13107200,
          "inodes_used": 902144,
          "access_model": "posix-mount-options",
          "mount_options": "rw,log=/dev/loglv00",
          "acl_summary": null,
          "backing_kind": "lvm-lv",
          "backing_ref": "datalv00",
          "group_ref": "datavg",
          "content_aggregates": {}
        }
      ],
      "storage_topology": {
        "groups": [
          {
            "group_name": "rootvg",
            "group_kind": "lvm-vg",
            "size_total_bytes": 68719476736,
            "size_free_bytes": 21474836480,
            "member_disk_count": 1,
            "state": "active",
            "extent_size_bytes": 134217728,
            "capabilities": { "max_lvs": 256, "max_pps": 1016, "vg_type": "scalable", "resiliency": null }
          },
          {
            "group_name": "datavg",
            "group_kind": "lvm-vg",
            "size_total_bytes": 214748364800,
            "size_free_bytes": 107374182400,
            "member_disk_count": 2,
            "state": "active",
            "extent_size_bytes": 268435456,
            "capabilities": { "max_lvs": 512, "max_pps": 2032, "vg_type": "scalable", "resiliency": null }
          }
        ],
        "volumes": [
          { "volume_name": "hd4", "volume_kind": "lvm-lv", "group_ref": "rootvg", "size_bytes": 2147483648, "type": "jfs2", "redundancy": "none" },
          { "volume_name": "datalv00", "volume_kind": "lvm-lv", "group_ref": "datavg", "size_bytes": 107374182400, "type": "jfs2", "redundancy": "mirror" }
        ],
        "disks": [
          { "disk_name": "hdisk0", "size_bytes": 68719476736, "free_bytes": 21474836480, "group_ref": "rootvg", "media_type": null },
          { "disk_name": "hdisk1", "size_bytes": 107374182400, "free_bytes": 53687091200, "group_ref": "datavg", "media_type": null },
          { "disk_name": "hdisk2", "size_bytes": 107374182400, "free_bytes": 53687091200, "group_ref": "datavg", "media_type": null }
        ]
      },
      "paging": [
        { "name": "hd6", "kind": "aix-paging-lv", "size_total_bytes": 8589934592, "size_used_bytes": 858993459 }
      ]
    },
    {
      "host_id": "RHEL-VM-1180-UUID",
      "hostname": "rhel9web1180",
      "gsma_code": "FR-LYO-DC2",
      "host_role_env": "prod",
      "platform": "linux",
      "os_family": "RedHat",
      "os_distribution": "RHEL",
      "os_version": "9.4",
      "architecture": "x86_64",
      "scan_timestamp": "2026-06-12T02:05:00Z",
      "collection_method": "ssh-shell",
      "scan_confidence": 0.98,
      "filesystems": [
        {
          "mount_point": "/",
          "label": "root",
          "fstype": "xfs",
          "size_total_bytes": 53687091200,
          "size_used_bytes": 16106127360,
          "size_available_bytes": 37580963840,
          "reserved_bytes": null,
          "inodes_total": 26214400,
          "inodes_used": 184320,
          "access_model": "posix-mount-options",
          "mount_options": "rw,relatime,attr2,inode64,noquota",
          "acl_summary": null,
          "backing_kind": "partition",
          "backing_ref": "/dev/sda2",
          "group_ref": null,
          "content_aggregates": {}
        },
        {
          "mount_point": "/boot/efi",
          "label": null,
          "fstype": "vfat",
          "size_total_bytes": 629145600,
          "size_used_bytes": 12582912,
          "size_available_bytes": 616562688,
          "reserved_bytes": null,
          "inodes_total": null,
          "inodes_used": null,
          "access_model": "posix-mount-options",
          "mount_options": "rw,relatime,fmask=0077,dmask=0077",
          "acl_summary": null,
          "backing_kind": "partition",
          "backing_ref": "/dev/sda1",
          "group_ref": null,
          "content_aggregates": {}
        }
      ],
      "storage_topology": {
        "groups": [],
        "volumes": [
          { "volume_name": "/dev/sda1", "volume_kind": "partition", "group_ref": null, "size_bytes": 629145600, "type": "gpt-part", "redundancy": null },
          { "volume_name": "/dev/sda2", "volume_kind": "partition", "group_ref": null, "size_bytes": 53687091200, "type": "gpt-part", "redundancy": null },
          { "volume_name": "/dev/sda3", "volume_kind": "partition", "group_ref": null, "size_bytes": 4294967296, "type": "gpt-part", "redundancy": null }
        ],
        "disks": [
          { "disk_name": "/dev/sda", "size_bytes": 64424509440, "free_bytes": 5813305344, "group_ref": null, "media_type": "ssd" }
        ]
      },
      "paging": [
        { "name": "/dev/sda3", "kind": "linux-swap-partition", "size_total_bytes": 4294967296, "size_used_bytes": 0 }
      ]
    },
    {
      "host_id": "UBU-VM-2210-UUID",
      "hostname": "ubu22db2210",
      "gsma_code": "FR-PAR-DC1",
      "host_role_env": "prod",
      "platform": "linux",
      "os_family": "Debian",
      "os_distribution": "Ubuntu",
      "os_version": "22.04",
      "architecture": "x86_64",
      "scan_timestamp": "2026-06-12T02:12:30Z",
      "collection_method": "ssh-shell",
      "scan_confidence": 0.98,
      "filesystems": [
        {
          "mount_point": "/",
          "label": "root",
          "fstype": "ext4",
          "size_total_bytes": 32212254720,
          "size_used_bytes": 12884901888,
          "size_available_bytes": 17716740096,
          "reserved_bytes": 1610612736,
          "inodes_total": 1966080,
          "inodes_used": 95000,
          "access_model": "posix-mount-options",
          "mount_options": "rw,relatime,errors=remount-ro",
          "acl_summary": null,
          "backing_kind": "partition",
          "backing_ref": "/dev/sda2",
          "group_ref": null,
          "content_aggregates": {}
        },
        {
          "mount_point": "/boot",
          "label": null,
          "fstype": "ext4",
          "size_total_bytes": 1073741824,
          "size_used_bytes": 268435456,
          "size_available_bytes": 751619277,
          "reserved_bytes": 53687091,
          "inodes_total": 65536,
          "inodes_used": 312,
          "access_model": "posix-mount-options",
          "mount_options": "rw,relatime",
          "acl_summary": null,
          "backing_kind": "partition",
          "backing_ref": "/dev/sda1",
          "group_ref": null,
          "content_aggregates": {}
        }
      ],
      "storage_topology": {
        "groups": [],
        "volumes": [
          { "volume_name": "/dev/sda1", "volume_kind": "partition", "group_ref": null, "size_bytes": 1073741824, "type": "gpt-part", "redundancy": null },
          { "volume_name": "/dev/sda2", "volume_kind": "partition", "group_ref": null, "size_bytes": 32212254720, "type": "gpt-part", "redundancy": null },
          { "volume_name": "/dev/sda3", "volume_kind": "partition", "group_ref": null, "size_bytes": 4294967296, "type": "gpt-part", "redundancy": null }
        ],
        "disks": [
          { "disk_name": "/dev/sda", "size_bytes": 38654705664, "free_bytes": 1073741824, "group_ref": null, "media_type": "ssd" }
        ]
      },
      "paging": [
        { "name": "/dev/sda3", "kind": "linux-swap-partition", "size_total_bytes": 4294967296, "size_used_bytes": 268435456 }
      ]
    },
    {
      "host_id": "SLES-VM-3307-UUID",
      "hostname": "sles15app3307",
      "gsma_code": "FR-PAR-DC1",
      "host_role_env": "preprod",
      "platform": "linux",
      "os_family": "Suse",
      "os_distribution": "SLES",
      "os_version": "15-SP5",
      "architecture": "x86_64",
      "scan_timestamp": "2026-06-12T02:10:00Z",
      "collection_method": "ssh-shell",
      "scan_confidence": 0.95,
      "filesystems": [
        {
          "mount_point": "/",
          "label": "ROOT",
          "fstype": "btrfs",
          "size_total_bytes": 42949672960,
          "size_used_bytes": 15032385536,
          "size_available_bytes": 27917287424,
          "reserved_bytes": null,
          "inodes_total": null,
          "inodes_used": null,
          "access_model": "posix-mount-options",
          "mount_options": "rw,relatime,space_cache=v2,subvolid=257,subvol=/@/.snapshots/1/snapshot",
          "acl_summary": null,
          "backing_kind": "lvm-lv",
          "backing_ref": "lv_root",
          "group_ref": "system",
          "content_aggregates": {}
        }
      ],
      "storage_topology": {
        "groups": [
          {
            "group_name": "system",
            "group_kind": "lvm-vg",
            "size_total_bytes": 53687091200,
            "size_free_bytes": 8589934592,
            "member_disk_count": 1,
            "state": "active",
            "extent_size_bytes": 4194304,
            "capabilities": { "max_lvs": null, "max_pps": null, "vg_type": null, "resiliency": null }
          }
        ],
        "volumes": [
          { "volume_name": "lv_root", "volume_kind": "lvm-lv", "group_ref": "system", "size_bytes": 42949672960, "type": "linear", "redundancy": "none" },
          { "volume_name": "/dev/system/lv_swap", "volume_kind": "lvm-lv", "group_ref": "system", "size_bytes": 2147483648, "type": "linear", "redundancy": "none" }
        ],
        "disks": [
          { "disk_name": "/dev/sda", "size_bytes": 53687091200, "free_bytes": 8589934592, "group_ref": "system", "media_type": "ssd" }
        ]
      },
      "paging": [
        { "name": "/dev/system/lv_swap", "kind": "linux-swap-lv", "size_total_bytes": 2147483648, "size_used_bytes": 268435456 }
      ]
    },
    {
      "host_id": "WIN-SRV-7702-UUID",
      "hostname": "winsrv7702",
      "gsma_code": "FR-LYO-DC2",
      "host_role_env": "prod",
      "platform": "windows",
      "os_family": "Windows",
      "os_distribution": "WindowsServer",
      "os_version": "10.0.20348",
      "architecture": "AMD64",
      "scan_timestamp": "2026-06-12T02:15:00Z",
      "collection_method": "winrm-powershell",
      "scan_confidence": 0.97,
      "filesystems": [
        {
          "mount_point": "C:",
          "label": "System",
          "fstype": "ntfs",
          "size_total_bytes": 137438953472,
          "size_used_bytes": 64424509440,
          "size_available_bytes": 73014444032,
          "reserved_bytes": null,
          "inodes_total": null,
          "inodes_used": null,
          "access_model": "ntfs-acl",
          "mount_options": null,
          "acl_summary": {
            "inheritance_enabled": true,
            "ace_count": 6,
            "well_known_principals": ["SYSTEM", "BUILTIN\\Administrators", "BUILTIN\\Users"],
            "non_well_known_ace_count": 0,
            "has_everyone_write": false
          },
          "backing_kind": "partition",
          "backing_ref": "disk0-part2",
          "group_ref": null,
          "content_aggregates": {}
        },
        {
          "mount_point": "D:",
          "label": "Data",
          "fstype": "refs",
          "size_total_bytes": 2199023255552,
          "size_used_bytes": 879609302220,
          "size_available_bytes": 1319413953332,
          "reserved_bytes": null,
          "inodes_total": null,
          "inodes_used": null,
          "access_model": "ntfs-acl",
          "mount_options": null,
          "acl_summary": {
            "inheritance_enabled": true,
            "ace_count": 4,
            "well_known_principals": ["SYSTEM", "BUILTIN\\Administrators"],
            "non_well_known_ace_count": 1,
            "has_everyone_write": false
          },
          "backing_kind": "virtual-disk",
          "backing_ref": "vdisk-data01",
          "group_ref": "Pool-Data",
          "content_aggregates": {}
        }
      ],
      "storage_topology": {
        "groups": [
          {
            "group_name": "Pool-Data",
            "group_kind": "storage-pool",
            "size_total_bytes": 4398046511104,
            "size_free_bytes": 2199023255552,
            "member_disk_count": 4,
            "state": "Healthy",
            "extent_size_bytes": null,
            "capabilities": { "max_lvs": null, "max_pps": null, "vg_type": null, "resiliency": "Mirror" }
          }
        ],
        "volumes": [
          { "volume_name": "disk0-part2", "volume_kind": "partition", "group_ref": null, "size_bytes": 137438953472, "type": "gpt-part", "redundancy": "none" },
          { "volume_name": "vdisk-data01", "volume_kind": "virtual-disk", "group_ref": "Pool-Data", "size_bytes": 2199023255552, "type": "Fixed", "redundancy": "mirror" }
        ],
        "disks": [
          { "disk_name": "Disk0", "size_bytes": 137975824384, "free_bytes": 0, "group_ref": null, "media_type": "ssd" },
          { "disk_name": "PhysicalDisk1", "size_bytes": 1099511627776, "free_bytes": 549755813888, "group_ref": "Pool-Data", "media_type": "hdd" },
          { "disk_name": "PhysicalDisk2", "size_bytes": 1099511627776, "free_bytes": 549755813888, "group_ref": "Pool-Data", "media_type": "hdd" },
          { "disk_name": "PhysicalDisk3", "size_bytes": 1099511627776, "free_bytes": 549755813888, "group_ref": "Pool-Data", "media_type": "hdd" },
          { "disk_name": "PhysicalDisk4", "size_bytes": 1099511627776, "free_bytes": 549755813888, "group_ref": "Pool-Data", "media_type": "hdd" }
        ]
      },
      "paging": [
        { "name": "C:\\pagefile.sys", "kind": "windows-pagefile", "size_total_bytes": 17179869184, "size_used_bytes": 2147483648 }
      ]
    }
  ]
}
```
