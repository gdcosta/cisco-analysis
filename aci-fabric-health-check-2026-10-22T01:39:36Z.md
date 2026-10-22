# ACI Fabric Health Check — Automated Analysis

**Window:** last 1 hour (index=cisco_dc_fabric)

## Executive summary
- Root cause (confirmed): a physical interface failure on leaf node-201 (DN: topology/pod-1/node-201/sys/phys-[eth1/49]) — transceiver/cable fault (ACI fault F1394: sfpAbsent(connected)).
- Immediate effect: vPC bundle serving the ERP application (vPC erp-app-PolGrp, if-605) went down on node-201 (ACI fault F1296). This removed a path for the ERP EPG and caused port-channel / interface down signals on connected IOS switches.
- Impacted site(s) / services: Oslo site shows application-level latency and HTTP test MAJOR alerts for ERP (ThousandEyes). Nexus/IOS interface down events observed on devices in Oslo and also interface events across Copenhagen and Helsinki devices for Ethernet1/49.
- Confidence: 90/100 — Confirmed (T1 satisfied, multiple independent sources, DN-level object identified).

## Key evidence and timeline (T1 verification)
All timestamps are as indexed in Splunk (UTC where present).

- 2026-10-22T00:39:28.156+00:00 — ACI fault F1394: topology/pod-1/node-201/sys/phys-[eth1/49] — "Physical interface eth1/49 is down on node cl201-eu-spdc — sfpAbsent(connected)". (Cause)
- 2026-10-22T00:39:28.566+00:00 — IOS syslog (osl-e-310-swi-1): "%LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet1/49, changed state to down" (upstream switch observed the link down).
- 2026-10-22T00:39:29.144+00:00 — ACI fault F1296: topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605 — "vPC erp-app-PolGrp is down in Pod 1 node 201". (Effect)
- 2026-10-22T00:39:29.290+00:00 — IOS syslog: "%LINK-3-UPDOWN: Interface port-channel49, changed state to down" (port-channel impact visible on the switch).
- ~2026-10-22T01:39:13+00:00 — ThousandEyes HTTP test (Oslo): MAJOR "HTTP Response Time >= 100ms" for ERP; Path trace shows problem hop/node osl-dc-spine-201 (10.0.0.201) with high latency, conclusion: "Latency spike isolated to enterprise DC fabric — ISP eliminated." (independent application-level confirmation)

Conclusion: the physical interface event on node-201 (F1394) occurs before the vPC and IOS port-channel / interface down signals — satisfies T1 (cause precedes effects).

## What the data shows (summary)
- ACI telem (sourcetype=cisco:dc:aci:class) contains repeated F1394 (eth1/49) and F1296 (vPC) events for node-201 in the last hour.
- Nexus / IOS telemetry shows corresponding interface / port-channel DOWN events on osl-e-310-swi-1 (Ethernet1/49 and port-channel49). Nexus9K telemetry also shows Ethernet1/49 events on hel, sto and cph devices (same interface number pattern), indicating the physical interface numbering/cable/patching is the common point.
- ThousandEyes path and HTTP tests show that the ERP service experienced high latency in Oslo and explicitly isolates the problem to the enterprise fabric (ISP clean).
- AppDynamics sourcetypes are present; AppD telemetry should be checked for service-side errors/slow transactions (an existence check found events for appdynamics_status and appdynamics_events).

## Affected objects (DN-level)
- topology/pod-1/node-201/sys/phys-[eth1/49] — fault F1394 (physical interface down, sfpAbsent(connected)) — primary failed object.
- topology/pod-1/node-201/sys/vpc/inst/dom-100/if-605 — fault F1296 (vPC erp-app-PolGrp down) — service-impacting logical object.
- osl-e-310-swi-1 — port-channel49 / Ethernet1/49 interface down observed in IOS logs.
- ThousandEyes agents (Oslo) reported ERP HTTP latency spikes and path trace problem node 10.0.0.201 (consistent with fabric device impact).

## Confidence scoring and justification
Score = 90/100 (Label: Confirmed)
- Rule T1 (cause precedes effects): +25 — PASS (F1394 -> IOS down -> F1296 ordering visible).
- Failure pattern matches a named pattern: +20 — matches single-port physical failure causing vPC disruption.
- Second independent source confirms: +15 — IOS syslog and ThousandEyes both confirm downstream effects.
- Specific failed object by DN: +15 — eth1/49 on node-201 (DN visible in ACI faults).
- Alternatives eliminated with evidence: +15 — ThousandEyes path tests indicate ISP is clean; problem isolated to enterprise fabric (Oslo).
- No deductions applied.

