# Integrated Web Security Assessment, Vulnerability Management, Mini-SIEM & Incident Response Lab

## Task 5 Capstone Project — ApexPlanet Cybersecurity & Ethical Hacking Internship

**Author:** Pratham Dodhiwala
**Program:** ApexPlanet Software Pvt. Ltd. — Cybersecurity & Ethical Hacking Internship
**Task:** Task 5 — Capstone Project & Incident Response (Days 49-60)

---

## 1. Executive Summary

This capstone project combines four core cybersecurity disciplines into one
integrated engagement: web application penetration testing, network vulnerability
assessment, security information and event management (SIEM), and formal incident
response. Testing was conducted exclusively against intentionally vulnerable,
self-hosted lab systems (DVWA and Metasploitable2) on an isolated VirtualBox
network, with no real-world targets involved at any stage.

Five critical/high-severity findings were confirmed on DVWA (SQL Injection, Stored
XSS, Reflected XSS, Local File Inclusion, and a CSRF proof-of-concept partially
mitigated by browser policy), and two remotely exploitable services were confirmed
and exploited on Metasploitable2 (UnrealIRCd backdoor, distcc RCE). A custom
ELK-based SIEM was built to detect the resulting attack traffic in near real time,
and a full incident response lifecycle — containment, eradication, and recovery —
was executed and independently verified against the Metasploitable2 compromise.

## 2. Project Objectives

- Conduct a controlled web application penetration test against DVWA, covering
  the OWASP Top 10 categories represented in the application
- Perform a network vulnerability assessment of the lab's Metasploitable2 host,
  including reconnaissance, scanning, and controlled exploitation
- Design and deploy a functional mini-SIEM using the Elastic stack, capable of
  ingesting both web and authentication log data
- Build a safe, local phishing awareness simulation demonstrating common social
  engineering red flags
- Simulate a realistic security incident, detect it via the SIEM, and execute a
  full incident response lifecycle with independently verified remediation

## 3. Scope

**In scope:**
- DVWA (Damn Vulnerable Web Application), hosted locally on the Kali attacker VM
- Metasploitable2 (192.168.56.101), an intentionally vulnerable target VM
- Kali Linux (192.168.56.10 host-only / bridged network), the attacker/lab platform
- A locally-hosted phishing awareness simulation page
- A Dockerized Elastic Stack (Elasticsearch + Kibana) SIEM

**Out of scope:**
- Any system, network, or individual outside the isolated lab environment
- Real-world phishing targets or credential collection

## 4. Authorization & Ethics

All testing was conducted against systems owned and controlled by the author,
within an isolated VirtualBox lab network with no exposure to the public internet
or third parties. No real credentials, real individuals, or production systems
were involved at any point. The phishing simulation was confined to a local-only
web server with no real-world distribution.

## 5. Lab Architecture

| Component | Role | Details |
|---|---|---|
| Kali Linux | Attacker platform | 192.168.56.10 (host-only), bridged to LAN for internet access |
| Metasploitable2 | Vulnerable target | 192.168.56.101, host-only network |
| DVWA | Vulnerable web application | Co-located on Kali (Apache + MariaDB), PHP 8.4.23 |
| Elasticsearch + Kibana | SIEM backend | Docker containers on the Windows host, ports 9200/5601 |
| Filebeat | Log shipper | Installed on Kali, ships auth.log and Apache access.log |

## 6. Network Diagram
            [ Internet / LAN ]
                    |
          (Bridged Adapter, DHCP)
                    |
             [ Kali Linux ]---(Host-only 192.168.56.0/24)---[ Metasploitable2 ]
             192.168.56.10                                   192.168.56.101
             (Attacker + DVWA host)
                    |
          (LAN, Windows host IP)
                    |
          [ Windows Host: Docker ]
          Elasticsearch :9200
          Kibana :5601## 7. Tools Used

## 7. Tools Used 
 Tool | Purpose |
