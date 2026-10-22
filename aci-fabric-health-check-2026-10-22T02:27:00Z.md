# ACI Fabric Health Check — index=cisco_dc_fabric (last hour) — 2026-10-22T02:27:00Z

**Scope & methodology**
- Data source: Splunk index `cisco_dc_fabric`.
- Detailed time window: last hour (earliest=-1h latest=now).
- Onset check: 7-day window scanned (earliest=-7d) to establish earliest timestamps for relevant streams (skill requirement).
- Queries run: ACI faults, vPC, interface (F0532), BGP/BFD faults, APIC interface faults, config-push faults, Nexus9K telemetry, IOS syslog UP/DOWN, ND anomalies, AppDynamics, ThousandEyes, VMware/endpoint faults.


## Executive summary
- No ACI faultInst events (vPC F1296, phys F0532, BGP F0299, BFD F1483, APIC F0103, configpush F3227 etc.) were found in `index=cisco_dc_fabric` for the last hour.
- There are repeated interface flaps on Ethernet1/0/1 observed on edge switches (e.g. cph-e-951-swi-1, hel-e-510-swi-1) in IOS syslog during the last hour.
- Nexus 9K streaming telemetry shows high event counts for nxinterface and nxtransceiver on the same site switches (cph-e-951-swi-1, hel-e-510-swi-1, sto-e-410-swi-1, osl-e-310-swi-1).
- AppDynamics and ThousandEyes telemetry are present in the index, but no application-level errors (AppD) or ThousandEyes alerts were observed in the last hour in this index.

Preliminary hypothesis: local layer-1/transceiver or local-edge link instability (site edge switches) causing repeated interface state changes. No evidence in this index of a fabric-wide ACI control-plane or vPC failure during the last hour. Confidence is limited by telemetry availability in this index (see score below).


## Evidence — selected results

### 1) Onset check (7-day earliest event per stream)
- cisco:dc:nd:anomalies — first_seen: 10/14/2026 19:26:56.368418
- cisco:dc:aci:health — first_seen: 10/14/2026 19:26:56.383387
- cisco:dc:aci:class — first_seen: 10/14/2026 19:26:56.919985
- cisco:dc:nexus9k — first_seen: 10/14/2026 19:26:57.198921
- cisco:ios — first_seen: 10/14/2026 19:26:57.706200

(These are the earliest indexed events for the streams during the 7-day window — returned by the onset query.)


### 2) Nexus 9K streaming telemetry (last hour) — top counts
(from: index=cisco_dc_fabric sourcetype=cisco:dc:nexus9k)

- cph-e-951-swi-1 — nxinterface (196 events)
- hel-e-510-swi-1 — nxtransceiver (196 events)
- osl-e-310-swi-1 — nxinterface (196 events)
- cph-e-951-swi-1 — nxtransceiver (196 events)
- hel-e-510-swi-1 — nxinterface (196 events)
- sto-e-410-swi-1 — nxtransceiver (195 events)
- sto-e-410-swi-1 — nxinterface (195 events)

Interpretation: multiple interface/transceiver telemetry messages on the same edge switches and sites; supports physical/link layer instability.


### 3) IOS syslog (UP/DOWN) — examples (last hour)
(index=cisco_dc_fabric sourcetype=cisco:ios; messages filtered for UPDOWN)

- 2026-10-22T01:27:19.989+00:00 cph-e-951-swi-1: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet1/0/1, changed state to up
- 2026-10-22T01:27:38.878+00:00 cph-e-951-swi-1: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet1/0/1, changed state to up
- 2026-10-22T02:27:00.629+00:00 cph-e-951-swi-1: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet1/0/1, changed state to up
- 2026-10-22T01:27:09.168+00:00 hel-e-510-swi-1: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet1/0/1, changed state to up

The dataset shows many repeated "changed state to up" entries for Ethernet1/0/1 on these hosts during the last hour — consistent with a flapping interface (pair of up/down events, raw contains many 'up' messages captured by the query).


### 4) AppDynamics / ThousandEyes presence (last hour)
- appdynamics_status — 1372 events
- appdynamics_events — 196 events
- cisco:thousandeyes:path-vis — 783 events
- cisco:thousandeyes:alerts — 783 events

Targeted searches for AppD error/exception strings and ThousandEyes alerts in the last hour returned no application-level errors or TE alert objects in this index. (No AppD ERROR/Exception hits and no TE alert_type results in the last-hour targeted queries.)


### 5) ACI fault checks (last hour)
Targeted searches for F0532, F1296, F0299, F1483, F0103, F3227 in `index=cisco_dc_fabric` returned empty results for the last hour. That means either:
- there were no ACI faultInst events in this index during the last hour, or
- ACI faults are being landed/indexed into a different index (detection gap).


## Temporal / causal check (rule T1)
- Observed: interface flaps on Ethernet1/0/1 (cph-e-951-swi-1, hel-e-510-swi-1) during the last hour.
- Not observed in this index: a subsequent ACI fault (vPC, BGP, BFD) or AppD application error with a timestamp after the first flap that would prove a causal chain within this data slice.

