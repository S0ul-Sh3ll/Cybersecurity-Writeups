## TryHackMe: Fusion Corp
Welcome to my writeup for **Fusion Corp**. On the surface, it’s just another corporate entity, but beneath the corporate veneer lies a network waiting to be compromised. In this walkthrough, we will be diving headfirst into an Active Directory environment, navigating through misconfigurations, extracting sensitive data, and ultimately climbing the ladder to Domain Admin. If you're looking to sharpen your internal network penetration testing skills, this room is the perfect training ground. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

## 🎯 Objective
* Find the **User** flag.
* Escalate privileges and find the **Root** flag.

Start up the lab and connect using Open VPN to your own Kali instance (highly recommended)

## 🔍 1. Initial Reconnaissance (Port Scanning)
Every successful penetration test starts with reconnaissance. We need to find out what doors (ports) are open on the target machine and what services are running behind them so we can look for potential entry points. 

Before initiating the enumeration phase, let's export the target IP address as an environment variable for easier reference throughout the lab: `export target=<TARGET_IP>`

Then, we use the following **Nmap** command:
```
nmap -p- -Pn -A --min-rate 2000 --initial-rtt-timeout 50ms --max-rtt-timeout 150ms --max-retries 1 --stats-every 1m $target --open
```

Here is exactly what each flag does:

* **`-p-`**: Scans **all 65,535 ports** instead of just the top 1,000 default ports. Backdoors or uncommon services love to hide on higher ports.
* **`-Pn`**: Treats the host as online and skips the initial ping discovery. This is crucial because many Windows targets disable ICMP (ping) by default, which can cause Nmap to falsely report that the machine is down.
* **`-A`**: This is the "Aggressive" scan flag. It enables **OS detection**, **version scanning** (to see what software version is running on a port), and runs **default NSE scripts** to find basic vulnerabilities.
* **`--min-rate 2000`**: Forces Nmap to send packets at a minimum rate of 2,000 packets per second. This drastically speeds up the scan.
* **`--initial-rtt-timeout 50ms` / `--max-rtt-timeout 150ms`**: Controls how long Nmap waits for a response to a packet. By capping it at 150ms, we ensure Nmap doesn't hang forever on a slow or dropped packet.
* **`--max-retries 1`**: If a port doesn't respond, Nmap will only try to probe it one more time before moving on. This keeps the scan fast.
* **`--stats-every 1m`**: Prints a status update in your terminal every 1 minute so you know the scan hasn't frozen.
* **`--open`**: Efficient filter used to only show ports that are actively listening and accepting connections.

 ⚠️ **Beginner Tip:** While these speed optimizations (`--min-rate`, aggressive timeouts) are perfect for CTFs like TryHackMe, they can be very noisy and disruptive in real-world professional engagements. In a real pentest, this could trigger firewalls, alert blue teams, or crash fragile services!

 _Figure 1: Nmap Scan Result_

 <img width="1078" height="687" alt="image-276" src="https://github.com/user-attachments/assets/e3d60569-b549-4473-8559-15c7734bf0e5" />
<img width="1071" height="364" alt="image-277" src="https://github.com/user-attachments/assets/ccf14034-00ce-454c-a06e-a07e3e6760ec" />


## 2.🧩 Analyzing the Scan Results

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

    "What is this server's primary job, and how can I use this information?"

#### <ins>The Big Picture: What is this machine?</ins>
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

 When you look closely at this output, the machine practically screams its identity. We aren't just looking at a random server; we are looking at a Windows Domain Controller (DC) for the domain fusion.corp.

<ins>1. The Active Directory Core </ins>

These ports are the foundation of Windows identity management. If you see this specific cluster open together, you are 100% looking at a Domain Controller:

    Port 53 (DNS): Active Directory relies entirely on DNS to map domain services.

    Port 88 (Kerberos): The default authentication protocol for Active Directory. This handles ticket grants (TGTs and TGSs).

    Ports 389 & 3268 (LDAP / Global Catalog): Used to query information about users, computers, and groups within the active directory tree.

<ins>2. File Sharing & Internal Communications</ins>

    Ports 139 & 445 (NetBIOS / SMB): Server Message Block. This is highly valuable for initial enumeration. We can check if "Null Sessions" or anonymous logins are allowed to read shared folders (SYSVOL or NETLOGON).

    Ports 135, 593, & High Ports (RPC): Remote Procedure Calls. Windows uses these for services to talk to one another.