|---|---|
| Nmap | Network reconnaissance and port/service scanning |
| Metasploit Framework | Exploitation of UnrealIRCd and distcc vulnerabilities |
| Burp Suite / manual browser testing | DVWA web application testing |
| Filebeat | Log shipping to Elasticsearch |
| Elasticsearch + Kibana | Log storage, search, and visualization (SIEM) |
| John the Ripper (prior task) | Password hash cracking (Task 4, referenced) |

## 8. Methodology

Testing followed a standard structured approach at each phase:

**Web application testing:** Reconnaissance → Enumeration → Vulnerability
Identification → Controlled Exploitation → Evidence Collection → Risk Rating →
Mitigation → Retest (where applicable).

**Network vulnerability assessment:** Host discovery → Port/service scanning →
Vulnerability correlation → Prioritization → Controlled exploitation → Evidence
collection.

**Incident response:** Preparation → Identification (Detection) → Containment →
Eradication → Recovery → Lessons Learned, following the standard IR lifecycle.

## 9. Reconnaissance & Scanning

A full TCP port/service/OS scan of Metasploitable2 was conducted:
21 open ports were identified, including several with known critical
vulnerabilities: vsftpd (backdoor, exploited in a prior task), UnrealIRCd 3.2.8.1
(trojaned backdoor), distccd (unauthenticated RCE), NFS, and an unauthenticated
VNC service. Full results are recorded in `nmap_metasploitable2_full.txt`.

## 10. Network Vulnerability Assessment

Two critical remotely-exploitable services were identified and confirmed on
Metasploitable2 via the Nmap scan and subsequent manual verification:

| ID | Vulnerability | Severity | Component | CVE |
|---|---|---|---|---|
| V-01 | UnrealIRCd 3.2.8.1 trojaned backdoor | Critical | UnrealIRCd, port 6667 | CVE-2010-2075 |
| V-02 | distcc unauthenticated remote command execution | High | distccd, port 3632 | CVE-2004-2687 |

### V-01: UnrealIRCd 3.2.8.1 Backdoor
- **Description:** The official UnrealIRCd 3.2.8.1 download between Nov 2009 and
  Jun 2010 was replaced with a trojaned version containing a hidden backdoor that
  executes arbitrary shell commands sent via a specific trigger string over the
  IRC protocol.
- **Testing methodology:** Metasploit module
  `exploit/unix/irc/unreal_ircd_3281_backdoor` was used against port 6667.
- **Command:**
- **Observed result:** Meterpreter session opened; `getuid` confirmed `root`;
  `sysinfo` confirmed target OS (Ubuntu 8.04, kernel 2.6.24-16-server).
- **Security impact:** Complete remote root compromise with no authentication
  required.
- **Root cause:** Supply-chain compromise of the distributed binary; no code
  vulnerability in the traditional sense, but a trusted-download integrity failure.
- **Recommended mitigation:** Verify binary checksums/signatures before
  installation; upgrade to a patched UnrealIRCd version; restrict IRC service
  exposure via firewall.
- **Status:** RESOLVED — service killed, autostart removed, binary disabled
  (see Section 13, Containment & Eradication).

### V-02: distcc Unauthenticated Remote Code Execution
- **Description:** distcc's daemon (distccd) accepts and executes arbitrary
  compiler commands from any client with no authentication mechanism by design.
- **Testing methodology:** Metasploit module `exploit/unix/misc/distcc_exec`
  against port 3632. Default payload (`cmd/unix/reverse_bash`) failed due to
  a `/dev/tcp` incompatibility on the target's shell; payload was switched to
  `cmd/unix/reverse_perl`, which succeeded.
- **Observed result:** Command shell session opened; `id` confirmed
  `uid=1(daemon)`, a lower-privilege account than root, consistent with distccd's
  unprivileged execution context.
- **Security impact:** Remote code execution as the `daemon` user; while not
  immediately root, this provides a foothold for further privilege escalation.
- **Root cause:** distccd designed without authentication, intended only for use
  on fully trusted internal networks — exposed here without that isolation.
- **Recommended mitigation:** Never expose distccd to untrusted networks; if
  required, restrict access via firewall to known build-farm hosts only.
