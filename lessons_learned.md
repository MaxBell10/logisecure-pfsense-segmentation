# Lessons Learned — P2 pfSense Network Segmentation

## Overview

This document reflects my honest experience building P2 of the LogiSecure SA portfolio. Where P1 was about identity and detection (AD, SIEM), P2 is about enforcement — making the network itself a security boundary. That shift from "monitoring what happens" to "preventing it from happening" introduced a completely different category of challenges: firewall rule semantics, DNS architecture, and the surprising complexity of making a simple homelab actually route traffic correctly.

This lab was built on VirtualBox 7.x with pfSense CE 2.7.2, running a LAN/DMZ/WAN segmented network on top of the P1 Active Directory infrastructure.

---

## Main Challenges

### pfSense update 2.7.2 → 2.8.1 — impossible from VirtualBox NAT

The first thing I tried after installing pfSense was updating to 2.8.1 (the latest stable version). Both the WebGUI update path and the console option 13 (Update from console) failed with the same errors:

```
ERROR: It was not possible to determine pkg remote version
pkg-static: An error occurred while fetching package
Unable to update repository pfSense-core
```

The root cause: pfSense running behind VirtualBox NAT cannot reach Netgate's package repositories. The WAN interface (`10.0.2.15`) can ICMP to `8.8.8.8`, but the pkg infrastructure uses HTTPS to specific Netgate CDN endpoints that appear to be blocked or unreachable through the VirtualBox NAT layer.

**Fix applied:** None found. Stayed on 2.7.2 and documented it.

**Production difference:** In a real environment, firmware updates for network appliances are planned events with a dedicated maintenance window, a tested rollback procedure, and guaranteed connectivity to vendor update repositories (either direct internet or a local mirror). "Unable to update" on a production firewall would be a severity-1 issue, not a footnote. The lesson here is that the update path needs to be validated *before* you need it.

---

### Firewall rule order — first-match semantics, and how I broke it

The most important technical lesson of P2: **pfSense evaluates firewall rules top-to-bottom, and stops at the first match.** Getting the order wrong doesn't just make a rule "weaker" — it silently makes another rule completely unreachable.

The mistake that happened: the "LAN to WAN" rule (destination: `*`, ports: 80/443) was positioned *above* the "LAN to DMZ" rule (destination: `DMZ subnets`, port: 443). Because `*` matches everything including the DMZ subnet, LAN-to-DMZ traffic was being handled by the WAN rule before the DMZ rule was ever evaluated. The DMZ rule existed but was unreachable — a silent security gap.

A second issue: during drag-and-drop reordering, the ⚓ (anchor) icon in pfSense opens an "Add rule below" form — not an edit form. I clicked it expecting to edit, filled in different values, and saved, which created a duplicate rule while leaving the original in place (and sometimes appearing to replace it). The ✏️ pencil and the ⚓ anchor look visually similar in a small row and are easy to confuse.

**Fix applied:** Delete corrupted rules, recreate them cleanly using the "↓ Add" button (adds to the bottom of the list), then drag to the correct position. Establish a clear rule order before implementing: intra-zone → specific destination → general destination → default deny.

**Lesson:** In any firewall platform — pfSense, iptables, AWS Security Groups, Palo Alto — rule order is as critical as rule content. Before implementing, write the intended order on paper (or in a justification document like `firewall-rules-justification.md`) and verify the order after every change by reading the rules table top-to-bottom and mentally walking through each test traffic flow.

---

### DNS resolution — a five-step failure chain in VirtualBox NAT

This was the most time-consuming challenge of P2. What looked like a simple "configure DNS" step turned out to be a cascade of five interlocked failures. Understanding each failure required understanding the one before it.

**Failure 1 — DNSSEC blocked Unbound.** With Unbound (DNS Resolver) in forwarding mode and DNSSEC enabled, DNS queries returned SERVFAIL. Disabling DNSSEC reduced the timeout from 10+ seconds to an immediate failure — progress, but still not working.

