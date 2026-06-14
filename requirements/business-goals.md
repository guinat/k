# Business Goals

> Registre des objectifs metier derives des 5 objectifs du projet. Format demande : `Id;Area;Description` (+ details et KPIs ci-dessous). Voir aussi `business-goals.csv`.
>
> **Périmètre :** parc multi-OS — AIX · Linux toutes distros (RHEL/Oracle/Rocky/Alma, Debian/Ubuntu, SUSE/SLES) · Windows — tous de plein droit. Les objectifs ci-dessous sont OS-agnostiques.

## Business Goals

| Id | Area | Description |
|----|------|-------------|
| B-01 | Risk Reduction | Prevent unplanned outages caused by full filesystems by continuously tracking size_used_bytes vs size_total_bytes per FS and inode exhaustion (inodes_used vs inodes_total), flagging saturation before it causes application crashes, failed writes, or AIX/Linux boot failures on critical mounts (/, /var, /tmp). |
| B-02 | Capacity Planning | Enable accurate storage growth forecasting by collecting usage metrics repeatedly over time (UTC time series) and computing growth_rate and projected_days_to_full, so procurement and expansion of VG/PV capacity is planned proactively instead of reacting to emergencies. |
| B-03 | Cost Saving | Reduce storage spend by identifying useless, obsolete or stale data via mtime_age_histogram, last_modified_age_days and large_alloc_histogram, allowing reclamation of allocated-but-wasted capacity and deferral of disk purchases across the >=500 server fleet. |
| B-04 | Risk Reduction | Eliminate client/business data sitting on system partitions by detecting data_present_on_system_flag and system_partition_unexpected_usage_bytes (data on rootvg/system mounts), reducing data-loss exposure, backup-scope errors, and compliance breaches. |
| B-05 | Compliance | Enforce storage configuration policies by comparing actual placement against machine-readable golden image models (RHEL 8/9, AIX hd-series, other distros) and producing auditable deviation reports with severity, supporting internal and client audit obligations. |
| B-06 | Standardization | Drive uniform filesystem and volume-group structures across the mixed AIX + Linux fleet by normalizing account-specific naming and measuring deviation_type counts against reference standards, lowering the cognitive load and error rate of operations teams. |
| B-07 | Operational Efficiency | Accelerate incident resolution by delivering prioritized, actionable remediation recommendations (recommendation_text with severity and confidence_score) so engineers act on a ranked worklist instead of manually diagnosing each host. |
| B-08 | Reliability | Improve fleet reliability by validating correct separation of datavg from rootvg and presence of expected separate filesystems (is_separate_filesystem), preventing single-VG saturation from taking down the operating system. |
| B-09 | Scalability | Support smooth fleet growth by operating a metadata-only, Ansible-collected pipeline that scales to 500+ heterogeneous servers, with snapshot_id/scan_timestamp versioning enabling onboarding of new hosts and standards without re-architecture. |
| B-10 | Cost Saving | Optimize allocated-vs-used capacity by surfacing over-provisioned volume groups (vg_free_bytes vs vg_size_bytes) and under-utilized logical volumes, enabling reallocation of free PE/PV capacity instead of provisioning new storage. |
| B-11 | Risk Reduction | Reduce security and integrity risk by reporting world_writable_dir_count and non-compliant mount_options (e.g. missing nodev/nosuid on data mounts), tightening the attack surface on system partitions. |
| B-12 | Operational Efficiency | Increase analyst trust and throughput by routing only low-confidence findings (confidence_score below threshold) to account teams for validation, while auto-confirming high-confidence deviations, minimizing manual review effort. |
| B-13 | Capacity Planning | Provide management visibility through dashboards and structured data aggregating capacity, deviation and forecast KPIs across accounts (gsma_code) and environments (prod/test), enabling data-driven storage governance decisions. |

## Détails

**B-01 — Prevent full-filesystem outages.** Business value: an outage from a 100% full / or /var is one of the most common and costly incidents on both AIX and Linux; catching it before it happens avoids downtime and emergency interventions. Maps to objectives 3 (reliability) and 5 (risk reduction). KPI: number of filesystems reaching >90% / 100% usage per month; count of saturation-related incidents avoided (predicted-and-remediated before threshold breach).

