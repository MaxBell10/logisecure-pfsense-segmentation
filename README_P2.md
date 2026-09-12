# P2 — pfSense Network Segmentation

<img src="./p2_logo.svg" alt="P2 Logo" width="100%"/>

> **LogiSecure SA** · Enterprise Security Programme
> Firewall : `logisecure-pfsense` · LAN : `10.10.10.1` · DMZ : `10.10.20.1` · WAN : NAT

## Objective

Implement network segmentation for **LogiSecure SA** using pfSense CE as the perimeter firewall, enforcing zone isolation between the IT LAN, the DMZ, and the WAN. Deploy Suricata as an IDS/IPS and validate the infrastructure with OpenVAS vulnerability scanning.

Builds on [P1 — Active Directory & SIEM Foundation](https://github.com/MaxBell10/logisecure-active-directory) — pfSense is inserted as the default gateway for the existing DC01/WKS01/Wazuh infrastructure.

---

## Lab Architecture

```
INTERNET / WAN (VirtualBox NAT)
        │
[pfSense CE 2.7.2 — logisecure-pfsense.lab.local]
        │   WAN  : 10.0.2.15 (DHCP NAT)
        │   LAN  : 10.10.10.1/24
        │   DMZ  : 10.10.20.1/24
        │
        ├── [LAN IT — 10.10.10.0/24]
        │       DC01    10.10.10.10  Windows Server 2022
        │       WKS01   10.10.10.20  Windows 10 Pro
        │       Wazuh   10.10.10.30  Wazuh OVA v4.14.5
        │
        └── [DMZ — 10.10.20.0/24]
                Kali Linux  10.10.20.10  Test / attack host
                Web supplier-facing (planned)
                Cowrie honeypot → logisecure-honeypot-threat-intel (P8)
```

| VM | OS | IP | Role |
|---|---|---|---|
| logisecure-pfsense | FreeBSD (pfSense CE 2.7.2) | WAN: DHCP / LAN: 10.10.10.1 / DMZ: 10.10.20.1 | Firewall, Router, IDS/IPS |
| LOGISECURE-DC01 | Windows Server 2022 | 10.10.10.10 | Domain Controller, DNS (→ pfSense) |
| LOGISECURE-WKS01 | Windows 10 Pro | 10.10.10.20 | Domain-joined workstation |
| LOGISECURE-WAZUH | Ubuntu (OVA) | 10.10.10.30 | Wazuh SIEM — receives Suricata alerts |
| Kali Linux (test) | Kali Linux | 10.10.20.10 | Segmentation + detection test host in DMZ |

---

## What Was Built

### 1. pfSense — Base Configuration

pfSense CE 2.7.2 deployed in VirtualBox with 3 interfaces (WAN/LAN/DMZ). Configured via console (interface assignment, IP setup) then hardened via WebGUI.

| Setting | Value |
|---|---|
| Hostname | `logisecure-pfsense` |
| Domain | `lab.local` |
| WebGUI | `https://10.10.10.1` (HTTPS port 443) |
| DNS | dnsmasq (Forwarder) → `10.0.2.3` (VirtualBox NAT DNS) |
| LAN interface | `em1` — `10.10.10.1/24` |
| DMZ interface | `em2` — `10.10.20.1/24` |

![pfSense Dashboard](screenshots/01_interfaces_webgui/08_webgui_dashboard_clean.png)

**Firewall Aliases:**

| Alias | Type | Value | Usage |
|---|---|---|---|
| `WEB_PORTS` | Port | 80, 443 | HTTP + HTTPS rules |
| `INTERNAL_NETS` | Network | 10.0.0.0/8 | All internal segments |
| `SURICATA_LAN_ONLY` | Network | 10.10.10.0/24 | Custom `$HOME_NET` for the DMZ Suricata instance (see §5) |

---

### 2. Firewall Rules — 12 Rules · 3 Interfaces

**KPI: ≥ 8 rules documented → 12 rules implemented ✅**

**LAN interface (5 rules) — Default Deny:**

| # | Action | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| 1 | ✅ Pass | LAN subnets | LAN subnets | Any | Intra-zone — AD, Wazuh, DNS |
| 2 | ✅ Pass | LAN subnets | DMZ subnets | 443 | LAN → DMZ HTTPS only |
| 3 | ❌ Block+log | LAN subnets | DMZ subnets | Any | Block LAN → DMZ non-HTTPS |
| 4 | ✅ Pass | LAN subnets | Any | WEB_PORTS | LAN → Internet HTTP/HTTPS |
| 5 | ❌ Block+log | LAN subnets | Any | Any | **Default deny LAN** |

> The LAN tab also displays a sixth entry, the **Anti-Lockout Rule**, generated automatically by pfSense to prevent administrative lockout from the WebGUI. It is not user-defined and is excluded from the rule count above.

**DMZ interface (4 rules) — Default Deny:**

| # | Action | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| 6 | ❌ Block+log | DMZ subnets | LAN subnets | Any | **CRITICAL — Block DMZ → LAN** |
| 7 | ❌ Block+log | DMZ subnets | INTERNAL_NETS | Any | Block DMZ → all internal nets |
| 8 | ✅ Pass | DMZ subnets | Any | WEB_PORTS | DMZ → Internet (updates) |
| 9 | ❌ Block+log | DMZ subnets | Any | Any | **Default deny DMZ** |

> Rules 6 and 7 overlap by design: `INTERNAL_NETS` (10.0.0.0/8) already contains the LAN subnet. Rule 6 exists as an explicit, separately named and separately logged control for the single most critical flow in the architecture, so that DMZ→LAN attempts are distinguishable at a glance in the firewall log rather than buried in a generic internal-networks deny.

**WAN interface (3 rules):**

| # | Action | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| 10 | ✅ Pass | Any | DMZ subnets | 443 | WAN → DMZ HTTPS (supplier) |
| 11 | ❌ Block+log | Any | LAN subnets | Any | Block WAN → LAN |
| 12 | ❌ Block+log | Any | Any | Any | Default deny WAN |

![LAN Rules](screenshots/02_firewall_rules/10_lan_rules.png)

![DMZ Rules](screenshots/02_firewall_rules/11_dmz_rules.png)

![WAN Rules](screenshots/02_firewall_rules/12_wan_rules.png)

Full rationale for each rule in [`firewall-rules-justification.md`](./firewall-rules-justification.md).

---

### 3. Segmentation Tests — Proof of Isolation

**Test 1 — DMZ → LAN: BLOCKED ✅**

Kali Linux (10.10.20.10) placed in DMZ attempts to ping DC01 (10.10.10.10). pfSense logs one blocked entry per second under rule `CRITICAL - Block DMZ to LAN`.

![CRITICAL Block DMZ to LAN](screenshots/03_segmentation_tests/16_log_dmz_lan_blocked.png)

**KPI: 100% DMZ→LAN traffic blocked — confirmed by firewall logs ✅**

**Test 2 — LAN → Internet: PASSING ✅**

DC01 browses `https://www.google.com` via pfSense (NAT + dnsmasq forwarding). Confirmed via Diagnostics > DNS Lookup (3 msec) and browser.

![LAN to Internet](screenshots/03_segmentation_tests/17_test_lan_internet_ok.png)

**Test 3 — DMZ → LAN: BLOCKED *and* DETECTED ✅**

A port scan from Kali (10.10.20.10) against DC01 (10.10.10.10) exercises both control layers on the same flow.

The scan reports every port as `filtered` — pf drops each SYN at the DMZ interface under `CRITICAL - Block DMZ to LAN`, one log entry per attempt. Prevention holds.

Suricata, running as a passive IDS on that same interface, still inspected the traffic and matched **ET SCAN** signatures (SID 2002910, SID 2010936) with `src_ip: 10.10.20.10` → `dest_ip: 10.10.10.10` on `em2` — mapping to **MITRE ATT&CK T1046 — Network Service Discovery**. Detection is not downstream of the firewall decision: the sensor reads what arrives on the interface, whether or not pf subsequently drops it.

Reaching this result required correcting the instance's `$HOME_NET` definition — see §5. Until that change, the sensor was receiving and logging the traffic while being structurally unable to alert on it.

Validated bottom-up, one layer at a time:

| Layer | Check | Result |
|---|---|---|
| 1 — Packets on interface | `tcpdump -i em2 -nn host 10.10.10.10` | SYN captured from 10.10.20.10 ✅ |
| 2 — Engine processing | DMZ instance `eve.json` | Flows logged on `em2` ✅ |
| 3 — Rule eligibility | `$HOME_NET` / `$EXTERNAL_NET` in generated config | DMZ moved to EXTERNAL_NET ✅ |
| 4 — Signature match | ET SCAN alert | `alert` event, correct src/dst ✅ |
| 5 — Firewall decision | `Status → System Logs → Firewall` | Block, rule `CRITICAL - Block DMZ to LAN` ✅ |

![tcpdump SYN captured on em2](screenshots/04_suricata/27_pfsense_tcpdump_em2_syn_captured.png)

![Suricata DMZ alert matched](screenshots/04_suricata/30_suricata_dmz_alert_matched.png)

![Firewall block DMZ to LAN](screenshots/04_suricata/31_pfsense_firewall_block_dmz_lan.png)

The full diagnostic path to this result — including two rounds of incorrect conclusions before the real cause was found — is documented in [`lessons_learned_P2.md`](./lessons_learned_P2.md).

---

### 4. DNS Configuration — Full Troubleshooting Chain

DNS resolution required a 5-step fix due to VirtualBox NAT constraints. Documented in detail in [`lessons_learned_P2.md`](./lessons_learned_P2.md).

| Step | Action | Result |
|---|---|---|
| 1 | Disabled DNSSEC in Unbound | Reduced timeout from ∞ to SERVFAIL |
| 2 | Switched DNS Resolver → DNS Forwarder (dnsmasq) | Simpler, no DNSSEC |
| 3 | Added `10.0.2.3` as DNS Server 1 in General Setup | VirtualBox NAT DNS — 75 msec |
| 4 | Enabled DNS Server Override | DHCP-provided DNS added automatically |
| 5 | `Set-DnsServerForwarder -IPAddress "10.10.10.1"` on DC01 | Browser DNS chain complete |

**Final DNS chain:** `DC01 DNS → pfSense dnsmasq (3 msec) → 10.0.2.3 → Internet`

![DNS Lookup pfSense](screenshots/03_segmentation_tests/18_pfsense_dns_lookup_google.png)

---

### 5. Suricata IDS/IPS

> **Status: Detection validated** — three instances deployed across all interfaces, ET Open rulesets applied, `$HOME_NET` tuned for internal segment monitoring, cross-segment detection test passed. SIEM integration remains open.

| Interface | Instance | Blocking mode | Role |
|---|---|---|---|
| WAN (`em0`) | WAN IPS | **Inline IPS** | Perimeter — blocks on match |
| LAN (`em1`) | LAN IDS | Disabled (passive) | Internal visibility — alerts only |
| DMZ (`em2`) | DMZ IDS | Disabled (passive) | DMZ visibility — alerts only |

![Suricata interfaces running](screenshots/04_suricata/24_suricata_interfaces_running.png)

**`$HOME_NET` tuning — required for east-west detection.**

pfSense's default `Home Net` covers *all* locally attached networks, LAN and DMZ alike. Since ET Open scan signatures are written as `alert tcp $EXTERNAL_NET any -> $HOME_NET any`, a DMZ→LAN scan is `$HOME_NET → $HOME_NET` and can never match — the sensor logs the traffic and stays silent. All surface indicators (green status, correct interface, rules enabled, flows in `eve.json`) remain healthy while detection coverage is nil.

Both internal sensors therefore run a custom `Home Net` restricted to the LAN subnet, so that anything sourced outside `10.10.10.0/24` is evaluated as external:

| Setting | Value |
|---|---|
| Firewall alias | `SURICATA_LAN_ONLY` = `10.10.10.0/24` |
| Suricata Pass List | `lan_only` — all five auto-generated groups unchecked, alias referenced |
| DMZ instance `Home Net` | `lan_only` |
| LAN instance `Home Net` | `lan_only` |
| `External Net` (both) | `default` (= everything not in Home Net) |

Resulting engine configuration, verified in the generated YAML rather than in the UI:

```yaml
# DMZ instance (em2)
HOME_NET: "[10.10.10.0/24, 10.10.20.1/32, 127.0.0.1/32, ::1/128]"
EXTERNAL_NET: "[!$HOME_NET]"

# LAN instance (em1)
HOME_NET: "[10.10.10.0/24, 10.10.10.1/32, 127.0.0.1/32, ::1/128]"
EXTERNAL_NET: "[!$HOME_NET]"
```

In both cases the DMZ subnet is gone and only the sensor's own interface address remains, which is expected.

The WAN instance keeps the default `Home Net` deliberately: it is a perimeter sensor, and "everything internal is home, everything else is external" is exactly the semantics it needs.

> The *Local Networks* checkbox in the Pass List re-adds every locally attached subnet, DMZ included. Leaving it checked silently reverts the fix while the UI shows the custom list as applied.

**Scope of each internal sensor.** The LAN instance only observes traffic transiting `em1`. Since pf drops DMZ→LAN packets on ingress at `em2`, they never reach the LAN interface, and the LAN sensor stays silent on that scenario by design — the DMZ instance is the one that covers it. The LAN sensor's value lies elsewhere: LAN-sourced traffic toward the DMZ, and inbound traffic routed to the LAN.

**Known architectural limitation.** A gateway-mounted IDS only observes inter-segment traffic. Intra-VLAN reconnaissance and lateral movement inside `10.10.10.0/24` never traverse pfSense and are structurally invisible to these sensors. In this lab the Wazuh agents on DC01 and WKS01 partially cover that blind spot; closing it properly requires a SPAN/mirror port or a network TAP feeding the sensor.

---

### 6. OpenVAS / Greenbone — Vulnerability Scanning

> **Status: Planned** — Greenbone Community Edition to be deployed via Docker for DC01 vulnerability assessment.

| KPI | Target | Status |
|---|---|---|
| Critical vulnerabilities on DC01 (before) | Documented | 📋 Pending |
| Critical vulnerabilities on DC01 (after) | **0** | 📋 Pending |
| Scan report before/after | Produced | 📋 Pending |

---

## Tech Stack

![pfSense](https://img.shields.io/badge/pfSense-CE_2.7.2-1F3864?style=flat&logo=pfsense&logoColor=white)
![Suricata](https://img.shields.io/badge/Suricata-IDS%2FIPS-E05252?style=flat&logoColor=white)
![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=flat&logo=kalilinux&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox_7.x-183A61?style=flat&logo=virtualbox&logoColor=white)
![Windows Server](https://img.shields.io/badge/Windows_Server_2022-0078D4?style=flat&logo=windows&logoColor=white)

| Tool | Version | Usage |
|---|---|---|
| pfSense CE | 2.7.2-RELEASE | Firewall, Router, DNS Forwarder |
| Suricata | (pfSense package) | IDS/IPS — 3 instances, ET Open rulesets |
| VirtualBox | 7.x | Hypervisor — internal networks |
| Kali Linux | Rolling | Segmentation + detection test host (DMZ) |
| Greenbone / OpenVAS | Community Edition | Vulnerability scanner |
| dnsmasq | (pfSense built-in) | DNS Forwarder → 10.0.2.3 |

---

## Repository Structure

```
logisecure-pfsense-segmentation/
├── README.md
├── p2_logo.svg
├── lessons_learned_P2.md
├── firewall-rules-justification.md
└── screenshots/
    ├── 00_pfsense_base/         # Console — interfaces, hostname, audit log
    ├── 01_interfaces_webgui/    # WebGUI — login, dashboard, general setup, port
    ├── 02_firewall_rules/       # Aliases, LAN/DMZ/WAN rules
    ├── 03_segmentation_tests/   # Kali DMZ config, blocked logs, internet test
    ├── 04_suricata/             # Suricata config, HOME_NET fix, detection test
    └── 05_openvas/              # Greenbone scan reports before/after
```

---

## KPI Dashboard

| KPI | Target | Actual | Status |
|---|---|---|---|
| Firewall rules documented | ≥ 8 | **12 rules** | ✅ |
| DMZ→LAN traffic blocked | 100% | **Confirmed by logs** | ✅ |
| Default deny on each interface | Yes | **3 interfaces** | ✅ |
| Critical logging on blocked flows | Yes | **7 logged rules** | ✅ |
| Suricata alerts detected | ≥ 1 test | **ET SCAN / T1046 validated** | ✅ |
| Sensor `$HOME_NET` scoped for east-west detection | Both internal sensors | **LAN + DMZ instances** | ✅ |
| Suricata → Wazuh integration | Operational | `TBD` | 📋 |
| OpenVAS critical vulns after remediation | 0 | `TBD` | 📋 |

---

## Status

| # | Step | Status |
|---|---|---|
| 1 | pfSense VM + 3 interfaces (WAN/LAN/DMZ) | ✅ Done |
| 2 | WebGUI config (hostname, DNS, port 443) | ✅ Done |
| 3 | Firewall Aliases (WEB_PORTS, INTERNAL_NETS) | ✅ Done |
| 4 | LAN rules — 5 rules incl. default deny | ✅ Done |
| 5 | DMZ rules — 4 rules incl. default deny | ✅ Done |
| 6 | WAN rules — 3 rules incl. default deny | ✅ Done |
| 7 | Segmentation test — DMZ→LAN blocked (logs) | ✅ Done |
| 8 | DNS troubleshooting — dnsmasq + DC01 forwarder | ✅ Done |
| 9 | Gateway switchover — DC01 | ✅ Done |
| 10 | Gateway switchover — WKS01 + Wazuh | ✅ Done |
| 11 | Suricata — interface config + rulesets | ✅ Done |
| 12 | Suricata — `$HOME_NET` scoping + cross-segment detection test | ✅ Done |
| 13 | Suricata — `$HOME_NET` scoping on LAN instance | ✅ Done |
| 14 | Suricata → Wazuh integration (eve.json) | 📋 Planned |
| 15 | OpenVAS — deployment + DC01 scan | 📋 Planned |
| 16 | OpenVAS — remediation + rescan (0 critical) | 📋 Planned |