**Failure 2 — VirtualBox NAT blocks outbound UDP 53 to external servers.** pfSense could ICMP to `8.8.8.8` (confirmed via Diagnostics → Ping, 3/3 packets received, ~67ms). But DNS queries to `8.8.8.8` on UDP port 53 returned no response. VirtualBox NAT passes ICMP but restricts outbound UDP 53 to anything except its own virtual DNS at `10.0.2.3`. This is not documented prominently in VirtualBox's NAT documentation.

**Failure 3 — Unbound cannot forward through VirtualBox NAT.** Even switching to forwarding mode and pointing to `10.0.2.3`, Unbound timed out. The issue appears to be that Unbound's upstream query mechanism doesn't work cleanly through VirtualBox NAT.

**Failure 4 — Switching to dnsmasq fixed pfSense's own resolution.** Disabling the DNS Resolver and enabling the DNS Forwarder (dnsmasq) with `10.0.2.3` as DNS Server 1 resolved the issue for pfSense itself — Diagnostics → DNS Lookup returned `google.com → 192.0.0.88` in 75ms via `10.0.2.3`.

**Failure 5 — DC01's DNS forwarder still pointed nowhere useful.** The browser on DC01 uses DC01's own Windows DNS server (`127.0.0.1`), not pfSense directly. DC01's DNS forwarder for external queries wasn't configured to use pfSense. Fixed with:

```powershell
Set-DnsServerForwarder -IPAddress "10.10.10.1"
```

**Final working chain:**
```
Browser (DC01) → DC01 DNS (127.0.0.1) → pfSense dnsmasq (10.10.10.1, 3ms) → 10.0.2.3 (75ms) → Host DNS
```

**Production difference:** In production, DNS architecture is designed before the network is built, not discovered through troubleshooting. The DNS forwarder chain, the upstream resolvers, and the split-horizon configuration (internal names vs external) are documented in the network design. VirtualBox NAT is a homelab-specific constraint with no production equivalent — in a real pfSense deployment with a proper WAN interface, querying `8.8.8.8` or `1.1.1.1` directly would work without issue.

---

### Interface IP assignment — wrong number, wrong interface

During console setup (option 2 — Set interface(s) IP address), the interface list is numbered starting from 1: 1=WAN, 2=LAN, 3=DMZ. My initial assumption was that LAN might be 1 (since WAN is sometimes listed separately or numbered differently). Entering the wrong number assigned the DMZ IP (`10.10.20.1`) to the LAN interface.

The console showed the change immediately, and the error was visible, but it required going through the configuration sequence twice to correct both interfaces.

**Fix applied:** After any IP assignment, read the full interface summary on the console and verify *all three* addresses before moving to the WebGUI. Never proceed to the next step without confirming the previous one visually.

**Lesson:** Console configuration of network appliances has no undo. Read before you press Enter. After each change, verify the full state — not just the field you just changed.

---

### pfSense WebGUI on port 8443 — not the default

The standard pfSense documentation shows WebGUI access at `https://[LAN IP]` (port 443). This VM had a previous lab configuration that had changed the port to 8443. The first browser connection attempt to `https://10.10.10.1` timed out with `ERR_CONNECTION_TIMED_OUT`.

**Fix applied:** Try `https://10.10.10.1:8443` — it worked. Then changed the port back to 443 via System → Advanced → Admin Access.

**Lesson:** Before assuming a service is broken, verify the port. pfSense keeps its last configuration across reinstalls if the same disk image is reused. Always check *what the previous state was* before diagnosing a problem.

---

### CRITICAL rule not appearing in logs — destination mismatch

After setting up the DMZ rules and testing with Kali pinging DC01 (`10.10.10.10`), the logs showed `Block DMZ to all internal networks` firing — but not `CRITICAL - Block DMZ to LAN`, the rule I specifically wanted to validate.

The reason: the earlier test had been pinging pfSense's own DMZ interface (`10.10.20.1`), not DC01. The address `10.10.20.1` is in `INTERNAL_NETS` (10.0.0.0/8) but is *not* in `LAN subnets` (10.10.10.0/24). So the INTERNAL_NETS rule fired first, while the CRITICAL rule (which matches LAN subnets specifically) was only triggered once the ping target was changed to `10.10.10.10`.

