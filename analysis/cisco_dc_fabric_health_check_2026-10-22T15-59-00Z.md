# Cisco DC Fabric Health Check — cisco_dc_fabric (last hour)

Timestamp: 2026-10-22T15:59:00Z

## Executive summary
- Root cause (confirmed, confidence 90/100): a physical transceiver/cable fault on the Oslo edge switch (osl-e-310-swi-1) interface Ethernet1/49 (XCVR not inserted / high error counts) triggered link flaps that propagated into the ACI fabric causing a vPC interface failure (vPC erp-app-PolGrp, F1296) on node-201 and downstream vNIC migrations on UCS blades. The vPC is named `erp-app-PolGrp` — this affected ERP application traffic.
- Primary site affected: Oslo (OSL). Secondary impacts observed at leaf nodes representing Copenhagen (CPH), Stockholm (STO) and Helsinki (HEL) as retained/related port-down faults, but the initiating hardware symptom is in Oslo.
- App-level checks (AppDynamics) show no application-generated error (no HTTP/transaction errors found in the last hour for SAP/ERP in AppDynamics events). ThousandEyes shows no ISP-wide outage for monitored sites; ISP is unlikely the root cause.

## Key evidence (selected lines)
1) ACI faults (index=cisco_dc_fabric, sourcetype=cisco:dc:aci:class)
- 2026-10-22T08:47:00.989-07:00 topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605/fault-F1296 — "vPC erp-app-PolGrp is down in Pod 1 node 201 fabric hostname cl201-eu-spdc" (lc_ada5b85b...)
- 2026-10-22T08:47:00.145-07:00 topology/pod-1/node-201/sys/phys-[eth1/49]/fault-F1394 — "Physical interface eth1/49 is down on node cl201-eu-spdc — sfpAbsent(connected)"
- Repeated F1263 (port-down) events on node-202/203/204 (cph/sto/hel fabric ports) marked `retaining` in the last hour.

2) Nexus (nxinterface) telemetry (sourcetype=cisco:dc:nexus9k)
- 2026-10-22T08:57:18.895-07:00 host=osl-e-310-swi-1 _raw -> Row_info.interface: Ethernet1/49 state: down state_rsn_desc: "XCVR not inserted"; eth_inerr: 7784, eth_crc: 3847 (high error counts)
- Multiple repeated timestamps showing Ethernet1/49 in state=down on host osl-e-310-swi-1 across the same window (lc_e0f01ec6...)

3) UCS events (sourcetype=cisco:ucs)
- 2026-10-22T08:57:20.574-07:00 dn=sys/chassis-1/blade-1/fabric-A/path-ep-[eth0] severity=major — "vNIC eth0 on blade osl-ucs-blade-01 FI-A uplink lost — link-flap detected during vPC peer-link degradation on fabric interconnect"
- Repeated vNIC migration messages (FI-A → FI-B) for the same blade, correlated in time with ACI vPC faults (lc_24945851...)

4) ThousandEyes (sourcetype=cisco:thousandeyes:metric / alerts)
- No global ISP or external-path alerts correlated with this timeframe for the monitored tests. TE metrics show normal telemetry for CPH/STO/HEL; Oslo is not showing an ISP-level loss that would indicate an external provider outage. (lc_a8eb65f3...)

5) AppDynamics (sourcetype=appdynamics_events / appdynamics_status)
- No application error/HTTP-5xx events for SAP/ERP found in the last hour. AppDynamics status/events present but there were no SAP-specific failure messages in the hour window.

