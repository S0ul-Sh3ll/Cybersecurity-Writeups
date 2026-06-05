# TryHackMe: VulnNet Roasted – Walkthrough

Welcome to my writeup for **VulnNet: Roasted**, an excellent beginner-to-intermediate Active Directory room on TryHackMe. We’ll walk through the process of compromising this machine from initial enumeration to domain admin privileges. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

## 🎯Objective
* **User Flag:** Gain initial access and locate the user flag.
* **Root Flag:** Escalate privileges to Domain Admin and capture the root flag.

Start up the lab and connect using Open VPN to your own Kali instance (highly recommended)

## 🔍 1. Initial Reconnaissance (Port Scanning)

Every successful penetration test starts with reconnaissance. We need to find out what doors (ports) are open on the target machine and what services are running behind them so we can look for potential entry points. 

To do this quickly and thoroughly, we use the following **Nmap** command:
```
nmap -p- -Pn -A --min-rate 2000 --initial-rtt-timeout 50ms --max-rtt-timeout 150ms --max-retries 1 --stats-every 1m 10.112.155.142
```

Here is exactly what each flag does:

* **`-p-`**: Scans **all 65,535 ports** instead of just the top 1,000 default ports. Backdoors or uncommon services love to hide on higher ports.
* **`-Pn`**: Treats the host as online and skips the initial ping discovery. This is crucial because many Windows targets disable ICMP (ping) by default, which can cause Nmap to falsely report that the machine is down.
* **`-A`**: This is the "Aggressive" scan flag. It enables **OS detection**, **version scanning** (to see what software version is running on a port), and runs **default NSE scripts** to find basic vulnerabilities.
* **`--min-rate 2000`**: Forces Nmap to send packets at a minimum rate of 2,000 packets per second. This drastically speeds up the scan.
* **`--initial-rtt-timeout 50ms` / `--max-rtt-timeout 150ms`**: Controls how long Nmap waits for a response to a packet. By capping it at 150ms, we ensure Nmap doesn't hang forever on a slow or dropped packet.
* **`--max-retries 1`**: If a port doesn't respond, Nmap will only try to probe it one more time before moving on. This keeps the scan fast.
* **`--stats-every 1m`**: Prints a status update in your terminal every 1 minute so you know the scan hasn't frozen.

 ⚠️ **Beginner Tip:** While these speed optimizations (`--min-rate`, aggressive timeouts) are perfect for CTFs like TryHackMe, they can be very noisy and disruptive in real-world professional engagements. In a real pentest, this could trigger firewalls, alert blue teams, or crash fragile services!

 _Figure 1: Nmap Scan Result_

<img width="626" height="742" alt="image-191" src="https://github.com/user-attachments/assets/6cff1441-c3bd-48aa-88f4-082a11e8ae9d" />

 ## 2.🧩 Analyzing the Scan Results

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

    "What is this server's primary job, and how can I use this information?"

#### <ins>The Big Picture: What is this machine?</ins>
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

* **The Identity & Database Engine (LDAP - Port 389/3268):** This is the directory database containing all the objects in the network—users, computers, groups, and permissions. If we can query this, we can pull a complete directory of everyone who works here.
* **The Gatekeeper (Kerberos - Port 88):** This is the authentication protocol. Instead of sending passwords across the network constantly, Windows uses Kerberos to issue ticket grants. If we can talk to Port 88, we can validate usernames or attempt to intercept ticket hashes.
* **The Postman & File Share (SMB - Port 445 & NetBIOS - Port 139):** This handles network data transfers and shares. Domain Controllers often host shared folders (like `SYSVOL`) containing group policies, logon scripts, and occasionally, accidentally leaked passwords.
* **The Navigator (DNS - Port 53):** Active Directory cannot function without DNS. It keeps track of where the Domain Controller lives and routes network traffic to the right services.

Conclusion: This machine is a **Windows Domain Controller (DC)**. 

 Domain Controller is the absolute "brain" of a Windows Active Directory network. It handles all security authentication requests, verifies user passwords, and dictates who has access to what files across the entire network ecosystem. 

### <ins>Open Ports Reference Table</ins>

