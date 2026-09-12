# Firewall Rules — LogiSecure SA · pfSense Segmentation

> **Repo:** `logisecure-pfsense-segmentation` · `firewall-rules-justification.md`
> **Principle:** Default deny on every interface + least privilege + DMZ→LAN absolutely blocked.
> **Target KPI:** ≥ 8 rules documented · 100% DMZ→LAN traffic blocked.

---

## Step 0 — Aliases (Firewall > Aliases > Add)

Create these two aliases **before** the rules, for readable and auditable configuration.

| Name | Type | Value(s) | Description |
|---|---|---|---|
| `WEB_PORTS` | Port | `80`, `443` | HTTP + HTTPS |
| `INTERNAL_NETS` | Network | `10.0.0.0/8` | All internal lab segments |

> `LAN net` (10.10.10.0/24) and `DMZ net` (10.10.20.0/24) are built-in pfSense macros — no need to create them.

---

## LAN interface (Firewall > Rules > LAN)

⚠️ **Delete first:** the default `Default allow LAN to any rule` entries (IPv4 + IPv6) before adding the rules below.

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | ✅ Pass | TCP/UDP | LAN net | LAN net | any | — | LAN intra-zone |
| 2 | ✅ Pass | TCP | LAN net | DMZ net | `443` | — | LAN → DMZ HTTPS only |
| 3 | ❌ Block | * | LAN net | DMZ net | any | ✓ | Block LAN → DMZ (everything but HTTPS) |
| 4 | ✅ Pass | TCP | LAN net | any | `WEB_PORTS` | — | LAN → Internet HTTP/HTTPS |
| 5 | ❌ Block | * | LAN net | any | any | ✓ | **Default deny LAN** — catch-all |

> pfSense also displays an **Anti-Lockout Rule** at the top of this tab. It is generated automatically to prevent administrative lockout from the WebGUI, is not user-defined, and is excluded from the rule count.

**Rationale:**

- **Rule 1** — AD flows (Kerberos, LDAP, RPC), DNS (DC01) and Wazuh agent traffic (port 1514) must circulate freely on the LAN. Blocking intra-LAN traffic would break the P1 infrastructure.
- **Rule 2** — Administrators reach the supplier-facing DMZ web server over HTTPS only. HTTP is forbidden — no cleartext traffic toward the DMZ.
- **Rule 3** — Any LAN→DMZ connection other than TCP/443 is blocked and logged. Surfaces unauthorised access attempts toward the DMZ.
- **Rule 4** — Internet browsing and updates for IT workstations, restricted to HTTP/HTTPS.
- **Rule 5** — Catch-all: any LAN flow not covered by the preceding rules is blocked and logged. Conforms to the *default deny* principle (NIS2 Art. 21, ISO 27001 A.8.20).

---

## DMZ interface (Firewall > Rules > DMZ)

> The DMZ has **no default rules** — everything is implicitly blocked. Rules are added in the order below.

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 6 | ❌ Block | * | DMZ net | LAN net | any | ✓ | **CRITICAL: Block DMZ → LAN** |
| 7 | ❌ Block | * | DMZ net | `INTERNAL_NETS` | any | ✓ | Block DMZ → all internal segments |
| 8 | ✅ Pass | TCP | DMZ net | any | `WEB_PORTS` | — | DMZ → Internet (updates) |
| 9 | ❌ Block | * | DMZ net | any | any | ✓ | **Default deny DMZ** — catch-all |

**Rationale:**

- **Rule 6** — The most critical rule in the project: a compromised DMZ host (Cowrie honeypot, supplier web server) must **never** be able to initiate a connection toward the IT LAN. A violation means direct lateral movement (EBIOS RM scenario B). Logging is mandatory as segmentation evidence.
- **Rule 7** — Belt and suspenders. Rules 6 and 7 deliberately overlap, since `INTERNAL_NETS` (10.0.0.0/8) already contains the LAN subnet. Rule 6 exists as a separately named and separately logged control for the single most critical flow, so DMZ→LAN attempts are identifiable at a glance in the firewall log rather than buried in a generic internal-networks deny. Rule 7 additionally covers future OT segments (10.10.30.x, 10.10.40.x) and internal ranges not yet deployed.
- **Rule 8** — DMZ servers need OS and application updates. Restricted to outbound HTTP/HTTPS toward the Internet only; rules 6 and 7 already block any path toward internal networks above it.
- **Rule 9** — DMZ catch-all. Any uncovered flow (direct DNS, unauthorised outbound SMTP) is blocked and logged.