<ins>3. Remote Management (The Potential Footholds)</ins>

    Port 3389 (RDP): Remote Desktop Protocol. If we harvest cleartext credentials later, we might be able to log right into a graphical session.

    Port 5985 (WinRM): Windows Remote Management (HTTP). This is the command-line equivalent of RDP. If we find valid credentials for a user in the "Remote Management Users" or "Administrators" group, we can spawn a shell here using tools like evil-winrm.

<ins>4. The Web Layer</ins>

    Port 80 (HTTP): Running Microsoft IIS 10.0. The Nmap script notes it's using an "eBusiness Bootstrap Template." This tells us there is a web application hosted here, which is a prime target for finding leaked credentials, usernames, or software vulnerabilities.


## 3. 📁 SMB and LDAP Enumeration
Our first goal is to check what access we have without valid credentials or if we can get usernames without them.
To do this, we use netexec:
```
netexec smb $target -u 'guest' -p '' --shares
netexec smb $target -u '' -p '' --shares
netexec smb $target -u 'guest' -p '' --rid-brute
netexec smb $target -u '' -p '' --rid-brute
netexec ldap $target -u '' -p '' --users
```

Figure 2: SMB and LDAP Enumeration Attempt Failed

<img width="1082" height="484" alt="image-278" src="https://github.com/user-attachments/assets/fd7aa934-75eb-4c37-b7ea-af8a03a11484" />

Our initial unauthenticated sweep on the core infrastructure has yielded a clean sheet for the defense:

    Null sessions cannot list shares or brute-force RIDs.

    The Guest account is disabled.

    LDAP requires a successful bind.

## 4. 🌐 Web Server Enumeration
Since our infrastructure checks came up empty, our next move is to shift our focus entirely to the web server running on Port 80. Before we start aggressively fuzzing directories, we need to ensure our system resolves the domain correctly.

#### Step 1: Updating the /etc/hosts File

Windows IIS web servers often utilize Virtual Hosting. This means the server checks the Host header in our HTTP requests to decide which website to serve. If we navigate to just the IP address, we might miss the actual target site.

Let's append the IP and domain to our local /etc/hosts file:
```
echo "10.114.189.37 fusion.corp" | sudo tee -a /etc/hosts
```

Now, we can reliably access the site via http://fusion.corp in our browser or through our command-line tools.

_Figure 3: Website Usernames_

<img width="1280" height="714" alt="image-279" src="https://github.com/user-attachments/assets/67ffbc60-02dd-40a9-8a17-c7f97815865e" />

We find some usernames that we can use for kerbrute.

#### Step 2: Directory Fuzzing (Web Reconnaissance)

To discover hidden files, directories, or backup folders that aren't linked on the main homepage, we will use a directory brute-forcer like ffuf or gobuster. We will use a standard wordlist (common.txt or directory-list-2.3-medium.txt) and filter out common false positives (like 404 errors) to keep our output clean.
Bash
```
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://fusion.corp/FUZZ -e .txt,.html,.php -ic
```

_Figure 4: Fuzzing for Hidden Directories_

<img width="1078" height="488" alt="image-281" src="https://github.com/user-attachments/assets/b6f3f12c-505d-429d-ae47-8998f5d7252f" />

The ffuf scan finishes and gives us a clear look into the directory structure of the web server. Most of the findings are standard bootstrap template assets (css, js, lib, img), but one directory immediately jumps out as highly interesting: `/backup.`

When we navigate to http://fusion.corp/backup/, directory listing is enabled, revealing a single file sitting in the open:

_Figure 5: Employee.ods file_

<img width="1003" height="229" alt="image-282" src="https://github.com/user-attachments/assets/dc87c511-6867-4963-9407-0228ce9b8acf" />

📂 employees.ods (OpenDocument Spreadsheet)

A spreadsheet named "employees" located inside a backup directory on a Domain Controller is an absolute goldmine. This likely contains the organizational chart, full names, email addresses, or possibly even temporary passwords for the company's workforce.

Since it's an .ods file, we can open it using LibreOffice Calc on Kali Linux to see its content.

_Figure 6: employee.ods file content_

<img width="285" height="271" alt="image-283" src="https://github.com/user-attachments/assets/32dcac95-8a00-4ddf-b31c-5b2499ce6d12" />

## 5. 🎯 Moving to Kerberos: Verification & Exploitation
Now that we have harvested a solid list of potential identities from the employees.ods file, we can pivot back to our Active Directory core on Port 88. Instead of guessing blindly, our strategy splits into a clean two-step operation:

    Username Validation: Use kerbrute to verify which harvested handles are actual, active accounts on the Domain Controller.

    AS-REP Roasting: Query the verified accounts to see if any have pre-authentication disabled, allowing us to request a ticket-granting-ticket (TGT) hash that we can crack offline.

