# Web Application Pentesting — Writeups

A collection of writeups from TryHackMe rooms focused on web application exploitation, covering RCE, privilege escalation, and post-exploitation across both Linux and Windows targets.

## Index

| Room | Platform | Difficulty | Focus Areas | Writeup |
|------|----------|------------|-------------|---------|
| Smol | TryHackMe | Medium | WordPress plugin SSRF/LFI, credential extraction, RCE via Hello Dolly backdoor, sudo misconfiguration | [smol.md](./smol.md) |
| Cheese CTF | TryHackMe | Medium | PHP filter chain RCE, writable `authorized_keys`, systemd timer + GTFOBins privesc | [cheese-ctf.md](./cheese-ctf.md) |
| Rabbit Store | TryHackMe | Medium | JWT manipulation, mass assignment, SSTI → RCE, RabbitMQ/Erlang cookie exploitation | [rabbit-store.md](./rabbit-store.md) |
| You Got Mail | TryHackMe | Medium | SMTP enumeration, password spraying, phishing, Meterpreter, Windows hash cracking | [you-got-mail.md](./you-got-mail.md) |

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

## Disclaimer

These writeups document personal practice on intentionally vulnerable machines hosted on TryHackMe, for educational and skill-development purposes only.