---

## WAN interface (Firewall > Rules > WAN)

> pfSense blocks all inbound WAN traffic by default (implicit deny). These rules make the posture **explicit and logged**, for traceability and evidence capture.

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 10 | ✅ Pass | TCP | any | DMZ net | `443` | — | WAN → DMZ HTTPS (supplier access) |
| 11 | ❌ Block | * | any | LAN net | any | ✓ | Block WAN → LAN — no direct exposure |
| 12 | ❌ Block | * | any | any | any | ✓ | **Default deny WAN** — catch-all |

**Rationale:**

- **Rule 10** — The supplier-facing web service in the DMZ is reachable from the Internet over HTTPS only. Placed **above** the deny rules, since pfSense evaluates top-down and first match wins. The service itself is not yet deployed; the rule is pre-positioned so the exposure path is designed, reviewed and documented before anything is published rather than opened ad hoc under delivery pressure.
- **Rule 11** — The IT LAN must never be reachable from the Internet. An explicit rule ensures the firewall log captures every inbound attempt and feeds it to Wazuh — an implicit deny blocks silently and produces no evidence.
- **Rule 12** — Catch-all. Anything not matching rules 10 or 11 is blocked and logged, including scans against WAN-side services and traffic toward ports never intended to be exposed.

---

## Post-implementation verification

| Flow to test | Expected result | How to test | Status |
|---|---|---|---|
| DMZ → LAN (ping 10.10.10.10) | ❌ Blocked + logged | From Kali in DMZ | ✅ Verified — screenshot 16 |
| DMZ → LAN (port scan) | ❌ Blocked + logged, ✅ detected by Suricata | `nmap` from Kali, then firewall log + `eve.json` | ✅ Verified — screenshots 27, 30, 31 |
| WAN → LAN (any) | ❌ Blocked + logged | Status > System Logs > Firewall | ✅ Verified |
| LAN → Internet HTTPS | ✅ Passing | Browser on DC01 / WKS01 | ✅ Verified — screenshot 17 |
| LAN → DMZ :443 | ✅ Passing | `curl https://10.10.20.x` | 📋 Pending — no DMZ web service deployed yet |
| LAN → DMZ :80 | ❌ Blocked + logged | `curl http://10.10.20.x` | 📋 Pending — same reason |

> Firewall log captures showing blocked traffic are the **proof of segmentation** — mandatory screenshots for the repo.

> Detection of a blocked flow is a separate control from the block itself, and requires its own evidence. Both internal Suricata instances (LAN and DMZ) run a custom `$HOME_NET` restricted to `10.10.10.0/24`, so that DMZ-sourced traffic is evaluated as `$EXTERNAL_NET` — without it, ET Open scan signatures cannot match DMZ→LAN traffic at all. The WAN instance keeps the default, which is the correct semantics for a perimeter sensor. See §5 of the README.

The two pending tests depend on the supplier-facing DMZ web service, which is not yet deployed. Rules 2, 3 and 10 are in place and evaluated correctly by pf, but the end-to-end flow cannot be exercised against a host that does not exist. This is a *control implemented but not tested* state, distinct from *control tested and effective*, and is recorded as such rather than assumed to pass.

---

## KPI summary

| KPI | Target | Result |
|---|---|---|
| Rules documented | ≥ 8 | **12 rules** (5 LAN + 4 DMZ + 3 WAN) |
| DMZ→LAN traffic blocked | 100% | Rule 6 — explicit block + log, verified |
| DMZ→LAN attempt detected | ≥ 1 | Suricata DMZ IDS — ET SCAN SID 2002910 / 2010936 matched |
| Default deny on every interface | Yes | Rules 5, 9, 12 |
| Logging on critical blocked flows | Yes | Rules 3, 5, 6, 7, 9, 11, 12 — 7 logged rules |