| Port | Service | Significance for Penetration Testing |
| :--- | :--- | :--- |
| **53** | DNS | Domain Name System. Used to resolve hostnames. Essential for Active Directory mapping. |
| **88** | Kerberos | The authentication backbone of AD. We can look for user enumeration or **AS-REP Roasting** attacks here. |
| **135 / 593** | MSRPC | Microsoft RPC. Used for management traffic. Often useful for enumerating users or domain information via `rpcclient`. |
| **139 / 445** | SMB | Server Message Block. Handles file sharing. We will check this for anonymous access or null sessions to look for leaked files/credentials. |
| **389 / 3268** | LDAP | Lightweight Directory Access Protocol. The directory service holding all object info (users, groups, computers). Great for active enumeration. |
| **464** | kpasswd | Kerberos Password Change service. |
| **5985** | WinRM | Windows Remote Management. If we find valid credentials later, we can use this port via `evil-winrm` to get an interactive shell. |

### <ins>Crucial Active Directory Leakage</ins>

Looking closely at the output, Nmap's default scripts have leaked two critical pieces of information that we need to document immediately:

1. **The Domain Name:** `vulnnet-rst.local` (Discovered via the LDAP service string: `Domain: vulnnet-rst.local0.`)
2. **The Computer Name:** `WIN-2BO8M1OE1M1` (Discovered via the Service Info section)

> 📌 **Key Takeaway:** Before moving forward, we should map this domain name to the target IP address in our `/etc/hosts` file. Many Active Directory tools will fail to authenticating properly if they cannot resolve the domain name directly.

### <ins>Next Attack Vectors</ins>

Based on these results, our next logical steps for enumeration will be- **SMB Enumeration:** Checking for anonymous share access.

## 3.📁 SMB Enumeration
Armed with the knowledge that our target is an Active Directory Domain Controller, we start our active investigation with **SMB (Server Message Block) on Port 445**. 

Network administrators often leave certain shared folders open to the public, or accidentally configure them to allow unauthenticated access. To verify if this is the case, we use `NetExec` (formerly known as CrackMapExec).

We run a command to check for anonymous/guest access and list available network shares and this one worked:
```
netexec smb 10.112.155.142 -u 'guest' -p '' --shares
```

_Figure 2: SMB Shares Enumeration_

<img width="624" height="339" alt="image-192" src="https://github.com/user-attachments/assets/08e4e9f6-85e1-4dd0-94e0-058959a611e1" />

#### <ins>Key Takeaways from NetExec Scan</ins>
* **Successful Authentication:** The green `[+]` symbol next to `vulnnet-rst.local\guest:` indicates that the built-in **Guest** account is active and allows logons with an empty password.
* **System Details Confirmed:** NetExec officially confirms the operating system is **Windows Server 2019 Build 17763 x64**, running with packet signing enabled (`signing:True`).
* **Interesting Shares Discovered:**
   * `IPC$` (**READ**): Inter-Process Communication share. This could potentially let us query domain identifiers or list users via null sessions.
   * `VulnNet-Business-Anonymous` (**READ**): A custom-created network share that explicitly permits guest read access.
   * `VulnNet-Enterprise-Anonymous` (**READ**): Another custom network share open to guests.

## 4.👥 RID Brute Forcing
Instead of immediately jumping into the file shares, we can leverage our valid Guest session on SMB to gather critical intelligence about the network's user base. 

Because we have a valid session (even as a guest), we can perform a **Relative Identifier (RID) Brute Force** attack. In Windows environments, every user, group, and computer object has a Security Identifier (SID). The final numerical part of the SID is the RID. 
* Standard administrative accounts always have specific RIDs (e.g., `500` for Administrator, `501` for Guest).
* Custom domain users typically start at RID `1000` and count upward.

By querying these IDs sequentially, `NetExec` allows us to extract the actual usernames mapped to those IDs directly from the Domain Controller without requiring full domain credentials.

We execute the following command:
```
netexec smb 10.112.170.127 -u 'guest' -p '' --rid-brute
```

_Figure 3: netexec rid-brute user ids_

<img width="924" height="463" alt="image-193" src="https://github.com/user-attachments/assets/024e0508-f9b8-4432-94f2-0d6f53acf040" />

From this scan we have extracted the following usernames:
- Administrator
- Guest
- krbtgt
- WIN-2BO8M1OE1M1$
- enterprise-core-vn
- a-whitehat
- t-skid
- j-goldenhand
- j-leet

