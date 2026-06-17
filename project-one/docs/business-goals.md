# Business Goals

These are the business goals derived from the five project objectives (O1-O5). They translate the technical objectives into outcomes the organization can plan, fund, and measure. The fleet is heterogeneous and multi-OS: AIX, every Linux distribution (RHEL / Oracle Linux / Rocky / AlmaLinux, Debian / Ubuntu, SUSE / SLES), and Windows Server, all delivered together as first-class platforms in a single release. Accordingly, every goal below is expressed in OS-agnostic terms that map onto the single unified storage collection schema, so that the same business value is realized identically across all operating systems.

Throughout, "system vs data" is expressed through the derived `storage_class` (system | data). This generalizes the AIX `rootvg`/`datavg` separation idea to all platforms: AIX system vs data volume groups, Linux system vs data volume groups (or plain partitions), and the Windows C: system drive vs other data drives. The `rootvg`/`datavg` names are used in this document only as illustrative examples of the system-vs-data idea, not as the data model itself.

A reminder on the data model these goals rely on: capacities are always **exact bytes** (integers, never percentages or human-readable units); a single envelope file is produced per collection run; the per-host document is OS-agnostic; and derived values, namely `storage_class`, all percentages, growth rates, deviations, severities, and confidence, are computed downstream by the pipeline, not collected.

## Business Goals

| Id | Area | Description |
|----|------|-------------|
| B-01 | Risk Reduction | Prevent unplanned outages caused by filesystems filling up, by continuously tracking each filesystem's `size_used_bytes` against `size_total_bytes` and inode exhaustion (`inodes_used` vs `inodes_total`), flagging saturation before it causes application crashes, failed writes, or boot failures on critical mounts (for example `/`, `/var`, `/tmp` on Unix, and the C: system drive on Windows). |
| B-02 | Capacity Planning | Enable accurate storage growth forecasting by collecting usage metrics repeatedly over time (append-only UTC time series) and deriving growth rate and projected days-to-full, so that procurement and expansion of storage capacity is planned proactively at the storage-group and disk level instead of handled as an emergency. |
| B-03 | Cost Saving | Reduce storage spend by identifying useless, obsolete, or stale data through non-sensitive age and size signals (file/directory counts, largest-file size, oldest/newest modification times, and per-extension size totals; never filenames or contents), enabling reclamation of allocated-but-wasted capacity and deferral of disk purchases across the fleet of 500+ servers. |
| B-04 | Risk Reduction | Eliminate business and client data sitting on system partitions by detecting application data on filesystems whose derived `storage_class` is `system` and quantifying the unexpected bytes consumed there, reducing data-loss exposure, backup-scope errors, and compliance breaches. |
| B-05 | Compliance | Enforce storage configuration policies by comparing each host against the per-OS reference standards ("golden images") and producing auditable deviation reports with derived severity, supporting internal and client audit obligations across all operating systems. |
| B-06 | Standardization | Drive uniform filesystem and storage-topology structures across the mixed AIX, Linux, and Windows fleet by normalizing account-specific naming and measuring deviation counts against the reference standards, lowering the cognitive load and error rate of operations teams. |
| B-07 | Operational Efficiency | Accelerate incident resolution by delivering prioritized, actionable remediation recommendations (each carrying a derived severity and confidence) so engineers work from a ranked worklist instead of manually diagnosing each host. |
| B-08 | Reliability | Improve fleet reliability by validating correct separation of data storage from system storage (data filesystems on `storage_class = data`, system on `storage_class = system`) and the presence of expected separate filesystems, preventing a single saturating volume from taking down the operating system. |
| B-09 | Scalability | Support smooth fleet growth by operating a metadata-only, automation-collected pipeline that scales to 500+ heterogeneous servers, where run-level `snapshot_id` and per-host `scan_timestamp` enable onboarding of new hosts and new standards without re-architecture. |
| B-10 | Cost Saving | Optimize allocated-versus-used capacity by surfacing over-provisioned storage groups (`size_free_bytes` vs `size_total_bytes`) and under-utilized volumes, enabling reallocation of free capacity instead of provisioning new storage. |
| B-11 | Risk Reduction | Reduce security and integrity risk by reporting world-writable directory counts and non-compliant mount options (for example missing `nodev`/`nosuid` on data mounts) and the non-sensitive Windows ACL summary, tightening the attack surface on system partitions in line with CIS baselines. |
| B-12 | Operational Efficiency | Increase analyst trust and throughput by automatically performing confidence-based triage: high-confidence deviations are auto-confirmed, while low-confidence or high-impact findings are automatically withheld from auto-confirmation and surfaced distinctly, so analysts can focus their attention where it adds value. |
| B-13 | Capacity Planning | Provide management visibility through dashboards and structured data aggregating capacity, deviation, and forecast KPIs across accounts (`gsma_code`) and environments (`host_role_env`, e.g. prod/test), enabling data-driven storage governance decisions. |