- **Status:** RESOLVED — service killed and removed from init startup
  (see Section 13).

## 11. Web Application Penetration Testing (DVWA)

All testing was performed at DVWA's "Low" security setting to establish baseline
vulnerability presence, consistent with the methodology used in the prior Task 3
engagement.

### Finding W-01: SQL Injection
- **Location:** `/vulnerabilities/sqli/`
- **Payload (auth bypass):** `1' OR '1'='1`
- **Payload (data extraction):** `1' UNION SELECT user, password FROM users-- -`
- **Result:** Full user table dumped on the first payload; usernames and MD5
  password hashes extracted on the second, including a hash matching the known
  MD5 of the string "password" (`5f4dcc3b5aa765d61d8327deb882cf99`), demonstrating
  trivially crackable credentials in addition to the injection itself.
- **Severity:** Critical
- **Root cause:** User input concatenated directly into SQL queries with no
  parameterization.
- **Mitigation:** Prepared statements / parameterized queries; least-privilege
  database account; input validation.

### Finding W-02: Stored Cross-Site Scripting
- **Location:** `/vulnerabilities/xss_s/` (guestbook)
- **Payload:** `<script>alert('Stored XSS by Task5')</script>`
- **Result:** Payload persisted in the application database and executed on every
  subsequent page load for any visitor, confirmed via direct database query and
  successful alert execution.
- **Notable finding during testing:** A global Content-Security-Policy header
  (`script-src 'self'`), left over from unrelated prior CSP-module testing in
  `/etc/apache2/apache2.conf`, was initially blocking inline script execution
  site-wide. This was identified via HTTP header inspection (`curl -I`), traced to
  its source, and disabled to allow proper testing of DVWA's actual (unrelated)
  XSS vulnerability — a useful demonstration of defense-in-depth troubleshooting.
- **Severity:** High
- **Root cause:** No output encoding on stored user input.
- **Mitigation:** HTML-encode all output (`htmlspecialchars()`); apply a
  correctly-scoped CSP that doesn't block legitimate application functionality.

### Finding W-03: Reflected Cross-Site Scripting
- **Location:** `/vulnerabilities/xss_r/`
- **Payload:** `<script>alert('Reflected XSS by Task5')</script>` via the `name`
  URL parameter
- **Result:** Payload executed immediately on page load, confirmed via alert
  popup; no persistence, purely reflected from the request.
- **Severity:** High
- **Root cause:** User input echoed into HTML response without encoding.
- **Mitigation:** Output encoding on all reflected parameters.

### Finding W-04: Cross-Site Request Forgery (CSRF)
- **Location:** `/vulnerabilities/csrf/` (password change form)
- **Proof-of-concept:** A local HTML page with a hidden `<img>` tag targeting the
  password-change endpoint with a new password value, designed to fire silently
  if a logged-in DVWA user opened it.
- **Result:** The forged request reached the server (confirmed via Apache access
  log) but returned HTTP 302 (redirect to login) instead of the HTTP 200 a valid
  authenticated request receives. Investigation via `curl -I` on the login
  endpoint revealed the session cookie is set with `SameSite=Strict`, which
  browsers refuse to attach to any cross-origin request — including this PoC's
  cross-origin image load from a `file://` context.
- **Severity:** Medium (application-level vulnerability present; naive exploitation
  blocked by an orthogonal browser-level control)
- **Root cause:** DVWA implements no CSRF token at its Low security setting.
- **Mitigation:** Implement server-side CSRF tokens; do not rely on cookie
  SameSite policy alone, as it does not protect against same-site attack vectors
  or older/misconfigured clients.

### Finding W-05: Local File Inclusion
- **Location:** `/vulnerabilities/fi/`
- **Payload:** `?page=../../../../../../etc/passwd`
- **Result:** Full contents of the system's `/etc/passwd` file disclosed, confirmed
  by the presence of the actual Kali user account entry in the output.
- **Severity:** High
- **Note:** Remote File Inclusion (RFI) was not exploitable, as PHP's
  `allow_url_include` is disabled on this environment — a useful contrast showing
  LFI succeeding while RFI is blocked by a platform-level configuration setting.