#### Step 1: Prepping the Target List
First, take the usernames from the file and save them into a text file named users.txt on our machine.

#### Step 2: Running Kerbrute for Enumeration
kerbrute uses fast, stealthy Kerberos pre-authentication requests (AS-REQ) to check if a user exists. Because it doesn't try a password, it does not lock out accounts, making it safe to use against production domains.

Run the following command against the DC:
```
kerbrute userenum --dc $target -d fusion.corp users.txt
```

Keep an eye out for valid user hits tagged with a green [+]. This confirms the exact structure of the accounts inside the domain.

_Figure 7: Kerbrute found 1 valid id_

<img width="1079" height="224" alt="image-284" src="https://github.com/user-attachments/assets/e9d0d5f9-13fb-43d4-a3eb-92d8ed20d84a" />

#### Step 3: Launching the AS-REP Roast Attack
If an account has the flag "Do not require Kerberos preauthentication" enabled in its User Account Control (UAC) properties, any attacker can request an authentication ticket for that user from the DC. The DC will happily return an encrypted ticket component containing the user's password hash.

We can pull these down using the Impacket suite tool GetNPUsers:
```
impacket-GetNPUsers fusion.corp/ -dc-ip $target -usersfile users.txt -outputfile hashes.txt
```

_Figure 8: Asreproasting Hash Successful_

<img width="1080" height="252" alt="image-285" src="https://github.com/user-attachments/assets/d9513f36-3b60-4594-a835-2e1cad7f9f44" />

The Domain Controller smoothly handed over an AS-REP ticket component, throwing a raw password hash straight into our hashes.txt file.

#### Step 4: Cracking the Hash Offline
The $krb5asrep$23$ prefix tells us this is a Kerberos 5, AS-REP Roasting etype 23 (RC4-HMAC) hash. This format is notoriously fast to crack using standard consumer hardware.

We will feed this hash into hashcat using the rockyou wordlist.
```
hashcat -m 18200 hashes.txt /home/kali/rockyou.txt
```

_Figure 9: Hashcat Password Cracked_

<img width="1075" height="696" alt="image-286" src="https://github.com/user-attachments/assets/c28a1c24-ced3-4a6b-825a-00275a2ecbd0" />

hashcat successfully processes the lparker hash in under a second, tearing through the keyspace and yielding a cleartext match:

    Target Account: lparker Password: REDACTED

We now have our initial foothold into the fusion.corp domain! With valid domain credentials, the constraints of our previous unauthenticated attempts disappear.

## 6. 🪜 Leveraging Authenticated Footholds for Enumeration & Exploitation
Our immediate next step is to perform an authenticated sweep of the Domain Controller to see what doors this password unlocks and what internal data we can extract.

#### 1. Validating Shares & Local Rights
We will use netexec over SMB to check if lparker has access to read domain shares or if they happen to possess administrative rights (marked by a lucrative Pwn3d! tag in NetExec):
```
netexec smb $target -u 'lparker' -p '!!abbylvzsvs2k6!' --shares
```

_Figure 10: SMB Shares_

<img width="1083" height="197" alt="image-287" src="https://github.com/user-attachments/assets/f6b31ce0-cd04-49d5-8a4b-0555aa8d3616" />

We successfully authenticated to SMB, confirming our (READ) access to the default logon shares: NETLOGON and SYSVOL.

#### 2. Checking Remote Access (WinRM)
We need to see if this user is allowed to log in remotely to get an interactive shell. We can check WinRM capability smoothly:
```
netexec winrm $target -u 'lparker' -p '!!abbylvzsvs2k6!'
```

_Figure 11: WinRM Pwn3d!_

<img width="1072" height="70" alt="image-288" src="https://github.com/user-attachments/assets/8c87bf0e-243e-4cc3-9e72-a81ec57c1175" />

[+] fusion.corp\lparker:!!abbylvzsvs2k6! (Pwn3d!) 🚨
The (Pwn3d!) status tag on port 5985 is huge. It indicates that lparker has local administrative privileges or belongs to the Remote Management Users group on this machine. We can use this to establish an interactive command shell.

#### 3. Full Domain Enumeration via LDAP
Since anonymous LDAP binds failed earlier, we can now use lparker to authenticate a proper LDAP bind and safely dump the entire user list, group memberships, and object descriptions directly from the Active Directory database:
```
netexec ldap $target -u 'lparker' -p '!!abbylvzsvs2k6!' --users
```