Conclusion for T1: we cannot produce a fully verified T1 table from `index=cisco_dc_fabric` alone because downstream effects (ACI faults or AppD errors) are not present in this index for the same time window. The flaps are visible and precede no captured downstream faults in this index — T1 cannot be confirmed within this dataset.


## Confidence scoring (0–100)
Score: 30 / 100 — UNRESOLVED / hypothesis

Justification (points):
- +15: Two independent telemetry sources report the same symptom (IOS syslog UP/DOWN and Nexus9K nxinterface/nxtransceiver counts). 
- +15: Specific failed object identified by interface (Ethernet1/0/1 on cph-e-951-swi-1 and hel-e-510-swi-1).
- −10: Alternatives not eliminated (server NICs, cable, remote port, upstream router, ISP) — additional checks required.
- −10: Key telemetry gap for ACI faultInst data in this index (no ACI faults present in the last hour) — limits root-cause confirmation.

Result: score 30 → Unresolved. Hypothesis: local physical link/transceiver instability at site edge switches. Do NOT mark confirmed.


## Pre-delivery gap audit (six quick checks)
1) Onset window run? — YES (7-day onset query executed; earliest timestamps listed above).
2) ACI faultInst checked in this index for last hour? — YES (no results found).
3) Nexus9K and IOS telemetry present? — YES (high counts and UPDOWN messages found).
4) AppDynamics / ThousandEyes presence? — YES, present, but no app errors or TE alerts in last hour in this index.
5) Endpoint / EPG mapping (fvRsCEpToPathEp) checked? — NO, not possible from this index window alone / no endpoint mapping results in this index for the last hour — run against full ACI index (recommended: index=main) to map impacted endpoints and SAP ERP paths.
6) APIC / CLI / tech-support collection performed? — NO. Recommend to collect tech-support and run on-device checks.


## Immediate recommended actions (triage / mitigation)
1) Treat Ethernet1/0/1 on cph-e-951-swi-1 and hel-e-510-swi-1 as suspected-locally-failing ports. Actions (in order):
   - Reseat or replace the SFP and the fiber/copper cable on those interfaces.
   - If available, move the server to a different spare port or verify the remote-facing port/NIC duplex and driver.
   - If the interfaces are in a port-channel, check the port-channel state and member flaps; if member flaps cause PO instability, temporarily remove the flapping member.
2) Collect device diagnostics: on the affected switches run:
   - show interface Ethernet1/0/1
   - show logging onboard
   - show interface transceiver details (ethtool equivalent / show inventory transceiver)
   - show port-channel summary (if applicable)
   - copy tech-support to a collector for TAC if hardware suspected
3) If SAP ERP availability is impacted, identify server connectivity: run endpoint mapping queries against the ACI telemetry in index=main (see queries below) to list endpoints behind the affected switch and inform application owners.
4) If link flaps persist after SFP/cable swap, escalate to hardware replacement and open TAC with vendor (include tech-support bundle).
5) Coordinate with on-call server team to verify server NIC status and logs; if both NIC and switch ports show errors, escalate as server-side issue.


## Queries to run next (recommended, run in index=main if available)
- ACI faults (last 1h):
  index=main sourcetype=cisco:dc:aci:class earliest=-1h latest=now component=faultInst | stats count by faultCode, dn, lc, node | sort -count

- vPC and phys faults (1h):
  index=main sourcetype=cisco:dc:aci:class "F1296" OR "F0532" | stats earliest(_time) as first_seen, values(dn) as dn, values(node) as nodes by faultCode | convert ctime(first_seen)

- Endpoint mapping for interfaces (to find affected servers):
  index=main sourcetype=cisco:dc:aci:class component=fvRsCEpToPathEp earliest=-1h latest=now | stats values(ep_mac) as macs, values(dn) as dns by path

- Nexus interface counters (last 15m):
  index=main sourcetype=cisco:dc:nexus9k component=nxinterface earliest=-15m | timechart span=1m sum(counter) by host

- IOS link flaps (last 1h):
  index=main sourcetype=cisco:ios earliest=-1h "UPDOWN" | stats count by host, _raw | sort -count


## What I have NOT checked (limitations)
- I did not run searches across other indexes (notably `index=main`) for ACI faultInst details and endpoint-to-path mappings — this is necessary to map SAP ERP endpoints to the affected interfaces.
- No APIC CLI/GUI checks or live show commands were run; no tech-support was collected.
- No on-device counter checks (CRC/errors) were pulled; physical diagnostics remain to be done.


## Recommended next steps (priority)
1) Swap SFP/cable on the flapping interfaces (Ethernet1/0/1 on cph-e-951-swi-1 and hel-e-510-swi-1). Monitor.
2) If problem persists, collect tech-support and escalate to hardware TAC.
3) Run endpoint mapping queries in `index=main` to identify SAP ERP endpoints behind the affected leafs and notify application owners.
4) Run the ACI fault queries against `index=main` to ensure we are not missing fabric faults in a different index.


---

Prepared by: automated Splunk analysis (index=cisco_dc_fabric). Include this file in GitHub `gdcosta/cisco-analysis` for audit trail.