- **Root cause:** User-controlled `page` parameter passed directly into a PHP
  `include()` call with no path sanitization.
- **Mitigation:** Allowlist valid page values; strip directory traversal
  sequences; use `basename()` before any dynamic include.

## 12. Vulnerability Summary Table

| ID | Vulnerability | Severity | Component | Status |
|---|---|---|---|---|
| V-01 | UnrealIRCd backdoor | Critical | Metasploitable2 | RESOLVED |
| V-02 | distcc RCE | High | Metasploitable2 | RESOLVED |
| W-01 | SQL Injection | Critical | DVWA | Documented (training app) |
| W-02 | Stored XSS | High | DVWA | Documented (training app) |
| W-03 | Reflected XSS | High | DVWA | Documented (training app) |
| W-04 | CSRF | Medium | DVWA | Mitigated (partial, incidental) |
| W-05 | Local File Inclusion | High | DVWA | Documented (training app) |
## 13. Mini-SIEM Architecture

A functional SIEM was built using the Elastic Stack, deployed as Docker containers
on the Windows host to conserve resources on the resource-constrained Kali VM.

**Components:**
- **Elasticsearch 8.15.0** — log storage and indexing (port 9200)
- **Kibana 8.15.0** — visualization and search interface (port 5601)
- **Filebeat 8.19.20** — log shipper, installed directly on Kali

**Data sources ingested:**
- `/var/log/auth.log` — SSH authentication events (failed/successful logins)
- `/var/log/apache2/access.log` — DVWA/web application request traffic

**Network path:** Filebeat on Kali → Elasticsearch on the Windows host, addressed
via the host's LAN IP (192.168.0.110) rather than `localhost`, since Kali runs as
a separate VM bridged onto the same physical network as the host.

### Dashboards Built
1. **"Task 5 – Auth Activity Overview"** — Failed and Accepted SSH login event
   panels, demonstrating brute-force detection capability.
2. **"Incident Detection – Web & Auth Attack Timeline"** — A time-series bar chart
   showing event volume filtered on web-attack and auth-brute-force indicators,
   visually surfacing two distinct attack-activity spikes corresponding to the
   simulated incident's two phases.

### Verification
End-to-end pipeline functionality was confirmed at multiple points by generating
known test traffic (SSH login attempts, DVWA SQLi/XSS requests) and verifying
their appearance in Kibana's Discover view with correct timestamps and full
request content in the `message` field.

**Operational note:** During testing, an Elasticsearch outage (Docker container
downtime, approx. 14:25-14:44) caused a gap in log ingestion — Filebeat correctly
logged connection-refused errors during this window, and events generated during
the outage were not retroactively captured once the pipeline recovered (Filebeat
does not re-read past file positions once tracked). This was identified via
Filebeat's own systemd journal logs and resolved by restarting the Docker
containers; subsequent traffic was confirmed to ingest correctly. This is
documented here as a genuine operational lesson: **a SIEM's own availability is
itself part of what must be monitored.**

## 14. Security Awareness Phishing Simulation

**Objective:** Demonstrate, in a fully local and harmless environment, how a
credential-phishing page manipulates a user into entering sensitive information,
and teach recognition of the red flags that expose such attacks.

**Design:** A fake "SecureBank" login page was built and served locally on Kali
(`python3 -m http.server`, accessible only within the lab network). The page
includes:
- A persistent banner clearly marking it as a training simulation
- Urgency-based manipulation language ("account suspended in 24 hours")
- A visible red-flags panel identifying each manipulation technique used
- A "Verify Account" button that performs **no real data transmission** — verified
  via browser DevTools showing no network request fires on submission

**Safety compliance:** No real credentials were collected, no real-world targets
were used, and the simulation was never exposed outside the isolated lab network.
Full design rationale, threat model, and lessons-learned documentation is
maintained separately in `phishing-awareness/README.md`.

## 15. Incident Scenario

