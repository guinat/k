# Matrice d'applicabilité des champs par type de serveur

> Quels champs sont collectés / nullable-N/A / absents / différents selon l'OS et le système de fichiers. C'est le **rulebook du playbook** (complète la spec).

## Matrice d'applicabilite des champs

Legende des cellules : **collecte** = champ renseigne avec une vraie valeur ; **nullable (N/A)** = champ present dans le document mais a `null` car non applicable a cette combinaison OS/fstype ; **absent** = clef non emise (niveau de topologie vide) ; **different** = champ renseigne mais via une autre source/semantique. La commande source est indiquee entre parentheses.

| Champ | AIX (jfs2) | RHEL/Oracle (xfs) | Debian/Ubuntu (ext4) | SUSE/SLES (btrfs) | Linux sans LVM | Windows (NTFS/ReFS) |
|---|---|---|---|---|---|---|
| **host_id, hostname, gsma_code, host_role_env** | collecte (CMDB / `hostname`, `uname -n`) | collecte (CMDB / `hostname`) | collecte (CMDB / `hostname`) | collecte (CMDB / `hostname`) | collecte (CMDB / `hostname`) | collecte (CMDB / `hostname`, `$env:COMPUTERNAME`) |
| **platform** | collecte = `aix` (`uname -s`) | collecte = `linux` (`uname -s`) | collecte = `linux` (`uname -s`) | collecte = `linux` (`uname -s`) | collecte = `linux` (`uname -s`) | collecte = `windows` (`$PSVersionTable` / WinRM) |
| **os_family** | collecte = AIX (`oslevel`) | collecte = RedHat/OracleLinux (`/etc/os-release`) | collecte = Debian (`/etc/os-release`) | collecte = Suse (`/etc/os-release`) | collecte (`/etc/os-release`) | collecte = Windows (`Get-CimInstance Win32_OperatingSystem`) |
| **os_distribution** | collecte = AIX (`oslevel -s`) | collecte = RHEL/OracleLinux (`/etc/os-release`) | collecte = Ubuntu/Debian (`/etc/os-release`) | collecte = SLES (`/etc/os-release`) | collecte (`/etc/os-release`) | collecte = WindowsServer (`Win32_OperatingSystem.Caption`) |
| **os_version, architecture** | collecte (`oslevel -s`, `prtconf` / `uname -p`) | collecte (`/etc/os-release`, `uname -m`) | collecte (`/etc/os-release`, `uname -m`) | collecte (`/etc/os-release`, `uname -m`) | collecte (`/etc/os-release`, `uname -m`) | collecte (`Win32_OperatingSystem.Version`, `$env:PROCESSOR_ARCHITECTURE`) |
| **snapshot_id, scan_timestamp, collection_method, scan_confidence** | collecte (metadonnees pipeline) | collecte (metadonnees pipeline) | collecte (metadonnees pipeline) | collecte (metadonnees pipeline) | collecte (metadonnees pipeline) | collecte (metadonnees pipeline, `collection_method=winrm-powershell`) |
| *filesystems[]* | | | | | | |
| **mount_point** | collecte (`df -k`, `lsfs`) | collecte (`df -k`, `findmnt`) | collecte (`df -k`, `findmnt`) | collecte (`findmnt`, `btrfs filesystem show`) | collecte (`df -k`, `findmnt`) | different = lettre de lecteur / point de montage dossier (`Get-Volume`, `Get-Partition`) |
| **label** | collecte (`lslv`, `lsfs`) | collecte (`blkid`, `xfs_admin -l`) | collecte (`blkid`, `e2label`) | collecte (`blkid`, `btrfs filesystem label`) | collecte (`blkid`) | collecte (`Get-Volume.FileSystemLabel`) |
| **fstype** | collecte = jfs2 (`lsfs -c`) | collecte = xfs (`findmnt -no FSTYPE`) | collecte = ext4 (`findmnt -no FSTYPE`) | collecte = btrfs (`findmnt -no FSTYPE`) | collecte (`findmnt -no FSTYPE`) | collecte = ntfs/refs (`Get-Volume.FileSystem`) |
| **size_total_bytes, size_used_bytes, size_available_bytes** | collecte (`df -k` × 1024) | collecte (`df -B1` / `findmnt -b`) | collecte (`df -B1` / `findmnt -b`) | different = via `btrfs filesystem usage` / qgroups (`df` trompeur a cause des sous-volumes) | collecte (`df -B1` / `findmnt -b`) | collecte = octets natifs (`Get-Volume Size/SizeRemaining`, used = Size - SizeRemaining) |
| **reserved_bytes** | nullable (N/A) — jfs2 sans blocs reserves | nullable (N/A) — xfs sans blocs reserves | collecte (`tune2fs -l` : Reserved block count × Block size) | nullable (N/A) — btrfs sans blocs reserves | collecte si ext4 (`tune2fs -l`) / nullable si xfs | nullable (N/A) — NTFS/ReFS sans blocs reserves |
| **inodes_total, inodes_used** | collecte (`df -v` ou `df -i`) | collecte (`df -i`, `xfs_info`) | collecte (`df -i`) | collecte (`df -i` ; partage dynamiquement entre sous-volumes) | collecte (`df -i`) | nullable (N/A) — pas d'inodes sur NTFS/ReFS |
| **access_model** | collecte = `posix-mount-options` | collecte = `posix-mount-options` | collecte = `posix-mount-options` | collecte = `posix-mount-options` | collecte = `posix-mount-options` | collecte = `ntfs-acl` |
| **mount_options** | collecte (`mount`, `lsfs`) | collecte (`findmnt -no OPTIONS`) | collecte (`findmnt -no OPTIONS`) | collecte (`findmnt -no OPTIONS`, options de sous-volume) | collecte (`findmnt -no OPTIONS`) | different = resume ACL non sensible via `Get-Acl` (champ mount_options laisse a null) |
| **acl_summary** | nullable (N/A) — modele POSIX | nullable (N/A) — modele POSIX | nullable (N/A) — modele POSIX | nullable (N/A) — modele POSIX | nullable (N/A) — modele POSIX | collecte = resume ACL non sensible (`Get-Acl`) |
| **backing_kind** | collecte = lvm-lv (`lslv`, `lsvg -l`) | collecte = lvm-lv \| partition (`lsblk -o TYPE`, `lvs`) | collecte = lvm-lv \| partition (`lsblk -o TYPE`, `lvs`) | collecte = lvm-lv \| partition \| btrfs (`lsblk -o TYPE`) | collecte = partition \| virtual-disk (`lsblk -o TYPE`) | collecte = virtual-disk \| partition \| storage-space (`Get-Partition`, `Get-VirtualDisk`) |
| **backing_ref, group_ref** | collecte (nom LV / rootvg-datavg, `lsvg -l`) | collecte (nom LV / VG, `lvs -o lv_name,vg_name`) | collecte (nom LV / VG, `lvs`) | collecte (nom LV/VG ou device btrfs) | collecte = backing_ref (device) ; group_ref nullable (pas de VG) | collecte (`Get-Partition.DiskNumber`, ref pool/virtual-disk) |
| **content_aggregates** | collecte (agregats 2e vague non sensibles) | collecte (agregats 2e vague) | collecte (agregats 2e vague) | collecte (agregats 2e vague) | collecte (agregats 2e vague) | collecte (agregats 2e vague) |
| *storage_topology.groups[]* | | | | | | |
| **groups[] (niveau)** | collecte = lvm-vg (`lsvg`) | collecte si LVM = lvm-vg (`vgs`) | collecte si LVM = lvm-vg (`vgs`) | collecte si LVM = lvm-vg (`vgs`) / sinon absent | absent (plain partitions) | collecte si Storage Spaces = storage-pool (`Get-StoragePool`) / sinon absent (basic disk) |
| **group_name, group_kind** | collecte (`lsvg`, kind=lvm-vg) | collecte (`vgs -o vg_name`, kind=lvm-vg) | collecte (`vgs`, kind=lvm-vg) | collecte si LVM (`vgs`) | absent | collecte (`Get-StoragePool.FriendlyName`, kind=storage-pool) |
| **size_total_bytes, size_free_bytes** | collecte (`lsvg` : TOTAL/FREE PPs × PP size) | collecte (`vgs --units b -o vg_size,vg_free`) | collecte (`vgs --units b`) | collecte si LVM (`vgs --units b`) | absent | collecte (`Get-StoragePool Size/AllocatedSize`) |
| **member_disk_count** | collecte (`lsvg -p` / ACTIVE PVs) | collecte (`vgs -o pv_count`) | collecte (`vgs -o pv_count`) | collecte si LVM (`vgs -o pv_count`) | absent | collecte (`Get-PhysicalDisk` du pool) |
| **state** | collecte (`lsvg` VG STATE) | collecte (`vgs -o vg_attr`) | collecte (`vgs -o vg_attr`) | collecte si LVM (`vgs`) | absent | collecte (`Get-StoragePool.HealthStatus/OperationalStatus`) |
| **extent_size_bytes** | collecte = PP size (`lsvg` PP SIZE) | collecte = PE size (`vgs -o vg_extent_size --units b`) | collecte = PE size (`vgs --units b`) | collecte si LVM (`vgs`) | absent | nullable (N/A) — pas d'extents sur Storage Spaces |
| **capabilities.max_lvs, max_pps, vg_type** | collecte (`lsvg` : MAX LVs, MAX PPs, VG type big/scalable) | nullable (N/A) — concepts AIX | nullable (N/A) — concepts AIX | nullable (N/A) — concepts AIX | absent (pas de groups) | nullable (N/A) — concepts AIX |
| **capabilities.resiliency (Windows)** | nullable (N/A) | nullable (N/A) | nullable (N/A) | nullable (N/A) | absent (pas de groups) | collecte (`Get-StoragePool` / `Get-ResiliencySetting` : mirror/parity/simple) |
| *storage_topology.volumes[]* | | | | | | |
| **volumes[] (niveau)** | collecte = lvm-lv (`lslv`) | collecte si LVM = lvm-lv (`lvs`) / partition sinon | collecte si LVM = lvm-lv (`lvs`) / partition sinon | collecte si LVM = lvm-lv (`lvs`) | collecte = partition (`lsblk`) / groupe vide | collecte = virtual-disk (Storage Spaces) ou partition (basic disk) |
| **volume_name, volume_kind** | collecte (`lslv`, kind=lvm-lv) | collecte (`lvs -o lv_name`, kind=lvm-lv) | collecte (`lvs`, kind=lvm-lv) | collecte si LVM (`lvs`) | collecte (`lsblk -o NAME`, kind=partition) | collecte (`Get-VirtualDisk.FriendlyName` kind=virtual-disk \| `Get-Partition` kind=partition) |
| **group_ref** | collecte = rootvg/datavg (`lslv`) | collecte = VG (`lvs -o vg_name`) | collecte = VG (`lvs -o vg_name`) | collecte si LVM (`lvs`) | nullable (N/A) — pas de VG | collecte = pool ref (`Get-VirtualDisk`) / nullable si basic disk |
| **size_bytes** | collecte (`lslv` : LPs × PP size) | collecte (`lvs --units b -o lv_size`) | collecte (`lvs --units b`) | collecte si LVM (`lvs --units b`) | collecte (`lsblk -b -o SIZE`) | collecte (`Get-VirtualDisk.Size` / `Get-Partition.Size`) |
| **type** | collecte (`lslv` TYPE jfs2/jfs2log/paging) | collecte (linear/thin, `lvs -o lv_layout`) | collecte (linear/thin, `lvs -o lv_layout`) | collecte si LVM (`lvs`) | collecte (type partition, `lsblk -o PARTTYPE`) | collecte (`Get-VirtualDisk` / type partition GPT) |
| **redundancy** | collecte (`lslv` COPIES = mirroring) | collecte (mirror/raid si lvmraid, `lvs -o lv_attr`) | collecte (`lvs -o lv_attr`) | collecte (profil RAID btrfs si pertinent / `lvs`) | nullable (N/A) — partition simple | collecte = resiliency (`Get-VirtualDisk.ResiliencySettingName` mirror/parity/simple) |
| *storage_topology.disks[]* | | | | | | |
| **disk_name** | collecte (`lspv` hdiskN) | collecte (`pvs -o pv_name` / `lsblk`) | collecte (`pvs` / `lsblk`) | collecte (`pvs` / `lsblk`) | collecte (`lsblk -d -o NAME`) | collecte (`Get-Disk.Number/FriendlyName`, `Get-PhysicalDisk`) |
| **size_bytes** | collecte (`lspv` / `bootinfo -s hdiskN` × 1024) | collecte (`pvs --units b` / `lsblk -b -d -o SIZE`) | collecte (`pvs --units b` / `lsblk -b`) | collecte (`lsblk -b -d`) | collecte (`lsblk -b -d -o SIZE`) | collecte (`Get-Disk.Size` / `Get-PhysicalDisk.Size`) |
| **free_bytes** | collecte (`lspv` FREE PPs × PP size) | collecte si LVM (`pvs -o pv_free --units b`) / nullable sinon | collecte si LVM (`pvs --units b`) / nullable sinon | collecte si LVM (`pvs`) / nullable sinon | nullable (N/A) — pas de notion VG-free | collecte (`Get-PhysicalDisk` espace non alloue du pool) / nullable si basic disk |
| **group_ref** | collecte = rootvg/datavg (`lspv`) | collecte = VG si LVM (`pvs -o vg_name`) / nullable sinon | collecte = VG si LVM (`pvs`) / nullable sinon | collecte = VG si LVM / nullable sinon | nullable (N/A) — pas de VG | collecte = pool si Storage Spaces (`Get-PhysicalDisk` -> pool) / nullable si basic |
| **media_type** | collecte (`lsdev`, `lsmpio` / SSD-HDD si expose) | collecte (`lsblk -d -o ROTA` -> ssd/hdd) | collecte (`lsblk -d -o ROTA`) | collecte (`lsblk -d -o ROTA`) | collecte (`lsblk -d -o ROTA`) | collecte (`Get-PhysicalDisk.MediaType` SSD/HDD/Unspecified) |
| *paging[]* | | | | | | |
| **name, kind** | collecte = aix-paging-lv (`lsps -a`) | collecte = linux-swap-partition \| linux-swap-file (`swapon --show`) | collecte = linux-swap-partition \| linux-swap-file (`swapon --show`) | collecte (`swapon --show` ; zswap/swapfile possibles) | collecte (`swapon --show`) | collecte = windows-pagefile (`Win32_PageFileUsage`) |
| **size_total_bytes, size_used_bytes** | collecte (`lsps -a` : Size Mo × %Used) | collecte (`swapon --show=SIZE,USED --bytes`) | collecte (`swapon --show=SIZE,USED --bytes`) | collecte (`swapon --show --bytes`) | collecte (`swapon --show --bytes`) | collecte (`Win32_PageFileUsage AllocatedBaseSize/CurrentUsage` Mo -> octets) |

