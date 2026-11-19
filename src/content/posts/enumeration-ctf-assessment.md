---
title: Enumeration - CTF Assessment
published: 2026-11-19
tags: [eJPT, CTF, Enumeration]
category: CTF
---

EJPT Walkthrough: Assessment Methodologies – Enumeration CTF 1 

**Challenge Overview**

In this lab from the eJPT **Assessment Methodologies: Enumeration** module, we are given access to a Linux target machine at `target.ine.local`. Our objective is to enumerate the running services, find misconfigurations, and capture the flags (formatted as MD5 hashes).

**Objective for Flag 1:**

> *"There is a samba share that allows anonymous access. Wonder what's in there!"*

---

**Step 1: Network Scanning with Nmap**

**What is Nmap?**

**Nmap (Network Mapper)** is an open-source network scanning tool used to discover hosts, open ports and running services on a network. Think of it as knocking on every door of a building to see which ones are unlocked and who answers behind them.

**Initial Port Scan**

First, I ran a basic Nmap scan against the target to see which TCP ports were open:

nmap target.ine.local

**Results:**

- **Port 22 (SSH):** Secure Shell, used for remote command line login.
- **Port 139 (NetBIOS-SSN):** Used for file and printer sharing over older Windows/Samba networking protocols.
- **Port 445 (Microsoft-DS / SMB):** Used for modern Server Message Block (SMB) file sharing directly over TCP/IP.

**Service Version Enumeration**

Seeing ports **139** and **445** open immediately pointed toward **Samba**. To confirm the exact service and version running on port 445, I ran a targeted scan.

nmap -sV -p 445 target.ine.local

**Flags Explained:**

- `-sV`: **Service Version Detection.** Nmap probes the open port to determine the exact software name and version number.
- `-p 445`: **Port Specification.** Tells Nmap to scan only port 445 instead of the default top 1,000 ports, making the scan much faster.

The output confirmed the target was running **Samba smbd 4.6.2** on a Linux (Ubuntu) host.

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/cec8bc58-6762-48c9-8b60-b242c9454d88.png align="center")

**Step 2: Understanding Samba & Checking Shares with** `smbclient`

**What is Samba?**

**Samba** is an open-source implementation of the **SMB (Server Message Block)** protocol. It allows Linux and Unix systems to share files and printers seamlessly with Windows machines (and other Linux clients) over a network, typically operating over **TCP ports 139 and 445**.

**Listing Shares with** `smbclient`

`smbclient` is a command-line FTP-like client used to communicate with SMB/Samba servers. Since the hint mentioned anonymous access, I attempted to list all available network shares without providing a password:

smbclient -L target.ine.local -N

**Flags Explained:**

- `-L`: **List Shares.** Asks the server to display all available shared folders and resources.
- `-N`: **No Password.** Suppresses the password prompt so we can test for anonymous (null session) access.

**What Happened:**\
The output showed two default administrative shares (`print$` and `IPC$`), along with the server banner (`Samba, Ubuntu`). However, no custom user share showed up in the standard listing—which often means a share is **hidden** (or browseable is set to `no` in the Samba configuration).

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/b79f72d3-1791-4b73-a973-2231fa018505.png align="center")

**Step 3: Brute-Forcing Hidden Shares with** `enum4linux`

**What is** `enum4linux`**?**

`enum4linux` is a comprehensive Linux enumeration tool built specifically for extracting information from Windows and Samba systems. It automates queries for usernames, group memberships, workgroups, password policies, and shared folders.

**Discovering the Hidden Share**

Because `smbclient -L` did not show any custom shares, I used `enum4linux` with a wordlist to brute-force potential hidden share names:

enum4linux -s /root/Desktop/wordlists/shares.txt target.ine.local

Flags Explained: • -s : Share Brute-Forcing. Tells enum4linux to test every name inside the specified wordlist (shares.txt) against the target to see if a hidden share exists and accepts connections. Results:

1. The server confirmed it allows null sessions (username '', password '').
2. Under the Brute Force Share Enumeration section, enum4linux successfully discovered a hidden share named pubfiles that allows anonymous access!

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/1a5d4df5-5552-43ec-a5c9-aec9904b68f6.png align="center")

**What's Next?**

Now that we have confirmed the hidden share `pubfiles` exists and permits anonymous login, the next step is to connect directly to `//target.ine.local/pubfiles` using `smbclient` and retrieve **Flag 1**.

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/ce4f20bb-5f56-4e0a-b117-0e1db953b8f4.png align="center")

---

**Flag 2: User Enumeration & SMB Password Brute-Forcing**

**Objective for Flag 2:**

> *"One of the samba users have a bad password. Their private share with the same name as their username is at risk!"*

---

**Step 1: Enumerating Samba Users via RID Cycling**

