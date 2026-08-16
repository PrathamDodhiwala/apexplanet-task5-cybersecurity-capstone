# Post-Incident Report — Task 5 Capstone
## Integrated Web Security Assessment, Vulnerability Management, Mini-SIEM & Incident Response Lab

## 1. Incident Summary
A simulated multi-stage attack was conducted against the lab environment, combining
web application exploitation (DVWA) and network-level exploitation (Metasploitable2).
The attack was detected via a Filebeat-to-Elasticsearch-to-Kibana SIEM pipeline,
contained, eradicated, and formally verified as resolved.

## 2. Executive Summary
Two remote code execution vulnerabilities (UnrealIRCd backdoor, distcc RCE) were
identified and exploited on Metasploitable2 (192.168.56.101), yielding root and
unprivileged shell access respectively. In parallel, DVWA (co-located on the Kali
attacker host) was found vulnerable to SQL Injection, Stored XSS, Reflected XSS,
Local File Inclusion, and (at the application layer) CSRF. All findings were
detected in near-real-time via a custom-built ELK-based SIEM, and the confirmed
compromise on Metasploitable2 was fully contained, eradicated, and verified
remediated.

## 3. Incident Timeline
| Time (IST) | Event |
|---|---|
| Earlier session | UnrealIRCd 3.2.8.1 backdoor exploited on 192.168.56.101, root meterpreter session obtained |
| Earlier session | distcc RCE exploited on 192.168.56.101, shell obtained as uid=1(daemon) |
| 17:25 | SQL Injection (row bypass + UNION extraction) and Reflected XSS traffic generated against DVWA, captured in Apache access.log |
| 17:27 | Additional SQLi/XSS traffic, captured and shipped to Elasticsearch |
| ~17:5x | SSH brute-force pattern (2 failed + 1 successful login) generated against Kali (192.168.56.10), captured in auth.log |
| Detection | Kibana dashboard "Incident Detection – Web & Auth Attack Timeline" shows two distinct event spikes corresponding to the web and auth attack phases |
| Containment | Live processes for distccd (4 workers) and unrealircd killed via pkill on Metasploitable2 |
| Eradication | unrealircd autostart entry removed from /etc/rc.local; binary renamed to unrealircd.disabled; distccd removed from init via update-rc.d |
| Recovery | UnrealIRCd exploit re-attempted — result: Connection refused. Nmap confirms ports 3632 and 6667 now closed |

## 4. Detection
Detection was achieved via a custom mini-SIEM: Filebeat (installed on Kali) shipping
both `/var/log/auth.log` and `/var/log/apache2/access.log` to a Dockerized
Elasticsearch + Kibana stack running on the host machine. A Kibana Lens dashboard
panel visualizes event volume over time, filtered on indicators of the attack
(`message: "sqli"`, `message: "xss_r"`, `message: "Failed password"`), showing two
clear activity spikes corresponding to the web-attack and auth-brute-force phases.

## 5. Initial Analysis
- **Web vector:** DVWA's SQL Injection, XSS, and LFI vulnerabilities stem from
  unsanitized user input reaching database queries, HTML output, and file-include
  calls respectively, at DVWA's "Low" security setting.
- **Network vector:** Metasploitable2 exposed two remotely exploitable services —
  UnrealIRCd 3.2.8.1 (trojaned backdoor, CVE-2010-2075) and distccd (unauthenticated
  RCE, CVE-2004-2687) — both confirmed exploitable via Metasploit modules.

## 6. Impact
- Full database compromise on DVWA (usernames + password hashes extracted via SQLi)
- Persistent script execution capability on DVWA (Stored XSS)
- Local file disclosure on DVWA (LFI, confirmed via /etc/passwd read)
- Root-level remote code execution on Metasploitable2 (UnrealIRCd)
- Unprivileged remote code execution on Metasploitable2 (distcc), representing a
  potential pivot point for further privilege escalation

## 7. Containment
Live sessions on Metasploitable2 had already dropped by the time formal containment
began; containment therefore focused on actively terminating the vulnerable services
still running: `pkill -f distccd` and `pkill -f unrealircd`, verified via `ps aux`
showing zero matching processes.