## Regles de branchement Ansible

Le module de collecte branche d'abord sur `ansible_facts['system']` / `platform`, puis sur le fstype par filesystem. Une seule tache logique par OS, sorties normalisees vers le schema v2 (octets partout).

- **Detection initiale** : `gather_facts` (ou `setup`) -> `ansible_system` ∈ { AIX, Linux, Win32NT }. Le `connection` est deja choisi par l'inventaire : `ssh` pour AIX/Linux, `winrm` (ou `psrp`) pour Windows.

- **Branche AIX** (`when: ansible_system == "AIX"`) : connexion SSH. Topologie LVM via `lsvg`, `lsvg -l <vg>`, `lsvg -p <vg>`, `lslv <lv>`, `lspv` ; filesystems via `df -k` + `lsfs -c` (jfs2) ; paging via `lsps -a`. `reserved_bytes` force a null, `acl_summary` null, `capabilities.max_lvs/max_pps/vg_type` renseignes. Conversions Ko/PP -> octets cote parsing.

- **Branche Linux** (`when: ansible_system == "Linux"`), puis sous-branche par `fstype` (issu de `findmnt -no FSTYPE` par mount) :
  - **xfs (RHEL/Oracle/Rocky/Alma)** : `df -B1` + `df -i` + `xfs_info` ; `reserved_bytes` = null. `/boot/efi` collecte seulement si UEFI x86_64 (`test -d /sys/firmware/efi`).
  - **ext4 (Debian/Ubuntu)** : `df -B1` + `df -i` + `tune2fs -l` pour `reserved_bytes` (Reserved block count × Block size).
  - **btrfs (SUSE/SLES)** : NE PAS se fier a `df` ; tailles vraies via `btrfs filesystem usage <mp>` et `btrfs qgroup show` (sous-volumes + snapshots) ; `reserved_bytes` = null.
  - **Sous-branche LVM** (orthogonale au fstype, `when: lvs présent`) : `vgs/lvs/pvs --units b --nosuffix` -> groups/volumes/disks. Si pas de LVM (plain partitions) : `storage_topology.groups` absent, `volumes[]` = partitions (`lsblk -b`), `group_ref` null.
  - **paging** : `swapon --show --bytes` (distingue partition vs fichier swap).