Before we can test for weak passwords, we first need to find out which valid user accounts exist on the target machine. To do this, I used `enum4linux` with the `-r` flag.

enum4linux -r target.ine.local\\

Use code with caution.

**Flag & Concept Explained:**

- `-r`: **RID Cycling.** Every user and group on a Windows or Samba system is assigned a unique number called a **Relative Identifier (RID)** (for example, `1000`, `1001`, `1002`). Even when anonymous user listing is blocked, tools like `enum4linux` can cycle through a range of RIDs (`500–550` and `1000–1050` by default) to uncover valid usernames.

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/31ff2375-51ad-4e98-9995-ff42da9168db.png align="center")

**Discovered Users:**\
Looking at the output under the Unix User SID (`S-1-22-1`), `enum4linux` revealed four local user accounts on the system:

- `josh` (RID `1000`)
- `bob` (RID `1001`)
- `nancy` (RID `1002`)
- `alice` (RID `1003`)

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/16a936e3-f1b3-461f-8506-438cb2e393ae.png align="center")

---

**Step 2: Preparing Wordlists**

Next, I checked the `/root/Desktop/wordlists/` directory to see what password lists were available on the attack machine:

ls /root/Desktop/wordlists/\\

Use code with caution.

The folder contained `unix_passwords.txt`. I saved the four discovered usernames (`josh`, `bob`, `nancy`, and `alice`) into a custom file named `name.txt` inside the same directory so I could run an automated login attack.

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/3e2d7dbd-75d3-40ff-ba4c-5b3184c3bcb9.png align="center")

**Step 3: Brute-Forcing SMB Credentials with Metasploit**

**What is Metasploit?**

**Metasploit Framework (**`msfconsole`**)** is one of the most widely used penetration testing platforms. It includes thousands of pre-built modules for scanning, enumeration, brute-forcing, and exploitation.

To find which user had a weak password, I loaded Metasploit's SMB login scanner module.

use auxiliary/scanner/smb/smb_login

**Configuring Module Options**

I ran `show options` to view the required settings and configured the module with the following parameters.

set RHOSTS 192.100.91.3

set USER_FILE /root/Desktop/wordlists/name.txt

set PASS_FILE /root/Desktop/wordlists/unix_passwords.txt

run

Options Explained:

- `RHOSTS`: **Remote Hosts.** Sets the IP address of the target machine (`192.100.91.3`).
- `USER_FILE`: **Username Wordlist.** Points to our custom file containing the four enumerated usernames (`josh`, `bob`, `nancy`, `alice`).
- `PASS_FILE`: **Password Wordlist.** Points to `unix_passwords.txt` so Metasploit tests each password against every username.\\

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/cfd4b902-d18a-4e96-8f14-f0bdadb780dc.png align="center")

**Brute-Force Results**

After running the module, Metasploit tested combinations of the usernames and passwords until it found a valid match:

- **Valid Credentials Found:** `josh : purple`

**What's Next?**

With valid credentials (`josh` / `purple`) in hand—and knowing from the hint that the user's private share shares the same name as their username the next step is to log into `//target.ine.local/josh` using `smbclient` and retrieve **Flag 2**.

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/d84a2c29-3830-408a-92df-34368b79673c.png align="center")

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/8d4e8e8c-0e60-4b2e-898a-20f3545ab9ff.png align="center")

**Retrieving Flag 2 & Finding the Next Clue**

After logging into `josh`'s Samba share using the credentials `josh:purple` and downloading `flag2.txt`, I read the file on my local machine:

cat flag2.txt\\

Use code with caution.

Inside `flag2.txt`, we captured **Flag 2** (`FLAG2{c03914a1a70f4b2f9fe79f07b702d69d}`) along with a crucial hint for our next target:

> *"Psst! I heard there is an FTP service running. Find it and check the banner."*

**Flag 3: Full Port Scanning, Banner Grabbing & FTP Brute-Forcing**

**Objective for Flag 3:**

> *"Follow the hint given in the previous flag to uncover this one."*

---

**What is FTP?**

**FTP (File Transfer Protocol)** is a standard network protocol used to transfer files between a client and a server. By default, FTP runs on **TCP port 21**, which did not show up in our initial top-1,000 port scan meaning the administrator likely moved it to a non-standard port.

To find where FTP was hiding, I ran a full all-ports scan with Nmap:

nmap -p- -sV target.ine.local\\

**Flags Explained:**

- `-p-`: **Scan All Ports.** Tells Nmap to scan every single TCP port from **1 to 65,535**, rather than just the default top 1,000 ports.
- `-sV`: **Service Version Detection.** Identifies the exact service and version running on any open ports discovered.

**Results:**\
In addition to ports 22, 139, and 445, Nmap uncovered a hidden service running on a high port:

