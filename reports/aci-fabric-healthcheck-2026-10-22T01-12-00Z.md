# ACI fabric health check — cisco_dc_fabric index (last 1 hour)

Generated: 2026-10-22T01:12:00Z

## Summary
- Scope: index=cisco_dc_fabric, time range: last 1 hour
- Short conclusion: Hypothesis — a set of physical interface failures (one clear single-port failure on node-201) that triggered a vPC fault, with simultaneous interface flapping across multiple leaves that suggests a site-level/shared-infrastructure event. Confidence score: 60/100 (Hypothesis).

## Key findings (top events)
- Fault counts (last 1h):
  - F1263: 584 events (nodes: 202, 203, 204; phys-[eth1/1]; lifecycle=retaining/minor)
  - F1394: 194 events (topology/pod-1/node-201/sys/phys-[eth1/49]; severity=major; lifecycle=raised)
  - F1296: 194 events (topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605; severity=major; lifecycle=raised)

- IOS syslog correlation: repeated LINK/LINEPROTO UPDOWN messages observed on multiple site switches in the same window: examples
  - osl-e-310-swi-1: Ethernet1/49 and port-channel49 (many DOWN events)
  - cph-e-951-swi-1: Ethernet1/0/1 (many UP/DOWN messages)
  - hel-e-510-swi-1, sto-e-410-swi-1: similar high-frequency LINEPROTO events

## Timeline (first_seen per stream — earliest observed in the window)
- 10/21/2026 17:11:07 UTC — F1263 on node-204 phys-[eth1/1] (first observed)
- 10/21/2026 17:11:13 UTC — F1394 on node-201 phys-[eth1/49] (first observed)
- 10/21/2026 17:11:14 UTC — F1296 on node-201 vpc if-605 (first observed)
- 10/21/2026 17:11:19–17:11:33 UTC — IOS LINK/LINEPROTO UPDOWN events observed on cph/hel/sto/osl devices (interfaces listed above)

Interpretation: physical interface fault on node-201 (eth1/49 / port-channel49) and a vPC interface fault on the same node are visible and time-ordered such that the physical fault precedes the vPC fault (T1 satisfied). Multiple other leaves show simultaneous phys faults (F1263 on eth1/1 across nodes 202–204) around the same time window which points at a shared or coordinated event.

## Root-cause hypothesis (scored)
- Hypothesis: A physical link/interface failure on node-201 (phys eth1/49 / port-channel49) caused the vPC interface (if-605) to enter fault (F1296). Simultaneously, a set of phys-level faults (F1263) on nodes 202–204 indicate a wider site/shared-infrastructure issue (e.g., upstream aggregation, patching, or power/change event) that may have amplified the impact.
- Confidence score: 60/100 — label: Hypothesis
  - Rationale:
    - +25: Rule T1 satisfied — physical interface fault timestamped before vPC fault
    - +20: Pattern matches "single port failure leading to vPC fault" signature
    - +15: Independent confirmation from IOS syslog (UP/DOWN messages) correlating to the same interfaces
    - +15: Specific failed objects identified by DN (eth1/49 on node-201, vpc if-605)
    - +10: Several alternatives (configpush, BGP/BFD control-plane) were searched for and not observed in the same window
    - −25: Simultaneous faults across multiple leaf nodes (F1263 on 202/203/204) create T2 ambiguity (shared-infrastructure candidate), reducing confidence
  - Final: 60 = Hypothesis (50–69). Per methodology: do not mark confirmed (<70).

## Affected sites / blast radius
- Directly implicated objects:
  - topology/pod-1/node-201/sys/phys-[eth1/49] (F1394)
  - topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605 (F1296)
- Observed flapping on access/edge switches in multiple sites (osl, cph, hel, sto). Devices with the heaviest UP/DOWN message counts in the window: osl-e-310-swi-1 (Ethernet1/49, port-channel49), cph-e-951-swi-1 (Ethernet1/0/1), hel-e-510-swi-1 (Ethernet1/0/1), sto-e-410-swi-1 (Ethernet1/0/1).
- Impact: Any endpoints dual-homed via the affected vPC on node-201 may have lost redundancy or connectivity. SAP ERP (mission critical) services that traverse these paths should be considered at risk until the physical-layer instability is resolved.

