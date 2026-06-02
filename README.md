# TryHackMe - Wgel CTF Write-up

## Introduction

The objective of this room was to gain initial access to the target machine through web enumeration and then perform privilege escalation to obtain the root flag.

This challenge focuses on reconnaissance, directory enumeration, SSH authentication using a leaked private key, and privilege escalation through misconfigured sudo permissions.

---

# Reconnaissance

The first step was performing an Nmap scan to identify open ports and running services.

```bash
nmap 10.66.XXX.XXX -sS -sV -sC
```

<br>

<img width="804" height="310" alt="image" src="https://github.com/user-attachments/assets/19407278-58b1-41c1-8ef8-26723b22df91" />

<br><br>

After analyzing the results, I identified two interesting services:

* SSH running on port 22
* Apache HTTP Server running on port 80

With this information, I decided to investigate the web application first.

---

# Web Enumeration

Accessing the web server revealed the default Apache2 Ubuntu page.

<br>

<img width="844" height="817" alt="image" src="https://github.com/user-attachments/assets/5d0b6cc5-d95a-4557-85a7-dc92776c0e68" />

<br><br>

Since default pages often hide additional content, I performed directory enumeration against the web server.

<br>

<img width="995" height="407" alt="image" src="https://github.com/user-attachments/assets/636d7806-277e-4d7a-a517-7c22cb0d23bd" />

<br><br>

During the enumeration, I discovered a `/sitemap` directory.

Navigating to this location revealed a web application containing multiple pages and navigation sections. I manually explored the website and gathered useful information such as employee names, email addresses, and other exposed data.

Although this information was not immediately useful, documenting all discovered information is an important part of the reconnaissance phase.

After manually investigating the website, I decided to perform another round of directory enumeration, this time targeting the `/sitemap` directory specifically.

<br>

<img width="1485" height="769" alt="image" src="https://github.com/user-attachments/assets/10fbb633-fb42-46ea-b9e6-05f6ac304771" />

<br><br>

This enumeration revealed an interesting directory:

```text
/.ssh
```

<br>

<img width="995" height="482" alt="image" src="https://github.com/user-attachments/assets/84433da5-7c1d-4f88-b243-00dcd23c8f5c" />

<br><br>

Inside this directory, I found an exposed private SSH key named:

```text
id_rsa
```

<br>

<img width="1846" height="687" alt="image" src="https://github.com/user-attachments/assets/0a557545-d2e9-475c-b4ea-36a8326e49a9" />

<br><br>

Opening the file revealed a complete RSA private key.

<br>

<img width="707" height="560" alt="image" src="https://github.com/user-attachments/assets/eae4c1a8-f7b2-45f0-b09e-382eb2cea2a1" />

<br><br>

---

# Initial Access

To use the discovered key, I created a local file containing its contents.

<br>

<img width="265" height="56" alt="image" src="https://github.com/user-attachments/assets/e20bcbc1-8851-4d8e-9497-0cea68caf132" />

<br><br>

Next, I adjusted the file permissions to ensure SSH would accept the key.

```bash
chmod 600 id_rsa
```

<br>

<img width="277" height="60" alt="image" src="https://github.com/user-attachments/assets/a118d1dc-f2b5-4c1d-ae16-14719b171ce2" />

<br><br>

At this point, I still needed to identify the correct username.

Initially, I attempted to authenticate using dev names gathered from the website, but none were successful.

I then revisited the web pages and inspected their source code. While reviewing the source code of the default Apache page, I discovered a comment containing the username:

```text
jessie
```

<br>

<img width="618" height="215" alt="image" src="https://github.com/user-attachments/assets/9a07213b-64a9-4366-b75a-1f8d7cfa78d3" />

<br><br>

Using the leaked private key together with the discovered username, I successfully authenticated via SSH.

```bash
ssh -i id_rsa jessie@TARGET_IP
```

<br>

<img width="546" height="221" alt="image" src="https://github.com/user-attachments/assets/e89e4d85-3acb-48d5-bff2-c761a96442a4" />

<br><br>

Once inside the system, I explored the user's files and found the user flag.

```text
user_flag.txt
```

<br>

<img width="742" height="131" alt="image" src="https://github.com/user-attachments/assets/6632a351-60e0-449e-acfe-51eba348cea3" />

<br><br>

---

# Privilege Escalation

The room also requires obtaining the root flag.

To identify potential privilege escalation vectors, I checked the user's sudo permissions.

```bash
sudo -l
```

<br>

<img width="958" height="118" alt="image" src="https://github.com/user-attachments/assets/6e8d21d7-0401-4839-a94a-7407aff5ffc4" />

<br><br>

The output revealed the following privilege:

```text
(root) NOPASSWD: /usr/bin/wget
```

This means the user can execute `wget` as root without providing a password.

To understand how this permission could be abused, I searched GTFOBins and found a technique that allows `wget` to upload the contents of privileged files to an attacker-controlled system.

<br>

<img width="864" height="248" alt="image" src="https://github.com/user-attachments/assets/30be51ac-5f1c-4c3f-830c-9a9256d5667f" />

<br><br>

A listener was configured on the attacking machine:

```bash
nc -lvnp 4444
```

Then, from the target machine, `wget` was used to send the contents of the root flag file to the attacker's listener.

<br>

<img width="766" height="79" alt="image" src="https://github.com/user-attachments/assets/e1238a73-97cf-4a0c-8ed2-7f5ad5d312ba" />

<br><br>

The listener successfully received the contents of the file.

<br>

<img width="739" height="250" alt="image" src="https://github.com/user-attachments/assets/fbe14b33-44eb-414e-8005-c1a43e255f4b" />

<br><br>

With the contents of the root flag obtained, the challenge was successfully completed.

---

# Conclusion

This room provided a great demonstration of how exposed files on a web server can lead to full system compromise.

The attack chain consisted of:

1. Enumerating the web server.
2. Discovering the `/sitemap` directory.
3. Finding an exposed SSH private key.
4. Identifying the valid username through source code analysis.
5. Gaining SSH access.
6. Exploiting a misconfigured sudo permission on `wget`.
7. Retrieving the root flag.

The room highlights the importance of securing sensitive files, restricting web-accessible directories, and carefully reviewing sudo permissions.