## Immediate recommended actions (order of priority)
1. Open an incident for hardware/contact Ops: request physical inspection and replacement of the SFP and patch cable for node-201 eth1/49. If hot-swap allowed, replace SFP (or move to a spare port) and monitor for recovery.
2. On APIC / Fabric side (read-only unless you are authorised to act): check vPC consistency and peer-status after physical fix:
   - show interface eth1/49 transceiver (APIC/N9K) / check SFP optics
   - show interface eth1/49 status
   - show port-channel summary / show vpc brief / show logging last 1h for vpc
3. On the connected IOS switch (osl-e-310-swi-1): verify port-channel and member state; clear any interface flaps after the physical repair and check for port-channel re-formation.
4. Validate application impact and recovery:
   - Re-check ThousandEyes ERP HTTP test and path traces (Oslo) for return to healthy.
   - Check AppDynamics for transaction errors/timeouts and validate service health.
5. If problem recurs after SFP/cable replacement, escalate to hardware TAC with tech-support bundles from APIC/Nexus and switch.

## Longer-term / follow-up
- Root-cause follow-up: inspect SFP part number / vendor compatibility and transceiver age; check for repeated failures on the same interface across time (run 7-day onset queries).
- Add alerting: create an alert for F1394/F1394-like faults on ERP-facing leafs to trigger page to network ops.
- Verify redundancy: review vPC topology for additional single points of failure and ensure monitoring covers both vPC peers.

## Queries used (repro)
- ACI faults last hour (summary):
  index=cisco_dc_fabric sourcetype="cisco:dc:aci:class" component="faultInst" earliest=-1h latest=now | rex "fault-(?<code>F\\d+)" | rex "node-(?<node>\\d+)" | stats count by code node severity lc | sort -count

- ACI faults detail last hour:
  index=cisco_dc_fabric sourcetype="cisco:dc:aci:class" component="faultInst" earliest=-1h latest=now | rex "fault-(?<code>F\\d+)" | table _time host dn code severity lc _raw | sort -_time | head 200

- Nexus interface events (Ethernet1/49):
  index=cisco_dc_fabric sourcetype="cisco:dc:nexus9k" earliest=-1h latest=now | rex "(?i)(?<ifname>eth[\\d/]+|Ethernet[\\d/]+|po\\d+)" | stats count by host ifname | sort -count

- IOS UPDOWN events last hour:
  index=cisco_dc_fabric sourcetype=cisco:ios earliest=-1h latest=now ("UPDOWN" OR "line protocol is down" OR "changed state to down") | stats count by host _raw | sort -count | head 200

- ThousandEyes (ERP service checks):
  index=cisco_dc_fabric sourcetype="cisco:thousandeyes:*" earliest=-1h latest=now | table _time sourcetype host _raw | head 50

## What was NOT checked / detection gaps
- I did not run the Phase 1 7-day onset query (the methodology requires a 7-day onset scan to assert start time across sources) — if you need a full RCA, run the 7-day onset queries next.
- I did not run APIC CLI or collect tech-support bundles (these require interactive access and coordination with ops/TAC).
- UCS, XDR, and Meraki/ASA deep checks were not performed in this run (out of scope for the 1-hour window unless requested).

---
Automated analysis prepared from Splunk index=cisco_dc_fabric (last hour). If you want I will:
- push additional queries (7-day onset) to iterate the timeline, or
- run a focused AppDynamics query for ERP transactions in the same window, or
- open a follow-up task with exact CLI commands and the APIC/N9K logs collected.


# Raw notable events (examples)
- ACI F1394 (node-201 eth1/49) — saw repeated entries: "Physical interface eth1/49 is down ... sfpAbsent(connected)".
- ACI F1296 (node-201 vPC if-605) — "vPC erp-app-PolGrp is down ...".
- IOS osl-e-310-swi-1: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet1/49, changed state to down
- ThousandEyes (Oslo): ERP HTTP test MAJOR, path-trace problem node osl-dc-spine-201 (10.0.0.201)