**B-02 — Storage growth forecasting.** Business value: turns storage capacity from a reactive cost into a planned budget line, avoiding rushed purchases at premium prices and avoiding overcapacity. Maps to objectives 3 (smooth growth) and 4 (forecasting). KPI: forecast accuracy (predicted vs actual days-to-full, error %); lead time between full-prediction and remediation; % of capacity additions that were planned vs emergency.

**B-03 — Reclaim useless / obsolete data.** Business value: stale and obsolete data consumes paid capacity and lengthens backup windows; identifying it (without reading filenames, using age and size histograms only) lets teams archive or delete and defer purchases. Maps to objectives 2 (optimization) and 1 (cost). KPI: bytes identified as reclaimable; bytes actually reclaimed per quarter; storage purchase deferral in currency.

**B-04 — Remove client data from system partitions.** Business value: business/client data on rootvg or system mounts is rarely backed up the same way, inflates system FS, and can violate data-handling agreements; flagging it reduces data-loss and compliance exposure. Maps to objectives 2 (misplaced data), 5 (appropriate data on system partitions) and 1 (deviation detection). KPI: count of hosts with data_present_on_system_flag set; total system_partition_unexpected_usage_bytes; time-to-remediation.

**B-05 — Policy compliance via golden images.** Business value: a machine-readable reference per OS family makes compliance objective and auditable instead of relying on RHEL prose only; deviation reports become defensible audit evidence for clients. Maps to objectives 1 (deviation detection) and 5 (compliance). KPI: % of fleet compliant per golden image; number of open high-severity deviations; audit findings raised vs prior period.

**B-06 — Standardized structures.** Business value: consistent FS/VG layouts across a mixed AIX + Linux estate reduce human error, simplify automation, and make every server predictable to operate. Maps to objectives 4 (standardized structures) and 1. KPI: deviation_type count trend over time; % of hosts matching reference layout; variance in layouts per account.

**B-07 — Actionable, prioritized remediation.** Business value: severity- and confidence-ranked recommendations convert raw findings into a triaged worklist, cutting diagnosis time and ensuring the highest-impact issues are fixed first. Maps to objectives 4 (faster incident resolution) and 1. KPI: mean time to resolution for storage deviations; % of recommendations actioned; backlog of high-severity items.

**B-08 — RootVG / DataVG separation and reliability.** Business value: ensuring data lives on a separate datavg (and expected mounts are separate filesystems) prevents a runaway application from filling the OS volume group and crashing the host. Maps to objectives 3 (prevent failures) and 5. KPI: % of hosts with correct rootvg/datavg separation; count of hosts where a single VG hosts both system and data.

**B-09 — Scalable metadata-only pipeline.** Business value: a non-sensitive, Ansible-driven collection model scales across 500+ heterogeneous servers without per-host bespoke tooling, and snapshot/timestamp versioning lets new hosts and new standards be added cleanly. Maps to objectives 3 (scalability) and 4. KPI: number of hosts onboarded per release; collection success rate across the fleet; time to add a new OS golden image.

**B-10 — Over-provisioning optimization.** Business value: large gaps between vg_size_bytes and vg_free_bytes, or LVs allocated far above usage, represent paid-for-but-idle capacity that can be reallocated rather than expanded. Maps to objectives 2 (optimization) and 3 (smooth growth). KPI: aggregate vg_free_bytes that is reallocatable; number of VGs flagged over-provisioned; capacity reallocated vs newly purchased.

**B-11 — Security posture of storage config.** Business value: world-writable directories and weak mount options on system/data partitions widen the attack surface and breach hardening baselines; reporting them feeds remediation and audit. Maps to objectives 5 (risk reduction / compliance) and 1. KPI: world_writable_dir_count trend; number of mounts missing required nodev/nosuid/noexec options.

**B-12 — Confidence-based validation routing.** Business value: auto-confirming high-confidence deviations and escalating only low-confidence ones to account teams concentrates scarce human review where it adds value, raising trust in the system's output. Maps to objectives 1 (validate low-confidence findings) and 4. KPI: % of findings auto-confirmed vs escalated; false-positive rate on auto-confirmed findings; validation turnaround time.

**B-13 — Management dashboards and governance.** Business value: aggregating capacity, deviation, and forecast metrics by account (gsma_code) and environment (prod/test) gives leadership the visibility needed to govern storage decisions and prioritize investment. Maps to objectives 4 (forecasting / efficiency) and 1. KPI: dashboard adoption by account teams; number of governance decisions driven by the data; reduction in unmanaged deviations quarter over quarter.
