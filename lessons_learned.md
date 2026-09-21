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

### Where you scan from decides what the scan measures

Kali had sat in the DMZ for the whole segmentation and detection phase, and the obvious move was to leave it there and point OpenVAS at DC01. That scan would have produced a result — hosts unreachable, zero findings — and it would have measured nothing about the hosts. `CRITICAL - Block DMZ to LAN` drops the SYNs at `em2`; the report would have restated §3 of the README with a different tool.

A vulnerability assessment is not a segmentation test. It asks *what is wrong with these hosts*, not *can this attacker reach them*, and those two questions need two different vantage points. The assessment belongs in a position of trust — which is also how it is performed in practice, from inside the perimeter rather than from a demilitarised zone.

**Fix applied:** Move Kali to the LAN (`10.10.10.50`) for the scanning phase, and document the repositioning in the README as a methodological decision rather than burying it.

**Lesson:** Before running any scan, state what the result is supposed to prove. "Zero findings" from a blocked position and "zero findings" from a trusted position are the same sentence describing opposite situations.

**Production difference:** Scanner placement is an architectural decision. A single scanner behind a segmented network reports clean subnets it simply cannot reach, and that coverage gap is indistinguishable from a healthy result. Distributed scanners — one per segment, or credentialed agents — exist to remove the ambiguity, and every scan report should record the scanner's network position alongside its findings.

---

### A scan that finds nothing is not a result until the scanner has been checked against a baseline

The first OpenVAS scan finished cleanly: status `Done`, two hosts scanned, no errors reported, 30 minutes of runtime. DC01 — the domain controller, the most exposed asset in this lab — came back with **zero open ports, zero findings, operating system unidentified**.

That is a publishable-looking result, and it was false.

The contradicting evidence was already in this repository. The nmap capture from the Suricata phase (`25_kali_nmap_dc01.png`) documents the same Kali host, on the same segment, finding 13 open ports and a full AD service fingerprint. Re-running `nmap -Pn 10.10.10.10` to settle it took 8 seconds and returned the same 13 ports.

**Cause.** The target used the `All IANA assigned TCP` port list — several thousand ports. Windows hosts drop unsolicited SYNs silently instead of returning RST (nmap reports `987 filtered tcp ports (no-response)` for exactly that reason), so every closed port costs the scanner a full timeout rather than an immediate answer. DC01 spent 25 minutes exhausting timeouts and never reached the ports that were open. Nothing in the interface signals this: the task reports `Done`, and `Error Messages` reports `0 of 0`.

**Fix applied:** Build a port list scoped to the services expected in a Windows domain (`T:53,88,135,139,389,445,464,593,636,3268,3269,3389,5357,5985,5986`) and apply it through a cloned target — GVM locks the port list of any target that already has a report attached. On the rerun DC01 was identified as Windows and returned findings; the report's unfiltered port list grew from 2 entries to 11, and 7 CVEs were actually tested and closed against 0 in the first scan.

**Lesson:** A vulnerability scanner is an instrument, and an instrument that has never been calibrated produces numbers, not measurements. Cross-check any new scanner against a second, independent tool on at least one host whose exposure is already known, before trusting anything it reports. The dangerous failure mode is not the scan that errors out — it is the scan that returns a clean report, because a clean report is what everyone wants to read.

**Production difference:** A false negative in a vulnerability report is worse than no report: it manufactures documented, signed-off confidence in an exposure nobody has looked at. In audit terms, "0 findings" with no stated baseline and no stated scan configuration is not evidence of anything, and a scan whose coverage has never been verified should be reported as *not performed* rather than *passed*. The control that catches this is banal and rarely implemented — a known-vulnerable canary host inside every scan scope, whose findings must appear in the report for that report to be considered valid.

**Evidence:** `screenshots/05_openvas/33_openvas_scan1_hosts.png`, `screenshots/05_openvas/45_nmap_dc01_13_ports.png`, `screenshots/05_openvas/40_openvas_scan2_hosts.png`

---

### A scan marked `Done` can contain tests that never ran

The corrected scan returned three findings and, in a tab nobody opens by default, two error messages:

```
NVT timed out after 1800 seconds — Generic HTTP Directory Traversal / File Inclusion (Web Root) - Active Check — 10.10.10.10
NVT timed out after 600 seconds  — GNU Bash Shellshock (CVE-2014-6271/6278) - Active Check                     — 10.10.10.10
```

Two active checks against DC01 expired without producing a verdict. The task still displays `Done` at 100%, and the finding list says nothing about them. Those 40 minutes of timeout also explain the scan's shape: DC01 alone consumed 54 of the 58 minutes, while WKS01 finished in 5.

**Lesson:** "No finding" and "no result" are different statements that a scan report renders identically. For those two vectors the coverage is null, not negative, and the report cannot be read as evidence that DC01 is free of them. The `Error Messages` tab is the only place that distinction exists, and it is off the default path.

**Production difference:** Scan coverage is itself a metric, and reporting findings without reporting coverage overstates assurance. A mature vulnerability-management process tracks tests attempted against tests completed, treats a rising timeout count as an operational alert, and states exclusions explicitly — the same way a penetration test report states what was out of scope. A finding list with no coverage statement is an opinion.

**Evidence:** `screenshots/05_openvas/44_openvas_scan2_error_messages.png`

---

### An unauthenticated scan measures exposure, not patch level

The valid scan returned 0 Critical, 0 High and 0 CVE against a Windows Server 2022 domain controller and a Windows 10 workstation. Neither machine has been patched since the lab was built.

The two facts are not in contradiction. A black-box scan enumerates what answers on the network: open ports, service banners, protocol behaviour. It does not read the installed-updates list, the registry, or file versions — which is where almost every Windows CVE is actually visible. `0 CVE` means *nothing observable from the network without credentials*, and stretching it into *no vulnerabilities* is the entire distance between a scan and an assessment.

This has a direct consequence on the P2 KPI, originally written as *"critical vulnerabilities on DC01 → 0 after remediation"*. With nothing above Medium in the baseline there is nothing to remediate, so the KPI cannot be met — and ticking it would present a scan blind spot as a security outcome. It has been rewritten in terms the evidence supports rather than claimed.

**Fix applied:** None to the scan. The limitation is stated in §6 of the README, the KPI is restated, and a credentialed scan is listed as the next step rather than as an optional improvement.

**Lesson:** The severity distribution of a scan says as much about the scan configuration as about the targets. An unauthenticated Windows scan returning only informational findings is behaving normally — the correct reaction is to question the method, not to celebrate the result.

**Production difference:** Credentialed scanning is the default in mature vulnerability management, using a dedicated read-only service account with its own rotation and its own monitoring. Unauthenticated scanning keeps a place — it shows what an attacker without a foothold sees — but using it as the primary measure of patch compliance produces a report that trends green while the estate ages.

---

### Standing up the scanner was a project of its own

`apt install openvas` takes a minute. Reaching a state where a scan could actually run took the rest of the session.

`gvm-setup` failed twice before succeeding. PostgreSQL refused to work on databases whose collation version no longer matched the system's after a `full-upgrade` (fixed with `ALTER DATABASE ... REFRESH COLLATION VERSION`), and the `gvmd` database was missing entirely and had to be created by hand. The admin account then had to be created manually with `gvmd --create-user`, and the Feed Import Owner set by UUID before the interface behaved.

The longest single blocker was the feed import: an `UPDATE` on `scap2.cpes` ran for roughly three hours across 1.8 million rows — a known GVM performance characteristic rather than a fault — during which the interface displayed *"Feed is currently syncing. Scans are not available"* and the Port Lists count sat at 0. It resolved when the transaction committed and the schema swapped from `scap2` to `scap`.

One detail cost more time than it should have: a generated password containing `$$` was expanded by bash into the shell's PID, creating an account whose password was not the one on screen. Single quotes around the value fixed it.

**Lesson:** Budget the deployment of a security tool separately from its use, and never plan a scan for the same session as the installation. And any secret passed on a command line goes in single quotes, always — `$`, `!` and backticks are live characters in a shell, and the failure is silent because the command itself succeeds with the wrong value.

**Production difference:** This is the argument for containerised or appliance-based deployment. The Greenbone Community Container ships a pre-synced feed and removes the database bootstrap entirely. In production the scanner is infrastructure with its own maintenance window, monitoring and feed-freshness alerting — a scanner whose feed is three weeks stale reports clean on three weeks of new CVEs.