**Scenario:** Repeated unauthorized login attempts followed by suspicious web
application activity, consistent with an attacker probing multiple attack
surfaces after initial reconnaissance.

- **Incident ID:** INC-2026-T5-001
- **Attacker (simulated):** Kali Linux (192.168.56.10 / bridged LAN IP)
- **Targets:** DVWA (co-located web application), Kali's own SSH service
- **Initial access vector:** SQL Injection and XSS probing against DVWA, followed
  by SSH brute-force attempts
- **Indicators of Compromise:** See Section 11 (Root Cause) findings; SQLi/XSS
  payload signatures in Apache logs; repeated failed SSH authentication followed
  by a successful login from the same source
- **Severity:** High (combined web application compromise + credential
  brute-force success)
- **Business impact (simulated):** Full database compromise (DVWA), persistent
  script execution risk, and successful unauthorized system access via SSH

## 16. Detection

The incident was detected via the Kibana "Incident Detection – Web & Auth Attack
Timeline" dashboard panel, which visualizes ingested log volume over time. Two
distinct spikes are visible in the panel: one corresponding to the SQLi/XSS
traffic against DVWA (~17:25-17:27), and one corresponding to the SSH
brute-force/login pattern generated shortly after. This timeline was used to
reconstruct the attack sequence and informed the incident timeline documented in
Section 3 of the accompanying Post-Incident Report.

**Detection query used:**All timestamps referenced throughout this report are drawn directly from
Elasticsearch-indexed log data and Apache/auth.log entries, not estimated or
reconstructed after the fact.
## 17. Incident Response

Standard IR lifecycle (Identification → Containment → Eradication → Recovery →
Lessons Learned) was executed against the confirmed Metasploitable2 compromise.

### Identification
See Section 16 (Detection). The Kibana dashboard timeline surfaced the attack
activity; correlated with the known exploitation performed in Section 10, this
constituted positive identification of active compromise.

### Containment
At the time formal containment began, no live Metasploit sessions remained active
(both had already dropped). Containment therefore focused on terminating the
underlying vulnerable services still running on the target:Verified via `ps aux | grep -i "distcc\|ircd\|unreal"` showing zero matching
processes.

### Eradication
Root cause of persistence was identified in `/etc/rc.local`, which launched
UnrealIRCd on every system boot (`nohup /usr/bin/unrealircd &`). This entry was
removed, and the binary itself renamed to prevent any residual invocation:
distccd was removed from init startup:
### Recovery & Verification
The exact exploit previously used to compromise the target was re-attempted
post-remediation:
**Result:** `Rex::ConnectionRefused` — confirming the service no longer accepts
connections. An Nmap scan of the previously-open ports independently confirmed
remediation:
**Result:** Both ports show `closed`.

## 18. Remediation Matrix

| Finding | Original Risk | Mitigation | Retest Result | Final Status |
|---|---|---|---|---|
| UnrealIRCd backdoor | Critical | Service killed, autostart removed, binary disabled | Connection refused; port closed | RESOLVED |
| distcc RCE | High | Service killed, removed from init | Port closed | RESOLVED |
| DVWA SQL Injection | Critical | Documented; requires code-level fix | Not retested (intentionally vulnerable app) | ACCEPTED (by design) |
| DVWA Stored/Reflected XSS | High | Documented; requires output encoding | Not retested (intentionally vulnerable app) | ACCEPTED (by design) |
| DVWA LFI | High | Documented; requires input allowlisting | Not retested (intentionally vulnerable app) | ACCEPTED (by design) |
| DVWA CSRF | Medium | Partially mitigated by browser SameSite policy; app-level token fix still recommended | Naive PoC blocked | MITIGATED (partial) |

## 19. Lessons Learned

- **SIEM availability is part of detection capability.** A gap in Elasticsearch
  uptime during this engagement caused real attack traffic to be permanently lost
  before ingestion. Production SIEM deployments must monitor the health of the
  pipeline itself, not only the security events it processes.
- **Persistence must be checked explicitly during eradication.** Killing a running
  malicious process is insufficient without also checking and removing its
  autostart mechanism (`rc.local`, init.d, cron, systemd) — otherwise the same
  compromise would silently return on the next reboot.