## 8. Eradication
Root cause of persistence was traced to `/etc/rc.local`, which launched UnrealIRCd
on every boot (`nohup /usr/bin/unrealircd &`). This line was removed via `sed`, and
the binary itself renamed to `/usr/bin/unrealircd.disabled` to prevent any residual
invocation. distccd was removed from init startup via `update-rc.d -f distccd remove`.

## 9. Recovery
The exact UnrealIRCd exploit that previously yielded a root shell was re-run against
the target post-remediation. Result: `Rex::ConnectionRefused`, confirming the service
is no longer listening. An Nmap scan of the previously-open ports (3632, 6667)
confirms both now show `closed`.

## 10. Root Cause
- **DVWA findings:** Application-level input validation failures (no prepared
  statements, no output encoding, no path sanitization on includes), present by
  design at DVWA's Low security setting.
- **Metasploitable2 findings:** Deliberately outdated, vulnerable service versions
  (UnrealIRCd 3.2.8.1 trojaned build, distcc with no authentication) present by
  design as an intentionally vulnerable training image.

## 11. Indicators of Compromise (IOCs)
- SQLi payloads containing `' OR '1'='1` or `UNION SELECT` in Apache access logs
- XSS payloads containing `<script>` tags in request parameters
- Path traversal sequences (`../`) in the `page` parameter on `/vulnerabilities/fi/`
- Connections to TCP/6667 (IRC) and TCP/3632 (distcc) on Metasploitable2
- Repeated `Failed password` entries in auth.log followed by an `Accepted password`
  entry from the same source IP within a short window (brute-force pattern)

## 12. Evidence
- Kibana dashboards: "Task 5 – Auth Activity Overview", "Incident Detection – Web &
  Auth Attack Timeline"
- Nmap scan: nmap_metasploitable2_full.txt
- Screenshots: IMG-004 through IMG-013 (see screenshot checklist)
- Terminal session evidence for all exploitation, containment, eradication, and
  recovery steps (documented throughout this engagement)

## 13. Remediation Matrix
| Finding | Original Risk | Mitigation Applied | Retest Result | Status |
|---|---|---|---|---|
| UnrealIRCd 3.2.8.1 backdoor | Critical | Service killed, autostart removed, binary disabled | Connection refused, port closed | RESOLVED |
| distcc unauthenticated RCE | High | Service killed, removed from init | Port closed | RESOLVED |
| DVWA SQL Injection | Critical | Documented; requires prepared statements (code fix, not applied to intentionally-vulnerable app) | N/A — training app | ACCEPTED (by design) |
| DVWA Stored/Reflected XSS | High | Documented; requires output encoding | N/A — training app | ACCEPTED (by design) |
| DVWA LFI | High | Documented; requires path allowlisting | N/A — training app | ACCEPTED (by design) |
| DVWA CSRF | Medium | Mitigated incidentally by browser SameSite=Strict cookie policy; app-level fix (CSRF tokens) still recommended | Naive PoC blocked | MITIGATED (partial) |

## 14. Lessons Learned
- A SIEM pipeline is only as good as its uptime — a gap in Elasticsearch availability
  during this engagement caused real attack traffic to be lost before ingestion,
  underscoring the need for monitoring the monitoring stack itself.
- Modern browser defaults (SameSite cookie policies) can provide unintentional but
  real mitigation against classic attack PoCs, which is worth accounting for when
  assessing real-world exploitability versus theoretical vulnerability presence.
- Persistence mechanisms (rc.local, init.d entries) must be checked explicitly during
  eradication — killing a running process alone is insufficient if the service will
  simply restart on reboot.

## 15. Preventive Recommendations
- Implement prepared statements and output encoding across all DVWA-style
  vulnerabilities if this were a production application
- Apply CSRF tokens rather than relying on browser-level cookie policy as a
  de facto mitigation
- Monitor SIEM pipeline health itself (Elasticsearch/Kibana uptime) as part of the
  detection capability, not just the data it processes
- Regularly audit startup scripts (rc.local, systemd units, cron) for unauthorized
  or unnecessary service persistence