## Details

### B-01: Prevent full-filesystem outages

- **Business value:** an outage caused by a 100%-full filesystem (for example `/` or `/var` on Unix, the C: drive on Windows) is one of the most common and costly storage incidents on every platform; catching it before it happens avoids downtime and emergency interventions. Both byte exhaustion and inode exhaustion are covered, since a filesystem can fail with free space but no free inodes.
- **Mapped objectives:** O3 (reliability and full-filesystem prevention), O5 (risk reduction).
- **KPI:** number of filesystems crossing 90% / 100% usage per month; count of saturation-related incidents avoided (predicted and remediated before the threshold was breached).

### B-02: Storage growth forecasting

- **Business value:** turns storage capacity from a reactive cost into a planned budget line, avoiding rushed purchases at premium prices and avoiding overcapacity. Forecasting is only possible because collection is an append-only UTC time series.
- **Mapped objectives:** O4 (operational efficiency and forecasting), O3 (reliability through proactive capacity).
- **KPI:** forecast accuracy (predicted vs actual days-to-full, error %); lead time between a full-prediction and remediation; share of capacity additions that were planned versus emergency.

### B-03: Reclaim useless / obsolete data

- **Business value:** stale and obsolete data consumes paid capacity and lengthens backup windows; identifying it from non-sensitive signals only (file/directory counts, largest-file size, oldest/newest modification times, and per-extension size totals; never filenames or contents) lets teams archive or delete it and defer purchases.
- **Mapped objectives:** O2 (optimization of useless / obsolete / misplaced data), O4 (cost-efficient capacity).
- **KPI:** bytes identified as reclaimable; bytes actually reclaimed per quarter; storage-purchase deferral in currency.

### B-04: Remove business data from system partitions

- **Business value:** business and client data on filesystems whose derived `storage_class` is `system` (illustratively, AIX `rootvg` or other system mounts) is rarely backed up the same way, inflates system filesystems, and can violate data-handling agreements; flagging it reduces data-loss and compliance exposure.
- **Mapped objectives:** O2 (misplaced data), O5 (appropriate data on system partitions), O1 (deviation detection).
- **KPI:** count of hosts with application data detected on `storage_class = system`; total unexpected bytes consumed on system partitions; time-to-remediation.

### B-05: Policy compliance via golden images

- **Business value:** a per-OS reference standard makes compliance objective and auditable instead of relying on prose for a single OS; the resulting deviation reports become defensible audit evidence for clients. Because the reference standards exist for AIX, every Linux distribution, and Windows together, compliance is uniform across the fleet.
- **Mapped objectives:** O1 (deviation detection vs standards), O5 (compliance).
- **KPI:** share of the fleet compliant per golden image; number of open high-severity deviations; audit findings raised versus the prior period.

### B-06: Standardized structures

- **Business value:** consistent filesystem and storage-topology layouts across a mixed AIX, Linux, and Windows estate reduce human error, simplify automation, and make every server predictable to operate.
- **Mapped objectives:** O4 (standardized storage structures), O1 (deviation detection).
- **KPI:** deviation-count trend over time; share of hosts matching the reference layout; variance in layouts per account.

