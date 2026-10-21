# ACI Fabric Health Check — last 1 hour (index=cisco_dc_fabric)

Timestamp (analysis): 2026-10-21T05:46:50Z

## Executive summary

- Root cause (confirmed, confidence 90/100): Physical interface failure on leaf node cl201-eu-spdc (topology/pod-1/node-201/sys/phys-[eth1/49]) — observed as "Physical interface eth1/49 is down" with operStQual=sfpAbsent — caused the vPC interface (topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605) for `erp-app-PolGrp` to go down (fault F1296).  The remote device at the Oslo site (host: osl-e-310-swi-1) reports Ethernet1/49 and port-channel49 down in the same time window.

- Impact: Devices/endpoints served by the `erp-app-PolGrp` vPC (ERP application EPG/policy group) that rely on the leaf pair including node-201 have lost the interface on node-201; redundancy is reduced and workloads attached to that physical link (Oslo site) likely have partial or full loss of connectivity. Given the vPC group name (erp-app-PolGrp) this likely affects SAP ERP-related workloads.

## Key evidence (from cisco_dc_fabric, last 1 hour)

- ACI fault events (sourcetype=cisco:dc:aci:class):
  - topology/pod-1/node-201/sys/phys-[eth1/49]/fault-F1394 — count=195, severity=major, lc=raised
    - sample: "Physical interface eth1/49 is down on node cl201-eu-spdc — sfpAbsent(connected)"
    - first seen in window: 2026-10-21T04:47:10.645+00:00 (multiple repeated events)
  - topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605/fault-F1296 — count=195, severity=major, lc=raised
    - sample: "vPC erp-app-PolGrp is down in Pod 1 node 201"
    - first seen in window: 2026-10-21T04:47:11.518+00:00
  - Temporal relation: physical interface down (F1394) observed ~1s before vPC fault (F1296) — satisfies rule T1 (cause precedes effect).

- IOS / device logs (sourcetype=cisco:ios / cisco:dc:nexus9k):
  - Host osl-e-310-swi-1 appears in nexus events and IOS UPDOWN logs with Ethernet1/49 and port-channel49 in state "down" (count repeated, 195 events in window). This confirms the down state on the remote switch facing the leaf port.

- ACI text in fault descriptions explicitly references `erp-app-PolGrp` (vPC group) which ties the vPC to the ERP application policy group.

## Sites and scope

- Confirmed affected site: Oslo (device osl-e-310-swi-1) — interface Ether1/49 and port-channel49 are down.
- Confirmed affected leaf: node-201 (cl201-eu-spdc).
- Affected service/policy-group: erp-app-PolGrp (vPC if-605) — likely SAP ERP application EPG.
- Extent: All endpoints relying on that specific port-channel/vPC path (Oslo-hosted or connected equipment) lose connectivity or redundancy. Cannot fully enumerate endpoint list within the 1-hour window without endpoint registration queries (not found in this timeframe).

## Confidence scoring (skill matrix)

- Rule T1 passes (physical port down timestamp precedes vPC fault): +25
- Failure pattern matches Single-Port failure -> vPC cascade (physical -> vPC): +20
- Second independent source confirms (IOS/Nexus and ACI fault): +15
- Specific failed object identified by DN (topology/pod-1/node-201/sys/phys-[eth1/49]): +15
- Alternatives eliminated with evidence (no configpush/BGP/BFD/APIC controller faults in last 1h): +15

Total = 90/100 → Label: Confirmed

Justification: timestamps, multi-source confirmation (ACI + IOS + Nexus streaming), explicit DN and human-readable fault descriptions provide high confidence. No competing infra-wide event (configpush, APIC, BGP or BFD faults) was observed in the same window.

## Pre-delivery gap audit (six quick checks)

1) ACI fault telemetry available? Yes — cisco:dc:aci:class faults present and parsed.
2) Nexus 9K telemetry available? Yes — sourcetype=cisco:dc:nexus9k shows host osl-e-310-swi-1 activity.
3) IOS syslog available? Yes — cisco:ios shows interface UP/DOWN for osl-e-310-swi-1 (Ethernet1/49 and Po49 down).
4) Broad config push / controller change in window? No evidence of configpush (F3227) in last 1 hour.
5) App performance (AppD) checked? Not in this index (not checked — external to cisco_dc_fabric). Recommend checking AppD for application-side errors.
6) 7-day onset analysis available? No — user requested last 1 hour only. The methodology recommends a 7-day onset run to establish longer term context; not performed here.

## What has NOT been checked (limitations)

- Application-layer telemetry (AppDynamics) — not present in cisco_dc_fabric index.
- ISP/peering / upstream router logs outside cisco_dc_fabric — not checked here.
- Full endpoint membership / EPG-to-endpoint mapping (fvCEp, fvRsCEpToPathEp) did not return results within the 1-hour window — further queries over a slightly larger window or live APIC queries recommended.
- UCS, ThousandEyes, and XDR telemetry — not checked (may be in other indexes).