---

### Received is not understood — an IDS alert filed as "unknown problem"

With the pipeline wired, transport was proven link by link: Suricata writing its alerts to pfSense's syslog, pfSense forwarding them, and a `tcpdump` on the Wazuh host capturing them on arrival as `local1.notice` datagrams — the exact facility and priority configured on the sensor. Wazuh's archive held them too.

The alert log held nothing.

`wazuh-logtest` replays one line through the three stages of Wazuh's pipeline and shows which stage fails. With pfSense in its default BSD syslog format:

- **Pre-decoding** read `suricata[23921]:` as the *hostname*. pfSense omits the hostname when it forwards in BSD format, so the program name was taken for the machine name, and no program name was left.
- **Decoding**: no decoder matched.
- **Rules**: one still fired — generic rule **1002**, *"Unknown problem somewhere in the system"*, level 2, almost certainly on the word *Bad* in the alert category `Potentially Bad Traffic`. Level 2 sits under the alert threshold, so the event was discarded.

An IDS detection, received intact by the SIEM, misread, reclassified as an unknown system problem on a keyword, and dropped under a threshold. No error at any stage.

Switching pfSense to RFC 5424 put the hostname back into the message, but Wazuh's pre-decoder does not parse that format either: nothing extracted, no decoder, no rule at all.

**Fix applied:** keep RFC 5424 — well-formed, hostname included, fixed structure — and write a decoder for it: a prematch on the RFC 5424 header of Suricata's messages, then Wazuh's JSON decoder on the remainder. Three custom rules on top (`100200`–`100202`, README §7). Everything was validated in `wazuh-logtest` before the manager was restarted: a malformed rules file stops the manager, agents included.

**Lesson:** A SIEM that stores an event has not necessarily understood it. Transport and parsing are separate claims, proven by separate tools — `tcpdump` for the first, the platform's own parser test for the second. Neither syslog format pfSense can emit is parsed by Wazuh's stock pre-decoder, so every further pfSense source forwarded this way will need its own decoder.

**Production difference:** In a SOC, onboarding a log source ends with a parser test, not with "logs are arriving". A source that arrives unparsed is worse than a missing one: it inflates the source inventory, feeds the volume counters, and detects nothing. Parser coverage is a metric per source, and its regressions — a vendor changing a log format in an update — are a known cause of silent detection loss.

**Evidence:** `screenshots/06_wazuh_integration/51_wazuh_tcpdump_local1_notice.png`, `53_wazuh_archives_suricata_alerts.png`, `55_wazuh_logtest_bsd_hostname_misparse.png`, `56_wazuh_logtest_rfc5424_no_decoder.png`, `58_wazuh_logtest_rule_100201.png`

---

### A setting is changed when the saved configuration says so — not when the symptom disappears

To cut telemetry volume before connecting the SIEM, every protocol type was unchecked in the DMZ instance's EVE settings, the form was captured with all boxes empty, and the instance was restarted. The change was then declared effective on two observations: eight seconds of log during the engine restart showing no DNS events, and a `grep alert` returning only alerts.

Neither could have shown anything else. No DNS event could appear while the engine was restarting, and a search for `alert` excludes DNS lines by construction.

The next day, DNS events were still flowing into Wazuh, and the settings page showed every box checked. One save had gone through the day before — SYSLOG output, printable payload, no packet dump, all visible in the shape of the alerts — and the second never had. The screenshot had been taken between the clicks and a save that never happened.

**Fix applied:** uncheck again, save, **reopen the page and confirm the state survived the reload**, restart the instance — then prove it on the wire (next entry). The exported pfSense configuration was read afterwards to confirm the persisted values for the DMZ instance: output `syslog`, payload `only-printable`, packet dump `off`, every per-protocol EVE flag off.

**Lesson:** A screenshot of a form is not a configuration, and the disappearance of a symptom is not a verification — least of all when the window observed could not have contained the symptom. A change is proven by reading the persisted state back.

**Production difference:** This is the gap configuration management closes. A change recorded in a ticket and "verified" by the absence of complaints is indistinguishable from a change never applied. Desired-state tooling and config-as-code — here, an exported `config.xml` — turn "I think it's set" into a diff.