### B-07: Actionable, prioritized remediation

- **Business value:** severity- and confidence-ranked recommendations convert raw findings into a triaged worklist, cutting diagnosis time and ensuring the highest-impact issues are fixed first. Measurement and detection remain deterministic; the LLM layer only explains, prioritizes, and recommends.
- **Mapped objectives:** O4 (operational efficiency, faster incident resolution), O1 (deviation detection).
- **KPI:** mean time to resolution for storage deviations; share of recommendations actioned; backlog of high-severity items.

### B-08: System / data separation and reliability

- **Business value:** ensuring data lives on storage classified as `data` (illustratively a separate `datavg` on AIX, a data volume group on Linux, or a non-system drive on Windows), and that expected mounts are separate filesystems, prevents a runaway application from filling the system storage and crashing the host.
- **Mapped objectives:** O3 (reliability and prevention of failures), O5 (risk reduction).
- **KPI:** share of hosts with correct system/data separation; count of hosts where a single group or volume hosts both system and data.

### B-09: Scalable metadata-only pipeline

- **Business value:** a non-sensitive, automation-driven collection model scales across 500+ heterogeneous servers without per-host bespoke tooling, and run-level `snapshot_id` together with per-host `scan_timestamp` lets new hosts and new standards be added cleanly. The single unified schema is what keeps onboarding cheap as the fleet grows.
- **Mapped objectives:** O3 (reliability at scale), O4 (operational efficiency).
- **KPI:** number of hosts onboarded per release; collection success rate across the fleet; time to add a new OS golden image.

### B-10: Over-provisioning optimization

- **Business value:** large gaps between a storage group's `size_total_bytes` and `size_free_bytes`, or volumes allocated far above their usage, represent paid-for-but-idle capacity that can be reallocated rather than expanded.
- **Mapped objectives:** O2 (optimization), O4 (efficient, forecast-driven capacity).
- **KPI:** aggregate `size_free_bytes` that is reallocatable; number of storage groups flagged as over-provisioned; capacity reallocated versus newly purchased.

### B-11: Security posture of storage configuration

- **Business value:** world-writable directories, weak mount options on system or data partitions, and overly broad permissions widen the attack surface and breach hardening baselines; reporting them (using only the non-sensitive ACL summary on Windows, never named ACEs or SIDs) feeds remediation and audit.
- **Mapped objectives:** O5 (risk reduction and compliance, CIS baselines), O1 (deviation detection).
- **KPI:** world-writable directory-count trend; number of mounts missing required `nodev`/`nosuid`/`noexec` options; ACL-summary findings on Windows.

### B-12: Confidence-based triage

- **Business value:** the system automatically auto-confirms high-confidence deviations and automatically withholds low-confidence or high-impact ones from auto-confirmation, surfacing them distinctly so they stand out in the output. This confidence-based triage is performed entirely by the system, with no approval step, raising trust in the system's output.
- **Mapped objectives:** O1 (separation of low-confidence findings), O4 (operational efficiency).
- **KPI:** share of findings auto-confirmed versus withheld; false-positive rate on auto-confirmed findings; triage turnaround time.

### B-13: Management dashboards and governance

- **Business value:** aggregating capacity, deviation, and forecast metrics by account (`gsma_code`) and environment (`host_role_env`, e.g. prod/test) gives leadership the visibility needed to govern storage decisions and prioritize investment.
- **Mapped objectives:** O4 (forecasting and efficiency), O1 (deviation detection).
- **KPI:** dashboard adoption by account teams; number of governance decisions driven by the data; reduction in unmanaged deviations quarter over quarter.

## Next steps

These goals will be operationalized through artifacts that are out of scope for this document set and are tracked as upcoming work: the per-OS reference standards (golden images), the OS-by-OS field-applicability matrix, the field-by-field rationale and the Ansible-based collection, the LangGraph pipeline architecture, and a machine-validatable JSON Schema file.