We will copy these specific names into a local text file named users.txt for asreproasting.

## 5.🔥 AS-REP Roasting
When a user requests a login ticket from Active Directory (a Kerberos AS-REQ), the key distribution center (KDC) normally expects some pre-authentication data usually a timestamp encrypted with the user's password hash to prove they know their password before handing over a ticket.

However, if an account has a specific administrative setting explicitly enabled called "Do not require Kerberos preauthentication" (UF_DONT_REQUIRE_PREAUTH), anyone can request an authentication ticket (AS-REQ) on behalf of that username. The Domain Controller will automatically reply (AS-REP) with a Ticket-Granting Ticket (TGT) encrypted with that user's specific password hash.

As attackers, we can capture this encrypted reply, save it to our machine, and crack it offline using brute-force tools like John the Ripper or Hashcat.

We will use Impacket's GetNPUsers tool to query the Domain Controller against our users.txt file and pull down any available hashes:
```
impacket-GetNPUsers vulnnet-rst.local/ -dc-ip 10.112.170.127 -usersfile users.txt -outputfile uhashes.txt
```

_Figure 4: ASREProasting_

<img width="1191" height="221" alt="image-194" src="https://github.com/user-attachments/assets/32e98c8e-858d-44a5-a991-78f1a7dbe16c" />

While some accounts come back with errors, the user account t-skid had pre-authentication disabled! The tool successfully captured a valid Kerberos $krb5asrep$23$ hash string and outputted it directly to our uhashes.txt file.

## 6.🖥️ Offline Hash Cracking

Now that we have successfully captured the AS-REP hash for the user `t-skid` and saved it inside `uhashes.txt`, it is time to extract the plaintext password using **John the Ripper**. 

We will feed the hash file to John and use the standard, powerhouse wordlist `/usr/share/wordlists/rockyou.txt` (or custom location `/home/kali/rockyou.txt`) to brute-force it offline:
```
john --wordlist=/home/kali/rockyou.txt uhashes.txt
```

_Figure 5: Hash Cracking_

<img width="913" height="149" alt="image-195" src="https://github.com/user-attachments/assets/287918a2-5aa1-46ba-84cc-570956610cff" />

Within moments, John the Ripper matches the cryptographic hash signature and reveals our first set of valid domain credentials:
```
Username: t-skid
Password: tj072889*
```

## 7.🎟️ Kerberoasting

Whenever you gain your first set of valid domain credentials in an Active Directory environment, your immediate next step should always be to check for **Kerberoasting**. 

### What is Kerberoasting?
While AS-REP Roasting targets accounts that explicitly do not require Kerberos pre-authentication, **Kerberoasting** targets **Service Accounts**. These are standard user accounts mapped to a specific Service Principal Name (SPN), which allows them to run services like SQL databases, web servers, or file shares (CIFS).

The inherent security flaw in the Kerberos protocol is that **any authenticated domain user** can request a Service Ticket (TGS-REP) from the Domain Controller for **any service account** configured in the entire domain. 

When the Domain Controller handles this request, it encrypts the service ticket using the master password hash of that specific service account. As attackers, we can use our new credentials for `t-skid` to ask the domain controller for these tickets, dump them to our machine, and attempt to extract the plaintext password via offline brute-force cracking.

#### <ins>Executing the Attack with Impacket</ins>
We will use Impacket's `GetUserSPNs` tool to authenticate to the Domain Controller using our cracked credentials for `t-skid` and request tickets for all available service profiles:
```
impacket-GetUserSPNs -dc-ip 10.112.170.127 'vulnnet-rst.local/t-skid:tj072889*' -request
```

_Figure 6: Kerberoasting_

<img width="1184" height="367" alt="image-196" src="https://github.com/user-attachments/assets/40fa4a91-9c20-4c12-a9a9-85131af55b67" />

The tool successfully authenticates, scans the active database, and pulls down an encrypted service ticket hash of:

Service Account: enterprise-core-vn

Mapped SPN: CIFS/vulnnet-rst.local

Group Membership: CN=Remote Management Users

