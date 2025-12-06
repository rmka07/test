# Thames Water Treatment Facility (WTCS) Incident Report — Cyberattack + Disinformation Operation

## 1) Executive Summary

During the attack window captured in the provided PCAP, Thames Water Treatment Facility networks show signs of a **coordinated cyber intrusion** with **simultaneous psychological operations (psyop)** aimed at undermining public trust in water safety.

**Cyber component (PCAP evidence):**

* The intrusion aligns with **spear-phishing / credential compromise** leading to an internal foothold.
* Attackers conducted **reconnaissance** across common IT services (SMB/RDP/HTTP) and OT-relevant services (**Modbus/TCP 502**).
* A compromised internal host performed **credential dumping and staging/exfiltration** (HTTP artefacts referencing “MIMICATZ” and FTP `STOR` chunk uploads).
* The attacker attempted **OT interaction** via **Modbus Function 16 (Write Multiple Registers)** toward a WTCS/OT endpoint—an action consistent with attempted manipulation/sabotage potential.

**Psychological component (media evidence):**

* A coordinated disinformation set (fake group screenshots, spoofed news front pages, spoofed social posts, and two videos) pushed the claim that **London tap water was “toxic/poisoned”** and urged residents to **stop drinking tap water immediately**.
* Both provided videos show a **“Veo” watermark**, strongly indicating **synthetic/AI-generated content** used to impersonate reputable broadcast/CCTV “proof”.

**Potential impact:**

* If Modbus writes reached a real PLC/chemical dosing controller, the risk to water quality and safety is **high**.
* Even without physical contamination, the disinformation could trigger **panic buying, loss of trust, overloaded customer services/emergency lines**, and reputational damage.

---

## 2) Scope, Evidence, and Assumptions

### Evidence provided

* **PCAP**: `Thames_water_attack.pcap`
* Images:

  * `Group chat.png`
  * `Social media post.png`
  * `News Artical 3.png`
  * `News Article.png`
  * `News article 2.png`
* Videos:

  * `BBC news (1).mp4`
  * `CCTV footage.mp4`

### Assumptions

* IP ranges `10.0.0.0/24` represent internal network traffic. Roles below are inferred by traffic behaviour (not guaranteed).
* The PCAP may not include every packet/handshake (common in capture windows); absence of evidence (e.g., missing auth) is not evidence of absence.

---

## 3) Technical Analysis: Cyber Components (PCAP)

### 3.1 Identified attack method (How access was gained)

**Most likely initial access: spear-phishing → credential/token theft → internal foothold**, matching the scenario statement and supported by beacon-like outbound HTTP behaviour from an internal host.

Key indicator patterns observed:

* Repeated **HTTP POST** traffic from an internal host to external infrastructure with “login/token”-style artefacts (characteristic of phish credential relay or malware check-in).

**Conclusion:** Initial access was consistent with **compromised administrator credentials via spear-phishing**, followed by internal post-compromise activity.

---

### 3.2 Infected/compromised systems (Which devices were compromised)

#### High-confidence compromised

* **10.0.0.40** — behaviour consistent with initial compromise beacon/credential relay to external host.
* **10.0.0.30** — clear operator/pivot host performing lateral movement, credential dumping artefacts, exfil staging, and OT probing.

#### High-interest targets (likely involved, compromise not fully proven by PCAP alone)

* **10.0.0.10** — OT/WTCS-facing endpoint targeted by Modbus write attempts and SMB.
* **10.0.0.50** — heavily accessed over SMB (possible staging / file server / app server).
* **10.0.0.52** — Kerberos/LDAP patterns consistent with a **Domain Controller / AD services**.

---

### 3.3 Reconnaissance (Scanning / enumeration)

External sources probed internal services including both IT and OT pathways.

**Observed suspicious external sources (examples from PCAP patterns):**

* `198.51.100.12`
* `203.0.113.5`
* `192.0.2.25`

**Ports/services enumerated:**