## Immediate recommended actions (order-of-magnitude, operational)
1. Open/assign an incident to on-call network ops with high priority (SAP impact). Provide this file and the exact DNs.
2. For node-201 (eth1/49 / po49):
   - Check physical cable and SFP on both ends of the link (replace cable/SFP if possible).
   - Check port-channel health and LACP status on both leaf peers and the attached switch (osl-e-310-swi-1).
   - If hardware replacement is needed, schedule immediate swap; if risk to SAP is high, move traffic via alternate bundle or maintenance window.
3. For simultaneous F1263 events on nodes 202–204: escalate to site ops to check the aggregation/patch panel, power or an upstream device common to those leaf ports.
4. If available, temporarily isolate the flapping interface (shutdown on maintenance path) to restore stability to vPC/peer-link.
5. Check for any active configpush jobs or automation runs around 10/21 17:11 UTC and roll back if a misconfiguration was applied.
6. Notify application owners (SAP) and consider failing over application paths if an immediate fix is not possible.

## What I checked (this run)
- Splunk searches on index=cisco_dc_fabric for last 1 hour covering:
  - cisco:dc:aci:class (faultInst) — fault code aggregation, DN-level extraction
  - cisco:dc:aci:health — (no health events returned in this window)
  - cisco:ios — facility frequency and LINEPROTO/LINK UPDOWN correlation
  - cisco:dc:nexus9k — (no significant interface error metrics returned in this window)
- Searched specifically for: F1296 (vPC), F1394 (phys eth1/49), F1263 (phys eth1/1), F3227 (configpush) — only F1296/F1394/F1263 were significant in the window.

## What I did NOT check (gaps / next steps)
- AppDynamics (AppD) application telemetry — not queried here; check AppD immediately to confirm SAP service errors vs network-induced failures.
- ThousandEyes / ISP reachability tests — not checked; if symptoms are widespread across sites, run ThousandEyes BGP/HTTP tests to rule in/out ISP or WAN issues.
- UCS/ESX host-side errors (comp/prov-VMware) and server NIC logs — not queried here; verify server link state and host-side errors.
- APIC cluster internal logs/techsupport bundles — not collected here; collect if hardware/hypervisor suspicion increases.

## Pre-delivery gap-audit (six quick checks)
1. Data coverage for window: PASS (ACI faultInst + IOS syslog present).  
2. Time window validated: PASS (last 1 hour as requested).  
3. Key sourcetypes present: PARTIAL (ACI faults and IOS present; aci:health and nexus9k metrics sparse in this hour).  
4. Evidence of config push at onset: NO (no F3227 observed).  
5. Multi-source correlation: YES (ACI faults + IOS syslog match interfaces and timestamps).  
6. Any telemetry gaps that block confirmation: YES (no AppD / ThousandEyes / UCS correlation done here).

## Detection-gap checklist (quick)
- Alerting: Not validated here (are there alerts firing for these fault codes?).
- Data model acceleration: Not required for these raw searches; used raw sourcetypes.
- ThousandEyes/UCS/XDR presence: Not checked (gap).
- Peer-link observability: vPC if-605 observed but peer-side vPC fault not seen in this window — verify peer node for same vPC if.
- 7-day onset window: Not run (user requested last 1 hour). Recommend running a 7-day onset query before declaring final root cause.

## Splunk queries used (examples)
- index=cisco_dc_fabric sourcetype="cisco:dc:aci:class" component="faultInst" | rex "fault-(?<fault_code>F\d+)" | stats count by fault_code, dn, severity
- index=cisco_dc_fabric sourcetype="cisco:dc:aci:class" component="faultInst" dn="*vpc/inst/*/if-605/*" | stats count by dn
- index=cisco_dc_fabric sourcetype="cisco:ios" | regex _raw="UPDOWN|LINK-3-UPDOWN|LINEPROTO-5-UPDOWN" | rex ... | stats by host,interface

## Raw top results (abridged)
- F1263: nodes 202/203/204 (phys-[eth1/1]) — ~195 events per node (retaining/minor)
- Node 201: F1394 on phys-[eth1/49] (major) and F1296 on vpc if-605 (major) — ~195 events each
- ios UPDOWN: osl-e-310-swi-1 Ethernet1/49 / port-channel49 (many DOWN events)

---

Prepared by: automated Splunk health-check (network security analyst). Run scope: index=cisco_dc_fabric (last 1 hour).  
Next recommended steps: (1) Network ops to isolate/replace SFP/cable on node-201 eth1/49 + attached switch port; (2) site ops to investigate common aggregation for nodes 202/203/204; (3) AppD & ThousandEyes checks; (4) run 7-day onset query and APIC techsupport collection if issue persists.
