# Modèles de référence (« golden images ») — peuplés depuis la recherche sourcée

Référentiel **lisible par machine** des structures de stockage *attendues* par profil de serveur.
Les fichiers `*.yaml` sont **générés depuis `docs/storage-standards`** (recherche sur les docs
officielles + CIS + FHS) et **conservent les `source_id`** pour la traçabilité.

> ⚠️ Tailles en **octets** (interprétation ; la chaîne d'origine est dans `size_source`). **Vérifier
> contre les sources citées avant de figer** — les standards évoluent par version.

## Contenu (18 modèles + index)

| Couche | Fichiers |
|---|---|
| **Socle** | `base.yaml` (FHS 3.0 + CIS) |
| **RHEL family** *(+ Oracle/Rocky/Alma)* | `rhel-7.yaml` · `rhel-8.yaml` · `rhel-9.yaml` |
| **Debian / Ubuntu** | `debian-11.yaml` · `debian-12.yaml` · `ubuntu-2004.yaml` · `ubuntu-2204.yaml` · `ubuntu-2404.yaml` |
| **SUSE / SLES** *(btrfs)* | `sles-12.yaml` · `sles-15.yaml` |
| **AIX** *(rootvg hd\*)* | `aix-71.yaml` · `aix-72.yaml` · `aix-73.yaml` |
| **Windows Server** | `windows-server-2016.yaml` · `2019` · `2022` · `2025` |
| **Index / contrat** | `_index.yaml` (liste machine) · `_schema.md` (contrat des champs) |

## Modèle en couches

Tout hérite de `base.yaml` (FHS/CIS), **sauf Windows** (`parent: null` — NTFS/ACL/Storage Spaces
n'entrent pas dans FHS). RHEL couvre Oracle Linux / Rocky / AlmaLinux via `os_distribution`.

## Utilisation par le pipeline

1. Matcher l'hôte → un modèle via `(os_family/os_distribution, os_version)` ; aucun match ⇒ finding `no_reference`.
2. Résoudre l'héritage `base → modèle`.
3. Comparer le stockage collecté (schéma agnostique `filesystems[]` + `storage_topology`) au modèle :
   présence/séparation, tailles (`min/max_size_bytes`), `expected_fstype`, `expected_mount_options`,
   classification `system` vs `data` (généralise rootvg/datavg, tous OS).
4. Citer la règle via `source_id` dans les rapports de déviation.

Détail des champs : voir `_schema.md`. Vue documentaire complète + bibliographie : `docs/storage-standards.pdf`.
