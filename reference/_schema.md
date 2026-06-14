# Contrat des modèles de référence (`reference/*.yaml`)

Chaque fichier `reference/<model>.yaml` est un **golden image** : la configuration de stockage
*attendue* pour un profil de serveur (OS + version). Les fichiers sont **générés depuis la recherche
sourcée** (`docs/storage-standards.pdf`) et **conservent les `source_id`** pour la traçabilité.

> ⚠️ **À vérifier contre les sources citées avant de figer.** Les tailles en octets sont une
> *interprétation* de la chaîne source (préservée dans `size_source`). Les standards évoluent par version.

## Modèle en couches (héritage)

```
base.yaml                         ← socle FHS 3.0 + CIS (Linux Foundation / CIS)
  ├─ rhel-7 / rhel-8 / rhel-9     (parent: base)   [aussi Oracle Linux, Rocky, AlmaLinux]
  ├─ debian-11 / debian-12        (parent: base)
  ├─ ubuntu-2004 / 2204 / 2404    (parent: base)
  ├─ sles-12 / sles-15            (parent: base)   [btrfs + sous-volumes]
  └─ aix-71 / aix-72 / aix-73     (parent: base)
windows-server-2016/2019/2022/2025 (parent: null)  ← FHS ne s'applique pas (NTFS/ACL/Storage Spaces)
```

Un modèle enfant ne déclare que ce qui le distingue de son `parent` ; à la résolution, on fusionne
`parent → enfant` (l'enfant gagne). `_index.yaml` liste tous les modèles (clé de jointure :
`os_family` + version ; pour RHEL, `os_distribution` couvre aussi Oracle/Rocky/Alma).

## Champs d'un fichier

| Champ | Sens |
|---|---|
| `model_id` | identifiant du modèle (= nom de fichier) |
| `parent` | modèle parent dont on hérite (`base`), ou `null` |
| `os_family` | famille (RHEL family, Debian/Ubuntu, SUSE/SLES, AIX, Windows Server, FHS/CIS) |
| `os_distribution` | distributions couvertes par ce modèle (aliases, ex. RHEL ⇒ Oracle/Rocky/Alma) |
| `os_version` | version visée |
| `default_fstype` | système de fichiers par défaut de l'OS |
| `swap_guidance` | règle de dimensionnement swap/paging (texte) |
| `notes` | remarques de version |
| `expected_filesystems[]` | la liste des points de montage/volumes attendus (voir ci-dessous) |
| `sources{}` | dictionnaire `id → {title, publisher, url, version_or_date, retrieved}` référencé par les lignes |

### `expected_filesystems[]` — une entrée par point de montage / volume attendu

| Champ | Sens |
|---|---|
| `mount_point` | `/`, `/var`, `C:`, `EFI System Partition`… |
| `classification` | `system` / `data` / `boot` / `swap` / `mixed` |
| `requirement` | `mandatory` / `recommended` / `conditional` / `optional` |
| `mandatory` | booléen dérivé (`requirement == mandatory`) |
| `separate_filesystem_required` | `yes` / `no` / `conditional` |
| `expected_fstype` | type de FS attendu (xfs/ext4/btrfs/jfs2/ntfs/refs/vfat…) |
| `min_size_bytes` / `max_size_bytes` | bornes en **octets** (`null` = non spécifié / illimité) |
| `size_source` | `{min, max}` : la chaîne d'origine de la source (ex. `5.0 GiB`, `unbounded`) — fait foi |
| `expected_mount_options` | options/durcissement attendus (CIS : nodev/nosuid/noexec…) ; sur Windows, posture ACL |
| `source_id[]` | renvois vers `sources{}` (traçabilité) |
| `notes` | précisions |

## Comment le pipeline l'utilise

1. Match d'un hôte → modèle par `(os_family/os_distribution, os_version)` ; si aucun → finding `no_reference`.
2. Résolution de l'héritage `base → modèle`.
3. Comparaison du `filesystems[]`/`storage_topology` collecté (schéma agnostique) au modèle :
   présence/séparation, `min/max_size_bytes`, `expected_fstype`, `expected_mount_options`,
   `classification` (system vs data, généralise rootvg/datavg).
4. Les `source_id` permettent de citer la règle exacte dans les rapports de déviation.

## Régénération

Ces fichiers sont produits à partir de `docs/storage-standards` (recherche sourcée) ; pour les
reconstruire après une nouvelle recherche, relancer le générateur (cf. note projet).