_Figure 12: LDAP User Enumeration with new creds found_

<img width="1080" height="245" alt="image-289" src="https://github.com/user-attachments/assets/fd18c4b3-e14f-421a-8c46-061f32761986" />

Authenticating our bind reveals that the Active Directory environment has 5 domain users in total. The system administrator left a cleartext password inside the Description field for the user account jmurphy. This is a classic Active Directory misconfiguration that occurs during rapid onboarding or automated deployment scenarios.
`New Account: jmurphy`
`Password: REDACTED`

<ins>Since we have Pwn3d! rights with lparker over WinRM, we don't need to pivot immediately just yet. Let’s capture our first interactive session on the target box using evil-winrm.</ins>

Open up a new terminal window and connect:
```
evil-winrm -i $target -u 'lparker' -p 'REDACTED'
```

Once you establish your shell as lparker: Grab the User Flag. Check privileges.

_Figure 13: User 1 Flag over WinRM_

<img width="1075" height="206" alt="image-295" src="https://github.com/user-attachments/assets/b0abeb86-2adf-4cbb-be9d-2a347c8dfd80" />

_Figure 14: Privileges check_

<img width="1080" height="465" alt="image-296" src="https://github.com/user-attachments/assets/7326a7e7-c980-4128-b381-c9ca269ce750" />

No special privilege here for gaining Admin shell.

#### Now let's login as jmurphy to find the 2nd user flag:
```
evil-winrm -i $target -u 'jmurphy' -p 'REDACTED'
```

_Figure 15: User 2 Flag_

<img width="1076" height="599" alt="image-290" src="https://github.com/user-attachments/assets/2ce548dd-0e20-45f1-8a60-6bb2972101dd" />

_Figure 16: jmurphy Privileges_

<img width="1077" height="581" alt="image-291" src="https://github.com/user-attachments/assets/9f2e8ef4-a7a3-47c5-b1b5-ed4d303db057" />

Let's look closely at the whoami /all output to analyze why this user is the key to compromising the entire system.
The Analysis: Privilege & Group Overview

    Group Membership: BUILTIN\Backup Operators

    Crucial Privileges Enabled:

        SeBackupPrivilege: Grants full read access to any file on the operating system, bypassing all standard Access Control Lists (ACLs).

        SeRestorePrivilege: Grants full write access to any file on the system, regardless of file permissions.

By design, members of the Backup Operators group are granted these rights so they can back up critical system configurations without being blocked by file security permissions. As an attacker, we can exploit SeBackupPrivilege to copy the operating system's sensitive registry hives directly out of memory, allowing us to harvest administrative hashes offline.


## 7. 🔑 Privilege Escalation
To extract the local credentials database, we need to save two primary registry keys:

    SAM: Contains the local NT hashes (including the built-in local Administrator account).

    SYSTEM: Contains the boot key required to decrypt the data stored inside the SAM hive.


#### Create a temporary directory to store the exports
```
mkdir loot
cd loot
```

_Figure 17: create a new folder_

<img width="1073" height="177" alt="image-292" src="https://github.com/user-attachments/assets/1b1704bd-45a4-441b-abad-1135e74d565d" />

#### Save the SAM and SYSTEM hives
Use the following commands to take backup of these files in your new folder:
```
reg save HKLM\SAM sam.hiv
reg save HKLM\SYSTEM system.hiv
```

Once the .hiv files are successfully written to the disk on the target server, we need to pull them down onto our Kali Linux environment.

Evil-WinRM features a built-in download command utility that makes transferring files from the target machine directly to your local working directory incredibly simple. Run the following commands in your active terminal:
```
download sam.hiv
download system.hiv
```

_Figure 18: SAM.hiv & SYSTEM.hiv transfers_

<img width="1073" height="341" alt="image-293" src="https://github.com/user-attachments/assets/7462ac4c-9b6c-4109-b974-81ddf6c99238" />

#### Secrets Dumping via Impacket
With both files successfully downloaded onto your local Kali machine, we will use Impacket's secretsdump.py tool to parse the keys offline and decrypt the hashes  or use pass the hash:
```
impacket-secretsdump -sam sam.hiv -system system.hiv LOCAL
```

_Figure 19: Obtained Hashes_

<img width="1076" height="149" alt="image-294" src="https://github.com/user-attachments/assets/3b42d58d-cba8-4b9f-9a7d-e77c08d1a091" />

The highlight of our dump is the built-in local Administrator account (RID 500):