- **Platform-level controls can provide unintentional mitigation.** DVWA's CSRF
  vulnerability exists at the application layer with zero code-level protection,
  yet a naive proof-of-concept was blocked by the browser's default
  `SameSite=Strict` cookie policy. This distinction — vulnerable code vs.
  exploitable-in-practice — is important for accurate risk communication and
  should not be mistaken for the vulnerability being "fixed."
- **Defense-in-depth troubleshooting is a real skill.** Diagnosing why a known
  payload (Stored XSS) unexpectedly failed to execute — tracing it to an
  unrelated, leftover Content-Security-Policy header from earlier testing —
  required systematic elimination of variables (checking the database directly,
  inspecting raw HTTP headers, reviewing Apache config) rather than assuming the
  payload itself was wrong.

## 20. Preventive Recommendations

- Apply proper input validation and parameterized queries to eliminate SQL
  Injection risk at the source, rather than relying on detection alone
- Implement output encoding universally to prevent both stored and reflected XSS
- Replace reliance on browser SameSite cookie policy with explicit, server-side
  CSRF tokens
- Allowlist and sanitize any user-controlled input used in file inclusion or path
  operations
- Monitor SIEM pipeline component health (Elasticsearch, Kibana, Filebeat) as a
  first-class operational concern
- Regularly audit system startup mechanisms (rc.local, init.d, systemd units,
  cron) for unauthorized or unnecessary persistence

## 21. Conclusion

This capstone project successfully integrated web application security testing,
network vulnerability assessment, SIEM development, phishing awareness training,
and full-lifecycle incident response into a single cohesive engagement. All
findings are backed by real, reproducible evidence — command outputs, log data,
and screenshots — rather than theoretical description. The confirmed compromise
on Metasploitable2 was fully contained, eradicated, and independently verified as
remediated, demonstrating the complete security lifecycle from initial
exploitation through confirmed recovery.

## 22. References

- OWASP Top 10: https://owasp.org/www-project-top-ten/
- CVE-2010-2075 (UnrealIRCd backdoor)
- CVE-2004-2687 (distcc RCE)
- DVWA Project: https://github.com/digininja/DVWA/
- Elastic Stack Documentation: https://www.elastic.co/guide/
## Appendix A: Screenshot & Evidence Manifest

| Filename | Description |
|---|---|
| IMG-004-nmap-scan.png | Full Nmap scan results against Metasploitable2 |
| IMG-006-unrealircd-exploit.png | UnrealIRCd exploit output, root meterpreter session |
| IMG-006b-distcc-exploit.png | distcc exploit output, daemon-level shell |
| IMG-006-sqli-test.png | SQLi authentication bypass result |
| IMG-006b-sqli-hashes.png | SQLi UNION query, extracted password hashes |
| IMG-007-stored-xss.png | Stored XSS alert execution |
| IMG-007b-reflected-xss.png | Reflected XSS alert execution |
| IMG-008-csrf-exploit.png / Apache log evidence | CSRF PoC request and SameSite-blocked result |
| IMG-009-lfi-test.png | LFI /etc/passwd disclosure |
| IMG-008-elk-dashboard-init.png | Initial Kibana/Elasticsearch verification |
| IMG-009-security-event.png | First real SSH event visible in Kibana Discover |
| IMG-010 (dashboard) | "Task 5 – Auth Activity Overview" dashboard |
| IMG-010b (dashboard) | "Incident Detection – Web & Auth Attack Timeline" dashboard |
| IMG-011-phishing-sim.png | Phishing awareness simulation page |
| IMG-011-containment.png | Process termination verification (ps aux) |
| IMG-012-eradication.png | Cleaned /etc/rc.local, confirming persistence removed |
| IMG-013-retest.png | Post-remediation exploit failure + Nmap closed-port confirmation |

**Note:** Screenshots are currently split between the Windows host and the Kali
VM and require consolidation into a single evidence folder prior to final
submission.

---

*End of Report*
