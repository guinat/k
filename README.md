# Projet — Gestion de capacité & détection de déviations (stockage, multi-OS)

Système agentic (LangGraph) qui analyse les configurations de systèmes de fichiers et de
stockage sur un parc **hétérogène** (≥ 500 serveurs) pour : détecter les écarts aux standards,
optimiser le stockage, prévenir les saturations, améliorer l'efficacité opérationnelle et réduire
les risques.

**Périmètre OS :** AIX · Linux toutes distros (RHEL / Oracle / Rocky / Alma, Debian / Ubuntu,
SUSE / SLES) · **Windows** — tous traités **de plein droit** (au même niveau). Le modèle de données est **agnostique**
(une liste `filesystems[]` + une `storage_topology` neutre) ; la notion AIX `rootvg`/`datavg` est
généralisée en un champ dérivé `storage_class` (system / data) valable sur tous les OS.

Ce dépôt contient la **phase de cadrage data + spécification + architecture** (pas encore de code).
Tout est en documents / données structurées pour démarrer proprement.

## Arborescence

```
.
├── README.md                          ← ce fichier (index)
├── msg.txt                            ← message équipe data (FR + EN), multi-OS
├── objectives.txt · data.csv · redhat-std.txt   ← entrées fournies
│
├── docs/
│   ├── data-collection-spec.(pdf|md)         ← LE schéma de collecte agnostique (multi-OS)
│   ├── field-applicability-matrix.(pdf|md)   ← quels champs selon le type de serveur (AIX/RHEL/…/Windows)
│   ├── field-rationale-and-ansible.(pdf|md)  ← pourquoi chaque champ + collecte Ansible par OS
│   ├── storage-standards.(pdf|tex)           ← TOUS les standards de partitionnement par OS/version + sources (recherche)
│   └── data-gap-analysis.(pdf|md|tex)        ← diagnostic initial (sur le data.csv d'origine)
│
├── reference/                         ← golden images PEUPLÉS depuis docs/storage-standards (18 modèles)
│   ├── README.md · _schema.md · _index.yaml  ← héritage + contrat des champs + index machine
│   ├── base.yaml                             ← socle FHS 3.0 + CIS (parent commun)
│   ├── rhel-7|8|9.yaml                        ← (+ Oracle/Rocky/Alma via os_distribution)
│   ├── debian-11|12.yaml · ubuntu-2004|2204|2404.yaml
│   ├── sles-12|15.yaml (btrfs) · aix-71|72|73.yaml (rootvg hd*)
│   └── windows-server-2016|2019|2022|2025.yaml  (parent: null)
│
├── architecture/
│   ├── langraph-architecture.md              ← principes, State, nœuds, 5 diagrammes
│   ├── architecture-diagrams.pdf             ← les 5 diagrammes en un PDF
│   └── diagrams/                             ← chaque schéma en .mmd / .svg / .png (fond blanc) / .pdf
│
└── requirements/
    ├── business-goals.(md|csv)               ← objectifs métier (Id;Area;Description + KPIs)
    ├── requirements.(md|csv)                 ← exigences FR (fonctionnelles) + NFR (non fonct.)
    └── risk-register.(md|csv)                ← registre des risques (multi-OS)
```

## Ordre de lecture conseillé

1. **`objectives.txt`** — le quoi et le pourquoi.
2. **`docs/data-collection-spec.pdf`** — la spécification de collecte agnostique (le contrat data).
   Les niveaux d'obligation y sont calibrés pour couvrir **les 5 objectifs**.
3. **`docs/field-applicability-matrix.pdf`** — « plus ou moins de champs selon le type de serveur » :
   collecté / nullable-N/A / différent, par OS et fstype. C'est le rulebook du playbook.
4. **`docs/field-rationale-and-ansible.pdf`** — justification champ par champ + collecte Ansible
   par OS (AIX · Linux · Windows phase 2).
5. **`msg.txt`** — le message à envoyer à l'équipe data.
6. **`reference/README.md`** — comment sont structurés les standards (modèle en couches).
7. **`architecture/`** — comment la donnée sera traitée (pipeline LangGraph) + diagrammes.
8. **`requirements/`** — business goals, exigences, risques (pilotage / gouvernance).
9. **`docs/data-gap-analysis.pdf`** — le diagnostic initial (sur le `data.csv` d'origine, mono-CSV).

## Décisions clés actées

- **Schéma de stockage AGNOSTIQUE** : `filesystems[]` (dénominateur universel) + `storage_topology`
  neutre (groups / volumes / disks, discriminés par `*_kind`). Un seul schéma pour AIX, Linux, Windows.
- **Champs conditionnels / nullable** par OS+fstype : un champ non applicable = `null` (vérifié N/A),
  jamais absent. La **collecte branche par plateforme**, pas le schéma.
- **Tailles en octets** partout (jamais de Go arrondi ; pas de `ansible_lvm/size_g`).
- **`storage_class` (system / data)** généralise `rootvg`/`datavg` sur tous les OS, via les clés
  `backing_ref` / `group_ref`.
- **Windows de plein droit** : collecte PowerShell/WinRM et standards NTFS au même niveau qu'AIX/Linux.
- **Collecte répétée dans le temps** (série UTC) — indispensable à la prévision.
- **Règles vs LLM** : les règles mesurent/détectent (zéro hallucination sur un chiffre), le LLM trie /
  explique / recommande ; **human-in-the-loop** pour les findings à faible confiance.

## Génération des PDF & diagrammes

- Les PDF (`docs/*.pdf`, `requirements/*.pdf`) sont **générés depuis le Markdown** (source de vérité)
  via Chrome headless. Régénérer : `node /tmp/mdpdf/render.js <in.md> <out.pdf> "Titre" [portrait|landscape]`.
- Les diagrammes Mermaid (en anglais) sont en `.mmd` (source, inline dans le `.md`), `.svg`, `.png`
  (fond blanc) et `.pdf`. Régénérer : `mmdc -i diagrams/x.mmd -o diagrams/x.png -b white -s 3`.

## Reste à faire (prochaines étapes)

- [x] **Golden images peuplés** depuis la recherche sourcée (`reference/*.yaml`, 18 modèles, avec `source_id`).
      Reste : **validation SME** (surtout AIX) et resserrage des tailles douteuses signalées dans `docs/storage-standards`.
- [ ] **JSON Schema validable** (`schema/collection.schema.json`) encodant le schéma agnostique + la
      matrice d'applicabilité (contrat machine-vérifiable pour l'équipe data).
- [ ] **Playbook Ansible** de référence multi-OS (sample 8–12 serveurs incl. AIX, RHEL, Ubuntu, SUSE).
- [ ] **Collecte Windows** : rôle Ansible WinRM/PowerShell sur le même schéma (au même niveau qu'AIX/Linux).
- [ ] **POC LangGraph** : ingestion agnostique + normalisation + 1 branche (standards) bout-en-bout.
- [ ] Choix du **store time-series** et du **checkpointer** (Postgres/Redis) + rétention.
```