**Fix applied:** Ping DC01 directly (`10.10.10.10`) from Kali. The CRITICAL rule appeared immediately in the logs, one entry per second.

**Lesson:** When validating firewall rules, use traffic that *specifically* matches the intended rule — not traffic that might be caught by a broader rule upstream. The validation is only meaningful if you confirm which rule actually fired.

---

### A gateway IDS is blind to intra-segment traffic

The first Suricata validation attempt was an nmap scan from Kali (`10.10.10.50`) against DC01 (`10.10.10.10`), both sitting on the LAN. The scan itself worked perfectly — 13 open ports, hostname `LOGISECURE-DC01`, a full Active Directory service fingerprint. Suricata LAN produced nothing at all.

The reason is architectural, not a misconfiguration. Both hosts are in `10.10.10.0/24`, so their traffic is switched at layer 2 and never reaches pfSense. Suricata runs on the gateway's interfaces; traffic that does not cross the gateway is invisible to it, no matter how aggressive the scan.

**Fix applied:** Move Kali into the DMZ segment (`10.10.20.0/24`) so that any scan against DC01 has to traverse pfSense. This also makes the test more valuable, since a single scan then exercises segmentation enforcement and detection at the same time.

**Lesson:** A network IDS deployed on the firewall only sees inter-segment traffic. Reconnaissance and lateral movement *inside* a VLAN are a structural blind spot. Closing it requires either a SPAN/mirror port or a network TAP feeding the sensor, or host-based detection on the endpoints — the Wazuh agents on DC01 and WKS01 partially cover that gap in this lab.

**Production difference:** In production, this blind spot is a documented architectural decision rather than an oversight. East-west visibility has to be bought explicitly: mirror ports into the IDS, inline microsegmentation, or EDR/HIDS coverage on every host. An architecture document that states "IDS deployed" without specifying where the sensor sits and what it can actually observe is not an architecture document.

**Evidence:** `screenshots/04_suricata/25_kali_nmap_dc01.png`

---

### Confirm packets reach the interface before auditing the ruleset

After moving Kali to the DMZ, the alerts page was still empty. A long sequence of checks followed: instance running, `em2` correctly assigned, `emerging-scan.rules` enabled in the DMZ categories, hardware offloading disabled with a full pfSense reboot. Every check came back clean and none of them changed the result.

A single command on the pfSense shell moved the investigation down a layer:

```
tcpdump -i em2 -nn host <kali_dmz_address>
```

Zero packets. Whatever was wrong sat below anything Suricata controls — which was correct, and was the first useful result of the session.

It was also incomplete, for a reason worth recording: the address in that filter was the same unverified address the test itself was using. A capture filter is only as good as the value you put in it. Pointing `tcpdump` at an address the interface did not reliably hold produced a true "zero packets" that pointed at the wrong culprit for hours.

**Fix applied:** Adopt a bottom-up diagnostic order and hold to it — (1) do packets reach the interface? (`tcpdump`); (2) does the engine process them? (`eve.json`, `stats.log`); (3) does a rule match? (categories, SID); (4) *can* a rule match at all, given the engine's variable definitions? Before trusting step 1, confirm the values in the filter against the documented topology, not against what was typed earlier in the session.

**Lesson:** Start at the cheapest testable layer, but verify the inputs to that layer first. `tcpdump` eliminates half the hypothesis space in one command — and returns a confidently wrong answer if the filter is built on an assumption nobody checked.

**Production difference:** The same discipline applies with higher stakes. A SOC that escalates "the IDS isn't alerting" without first confirming the sensor is receiving traffic burns analyst time and can mask a real outage — a mirror port disabled during maintenance looks exactly like a quiet network. Sensor health monitoring (packet counters, drop rates, heartbeat) is a standing control, not an ad-hoc check.

---

### The test was wrong — but fixing the test was not enough

The first round of validation concluded that DMZ connectivity was structurally broken under VirtualBox and that the KPI could not be met. That conclusion was wrong, and the evidence contradicting it was already in this repository.

Two preconditions were broken, neither of them checked.

