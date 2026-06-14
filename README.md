# Web Application Pentesting — Writeups & Reports

A collection of writeups and reports from web application security testing, spanning TryHackMe CTF rooms and a structured penetration test report against a custom e-commerce application. Covers RCE, privilege escalation, and post-exploitation across Linux and Windows targets, plus formal vulnerability documentation (severity, CWE, OWASP mapping, PoC, remediation).

## Index

| Room | Platform | Difficulty | Focus Areas | Writeup |
|------|----------|------------|-------------|---------|
| Smol | TryHackMe | Medium | WordPress plugin SSRF/LFI, credential extraction, RCE via Hello Dolly backdoor, sudo misconfiguration | [smol.md](./smol.md) |
| Cheese CTF | TryHackMe | Medium | PHP filter chain RCE, writable `authorized_keys`, systemd timer + GTFOBins privesc | [cheese-ctf.md](./cheese-ctf.md) |
| Rabbit Store | TryHackMe | Medium | JWT manipulation, mass assignment, SSTI → RCE, RabbitMQ/Erlang cookie exploitation | [rabbit-store.md](./rabbit-store.md) |
| You Got Mail | TryHackMe | Medium | SMTP enumeration, password spraying, phishing, Meterpreter, Windows hash cracking | [you-got-mail.md](./you-got-mail.md) |
| Hack Smarter E-Commerce | Custom Lab | Medium/High | Full pentest report \u2014 broken access control, IDOR, stored XSS, SQL injection, unrestricted file upload to RCE | [hack-smarter-ecommerce.md](./hack-smarter-ecommerce.md) |

## Summary of Techniques Covered

- Web enumeration (dirsearch, rustscan, nmap)
- Custom wordlist generation (John, CeWL)
- Authentication brute-forcing and password spraying (ffuf, hydra)
- SSRF / LFI via `php://filter`
- Server-Side Template Injection (SSTI) leading to RCE
- Mass assignment and JWT manipulation
- Unrestricted file upload → RCE
- Phishing-based initial access (Metasploit, msfvenom)
- Windows hash extraction and offline cracking (hashdump, John, Hashcat)
- Linux privilege escalation (sudo misconfiguration, systemd timers, GTFOBins, SSH key persistence)
- Service-specific exploitation (RabbitMQ/Erlang, hMailServer)
- Business logic flaws and broken access control (client-side restriction bypass)
- IDOR-based privilege escalation
- Stored XSS and session hijacking
- SQL injection and database enumeration via UNION-based extraction

## Disclaimer

These writeups and reports document personal practice on intentionally vulnerable machines and lab environments (TryHackMe and custom-built applications), for educational and skill-development purposes only.