**Evidence:** `screenshots/06_wazuh_integration/62_suricata_dmz_eve_logged_unsaved.png`, `46_suricata_dmz_eve_final.png`

---

### Silence proves nothing without a positive control

Two sources of volume had to go before the SIEM was usable. pfSense's *Everything* forwarding sent the firewall log along with Suricata's alerts — one `filterlog` line per blocked SYN, roughly a thousand for a default nmap run, burying the two or three alerts it produced. And Suricata's EVE protocol telemetry — DNS, Kerberos, SMB — from a segment where a domain controller answers constantly.

Both were cut at the source. Proving it took more care than cutting it.

For the firewall log, the timestamp of the last `filterlog` line received (17:16:06 UTC) was compared with scan alerts still arriving afterwards (17:23:38): alerts flowing, firewall log stopped — so *System Events* carries Suricata's `LOCAL1` facility, and *Everything* is off.

For DNS, a last event at 17:29:05 still unchanged at 17:37:59 looked conclusive — but only if Kali had queried DNS during those nine minutes, which nothing guaranteed. A forced `nslookup` from Kali closed the gap: the query crossed `em2`, and the last DNS event stayed at 17:29:05.

**Lesson:** Silence is evidence only if something should have been seen. The cheapest way to know is to cause the event deliberately — a positive control — and confirm it does not appear. It is the empty OpenVAS scan in reverse: there, a clean report hid a blind scanner; here, a quiet log could have hidden a quiet source.

**Production difference:** SIEM ingestion has a cost — licence, storage, analyst attention — and the discipline is to log what will be acted on. Filtering at the source is the cheap end of that; proving the filter with injected test events belongs in the same change, not after it. The same method validates detection rules: a rule never fired on purpose is a rule never tested.

**Evidence:** `screenshots/06_wazuh_integration/52_wazuh_archives_everything_noise.png`, `48_pfsense_remote_logging_final.png`

---

### Severity is a claim — level 3 means "authorized"

The first working rule filed every Suricata alert at level 3, copied from the level of Wazuh's built-in Suricata rule. On Wazuh's scale, level 3 means *successful or authorized events* — valid logins, firewall accepts. A scan from the DMZ against the domain controller was being filed next to them.

**Fix applied:** a child rule, `100202`, for signatures starting with `ET SCAN`: level 6 — *frequent IDS events* on the same scale — mapped to MITRE ATT&CK **T1046**. Wazuh resolved the tactic (*Discovery*) and technique (*Network Service Discovery*) from the ID alone, and the dashboard's MITRE panel, empty until then, populated. The prefix match proved itself immediately: an `Oracle SQL port 1521` scan signature never seen during development was classified correctly on first arrival.

Non-scan alerts still fall to the parent rule at level 3. Mapping Suricata's own `alert.severity` to Wazuh levels would be the complete fix.

**Lesson:** A severity level is a statement about meaning, and copying a default copies someone else's triage decision without its context. The README had claimed T1046 since the Suricata phase; until this rule, the SIEM did not know it.

**Production difference:** Levels decide what pages someone, what enters an SLA, what gets suppressed. A scan detection filed as an authorized event is not bad data — it is an ignored alert. Severity mapping is a design decision reviewed per rule family, not an inherited default.

**Evidence:** `screenshots/06_wazuh_integration/59_wazuh_dashboard_rule_100201.png`, `60_wazuh_dashboard_rule_100202_mitre.png`

---

### The SIEM logs its own administrator

Diagnosing the integration required `logall`, which makes Wazuh archive everything it receives — including its own host's logs, and therefore `sudo`'s record of every command typed on it, arguments included.

The first consequence was comic: a `grep` for `"event_type":"alert"` in the archive returned the previous `grep`, whose command line contained the pattern. The same trap waited in the alerts file, since `sudo` itself triggers Wazuh alerts. The workaround was a character class: `10020[1]` matches the text `100201` in an alert, but not the literal `10020[1]` that `sudo` records for the command.

The second consequence was not comic. Wazuh's password tool takes the new password as a command-line argument. `sudo` logged it, and the SIEM archived it: the dashboard administrator's password sits in cleartext in the SIEM's own archive.