## Immediate recommended next steps (execute now)

1) Open an incident for physical link failure and page on-call network engineer; mark as high priority (SAP ERP impact).
2) Ask Oslo NOC/operator to:
   - Verify SFP/cable at the Oslo switch side (osl-e-310-swi-1) on Ethernet1/49 and Po49; reseat or replace the SFP and/or fiber patch immediately.
   - If available, move the fiber to a known-good transceiver and test.
3) Remediation on leaf node-201 side (co-ordinate with APIC/NOC):
   - Check SFP/optical module status on node-201 (cl201-eu-spdc) and reseat/replace if required.
   - Commands to run (on the relevant devices):
     - On APIC / CLI: show interface and fault details: "show fault \"topology/pod-1/node-201/sys/phys-[eth1/49]\"" (APIC UI or API)
     - On Nexus leaf (node-201): show interface transceiver and status (via NX-OS CLI or Nexus API): "show interface ethernet1/49 transceiver detail"; "show interface ethernet1/49"; "show port-channel brief"; "show vpc brief"; "show vpc consistency-parameters".
     - On Oslo switch (osl-e-310-swi-1): "show interface Ethernet1/49 status"; "show port-channel summary"; inspect logs for SFP errors.
4) If physical replacement is not immediately possible, consider rerouting traffic or enabling alternate paths (if present) to restore ERP connectivity while physical fix occurs.
5) After fix, validate: APIC and Nexus faults clear (lc changes to cleared), vPC recovers (F1296 clears), and application connectivity verified (AppD or SAP application test).
6) Document actions taken and times in the incident ticket.

## Splunk queries used (run against index=cisco_dc_fabric, last 1 hour)

- ACI faults summary:
  index=cisco_dc_fabric sourcetype=cisco:dc:aci:class "faultInst" earliest=-1h latest=now
  | rex field=_raw "\\\"dn\\\":\\\"(?<dn>[^\\\"]+)\\\"" max_match=0
  | rex field=_raw "fault-(?<code>F\\d{3,4})" max_match=0
  | rex field=_raw "\\\"severity\\\":\\\"(?<severity>[^\\\"]+)\\\"" max_match=0
  | rex field=_raw "\\\"lc\\\":\\\"(?<lc>[^\\\"]+)\\\"" max_match=0
  | stats count earliest(_time) as first_seen latest(_time) as last_seen values(severity) as severity values(lc) as lc by code,dn

- Major/critical faults:
  index=cisco_dc_fabric sourcetype=cisco:dc:aci:class "faultInst" earliest=-1h latest=now (("severity\\\":\\\"critical\\\") OR ("severity\\\":\\\"major\\\") OR "Critical" OR "Major")
  | rex ...
  | stats ... by code,dn,lc

- vPC F1296 check:
  index=cisco_dc_fabric sourcetype=cisco:dc:aci:class "fault-F1296" earliest=-1h latest=now
  | rex field=_raw "\\\"dn\\\":\\\"(?<dn>[^\\\"]+)\\\"" max_match=0
  | rex field=dn "node-(?<node>\\d+)"
  | rex field=dn "vpc/inst/[^/]+/if-(?<vpc_if>[^/]+)"
  | stats count earliest(_time) as first_seen latest(_time) as last_seen values(node) as nodes by vpc_if
  | where mvcount(nodes) > 1

- BGP / BFD / APIC controller faults (quick checks): searched for F0299, F1483, F0103 in last 1h — no results.

- Interface / IOS UP/DOWN (evidence from remote switch):
  index=cisco_dc_fabric sourcetype=cisco:ios earliest=-1h latest=now ("LINK-3-UPDOWN" OR "changed state to" OR "line protocol has changed" OR "UPDOWN")
  | rex field=_raw "Interface (?<intf>[A-Za-z0-9\\/\\.-]+).* changed state to (?<state>\\w+)" max_match=0
  | stats count by host,intf,state

- Interface mapping search (eth1/49 / po49 / if-605):
  index=cisco_dc_fabric sourcetype=cisco:dc:aci:class earliest=-1h latest=now ("eth1/49" OR "po49" OR "if-605")
  | rex field=_raw "\\\"dn\\\":\\\"(?<dn>[^\\\"]+)\\\"" max_match=0
  | stats count earliest(_time) as first_seen latest(_time) as last_seen values(_raw) as samples by dn


## Immediate follow-ups I will perform if you want me to (pick one or more):
- Run endpoint-to-path discovery for node-201/eth1/49 and list EPG/tenant membership (requires a slightly larger time window or APIC queries).
- Run a 7-day onset query (recommended by methodology) to find when this problem first appeared.
- Check related indexes (AppD, ThousandEyes, UCS, xdr) to enumerate application and service impact.

---

(Report auto-uploaded to repository 'gdcosta/cisco-analysis' under reports/aci-fabric-analysis-2026-10-21T05-46-50.md)