Notice the `MemberOf` field! This account is a member of the Remote Management Users group. In Windows AD environments, members of this group are typically granted permissions to log in remotely via WinRM. If we can successfully crack this newly captured $krb5tgs$23$ hash, we won't just have a regular user account, we may get an interactive command terminal on the system.

We will copy this massive Kerberos $krb5tgs$ ticket string into a file named khash.txt on our attacking platform. Next, we will spin up John the Ripper to crack this hash offline and see if we can compromise the enterprise-core-vn account. (Notice if we cannot, we'll use the hash itself for login)

## 8.🖥️ Offline Hash Cracking
With the Kerberoasting ticket for `enterprise-core-vn` safely saved into our local `khash.txt` file, we return to **John the Ripper** to break the cryptographic wrapper. 

Because Kerberoasted tickets format slightly differently than standard user authentication requests, we explicitly append the `--format=krb5tgs` flag. This helps optimize John's internal parsing engine to speed up the dictionary attack against our `rockyou.txt` wordlist:
```
john --format=krb5tgs --wordlist=/home/kali/rockyou.txt khash.txt
```

_Figure 7: Hash Cracking_

<img width="691" height="140" alt="image-197" src="https://github.com/user-attachments/assets/fd30ad9d-a3dd-4a30-abc7-0dbeaf2e0d02" />

We have successfully exposed the complex plaintext password for the service account! Our updated active credentials profile is:
```
Username: enterprise-core-vn
Password: ry=ibfkfv,s6h,
```


## 9.🗄️ Gaining Initial Access & Capturing the User Flag

With a valid set of credentials for an account that belongs to the **Remote Management Users** group (`enterprise-core-vn`), we will use **Evil-WinRM**, gain shell and capture the user flag. 
```
evil-winrm -u 'enterprise-core-vn' -p 'ry=ibfkfv,s6h,' -i 10.112.179.143 -N
```

_Figure 8: Capture the User Flag_

<img width="597" height="638" alt="image-198" src="https://github.com/user-attachments/assets/2c298538-cf2d-45ca-a846-e0bf2aac2247" />

## 10.🔎 Privilege Enumeration
With an active shell session as `enterprise-core-vn`, our ultimate objective is to escalate our privileges to local `SYSTEM` or become a `Domain Admin`. 

Our first step upon landing on a Windows machine is to run a comprehensive token and membership check. This helps us see if our current user account holds any powerful rights or administrative properties that we can immediately exploit.

We execute the following command:
```
whoami /all
```

_Figure 9: Checking privileges_

<img width="909" height="471" alt="image-199" src="https://github.com/user-attachments/assets/fbf3d295-8612-4e34-b73f-a422ffd2a227" />

Since there are no obvious configuration flaws or dangerous tokens instantly visible within our current user profile. We can use our terminal to navigate the file system, explore the contents of these directories, and check if any network administrators or automated tasks left sensitive log files, configuration parameters, or scripts lying around.

## 11.🔑 Credential Hunting
Since looking through directories manually can be time-consuming, we can use an alternative, highly efficient method: running an automated search directly through our `Evil-WinRM` shell to scan the filesystem for interesting filenames. 

We can search the entire `C:\` drive for any files containing the word "password" in the name, while telling PowerShell to suppress access errors for folders we don't have permission to view:
```
Get-ChildItem -Path C:\ -Include *password* -File -Recurse -ErrorAction SilentlyContinue
```

_Figure 10: Credentials Hunting_

<img width="1034" height="585" alt="image-200" src="https://github.com/user-attachments/assets/a383ef12-4a15-4bc9-a536-ac521a6521c8" />


_Figure 11: RedetPassword.vbs script_

<img width="606" height="112" alt="image-201" src="https://github.com/user-attachments/assets/c49bdf8c-dfad-4c9e-89df-aec8f6f53648" />

The search successfully crawls the system drive and flags an incredibly interesting file hidden deep within the Active Directory Group Policy deployment structure. SYSVOL directory is a domain-wide share that replicates across all Domain Controllers. Group Policy Objects (GPOs) and logon/logoff scripts are stored here so every computer in the network can execute them. Historically, administrators frequently hardcoded administrative credentials directly into these logon scripts (like Visual Basic .vbs or PowerShell .ps1 scripts) to map drives, install software, or automatically reset standard user passwords across the network.

We navigate straight to the physical directory path using our shell and use the type command to output the plain text contents of the Visual Basic Script.

_Figure 12: New Credentials_

<img width="804" height="270" alt="image-202" src="https://github.com/user-attachments/assets/fb8f2de1-5810-4f9c-a1c7-c93989ea9001" />

Upon analyzing the code inside ResetPassword.vbs, we locate a hardcoded string revealing a brand new set of Active Directory domain credentials:
```
UserName = "a-whitehat"
Password = "bNdKVkjv3RR9ht"
```

Now that we have successfully extracted these fresh credentials from the automated script, our immediate next step is to test them against our remote administration access. We will try logging back into WinRM using this new identity to see if this account holds elevated privileges or places us on a much faster escalation path to full Domain Admin.
```
evil-winrm -u 'a-whitehat' -p 'bNdKVkjv3RR9ht' -i 10.114.168.250
whoami /all
```

_Figure 13: Admin Rights_

<img width="1035" height="629" alt="image-205" src="https://github.com/user-attachments/assets/5fb2927e-5527-4fd9-b8d6-76e76fe22ed6" />

<ins>We see 2 critical findings:</ins>

* **VULNNET-RST\Domain Admins:** This user account is an explicit member of the Domain Admins group. In Active Directory, this group grants unrestricted administrative control over every single computer, user account, server, and setting across the entire domain infrastructure.

* **BUILTIN\Administrators:** This confirms our local administrative status on this specific Domain Controller machine.

Because we are officially operating within the security context of a Domain Administrator, we have complete read rights over any folder on the file system.

We can navigate straight to the standard repository path where Windows stores the highest privilege administrative artifacts, the local Administrator's Desktop directory to claim our final objective.

_Figure 14: Failed Attempt at Root Flag_

<img width="975" height="469" alt="image-206" src="https://github.com/user-attachments/assets/fefa59e6-848c-4b0a-995c-2c8ab348a1dc" />

Even though whitehat is an admin it cant read the file
Even though our account `a-whitehat` belongs to the **Domain Admins** and **Local Administrators** groups, Windows security engines occasionally apply explicit File System Access Control Lists (ACLs) that block direct read commands, resulting in the `Access to the path is denied` error we just hit.

## Step 12:🗄️ Extracting the Active Directory Database via Secretsdump
Since our account has full Domain Admin privileges along with `SeBackupPrivilege`, we don't have to limit ourselves to fighting local file permissions to get the flag. Instead, we can extract the entire Active Directory database directly from the memory structures or the cryptographic storage engines of the Domain Controller.

To do this, we will drop out of our interactive shell temporarily (or open a new terminal tab on our Kali machine) and use Impacket’s powerful **`secretsdump.py`** utility. 

#### How Secretsdump Works
As a Domain Admin, `secretsdump.py` allows us to communicate with the Domain Controller's replication engine using the **DRSUAPI (Directory Replication Service Remote Protocol)**. The tool essentially pretends to be another Domain Controller requesting a backup sync. The target DC will willingly hand over the entire cryptographic active directory database, including the NT hashes for **every single user on the domain** (including the core local `Administrator` account).

Run the following command from your local Kali Linux terminal:
```
impacket-secretsdump 'vulnnet-rst.local/a-whitehat:bNdKVkjv3RR9ht@10.114.168.250' -just-dc
```

_Figure 15: Secrets Revealed_

<img width="912" height="556" alt="image-207" src="https://github.com/user-attachments/assets/b349fae0-4b8b-4e33-a24c-f50eb04eb5a8" />

Now that we have the raw hashes, we don't need to spend time attempting to crack the complex password of the root Administrator account. Instead, we can take advantage of a core architectural property of Windows authentication: Pass-the-Hash (PtH). Because the Windows authentication engine uses the NT hash value directly to verify credentials over the network, we can use the raw hash string as a key to sign right back into our administration console.

From the output data, we isolate the built-in local Administrator user entry:
```
    Username: Administrator
    NT Hash: c2597747aa5e43022a3a3049a3c3b09d
```
(Note: The aad3b435b51404eeaad3b435b51404ee string is simply the empty/default LM hash padding placeholder, which we can ignore.)

We pass this hash directly into Evil-WinRM using the -H flag:
```
evil-winrm -u 'Administrator' -H 'c2597747aa5e43022a3a3049a3c3b09d' -i 10.114.168.250
```

_Figure 16: Capture the Root Flag_

<img width="918" height="238" alt="image-209" src="https://github.com/user-attachments/assets/b1b68991-4e0b-4909-8ed7-0719b268adfe" />

We have successfully completed the walkthrough for VulnNet: Roasted! This lab provides an excellent showcase of how minor misconfigurations can build up into a full domain compromise.
---

### <ins>The Attack Chain Recap</ins>

1. **Reconnaissance (Port Scanning)**
   * Performed an optimized, aggressive `nmap` scan to discover open ports (**53, 88, 139, 445, 389**).
   * Grouped the open services to quickly identify the target machine's ecosystem as a **Windows Domain Controller (DC)** managing the domain `vulnnet-rst.local`.

2. **SMB Enumeration**
   * Used `NetExec` to look for unauthenticated guest or null session access.
   * Discovered that the built-in **Guest** account was active with no password, exposing two custom data shares: `VulnNet-Business-Anonymous` and `VulnNet-Enterprise-Anonymous`.

3. **Relative Identifier (RID) Brute Forcing**
   * Exploited the active Guest SMB session to brute-force security identifiers sequentially via `NetExec --rid-brute`.
   * Successfully extracted a comprehensive list of real, valid human identities active within the AD domain (`t-skid`, `enterprise-core-vn`, `a-whitehat`, etc.).

4. **AS-REP Roasting**
   * Tested the compiled user list against the Kerberos engine using Impacket's `GetNPUsers` tool.
   * Discovered that the account **`t-skid`** had the structural vulnerability flag `"Do not require Kerberos preauthentication"` (`UF_DONT_REQUIRE_PREAUTH`) active.
   * Captured and downloaded an encrypted ticket hash (`$krb5asrep$23$`) for the account.

5. **Initial Access via Offline Cracking**
   * Passed the recovered AS-REP ticket hash into `John the Ripper` using the `rockyou.txt` dictionary wordlist.
   * Successfully exposed the plaintext credentials: `t-skid:tj072889*`.

6. **Kerberoasting (Lateral Movement)**
   * Authenticated as a valid user domain account (`t-skid`) to safely request Service Tickets (`TGS-REP`) for any Service Principal Names (SPNs) from the Domain Controller via `GetUserSPNs`.
   * Intercepted an encrypted service ticket hash for **`enterprise-core-vn`** and identified its critical membership in the **Remote Management Users** group.

7. **Pivoting to an Interactive Console Shell**
   * Used `John the Ripper` with the `--format=krb5tgs` flag to crack the service account ticket hash, exposing the plaintext password: `ry=ibfkfv,s6h,`.
   * Leveraged **Evil-WinRM** on port 5985 to spawn an interactive PowerShell command session as `enterprise-core-vn` and capture the `user.txt` flag.

8. **Automated Credential Hunting (Privilege Escalation)**
   * Executed an automated PowerShell discovery loop (`Get-ChildItem`) to hunt for passwords left on the filesystem.
   * Discovered a hardcoded administration script named `ResetPassword.vbs` saved inside the public **SYSVOL** domain share directory.
   * Read the script to find cleartext credentials for a Domain Administrator account: `a-whitehat:bNdKVkjv3RR9ht`.

9. **Domain Dominance & Database Extraction**
   * Connected via `Evil-WinRM` as `a-whitehat` and confirmed explicit membership inside the **Domain Admins** group.
   * Faced a restrictive File System Access Control List (ACL) blocking access to the final root file on the Administrator Desktop.
   * Dropped out to a local terminal and ran Impacket's `secretsdump.py` tool using the `DRSUAPI` replication method to dump the master Active Directory credential store (`NTDS.dit`).

10. **Pass-the-Hash Execution & Root Flag Capture**
    * Isolated the local built-in root `Administrator` account's raw NT hash: `c2597747aa5e43022a3a3049a3c3b09d`.
    * Bypassed all local system file access blocks completely by performing a **Pass-the-Hash (PtH)** login injection directly into `Evil-WinRM`.
    * Authenticated as the system head `Administrator` and successfully read the contents of `system.txt` to capture the root flag: `THM{ad_compromised_master_of_roasted}`