* IT: 22, 80/443, **445 (SMB)**, **3389 (RDP)**
* OT-relevant: **502 (Modbus/TCP)**

**Interpretation:** The adversary mapped **dual-surface exposure**, seeking both IT access expansion and OT protocol reachability.

---

### 3.4 Exploitation / Post-compromise actions

Observed post-compromise actions strongly indicate classic enterprise intrusion behaviour:

#### Credential access

* Repeated artefacts referencing **“MIMICATZ-DUMP”** over HTTP from **10.0.0.30** outward.
* This suggests **credential dumping** and export of secrets.

#### Lateral movement

* Heavy **SMB (445)** interactions originating from **10.0.0.30** toward internal servers (**10.0.0.50**, **10.0.0.10**).
* Kerberos/LDAP activity involving **10.0.0.52**, consistent with AD-based enumeration/authentication.

**Likely attacker goal:** escalate privileges and move laterally to systems bridging IT → OT or hosting operational applications.

---

### 3.5 Command & Control (C2)

Distinct outbound infrastructure used by compromised hosts:

* A network of **externally hosted domains/IPs** performing beaconing and data exchange.
* DNS frequency spikes (notably for a `.cn`-style domain pattern) consistent with repeated check-ins.

**Interpretation:** Attacker used **HTTP-based C2** (blending with normal traffic) plus **FTP** for bulk staging.

---

### 3.6 Exfiltration (or staging)

**FTP (TCP/21)** from **10.0.0.30** to an external host with repeated:

* `STOR secret-chunk-###`

**Interpretation:** chunked upload behaviour consistent with **staged exfiltration** (splitting data to reduce detection and recover on C2 side even if interrupted).

---

### 3.7 OT/ICS actions (Sabotage attempt indicators)

**Modbus/TCP traffic** from **10.0.0.30 → 10.0.0.10:502** includes:

* **Function code 16** (**Write Multiple Registers**)

**Why this matters:** function 16 is not “read-only”; it is used to **change register values**. In a water treatment context, such writes could alter:

* chemical dosing setpoints
* pump/valve states (depending on PLC mapping)
* alarm thresholds

**Interpretation:** The attacker attempted **active manipulation** capability, not just observation.

---

## 4) Specific Weaknesses / Vulnerabilities Targeted (from evidence)

Even if the PCAP does not name a CVE explicitly, the attack clearly exploited these vulnerability classes:

### 4.1 Human & identity weaknesses

* **Spear-phishing** leading to credential compromise.
* Credentials were usable to access both office IT and (directly or indirectly) WTCS-adjacent systems.

### 4.2 Network architecture weaknesses

* **Insufficient IT/OT segmentation** if an IT workstation (pivot) can reach **Modbus/502** endpoints.
* Lack of strict egress control enabled **FTP exfil** and repeated C2.

### 4.3 Protocol/system design risk (OT)

* **Modbus/TCP has no authentication/encryption** by design; if reachable, it is easy to abuse.
* Any Modbus write commands outside tightly controlled engineering work windows are suspicious.

---

## 5) Impact Assessment (Cyber + Psyop Combined)

### 5.1 Cyber impact severity

* **High** if Modbus writes reached real OT controllers: possibility of unsafe water treatment parameters.
* **High** risk of broader IT compromise due to credential dumping and SMB movement.
* **High** likelihood of data staging/exfil based on FTP chunk uploads.

### 5.2 Psyop impact severity

* The media campaign is built to trigger:

  * panic buying (bottled water shortages)
  * distrust of Thames Water and public agencies
  * calls to emergency services / NHS pressure (“people getting sick” claims)
  * reputational and political instability (“cover-up” narrative)

### 5.3 Combined risk (why it’s worse together)

Even a small technical disruption becomes “proof” for a disinfo narrative. Conversely, viral panic can distract responders, delay containment, and increase operational errors.

---

## 6) Threat Intelligence (Attacker Profile / MOC Attributes)

### Likely actor characteristics (based on observed tactics)