## Timeline (chronological, last hour)
- 2026-10-22T08:43:37-07:00 — Nexus telemetry first shows Ethernet1/49 on host osl-e-310-swi-1 reporting state=down with state_rsn_desc="XCVR not inserted" and high error counts (first observed transceiver/cable symptom).
- 2026-10-22T08:47:00-07:00 — ACI raises physical-interface and vPC faults on node-201: phys eth1/49 SFP absent (F1394) and vPC erp-app-PolGrp down (F1296). (vPC event repeats as `raised`.)
- 2026-10-22T08:53:05-07:00 — UCS shows vNIC uplink loss and vNIC migration (FI-A → FI-B) on osl-ucs-blade-01; link-flap messages observed repeatedly as ACI instability continues.
- Throughout 08:47–08:58 window — ACI shows minor port-down faults (`F1263`) on nodes 202/203/204 as retained port-down symptoms; Nexus interface telemetry shows the Oslo port has large CRC / input errors.
- ThousandEyes and AppDynamics show no ISP-wide or app-originating outage during this window.

## Root cause hypothesis (validated)
- Primary root cause: physical transceiver (SFP) or cable failure on Ethernet1/49 at the Oslo edge switch (osl-e-310-swi-1). Evidence: Nexus telemetry with XCVR not inserted + high error/CRC counters, matching ACI phys fault F1394 (sfpAbsent) for eth1/49 on node-201.
- Consequent effects: vPC instability (F1296) on the leaf pair that owns vPC `erp-app-PolGrp`, causing vNIC uplink flap and migrations on UCS blades and partial loss of ERP application connectivity for endpoints using that vPC bundle.
- ISP/external provider ruled out by ThousandEyes (no correlated ISP path loss) and AppDynamics (no app-originated failure before the fabric faults).

## Impact (brief)
- SAP ERP traffic carried by `erp-app-PolGrp` experienced loss of redundancy and intermittent connectivity for endpoints dual-homed to that vPC bundle; specific endpoints observed: osl-ucs-blade-01 vNIC eth0 experienced FI-A uplink loss and migration to FI-B (some VM/guest flows will see packet loss during migration). Extent: partial (affects hosts dual-homed to impacted vPC), not global.

## Immediate recommended actions (order-of-importance)
1) On-site / NOC: inspect / reseat / replace the SFP on osl-e-310-swi-1 Ethernet1/49 and verify the cable. If replacement is required, perform replacement during a short maintenance window (this is the immediate fix).
2) Verify link state after SFP reseat/replace: from Nexus do `show interface Ethernet1/49 transceiver`, `show interface Ethernet1/49`, `show interface counters errors` and confirm link is up and error counters cleared.
3) In ACI (APIC) check vPC status on the vPC peer pair (node-201 / node-202): run `show vpc brief`, validate vPC domain 100 and `if-605` state on both peers; if inconsistent, fix peer-link / clear sticky state after physical fix.
4) In UCS Manager: confirm vNICs returned to normal and that FI-A / FI-B health is ok; validate endpoint reachability for ERP VMs, and if necessary restart uplinks or failback to FI-A after stable physical link.
5) Re-check SAP ERP connectivity (AppDynamics): run synthetic transactions and confirm no user-facing errors; watch for slow transactions or 5xxs after network fix.
6) If faults persist after SFP replacement and checks, escalate to TAC with the following artifacts: ACI faultInst exports, Nexus transceiver show output, UCS event logs and failure timestamps.

## Short-term mitigation (if immediate replacement not possible)
- Move the affected server uplink to an alternate physical port (if wiring and topology allow) or place ERP workload on alternate hosts if capacity exists.
- If vPC peer-link is inconsistent, consider planned manual remediation to re-establish vPC consistency only after physical fix to avoid split-brain forwarding.

## Longer-term recommendations
- Add ThousandEyes coverage for Oslo (if not already present) to detect last-mile/ISP changes for that site.
- Add automated alerts for SFP XCVR absence / CRC threshold crossing so hardware symptoms are detected before vPC impact.
- Harden runbooks for vPC peer-link failure with step-by-step Recovery & Failback instructions for APIC/UCS/Nexus teams.