- **Branche Windows** (`when: ansible_os_family == "Windows"`) : connexion WinRM/PSRP, modules `ansible.windows.win_powershell` / `win_shell`. Inventaire PowerShell : `Get-Volume`, `Get-Partition`, `Get-Disk`, `Get-PhysicalDisk`, `Get-StoragePool`, `Get-VirtualDisk`, `Win32_PageFileUsage`, `Get-Acl` (resume ACL non sensible). Mapping : pool -> `groups[]` (storage-pool), virtual disk -> `volumes[]` (virtual-disk), basic disk -> partitions en `volumes[]` avec `groups` absent. `inodes_*` = null, `mount_options` = null (remplace par `acl_summary`, `access_model=ntfs-acl`), `reserved_bytes` = null. Tailles deja en octets natifs. Volume systeme (BootDevice / emplacement de Windows) -> `storage_class=system`, autres -> `data`.

- **Derivation post-collecte (commune)** : lien filesystems <-> topologie par clefs `group_ref`/`backing_ref` ; `storage_class` (system vs data) derive du group_ref (rootvg / C: / volume de boot = system, datavg / autres = data). Aucune branche n'omet un champ obligatoire : les champs OS-specifiques sont mis a `null`, jamais supprimes.

## Principe nullable vs absent

Deux mecanismes distincts, a ne pas confondre :