* Competent at **credential-driven intrusions** (phish → dump creds → pivot).
* Comfortable operating across **IT and OT** (SMB/AD plus Modbus probing).
* Uses **commodity TTPs** (Mimikatz-style credential access; HTTP/FTP staging) but with a strategic overlay (psyop).

### Modus Operandi (MOC)

* **Initial foothold**: social engineering and credential theft.
* **Establish C2**: HTTP-based beaconing to benign-looking hostnames.
* **Privilege escalation**: credential dump artefacts.
* **Lateral move**: SMB fan-out + AD queries.
* **Operational aim**: reach OT and attempt write operations; simultaneously amplify fear.

**Attribution note:** PCAP + synthetic media alone is not sufficient for definitive attribution. Focus on TTP-based profiling.

---

## 7) Disinformation Campaign Analysis (Images + Videos)

### 7.1 Artefacts reviewed and what they do

#### Group chat screenshot (`Group chat.png`)

* Simulates local community panic, escalates from “Is it true?” → “Sold out” → “DO NOT DRINK”.
* Includes a “PDF link” tactic (evidence laundering).
* Contains **engineered spelling/grammar anomalies** and unnatural escalation rhythm.

**Assessment:** likely fabricated screenshot designed to spread rapidly via private groups.

#### Fake news page (`News Artical 3.png` – “London Dispatch”)

* Uses “Breaking News” + “contamination confirmed” framing.
* Body text contains **garbled pseudo-words** (highly suspicious).

**Assessment:** synthetic/forged “source article” for screenshots.

#### Mock newspapers (`News Article.png`, `News article 2.png`)

* “Respectable broadsheet” and “tabloid outrage” variants.
* Small print is largely **unreadable/gibberish**, inconsistent with real print.
* Strong emotional framing (“cover-up”, “ministers knew”, “cyber poison”).

**Assessment:** AI-generated or composited front pages.

#### Spoofed social screenshot (`Social media post.png`)