**Fix applied:** none yet — rotating the password without exposing the new value, and purging the archive, are in Next Steps.

**Lesson:** Command-line arguments are logged — by shell history, by `sudo`, by process accounting — and logs end up in the SIEM. Secrets never go in `argv`. And a SIEM that monitors its own host ingests its administrators' mistakes along with everyone else's.

**Production difference:** A tool that takes secrets as arguments is a finding in itself; the alternatives are prompts, standard input, restricted files, or a secret manager. `sudo` log redaction exists for exactly this. A SIEM's access control deserves the scrutiny of the most sensitive data it holds — which, as here, can include its own credentials.

**Evidence:** `screenshots/06_wazuh_integration/54_wazuh_logtest_sudo_self_match.png`

---

### Moving a VM between segments is a two-layer change

Kali was moved back from the LAN (OpenVAS phase) to the DMZ by switching its VirtualBox adapter to `dmz-network`. The next scan reported `0 hosts up` in 1.84 seconds.

The fast failure was first read as a scan too short to cross an IDS detection threshold. It was nothing of the kind. NetworkManager still had the LAN profile active: Kali held `10.10.10.50/24` on the DMZ wire, treated DC01 as on-link, and sent ARP requests on a segment where nobody answers. No packet ever reached pfSense.

The Wazuh OVA showed the same class of fault from the other side: after a reboot, its interface came up with no IPv4 address at all. The `ifcfg-eth0` file was complete and correct — the deprecated `network-scripts` service simply does not apply it at boot, and `ifup eth0` has to be run by hand.

**Fix applied:** `nmcli con up dmz-static` on Kali — the LAN and DMZ profiles now coexist and switch with one command. `ip addr` and `ip route` checked after every move, on every machine, before any test.

**Lesson:** A VM's position on the network is defined at two layers — the virtual cable and the IP configuration — and changing one produces a machine that looks moved without being moved. A result that comes back unusually fast is worth reading as "nothing happened" before it is read as "something was blocked".

**Production difference:** The equivalent is a server re-patched to a new VLAN with its old static addressing. IPAM, DHCP reservations and configuration management exist to make the two layers move together — and post-change connectivity checks exist because they often don't.

---

## Positive Surprises

### Suricata was already installed and running

When checking Status → Services after completing the firewall rules, Suricata appeared in the list as `Running`. It had been installed during a previous lab on this pfSense image.

Worth revisiting with hindsight: this looked like a shortcut, and the obvious risk seemed to be inherited configuration from an unknown previous session. The fault that actually cost the most time was neither — the `$HOME_NET` default is what *any* fresh pfSense install ships with. Inheriting the package was harmless; trusting a default because the service looked configured was not.

### pfSense audit logging is automatic

The console displayed a syslog message during the first WebGUI login: `Successful login for user 'admin' from: 10.10.10.10 (Local Database)`. pfSense logs authentication events automatically, without any custom configuration. In a real deployment, this would be forwarded to a SIEM. In this lab, it connects directly to P1 (Wazuh can receive pfSense syslog — a future integration point).

### The CRITICAL rule worked on the first test

After all the complexity of the rule configuration (wrong orders, corrupted rules, anchor icon confusion), when the actual segmentation test ran — Kali in DMZ pinging DC01 — the CRITICAL rule fired immediately, one entry per second, with the correct source and destination. The firewall was doing exactly what it was designed to do. The gap between the configuration effort and the elegance of the result is worth noting.

### `wazuh-logtest` made the pipeline visible

For most of the integration, Wazuh behaved as a black box: events in, nothing out. `wazuh-logtest` replays a single line through pre-decoding, decoding and rule matching, shows the result of each stage, and reloads the rule files on every run — so a decoder or rule can be written and tested without restarting the manager or touching production. It turned "Wazuh ignores my alerts" into "pre-decoding reads the program name as the hostname", which is a problem with a fix.

---

## What I Would Do Differently in Production

Building on the honesty from P1: I am still in a learning phase. What follows is my current understanding, not production experience.

- **WAN adapter would use bridged or dedicated physical NIC, not NAT.** VirtualBox NAT introduces artificial constraints (blocked UDP 53, unreachable package repos) that don't exist in real deployments. The homelab limitation shaped the DNS architecture more than the security requirements did — that's backwards.