- **Port 5554 (FTP):** `vsftpd 2.0.8 or later`

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/60e9cbe0-55da-41ac-9a07-efaf16119227.png align="center")

**Step 2: Banner Grabbing with Netcat (**`nc`**)**

**What is Netcat?**

**Netcat (**`nc`**)** is known as the "Swiss Army knife" of networking. It reads and writes data across network connections using TCP or UDP, making it ideal for connecting directly to open ports and reading service welcome messages (known as **banner grabbing**).

Following the hint to *"check the banner,"* I connected to port `5554` using Netcat:

nc -nv 192.100.91.3 5554\\

**Flags Explained:**

- `-n`: **Numeric-Only IP.** Skips DNS resolution, making the connection faster and avoiding DNS errors.
- `-v`: **Verbose Mode.** Displays detailed connection status messages in the terminal.

**The Banner Clue:**

\
As soon as Netcat connected, the server printed a revealing welcome banner:

> `220 Welcome to blah FTP service. Reminder to users, specifically ashley, alice and amanda to change their weak passwords immediately!!!`

This banner handed us three target usernames with weak passwords: `ashley`, `alice`, and `amanda` (and we already knew `alice` was a valid local user on the machine from our earlier `enum4linux` scan!). After checking the banner, I typed `quit` to cleanly close the FTP session.

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/d62dd651-6a7e-4c21-a5d8-239333e20535.png align="center")

**Step 3: Brute-Forcing FTP Credentials with Hydra**

**What is Hydra?**

**THC-Hydra** is a fast, parallelized network login cracker that supports dozens of protocols—including FTP, SSH, SMB, and HTTP. It systematically tests combinations of usernames and passwords against a target service until it finds a valid login.

Using our username list (`name.txt`, which included `alice`) and `unix_passwords.txt`, I launched Hydra against the FTP service on port `5554`:

**bash**

hydra -L /root/Desktop/wordlists/name.txt -P /root/Desktop/wordlists/unix_passwords.txt [ftp://target.ine.local:5554](ftp://target.ine.local:5554)\\

Use code with caution.

**Flags Explained:**

- `-L <file>`: **Login (Username) List.** Specifies the file containing usernames to test (`name.txt`).
- `-P <file>`: **Password List.** Specifies the wordlist of passwords to try (`unix_passwords.txt`).
- [`ftp://target.ine.local:5554`](ftp://target.ine.local:5554): **Target Protocol & Port.** Tells Hydra to use the `ftp` protocol against `target.ine.local` on custom port `5554`.

**Brute-Force Results:**\
Hydra quickly found a valid login for `alice`:

- **Valid FTP Credentials:** `alice : pretty`

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/679178d4-f903-48f9-aa36-e3c28b37d9b6.png align="center")

**Step 4: Retrieving Flag 3**

After logging into the FTP server (`ftp target.ine.local 5554`) as `alice` with the password `pretty` and downloading `flag3.txt`, I checked its contents locally:

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/e69b3c41-9df9-4863-b3ff-d5b038c0f4d0.png align="center")

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/9e70f1aa-6c4c-43c7-95c7-dcd56fa4d728.png align="center")

**Flag 4: SSH Pre-Authentication Banner Enumeration**

**bjective for Flag 4:**

> *"This is a warning meant to deter unauthorized users from logging in."*

---

**Step 1: Connecting the Clue to SSH**

**What is SSH?**

**SSH (Secure Shell)** is a cryptographic network protocol used to securely access and manage remote machines over a command-line interface. By default, SSH operates on **TCP port 22**, which we discovered during our very first Nmap scan.

**Why the Clue Pointed to SSH**

In Linux administration, a message displayed before the password prompt to warn away unauthorized users is known as a **login warning banner** (often configured in `/etc/`[`issue.net`](http://issue.net) and enabled inside `/etc/ssh/sshd_config`). Because we had already enumerated Samba (Flags 1 & 2) and FTP (Flag 3), SSH on **port 22** was the remaining remote login service capable of displaying a pre authentication warning banner.

**Step 2: Triggering the SSH Login Banner**

To view the warning banner, I initiated an SSH connection to the target machine without needing a valid password:

ssh root@target.ine.local\\

**Command Explained:**

- `ssh`: Invokes the OpenSSH client to connect to port `22` on the target host.
- `root@target.ine.local`: Attempts to authenticate as the `root` user on `target.ine.local`, which forces the server to display its pre-authentication banner before asking for a password.

**Capturing Flag 4**

After accepting the host's ED25519 fingerprint by typing `yes`, the server printed a large legal warning banner and right at the bottom of the banner was **Flag 4** (`FLAG4{173f4dd69119471da7cb0c5219070791}`)!

![](https://cdn.hashnode.com/uploads/covers/6afe9176ea699cf3085ebd20/3e3936ee-1a39-4448-9707-1a5b7ef0784f.png align="center")

---
