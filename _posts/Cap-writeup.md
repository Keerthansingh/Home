---
title: "Cap - HackTheBox Writeup"
date: 2026-09-10
categories: [ctf]
tags: [hackthebox, linux, idor, pcap, capabilities]
---

Hey! Let’s break down **Cap**, an Easy Linux machine on HackTheBox that revolves around insecure direct object references (IDOR), cleartext credential exposure in PCAP files, and dangerous binary capabilities.

---

## 1. Reconnaissance

I started off running a standard Nmap scan against the target IP to see what ports and services were running:

```bash
nmap -sVC -p- 10.129.93.39
```
 ![Nmap scan](/ctf-writeups/assets/img/1.jpg)
 
The scan came back with **3 open TCP ports**:
* **Port 21**: FTP (`vsftpd 3.0.3`)
* **Port 22**: SSH (`OpenSSH 8.2p1`)
* **Port 80**: HTTP running a Gunicorn web server titled "Security Dashboard"

---

## 2. Enumeration & Finding the IDOR

Popping open the web browser to `http://10.129.93.39`, I landed on the Security Dashboard. Clicking around the sidebar, I triggered a feature called "Security Snapshot," which ran a quick packet capture and redirected my browser to `http://10.129.93.39/data/1`. 

![Security Snapshot sidebar option](/ctf-writeups/assets/img/2.png)

Noticing the `/1` at the end of the URL, I wondered if it was vulnerable to IDOR. I manually changed the URL path to `/data/0`. Sure enough, it loaded another user's private scan report containing 72 captured packets! The application didn't validate whether the logged-in user actually owned the requested ID, letting me browse other users' scans freely.

![Accessing another  /data/0](/ctf-writeups/assets/img/3.png)

---

## 3. Exploitation & Getting a Shell

Inside the scan data at `/data/0`, there was an option to download a packet capture file named `0.pcap`. I pulled it down and opened it up in Wireshark to see what was hidden inside.

![FTP USER and PASS captured in Wireshark](/ctf-writeups/assets/img/4.jpg)

![Wireshark stream confirming the credentials](/ctf-writeups/assets/img/5.jpg)

Filtering through the stream, I spotted unencrypted **FTP** traffic. Following the TCP stream revealed a login attempt where user `nathan` sent their password in cleartext:
* **Username**: `nathan`
* **Password**: `Buck3tH4TFORM3!`

Since port 22 (SSH) was open, I tested these credentials there. Logging in via SSH worked instantly:

```bash
ssh nathan@10.129.93.39
```

Once inside, I grabbed the user flag right away:
* **User Flag**: `7074363d194f7b877e87a5cee7c52499`

![SSH access and reading user.txt](/ctf-writeups/assets/img/6.jpg)

---

## 4. Privilege Escalation

To figure out how to escalate to root, I checked for files with special Linux capabilities assigned to them using `getcap`:

```bash
getcap -r / 2>/dev/null
```
![getcap output showing python3.8 capabilities](/ctf-writeups/assets/img/7.png)

The output flagged something very interesting: `/usr/bin/python3.8` had the `cap_setuid` capability enabled (`cap_setuid,cap_net_bind_service+eip`). This means the python binary can change its user ID during execution, effectively allowing a normal user to spawn a root-level process.

I leveraged this capability to pop a root shell with a quick one-liner:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

Running `whoami` confirmed I was running as `root`. I navigated to the root directory and grabbed the final flag:

![Root shell and reading root.txt](/ctf-writeups/assets/img/8.png)

* **Root Flag**: `33100bb86f58e26315ba29edc2e8d86e`