- **nullable (N/A)** : la clef EST presente dans le document JSON, avec la valeur `null`. On l'emploie quand le champ appartient au niveau de denominateur universel ou a un objet effectivement instancie, mais que la valeur n'a pas de sens pour cette combinaison OS/fstype. Exemples : `reserved_bytes` = null sur xfs/jfs2/btrfs/ntfs (seul ext4 le renseigne) ; `inodes_total/used` = null sur Windows ; `acl_summary` = null sur Unix ; `mount_options` = null sur Windows ; `capabilities.max_lvs/max_pps/vg_type` = null hors AIX ; `extent_size_bytes` = null sur Storage Spaces ; `group_ref`/`free_bytes` = null sur disque/partition hors LVM. Garder la clef preserve un schema stable, colonne-compatible, et permet de DISTINGUER "non applicable" de "non collecte".

- **absent** : la clef (ou le niveau entier de `storage_topology`) N'EST PAS emise, parce que la structure n'existe pas sur l'hote. C'est le cas des NIVEAUX de topologie vides : `storage_topology.groups` est absent (tableau vide) sur du Linux en partitions simples sans LVM et sur du Windows en disques basiques (pas de Storage Spaces). On ne fabrique pas de groupe fictif ; les filesystems pointent alors directement sur des `volumes[]` de type partition, et `group_ref` vaut null.

Regle d'or : un champ OBLIGATOIRE n'est jamais silencieusement supprime. S'il ne s'applique pas, il devient `null` (nullable) et reste auditable. Seuls les NIVEAUX de topologie structurellement inexistants sont `absent` (collection vide), ce qui se lit comme "cet hote n'a pas cette couche d'abstraction de stockage" et non comme "donnee manquante". Cette discipline garantit que l'ensemble obligatoire couvre les 5 objectifs sur chaque OS, tout en gardant les champs specifiques OS/fstype nullable plutot que perdus.