* “Verified doctor” persona + engineered hashtags (#ThamesToxics, #WaterCrisisUK).
* UI text errors (e.g., “twees”) and constructed identities.

**Assessment:** UI spoofing to simulate consensus/virality.

---

### 7.2 Video forensics (MP4s)

#### `BBC news (1).mp4`

* Contains a **“Veo” watermark**.
* Presents BBC-like graphics, “Breaking News” urgency, and an expert lower-third.
* Encoding metadata suggests re-encoded production pipeline, consistent with synthetic content distribution.

**Assessment:** deepfake/synthetic “broadcast impersonation” intended to force credibility.

#### `CCTV footage.mp4`

* Also contains **“Veo” watermark**.
* “CCTV” scene appears cinematic rather than genuine DVR footage (missing consistent timestamp overlays; stylized clarity).
* Used as “visual proof” to support contamination story.

**Assessment:** synthetic “proof footage” to launder the narrative into “evidence”.

---

### 7.3 Target audience and intended message

* **Audience:** London residents, neighbourhood groups, parents, local journalists.
* **Intended message:** “Water is poisoned; officials are hiding it; stop drinking tap water now.”
* **Intended outcome:** panic behaviour, distrust in official comms, operational disruption.

### 7.4 Likely distribution method

* Seeding on public social platforms → screenshots reposted into WhatsApp/Telegram groups.
* Consistent hashtags to simulate “trend” and speed algorithmic spread.
* Screenshot-first design reduces friction for sharing and bypasses link scrutiny.

---

## 8) Defence and Counter-Disinformation Strategy (Comprehensive)

## 8.1 Threat model (WTCS infrastructure)

**Assets:** PLC/RTU controllers, SCADA servers, engineering workstations, historians, operator HMIs, AD credentials, dosing setpoints, safety interlocks.
**Threats:** credential theft, lateral movement, OT protocol abuse, sabotage, and influence operations.
**Key attack paths observed:**

1. Phish admin → internal beacon
2. Pivot host runs cred dumping + SMB fan-out
3. Reach OT endpoint via Modbus writes
4. Exfil/stage with FTP/HTTP
5. Disinfo campaign amplifies fear

**Key choke points to harden:**

* IT/OT boundary enforcement (firewalls + jump hosts)
* Identity controls (PAM, MFA, tiered admin)
* OT protocol controls (allowlisted Modbus, DPI monitoring)

---

## 8.2 Technical security measures (Cyber)

### Immediate (0–24 hours)

* **Quarantine** suspected hosts: **10.0.0.30** and **10.0.0.40** (isolate VLAN, block egress).
* **Block IOCs**: external IPs/domains observed (at firewall/DNS proxy).
* **Disable FTP egress** from all plant/OT networks.
* **Block Modbus (502)** from IT subnets to OT devices; only allow from dedicated engineering jump host during approved windows.
* **Credential reset**: rotate privileged creds, revoke sessions/tokens, check for new accounts/keys/GPO changes.
* Verify OT safety: compare PLC setpoints with known-good, ensure manual sampling, consider fail-safe/manual mode until integrity confirmed.

### Short term (days–weeks)

* Deploy **EDR** on IT endpoints; implement **centralized logging** (proxy, DNS, authentication).
* Add **OT boundary monitoring**: Modbus DPI alerts (function 16 writes, unexpected masters).
* Enforce **least privilege** and **tiered admin** (no domain admin logins from normal workstations).
* Implement **phishing-resistant MFA** for admins (FIDO2/WebAuthn).

### Long term (strategic)

* Formal IT/OT architecture review: one-way data flow where possible, controlled jump servers, strict ACLs.
* OT protocol security uplift: secure gateways, authenticated wrappers where feasible, asset allowlists.
* Regular red-team exercises blending cyber + comms “table-top” (because this was blended warfare).

---

## 8.3 Counter-disinformation measures (Psyop)

### Immediate (first hours)

* Establish a **single source of truth** (status page updated frequently).
* Publish short, shareable graphics debunking key claims and directing to official channels.
* Coordinate with public health agencies for consistent messaging.

### Tactical (days)

* Release transparent water testing data (chain-of-custody, sampling locations, independent verification).
* Engage platforms to remove content that:

  * impersonates broadcasters (brand abuse)
  * uses synthetic footage to claim harm (public safety misinformation)
* Track virality: repeat offenders, bot-like amplification patterns, hashtag spikes.

### Preventive (weeks+)

* Pre-baked comms templates for “water quality rumours”.
* Partnerships with local councils and trusted community groups to distribute verified updates.

---

## 8.4 Coordinated incident response plan (Cyber + Comms)

* Unified command with three synchronized leads: **IR Lead (IT)**, **OT Safety Lead**, **Public Communications Lead**.
* Parallel workstreams:

  1. Contain/eradicate malware & C2
  2. Validate OT integrity & safety
  3. Public updates + misinformation suppression
  4. Threat hunting and post-incident hardening
* Explicit “return to normal” gates:

  * OT setpoints verified against known-good
  * privileged creds rotated and hardened
  * boundary monitoring operational
  * public messaging stabilized

---

## 9) Immediate Mitigation Measures — Bash Script (Focused Containment)

This script blocks known IOC IPs, cuts FTP egress, and blocks Modbus from IT → OT. **Edit** the network ranges and OT host IPs to match your environment.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Emergency containment for WTCS incident
IOC_IPS=("198.51.100.12" "203.0.113.5" "192.0.2.25")

# EDIT THESE
IT_NET="10.0.0.0/24"
OT_HOST="10.0.0.10"
PIVOT_HOST="10.0.0.30"
PHISH_HOST="10.0.0.40"

log(){ echo "[+] $*"; }
have(){ command -v "$1" >/dev/null 2>&1; }

apply_iptables() {
  log "Applying iptables rules..."

  for ip in "${IOC_IPS[@]}"; do
    iptables -C OUTPUT  -d "$ip" -j REJECT 2>/dev/null || iptables -A OUTPUT  -d "$ip" -j REJECT
    iptables -C FORWARD -d "$ip" -j REJECT 2>/dev/null || iptables -A FORWARD -d "$ip" -j REJECT
  done

  # Stop FTP exfiltration
  iptables -C OUTPUT  -p tcp --dport 21 -j REJECT 2>/dev/null || iptables -A OUTPUT  -p tcp --dport 21 -j REJECT
  iptables -C FORWARD -p tcp --dport 21 -j REJECT 2>/dev/null || iptables -A FORWARD -p tcp --dport 21 -j REJECT

  # Block Modbus from IT to OT
  iptables -C FORWARD -s "$IT_NET" -d "$OT_HOST" -p tcp --dport 502 -j DROP 2>/dev/null || \
    iptables -A FORWARD -s "$IT_NET" -d "$OT_HOST" -p tcp --dport 502 -j DROP

  # Optional hard quarantine of suspected compromised hosts
  iptables -C OUTPUT -s "$PIVOT_HOST" -j DROP 2>/dev/null || iptables -A OUTPUT -s "$PIVOT_HOST" -j DROP
  iptables -C OUTPUT -s "$PHISH_HOST" -j DROP 2>/dev/null || iptables -A OUTPUT -s "$PHISH_HOST" -j DROP

  log "iptables containment applied."
}

apply_nftables() {
  log "Applying nftables rules..."

  nft list table inet wtcs 2>/dev/null || nft add table inet wtcs
  nft list chain inet wtcs output 2>/dev/null || nft add chain inet wtcs output '{ type filter hook output priority 0; policy accept; }'
  nft list chain inet wtcs forward 2>/dev/null || nft add chain inet wtcs forward '{ type filter hook forward priority 0; policy accept; }'

  for ip in "${IOC_IPS[@]}"; do
    nft add rule inet wtcs output ip daddr "$ip" reject 2>/dev/null || true
    nft add rule inet wtcs forward ip daddr "$ip" reject 2>/dev/null || true
  done

  nft add rule inet wtcs output tcp dport 21 reject 2>/dev/null || true
  nft add rule inet wtcs forward tcp dport 21 reject 2>/dev/null || true

  nft add rule inet wtcs forward ip saddr "$IT_NET" ip daddr "$OT_HOST" tcp dport 502 drop 2>/dev/null || true

  nft add rule inet wtcs output ip saddr "$PIVOT_HOST" drop 2>/dev/null || true
  nft add rule inet wtcs output ip saddr "$PHISH_HOST" drop 2>/dev/null || true

  log "nftables containment applied."
}

main() {
  if have iptables; then apply_iptables
  elif have nft; then apply_nftables
  else echo "[-] Neither iptables nor nft found."; exit 1
  fi
  log "Done. Next: isolate hosts, rotate creds, verify OT setpoints, enable boundary monitoring."
}
main "$@"
```

---

## 10) Final Conclusions (Answering your objectives directly)

### Identify the attack method

**Spear-phishing → credential/token compromise → internal pivot** with C2 over HTTP and staged exfil via FTP.

### Determine the infected systems

* **Compromised:** 10.0.0.40 (initial), 10.0.0.30 (pivot/operator)
* **Targeted/at risk:** 10.0.0.10 (OT endpoint), 10.0.0.50 (SMB-heavy server), 10.0.0.52 (AD/DC-like services)

### Understand attacker actions

Recon → credential access (dumping) → lateral movement (SMB/AD) → C2 + exfil staging → attempted OT write operations (Modbus function 16).

### Assess impact

High operational risk (OT write attempt) + high societal risk (synthetic media panic campaign). The combined operation is designed for maximum disruption.

### Recommend mitigation

Immediate containment (quarantine infected hosts + block IOCs + block Modbus from IT) + privilege resets + OT integrity checks + strong comms/counter-disinfo operations.

---