Because this machine is the primary Domain Controller, the local administrator account typically shares the same security context or allows administrative control over the entire Active Directory domain infrastructure. Furthermore, Windows allows us to authenticate using the raw NT hash directly without needing to crack it back into text, a technique known as Pass-the-Hash (PtH).

#### Spawning the Root Shell
We can pass our newly acquired NT hash directly back into evil-winrm using the -H flag to claim our top-tier administrative shell session.
From your Kali Linux terminal, establish the final connection:
```
evil-winrm -i 10.114.189.37 -u 'Administrator' -H '2182eed0101516d0a206b98c579565e6'
```

_Figure 20: Unable to Login using Hash_

<img width="1076" height="209" alt="image-298" src="https://github.com/user-attachments/assets/7c698ecb-707f-4b4b-b65c-4bfcf6cfe3d0" />

It is incredibly common for evil-winrm to hang or fail when executing a Pass-the-Hash attack against the built-in Administrator account on certain targets. This typically happens due to LocalAccountTokenFilterPolicy configurations on Windows Server, which strip administrative tokens from local accounts connecting over remote management protocols (WinRM), even if they are part of the Administrators group.

Since Port 3389 (RDP) is wide open on this machine, we can switch tactics and use our captured Administrator NT Hash to log in via a graphical session using xfreerdp and its built-in restricted admin mode option.

We can use the /pth flag to feed the raw NT hash directly into the remote desktop client. Run the following command from your local Kali Linux machine:
```
xfreerdp /v:10.114.189.37 /u:Administrator /pth:2182eed0101516d0a206b98c579565e6 /gdi:sw /dynamic-resolution +clipboard
```

_Figure 21: RDP Login Failed_

<img width="1084" height="334" alt="image-299" src="https://github.com/user-attachments/assets/66f949b9-e47e-4753-9b24-c589ec2fdccc" />

Since remote token filter policies are blocking our Pass-the-Hash attempts via WinRM and RDP, let's take a step back and let our hardware do the heavy lifting. If we can break the Administrator's raw NT hash back into its cleartext password, the server won't be able to distinguish our remote login from a legitimate administrative session.

Use hashcat to decipher the hash:
```
hashcat -m 1000 -a 0 '2182eed0101516d0a206b98c579565e6' /home/kali/rockyou.txt
```

_Figure 22: Crack Failed_

<img width="1085" height="296" alt="image-300" src="https://github.com/user-attachments/assets/a18e2ef1-f872-4eed-a63f-2a12ee8076d1" />

No passwords found.

#### Alternative Strategy: Account Takeover via Backup Operators
Log back in to your working WinRM session as jmurphy (the account that possesses SeBackupPrivilege and SeRestorePrivilege). Because you have full token access there, you can override system settings directly or modify group parameters.

After trying resetting admin password, add jmurphy to admin group, ntds.dit copy into shadow volume, ntdsutil tool, SeBackupPrivilegeCmdLets.dll etc custom scripts, this is what finally worked: `PsCabesha-tools.git`

### Step 1: Clone the Repository
The first step involved cloning the PsCabesha-tools repository from GitHub into the working directory (/home/kali/Desktop/fncorp). This repository contains various post-exploitation and privilege escalation scripts.
```
git clone https://github.com/Hackplayers/PsCabesha-tools.git
```

### Step 2: Directory Navigation & Exploration
After cloning, navigate to:
```
cd PsCabesha-tools
ls
# Output: Credentials, Misc, Privesc, README.ME, Tuneles-PortFwd

cd Privesc
ls
```

The Privesc directory contained several specialized PowerShell scripts, including automated auditing tools and privilege escalation vectors (e.g., Sherlock.ps1, BypassUAC-CMSTP.ps1, and Backup-ToSystem.ps1).

#### <ins>Upload Backup-ToSystem.ps1 file using Evil-WinRM upload funtion.</ins>
After it uploads, use the following commands:
import-module .\Backup-ToSystem.ps1
Backup-ToSystem -command "net user newadmin Password123! /add"
Backup-ToSystem -command "net localgroup Administrators newadmin /add"

#### These commands are directed to create a new local user account (newadmin) with the specified password. 

_Figure 23: Backup-ToSystem.ps1_

<img width="1077" height="643" alt="image-307" src="https://github.com/user-attachments/assets/421dba59-871c-4f9c-8b37-8c57d90fe05a" />

Login with the newadmin user you just created and capture the root flag.

_Figure 24: Root Flag_

<img width="888" height="209" alt="image-308" src="https://github.com/user-attachments/assets/85d9030d-b658-4663-aa7a-4ed0acf01860" />