**The address was invented.** Every failed test used `10.10.20.50`, an address introduced mid-session that appears nowhere in the project documentation. Kali's documented DMZ address is `10.10.20.10`.

**The configuration was never persistent.** The address was applied with `ip addr add`, which writes to the running kernel only and sits entirely outside NetworkManager — the service actually managing `eth0` on Kali. NetworkManager held a DHCP profile for that interface, on a segment with no DHCP server, and reclaimed the interface repeatedly. The address disappeared between test cycles; at least two scans were launched from an interface holding no IPv4 address at all. A "network disconnected" notification appeared on the Kali desktop mid-session and was dismissed as noise.

Meanwhile the repository already held the counter-evidence. Screenshot 16 (`screenshots/03_segmentation_tests/16_log_dmz_lan_blocked.png`) shows Kali operating at `10.10.20.10` with traffic crossing pfSense and hitting the block rule.

**Fix applied:** Configure the address *through* NetworkManager rather than around it, using the documented address:

```
nmcli con add type ethernet ifname eth0 con-name dmz-static ip4 10.10.20.10/24 gw4 10.10.20.1
nmcli con up dmz-static
nmcli con show --active
```

With that corrected, `tcpdump` on `em2` captured the scan's SYN packets cleanly and `eve.json` logged the flows. The sensor was receiving traffic and processing it.

It still produced zero alerts. The invalid test had been masking a second, unrelated fault — documented in the next entry.

**Lesson:** A negative result is only evidence if the test was valid. But fixing a broken test does not guarantee the original conclusion was the only thing wrong: two faults can stack, and correcting the obvious one can leave the real one untouched while looking like progress. The harder failure here was allowing a conclusion to stand against contradicting evidence already held in the same repository. Before declaring something impossible, check whether you have already proven it possible.

**Production difference:** Blaming the platform is the most expensive wrong answer available. A finding that reads "control cannot be tested due to environment limitation" closes an investigation, reallocates budget, and sometimes drives a migration. The bar for that claim is the same as for any other finding: reproducible, and reconciled against every piece of evidence already on file. Ephemeral configuration applied by hand is also exactly what configuration management exists to eliminate — Ansible, Puppet or equivalent enforce desired state and make drift visible, where a change that silently disappears at the next reboot is an incident waiting for a maintenance window.

**Evidence:** `screenshots/04_suricata/26_kali_dmz_static_ip_nmcli.png`, `screenshots/04_suricata/27_pfsense_tcpdump_em2_syn_captured.png`

---

### The default `$HOME_NET` makes a gateway IDS blind to lateral movement

With the test finally valid — correct source address, persistent configuration, packets confirmed on `em2`, flows confirmed in `eve.json` — the DMZ instance still produced no alerts. A count across the entire lifetime of the log file returned zero:

```
grep -c alert /var/log/suricata/suricata_<id>_em2/eve.json
→ 0
```

Not zero for this scan. Zero ever. That reframes the problem: not a tuning issue, a structural one.

The cause is a single variable left at its default. On pfSense, `Home Net` defaults to *"local networks, WAN IPs, Gateways, VPNs and VIPs"* — which includes **every** locally attached segment, LAN and DMZ alike.

Nearly every ET Open scan signature is written as:

```
alert tcp $EXTERNAL_NET any -> $HOME_NET any (msg:"ET SCAN ...")
```

A scan from Kali (`10.10.20.10`, in HOME_NET) against DC01 (`10.10.10.10`, also in HOME_NET) is `$HOME_NET → $HOME_NET`. It cannot match. No scan type, no timing template, no target changes that — the rule was never eligible to fire.

This also explains the one alert the lab produced during the entire investigation: an external public IP reaching WKS01 on port 80. `$EXTERNAL_NET → $HOME_NET`. It was the only traffic in the right direction.

**Fix applied:** Redefine `Home Net` for the internal instances so the DMZ is treated as untrusted. pfSense does not accept a firewall alias directly in that field — it requires a Suricata Pass List:

1. `Firewall → Aliases → IP` → create `SURICATA_LAN_ONLY` = `10.10.10.0/24`.
2. `Services → Suricata → Pass Lists` → create `lan_only`, **uncheck all five auto-generated groups** — *Local Networks* in particular re-adds the DMZ and silently undoes the entire fix — then reference the alias.
3. `Services → Suricata → Interfaces → DMZ` → set `Home Net` = `lan_only`, leave `External Net` at default. Repeat for the LAN instance.

Verified in the generated configuration rather than in the UI:

```
# DMZ instance (em2)
HOME_NET: "[10.10.10.0/24, 10.10.20.1/32, 127.0.0.1/32, ::1/128]"
EXTERNAL_NET: "[!$HOME_NET]"
```

The DMZ subnet is gone; only pfSense's own interface address remains, which is expected. The same scan then produced alerts immediately — ET SCAN signatures 2002910 and 2010936, `src_ip: 10.10.20.10` → `dest_ip: 10.10.10.10` on `em2`.

The WAN instance keeps the default deliberately: it is a perimeter sensor, and "everything internal is home, everything else is external" is the correct semantics there. The distinction is the point — the fix is not "override the default everywhere", it is "define trust per sensor according to what that sensor is meant to distrust".

**Outcome:** KPI *"≥ 1 validated Suricata alert"* is **met**.

**Evidence:** `screenshots/04_suricata/28_suricata_dmz_no_alert_before_fix.png` (before), `screenshots/04_suricata/29_suricata_dmz_alert_query.png` and `screenshots/04_suricata/30_suricata_dmz_alert_matched.png` (after)

**Lesson:** The pfSense default is built for a perimeter sensor — outside attacking inside. Applied unchanged to an internal segment, it declares the attacker's own subnet trusted and disables the entire scan-detection ruleset for exactly the traffic the sensor was deployed to watch. The default is not wrong; it is right for one position and silently wrong for every other. Every surface indicator stayed green throughout: instance running, correct interface, `emerging-scan.rules` enabled, packets arriving, flows logged. None of them can reveal this. Only firing real attack traffic and demanding an alert does.

**Production difference:** This is a silent-failure class of misconfiguration, and a common one. An internal IDS on default variables reports healthy indefinitely while providing no east-west detection whatsoever — the worst available outcome, since the organisation believes it has coverage it does not have. Two controls follow: `HOME_NET`/`EXTERNAL_NET` must be defined per sensor according to what that sensor is meant to distrust, and detection coverage must be validated by firing known-signature traffic on a schedule rather than inferred from service health. In audit terms, a sensor whose detection path has never been proven end-to-end is *implemented but not tested*, and reporting it as effective is itself a finding.

---

### Prevention and detection are separate claims, each needing its own proof

Nmap from the DMZ against DC01 returns all scanned ports filtered. The firewall log shows why: `CRITICAL - Block DMZ to LAN` dropping each SYN at the DMZ interface, one log entry per attempt. Segmentation holds, and that is the intended behaviour rather than a fault to fix.

Suricata, running as a passive IDS on that same interface, still saw the traffic and matched ET SCAN signatures. Detection is not downstream of the firewall decision — the sensor inspects what arrives on the interface, whether or not pf subsequently drops it. That independence is the entire reason for placing an IDS behind a firewall instead of relying on the firewall alone.

The two layers were also proven on different artefacts: a firewall log entry for the block, an `eve.json` alert record for the detection. Neither would have revealed a failure in the other. For most of this investigation the firewall was working perfectly while the sensor detected nothing, and nothing in the block logs hinted at it.

**Lesson:** "The firewall blocked it" says nothing about whether the attempt was visible. "The IDS alerted" says nothing about whether the traffic was stopped. Both were demonstrated here on the same flow, with separate artefacts, because neither substitutes for the other.

**Production difference:** This is the difference between a blocked event nobody sees and a blocked event that generates a ticket. Firewall drop logs are volume; IDS alerts are signal. A report that conflates the two overstates coverage.

**Evidence:** `screenshots/04_suricata/30_suricata_dmz_alert_matched.png`, `screenshots/04_suricata/31_pfsense_firewall_block_dmz_lan.png`

---

## Positive Surprises

### Suricata was already installed and running

