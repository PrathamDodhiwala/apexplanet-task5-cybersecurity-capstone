# ApexPlanet Task 5 — Cybersecurity Capstone

## Integrated Web Security Assessment, Vulnerability Management, Mini-SIEM & Incident Response Lab

Capstone project for the ApexPlanet Cybersecurity & Ethical Hacking Internship,
Task 5 (Days 49-60): Capstone Project & Incident Response.

## Overview

This project combines four disciplines into one integrated engagement: web
application penetration testing (DVWA), network vulnerability assessment
(Metasploitable2), a custom-built mini-SIEM (ELK stack), and a full incident
response lifecycle — detection through verified recovery.

## Objectives

- Conduct a controlled web application penetration test against DVWA
- Perform a network vulnerability assessment against Metasploitable2
- Build a functional mini-SIEM using the Elastic Stack
- Create a local, harmless phishing awareness simulation
- Simulate, detect, and formally respond to a security incident

## Architecture

- **Kali Linux** (192.168.56.10) — attacker platform, hosts DVWA
- **Metasploitable2** (192.168.56.101) — vulnerable target
- **Elasticsearch + Kibana** — Docker containers on the host machine
- **Filebeat** — log shipper on Kali, ingests auth.log and Apache access.log

Full network diagram and lab setup details are in `reports/capstone-final-report.md`.

## Tools

Nmap, Metasploit Framework, DVWA, Elastic Stack (Elasticsearch, Kibana,
Filebeat), John the Ripper (referenced from prior task).

## Findings Summary

| Finding | Component | Severity | Status |
|---|---|---|---|
| UnrealIRCd 3.2.8.1 backdoor (CVE-2010-2075) | Metasploitable2 | Critical | Resolved |
| distcc RCE (CVE-2004-2687) | Metasploitable2 | High | Resolved |
| SQL Injection | DVWA | Critical | Documented |
| Stored XSS | DVWA | High | Documented |
| Reflected XSS | DVWA | High | Documented |
| CSRF | DVWA | Medium | Partially mitigated |
| Local File Inclusion | DVWA | High | Documented |

Full details, evidence, and remediation steps for every finding are in
`reports/capstone-final-report.md`.

## Mini-SIEM

Built with Elasticsearch + Kibana (Docker) and Filebeat (Kali), ingesting SSH
auth logs and Apache access logs. Dashboards include an auth activity overview
and a combined incident detection timeline. See `siem/` for configuration
notes.

## Incident Response

A simulated incident (web + SSH brute-force attack pattern) was detected via
the SIEM, formally contained and eradicated on Metasploitable2, and
independently verified as resolved via retest. Full timeline and lifecycle
documentation: `incident-response/reports/post-incident-report.md`.

## Phishing Awareness Simulation

A local, harmless simulation demonstrating common phishing red flags. No real
credentials are ever collected. See `phishing-awareness/`.

## Project Structure
├── reports/ # Final capstone report
├── incident-response/ # Post-incident report, timeline, evidence
├── nmap/ # Network scan output
├── web-pentest/ # DVWA findings and notes
├── siem/ # ELK configuration notes
├── phishing-awareness/ # Phishing simulation page + docs
├── diagrams/ # Network/architecture diagrams
├── screenshots/ # Evidence screenshots
└── scripts/ # Supporting scripts
## Ethical & Legal Disclaimer

All testing in this project was conducted exclusively against systems owned
and controlled by the author (DVWA, Metasploitable2) within an isolated,
non-internet-facing lab network. No real-world targets, individuals, or
production systems were involved at any stage. This repository is for
educational purposes only.

## Results

Two critical/high-severity network vulnerabilities were confirmed, exploited,
and fully remediated with independent verification. Five web application
vulnerabilities were identified and documented with proof-of-concept evidence.
A functional SIEM successfully detected the simulated incident in near
real-time.

## Future Improvements

- Consolidate screenshots currently split across host/guest machines into a
  single evidence archive
- Extend SIEM detection rules into automated Kibana alerting
- Add Logstash for more advanced log parsing/enrichment