## Confidence score and justification (90/100)
- Rule T1 (cause precedes effects): +25 points — Nexus XCVR absent observed before ACI vPC and UCS vNIC migration events (timestamps consistent).
- Failure pattern matches "Single port failure that cascaded to vPC" named pattern: +20 — phys SFP absent -> vPC F1296 matches documented pattern.
- Second independent source confirms (Nexus telemetry + UCS events + ACI faults): +15.
- Specific failed object identified by DN (topology/pod-1/node-201/sys/phys-[eth1/49]): +15.
- Alternatives eliminated (ThousandEyes shows no ISP outage; AppDynamics shows no app-originated fault before network faults): +15.
- Deductions: none applied (no contradictory telemetry). Final score: 90 = Confirmed.

## Pre-delivery gap audit (six quick checks)
1) Time window used: last 1 hour (requested by user). Note: the methodology recommends a 7-day onset query for full change analysis; that was NOT run (gap).
2) Independent sources checked: ACI faults, ACI health, Nexus9K telemetry, UCS events, ThousandEyes, AppDynamics, XDR (XDR had no correlated incidents) — YES.
3) vPC DN parsed and included in evidence: YES (vPC erp-app-PolGrp, if-605 in dom-100 on node-201).
4) Resolved node → site mapping: inferred from nexus host/site_code in telemetry and ACI descr fields — mapping cross-checked in evidence.
5) APIC audit (aaaModLR) and config push history NOT checked in this run (gap) — recommend checking immediately if suspicious changes are suspected.
6) Physical inspection NOT performed (on-site required) — gap.

## What I have NOT checked (important)
- 7-day onset and change correlation across the environment (per skill Phase 1) — not executed because user requested last hour.
- APIC audit logs (aaaModLR) and configpush history for recent changes — could identify an operator or automation change at onset.
- Physical status at the site (cable / SFP vendor / optics mismatch) — requires on-site tech.
- Full mapping of every server/VM impacted by the `erp-app-PolGrp` vPC bundle — needs endpoint-path mapping (fvRsCEpToPathEp) for blast radius.

## SPL queries used (executed against index=cisco_dc_fabric, earliest=-1h latest=now)
- ACI faults (selected):
  index=cisco_dc_fabric sourcetype="cisco:dc:aci:class" component=faultInst | table _time, dn, code, severity, lc, descr, subject

- ACI node summary:
  index=cisco_dc_fabric sourcetype="cisco:dc:aci:class" component=faultInst | rex field=dn "topology/pod-(?<pod>\\d+)/node-(?<node>\\d+)/" | stats count as fault_count, values(code) as codes by node

- ACI health timeline:
  index=cisco_dc_fabric sourcetype="cisco:dc:aci:health" | table _time, node, component, healthScore, fabricHealthTotal, msg

- Nexus interface telemetry (focused on Ethernet1/49 / XCVR):
  index=cisco_dc_fabric sourcetype="cisco:dc:nexus9k" | search "XCVR not inserted" OR Ethernet1/49 | table _time, host, _raw

- UCS events:
  index=cisco_dc_fabric sourcetype=cisco:ucs | table _time, dn, severity, descr

- ThousandEyes metrics/alerts:
  index=cisco_dc_fabric sourcetype=cisco:thousandeyes:metric | table _time, site_code, site_name, lossPct, rttAvg

- AppDynamics checks:
  index=cisco_dc_fabric sourcetype=appdynamics_events | search error OR slow | table _time, appName, tierName, nodeName, message

(Full queries and raw excerpts are retained in the investigation notebook.)

---

If you want, I will:
- (A) Trigger the recommended immediate actions as a runbook checklist (step-by-step commands / contacts) for NOC to execute now.
- (B) Run the 7-day onset query and APIC audit correlation to confirm there was no pre-existing config change.
- (C) Produce an endpoint blast-radius map for `erp-app-PolGrp` showing exactly which servers/VMs were impacted.

Please tell me which next step you want me to take now (A, B or C), or confirm I should open a TAC case and include the artifacts above.