When checking Status → Services after completing the firewall rules, Suricata appeared in the list as `Running`. It had been installed during a previous lab on this pfSense image.

Worth revisiting with hindsight: this looked like a shortcut, and the obvious risk seemed to be inherited configuration from an unknown previous session. The fault that actually cost the most time was neither — the `$HOME_NET` default is what *any* fresh pfSense install ships with. Inheriting the package was harmless; trusting a default because the service looked configured was not.

### pfSense audit logging is automatic

The console displayed a syslog message during the first WebGUI login: `Successful login for user 'admin' from: 10.10.10.10 (Local Database)`. pfSense logs authentication events automatically, without any custom configuration. In a real deployment, this would be forwarded to a SIEM. In this lab, it connects directly to P1 (Wazuh can receive pfSense syslog — a future integration point).

### The CRITICAL rule worked on the first test

After all the complexity of the rule configuration (wrong orders, corrupted rules, anchor icon confusion), when the actual segmentation test ran — Kali in DMZ pinging DC01 — the CRITICAL rule fired immediately, one entry per second, with the correct source and destination. The firewall was doing exactly what it was designed to do. The gap between the configuration effort and the elegance of the result is worth noting.

---

## What I Would Do Differently in Production

Building on the honesty from P1: I am still in a learning phase. What follows is my current understanding, not production experience.

- **WAN adapter would use bridged or dedicated physical NIC, not NAT.** VirtualBox NAT introduces artificial constraints (blocked UDP 53, unreachable package repos) that don't exist in real deployments. The homelab limitation shaped the DNS architecture more than the security requirements did — that's backwards.

- **DNS architecture would be documented before implementation.** The split-horizon design (internal names → DC01, external names → pfSense → upstream), the forwarder chain, and the fallback behavior would be drawn out before any VM is configured. Discovering the DNS chain through troubleshooting is expensive and error-prone.

- **Firewall rule sets would be version-controlled.** pfSense supports XML config export (`Diagnostics → Backup & Restore`). In production, every config change would be exported, committed to a Git repository with a change description, and reviewed before application — identical to infrastructure-as-code principles.

- **IDS trust boundaries would be defined per sensor, before deployment.** `HOME_NET` and `EXTERNAL_NET` decide which signatures are even eligible to fire. A sensor watching an internal segment needs a different definition from a perimeter sensor, and leaving both on the platform default disables east-west detection while every health indicator stays green.

- **Detection coverage would be validated on a schedule, not assumed.** Atomic tests or purple-team exercises firing known-signature traffic and confirming the alert lands in the SIEM. Service health tells you the daemon is alive; only a fired alert tells you the detection path works end to end.

- **pfSense updates would be tested in a staging environment first.** The inability to update 2.7.2 to 2.8.1 would be a blocker in production. A staging pfSense would receive the update first, connectivity and rule behavior would be validated, then production would follow.

- **DNSSEC would be enabled in production with proper upstream support.** Disabling DNSSEC was the correct fix for this homelab (VirtualBox NAT cannot support it properly), but in production DNSSEC provides an important layer of protection against DNS spoofing. The upstream resolvers would need to be DNSSEC-aware (Cloudflare 1.1.1.1, Google 8.8.8.8, or an internal resolver with DNSSEC validation).

- **Rule logging would be selective and forwarded to the SIEM.** Logging every blocked packet from every interface would generate too much noise. In production, logging would be tuned to capture high-value events (CRITICAL DMZ→LAN attempts, WAN→LAN probes) and forwarded to Wazuh via syslog, creating a unified detection layer across P1 and P2.

---

## Next Steps

- Integrate Suricata `eve.json` alerts into Wazuh for unified SIEM correlation
- Deploy Greenbone Community Edition (OpenVAS) and run the DC01 vulnerability assessment
- Remediate findings and rescan to zero critical vulnerabilities
- Export pfSense config backup as baseline (`Diagnostics → Backup & Restore → XML`)

---

*Part of the [LogiSecure SA Enterprise Security Programme](https://github.com/MaxBell10/logisecure-enterprise-security-program) — a 16-project cybersecurity portfolio.*
