# Cisco DC Fabric Health Check Report
**Date:** 2023-10-20
**Time Window:** Last 4 hours (2023-10-20T?? to now)

## Summary
- Critical internal fabric issues identified affecting the SAP_ERP_EMEA application.
- No external ISP or ThousandEyes alerts detected.
- Confidence score: **90 (Confirmed)**.

## Findings

### 1. ACI Fabric Health
| Component | Current Score | Previous Score | Max Severity |
|-----------|---------------|----------------|--------------|
| Fabric Health Total | 71 | 93 | major |
| Tenant `ERP_EMEA` Health | 55 | 90 | major |

The health scores dropped significantly, indicating major issues.

### 2. Fault Events (last 4h)
| Fault Code | Description | DN (sample) | Severity | Lifecycle |
|-----------|-------------|-------------|----------|-----------|
| **F1296** | vPC interface down (peer‑link) | `topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605/fault-F1296` | major | raised |
| **F1394** | Physical interface down (eth1/49) | `topology/pod-1/node-201/sys/phys-[eth1/49]/fault-F1394` | major | raised |
| **F1263** | Port down (minor) | `topology/pod-1/node-204/sys/phys-[eth1/1]/fault-F1263` | minor | retaining |

Multiple occurrences of each fault were observed across leaf nodes 201, 202, 203, 204.

### 3. Application Performance (AppDynamics)
| Transaction | Error % | SLA Status | Avg Response Time (ms) |
|-------------|---------|------------|------------------------|
| `SAP_HANA_DBQuery` | **47%** (max) | **VIOLATED** | 1510 |
| `SAP_ERP_SubmitPurchaseOrder` | ≤3% | NORMAL | 32‑69 |

The high error percentage and SLA violation for `SAP_HANA_DBQuery` correlate with the fabric issues.

### 4. ISP / External Network
- No ThousandEyes alerts or path‑visibility issues detected in the last 4 h.
- No ISP‑related fault codes observed.

## Impact Assessment
- **Affected Sites:** All EMEA sites served by the `eu-spdc` fabric (nodes `cl201-eu-spdc`, `cl202-eu-spdc`, etc.).
- **Services Impacted:** SAP ERP backend transactions, particularly HANA DB queries.
- **Extent:** Major degradation (error > 45 %, SLA breach) for SAP ERP; potential downstream impact on user transactions.

## Confidence Score
| Criterion | Points |
|-----------|--------|
| Rule T1 passes (cause precedes effect) | +25 |
| Failure pattern matches known peer‑link failure | +20 |
| Independent source (AppDynamics) confirms impact | +15 |
| Specific failed objects identified by DN | +15 |
| Alternatives (ISP) eliminated | +15 |
| **Total** | **90** (Confirmed) |

## Recommendations
1. **Immediate:** Investigate vPC peer‑link between leaf nodes 201‑202 (if‑605). Check physical links, transceivers, and LACP status.
2. **Physical Interface:** Verify eth1/49 on node‑201 for hardware or configuration issues.
3. **Application:** Coordinate with AppDynamics team to monitor SLA recovery; consider temporary routing adjustments if possible.
4. **Monitoring:** Enable alerts for vPC and critical interface faults to reduce detection latency.
5. **Post‑mortem:** After remediation, run the health‑check methodology again to verify restoration.

*Report generated automatically using the `splunk-cisco-dc-fabric-health-check` methodology.*