- **DNS architecture would be documented before implementation.** The split-horizon design (internal names → DC01, external names → pfSense → upstream), the forwarder chain, and the fallback behavior would be drawn out before any VM is configured. Discovering the DNS chain through troubleshooting is expensive and error-prone.

- **Firewall rule sets would be version-controlled.** pfSense supports XML config export (`Diagnostics → Backup & Restore`). In production, every config change would be exported, committed to a Git repository with a change description, and reviewed before application — identical to infrastructure-as-code principles.

- **IDS trust boundaries would be defined per sensor, before deployment.** `HOME_NET` and `EXTERNAL_NET` decide which signatures are even eligible to fire. A sensor watching an internal segment needs a different definition from a perimeter sensor, and leaving both on the platform default disables east-west detection while every health indicator stays green.

- **Detection coverage would be validated on a schedule, not assumed.** Atomic tests or purple-team exercises firing known-signature traffic and confirming the alert lands in the SIEM. Service health tells you the daemon is alive; only a fired alert tells you the detection path works end to end.

- **Scanner output would be calibrated against an independent tool before any report is issued.** A host of known exposure inside every scan scope, whose findings must appear for the report to be accepted. A vulnerability report with no baseline is unfalsifiable, and a false negative in one is worse than no report at all.

- **Vulnerability scans would be credentialed by default.** A dedicated read-only service account for the scanner, rotated and monitored like any other privileged identity. Unauthenticated scanning would remain as the complementary external view, never as the measure of patch compliance.

- **pfSense updates would be tested in a staging environment first.** The inability to update 2.7.2 to 2.8.1 would be a blocker in production. A staging pfSense would receive the update first, connectivity and rule behavior would be validated, then production would follow.

- **DNSSEC would be enabled in production with proper upstream support.** Disabling DNSSEC was the correct fix for this homelab (VirtualBox NAT cannot support it properly), but in production DNSSEC provides an important layer of protection against DNS spoofing. The upstream resolvers would need to be DNSSEC-aware (Cloudflare 1.1.1.1, Google 8.8.8.8, or an internal resolver with DNSSEC validation).

- **Rule logging would be selective and forwarded to the SIEM.** Logging every blocked packet from every interface would generate too much noise. In production, logging would be tuned to capture high-value events (CRITICAL DMZ→LAN attempts, WAN→LAN probes) and forwarded to Wazuh via syslog, creating a unified detection layer across P1 and P2.

- **Log sources would be onboarded with a parser test, not declared covered when data arrives.** Transport, parsing and rule matching are three separate claims; `wazuh-logtest` or its equivalent proves the last two before a source counts toward coverage.

- **Security alerts would travel over an authenticated transport.** Plain UDP syslog is unauthenticated — the source address is trivially spoofed, so anyone on the segment could inject alerts — unencrypted, and lossy under load. Syslog over TLS, or an agent-based forwarder, anywhere beyond an isolated lab.

- **Secrets would never be passed as command-line arguments.** Prompts, standard input or restricted files — otherwise `sudo` and the SIEM keep a copy.

- **Appliances would be managed over SSH from day one.** The hypervisor console cost a full session to a keyboard-layout mismatch and the absence of a clipboard. SSH from a management host removes both, and pfSense's `Diagnostics → Command Prompt` covers the firewall from a browser.

---

## Next Steps

- Forward the WAN inline IPS (and the LAN IDS) alerts to Wazuh through the same pipeline — the perimeter IPS currently blocks traffic the SIEM never sees
- Map Suricata's `alert.severity` to Wazuh levels, so non-scan alerts stop defaulting to level 3
- Rotate the Wazuh dashboard admin password without passing it on the command line, and purge the archive that recorded the current one
- Fix the duplicated rule ID `100001` flagged by `wazuh-logtest` — probably inherited from P1, and one of the two rules is silently ignored
- Make the Wazuh OVA network configuration persistent (NetworkManager instead of the deprecated `network-scripts`)
- Credentialed OpenVAS scanning and remediation — carried over to a dedicated vulnerability-management project

---

*Part of the [LogiSecure SA Enterprise Security Programme](https://github.com/MaxBell10/logisecure-enterprise-security-program) — a 16-project cybersecurity portfolio.*
