## TryHackMe: RazorBlack – Walkthrough
Welcome to my writeup for **RazorBlack**, an immersive Active Directory room on TryHackMe that perfectly demonstrates the classic internal attack lifecycle. In this machine, we pivot from zero footprint to total enterprise dominance by abusing native Kerberos mechanisms, exploiting information disclosures, and leveraging "Privilege by Design" administrative rights. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

## 🎯 Objective
- **User Flags:** Enumerate the Active Directory environment, exploit misconfigurations, and capture the flags for multiple user accounts (**Steven**, **Ljudmila**, **Tyson**, and **Xyan1d3**).
- **Root Flag:** Escalate privileges to Domain Admin and capture the root flag.

Start up the lab and connect using Open VPN to your own Kali instance (highly recommended)

## 📑 Questions Checklist
Below are the flags and answers we need to gather throughout this assessment:

- [ ] **What is the Domain Name?**
- [ ] **What is Steven's Flag?**
- [ ] **What is the zip file's password?**
- [ ] **What is Ljudmila's Hash?**
- [ ] **What is Ljudmila's Flag?**
- [ ] **What is Xyan1d3's password?**
- [ ] **What is Xyan1d3's Flag?**
- [ ] **What is the root Flag?**
- [ ] **What is Tyson's Flag?**
- [ ] **What is the complete top secret?**
- [ ] **Did you like your cookie?**

## 🔍 1. Initial Reconnaissance (Port Scanning)
Every successful penetration test starts with reconnaissance. We need to find out what doors (ports) are open on the target machine and what services are running behind them so we can look for potential entry points. 

Before initiating the enumeration phase, let's export the target IP address as an environment variable for easier reference throughout the lab: `export target=<TARGET_IP>`

Then, we use the following **Nmap** command:
```
nmap -p- -Pn -A --min-rate 2000 --initial-rtt-timeout 50ms --max-rtt-timeout 150ms --max-retries 1 --stats-every 1m $target
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

<img width="738" height="689" alt="image-210" src="https://github.com/user-attachments/assets/8bb6ff4f-cbab-421a-af55-57eaea74ea1c" />
<img width="736" height="681" alt="image-211" src="https://github.com/user-attachments/assets/2ef85ef0-69e1-4a2e-bf1b-299eb5342e01" />

 ## 2.🧩 Analyzing the Scan Results

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

    "What is this server's primary job, and how can I use this information?"

#### 🖼️<ins>The Big Picture: What is this machine?</ins>
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

* <ins>Active Directory Fundamentals</ins> (Ports 53, 88, 389, 636, 3268): We have DNS, Kerberos, and LDAP all running together. When you see this specific cluster of services, you are almost certainly looking at an Active Directory Domain Controller (DC).

* <ins>Windows File Sharing & RPC</ins> (Ports 135, 139, 445): Standard Windows internal communication and SMB file sharing. These are always prime targets for initial enumeration to check for anonymous login access.

* <ins>Remote Management</ins> (Ports 3389, 5985): Remote Desktop Protocol (RDP) and Windows Remote Management (WinRM). We can't do much with these yet, but once we score some valid credentials, these will be our front doors into the server.

* </ins>The Anomalies</ins> (Ports 111, 2049): Wait, rpcbind and nfs? Network File System (NFS) is traditionally a Unix/Linux file-sharing protocol. Seeing it exposed on a Windows Domain Controller is highly unusual and screams "misconfiguration." This should go straight to the top of our attack priority list.

Beyond the open ports, Nmap's scripting engine (-A) handed us some critical environmental intelligence:
* We have our first answer! By looking at the LDAP or RDP script outputs, we can see the Domain Name is <ins>raz0rblack.thm</ins> and the computer name is HAVEN-DC.

Based on our scan results, we will start with NFS Enumeration , NFS running on a Windows DC is suspicious, we'll check for openly mountable shares.

## 📂 3: NFS Enumeration
Seeing Network File System (NFS) on a Windows Domain Controller is a massive red flag. Because it is traditionally a Linux protocol, it often suffers from misconfigurations when deployed in a Windows environment, specifically, allowing anonymous access.

We would first run `showmount -e $target` to reveal the exact name of the exported share (in this case, it exposes a `/users` directory). Once we know the share name, we can mount it to our local Kali file system to explore it just like a normal folder. 

Newer versions of Kali Linux drop support for older NFS protocols by default. To make this work, we **must** force the connection to use version 3 by appending the `-o vers=3` flag.

Let's set up our workspace, mount the remote share, and see what is inside:
Create a local folder to act as our mount point:
`mkdir mnt`

Mount the target's share to our newly created folder:
```
sudo mount -t nfs -o vers=3 $target:/users razorblack/mnt/
```

Then list the contents of our mnt folder: `ls -la` and read the files to get the flag.
<ins>Remember, never try to cd into the mounted directory or you'll freeze and will have to restart kali.</ins>

_Figure 2: Mount and capture sbradley flag_

<img width="630" height="299" alt="image-213" src="https://github.com/user-attachments/assets/e781003b-3cc4-402b-a840-0dbb7cbc561f" />

Let's take a look at that spreadsheet also. You can open it right from your terminal using LibreOffice Calc(should be installed first):
```
localc mnt/employee_status.xlsx
```

_Figure 3: Excel file revealing employee names_

<img width="768" height="395" alt="image-214" src="https://github.com/user-attachments/assets/a35c50f5-e8cc-4fab-8d40-31051feebb94" />

<ins>In an Active Directory environment, having a list of employee names is half the battle. This gives us the raw material needed to generate a targeted username wordlist for kerbrute.</ins>

### <ins>Building a User Wordlist</ins>
Before we can launch any attacks against the Domain Controller, we need to convert the full names from our spreadsheet into valid Active Directory usernames.
Companies typically use standard naming conventions. Looking at Steven Bradley's text file (sbradley), we can confidently deduce that the company uses the first initial + last name format (flast). Let's create a file named users.txt and format the names accordingly:

- dport
- iroyce
- tvidal
- aedwards
- cingram
- ncassidy
- rzaydan
- lvetrova
- rdelgado
- twilliams
- sbradley
- clin

## 🔥4: Asreproasting
Now that we have our `users.txt` wordlist, we shouldn't just blindly fire attacks at the server. In a real-world scenario, repeatedly guessing passwords for invalid accounts is a quick way to trigger security alerts. Instead, we will use a tool called **Kerbrute** to validate which of these usernames actually exist in the Active Directory environment. Kerbrute does this efficiently and quietly by abusing the Kerberos pre-authentication mechanism.

Let's run Kerbrute against the Domain Controller:
```
kerbrute userenum users.txt  --dc $target -d raz0rblack.thm 
```

Out of the 12 names we generated from the spreadsheet, exactly three are active accounts on this domain:

    lvetrova (Ljudmila - the AD Admin)

    twilliams (Tyson)

    sbradley (Steven)

With our list narrowed down to three guaranteed valid targets, it is time to go on the offensive. Our first Active Directory attack will be AS-REP Roasting.

How does it work? By default, Kerberos requires users to prove their identity (pre-authentication) before it hands over a Ticket Granting Ticket (TGT). However, administrators sometimes disable this requirement for specific accounts. If "Do not require Kerberos preauthentication" is enabled, we can simply ask the Domain Controller for the user's TGT. The DC will hand it over, and importantly, a portion of that ticket is encrypted with the user's password. We can take this ticket offline and attempt to crack it.

To execute this, we will use Impacket's GetNPUsers script:
```
impacket-GetNPUsers raz0rblack.thm/ -dc-ip $target -usersfile users.txt -outputfile hashes.txt
```

_Figure 4: Kerbrute & Asreproasting_

<img width="628" height="522" alt="image-216" src="https://github.com/user-attachments/assets/64fa5d3d-8207-4544-a874-909bd55ba031" />

Success! While Ljudmila and Steven's accounts are secure against this specific misconfiguration, Tyson Williams (twilliams) is vulnerable. Impacket successfully extracted his AS-REP hash and saved it to our hashes.txt file.

Our next step is to take this hash, feed it to a cracking tool like Hashcat or John the Ripper, and see if we can uncover Tyson's plaintext password.

### 🖥️ Offline Hash Cracking
Now that we have successfully captured the AS-REP hash for the user twilliams and saved it inside uhashes.txt, it is time to extract the plaintext password using John the Ripper. We will feed the hash file to John and use the standard, powerhouse wordlist rockyou.txt to brute force it offline:
```
john --wordlist=/home/kali/rockyou.txt hashes.txt
```

_Figure 5: Hash Cracking_

<img width="628" height="171" alt="image-215" src="https://github.com/user-attachments/assets/f181c411-ebcb-4347-a695-89cb966aaca7" />

Within moments, John the Ripper matches the cryptographic hash signature and reveals our first set of valid domain credentials:
```
Username: twilliams
Password: Redacted
```


## 🔥5: Kerberoasting
Once we cracked Tyson Williams' (`twilliams`) AS-REP hash offline, we recovered his plaintext password: `redacted`. 

Having a valid set of domain credentials changes our position entirely. We can now authenticate to the Active Directory environment as an insider and look for a highly lucrative target configuration: **Service Principal Names (SPNs)**. This technique is called **Kerberoasting**.

**How does it work?** Any domain user can request a Kerberos service ticket (TGS) for any service registered in the Active Directory forest. When the Domain Controller hands over this ticket, it encrypts it using the password hash of the account running that service. If a service is running under a normal user account rather than a machine account, we can pull that ticket down, extract the hash, and crack it offline to steal that user's password.

We will use Impacket's `GetUserSPNs` script along with Tyson's credentials to query the Domain Controller and request these tickets:
```
impacket-GetUserSPNs -dc-ip $target 'raz0rblack.thm/twilliams:passwordhere' -request
```

_Figure 6: Kerberoasting_

<img width="627" height="541" alt="image-219" src="https://github.com/user-attachments/assets/d382b14e-3b44-4fd6-8452-0d3535cc96a8" />

Our request successfully pulled back an active SPN mapping:

```
Target User Account: xyan1d3

Group Membership: CN=Remote Management Users (This means if we crack this account's password, we can immediately log in remotely via WinRM!)

The Artifact: Impacket cleanly dumped a valid $krb5tgs$23$ Kerberos TGS hash.
```
Let's copy this massive hash block out of our terminal, append it to a file called khash.txt, and fire up our cracking rigs once again to discover Xyan1d3's password.

_Figure 7: Hash Cracking_

<img width="620" height="201" alt="image-220" src="https://github.com/user-attachments/assets/8a0ae43a-36ff-4ed4-8b3d-28941eae82b9" />

Within moments, John the Ripper matches the cryptographic hash signature and reveals xyan1d3 credentials. 

## 🔑 6: Initial Access
With a valid set of credentials for an account that belongs to the Remote Management Users group (enterprise-core-vn), we will use Evil-WinRM, gain shell and capture the user flag.

```
evil-winrm -u 'xyan1d3' -p 'passwordhere' -i $target -N
```

Our first step upon landing on a Windows machine is to run a comprehensive token and membership check. This helps us see if our current user account holds any powerful rights or administrative properties that we can immediately exploit.

We execute the following command:
```
whoami /all
```

_Figure 8: WinRM_

<img width="624" height="585" alt="image-221" src="https://github.com/user-attachments/assets/95fe8f70-2322-4c27-a6e6-8f23dc4c8724" />

_Figure 9:  Privileges Information_

<img width="626" height="173" alt="image-222" src="https://github.com/user-attachments/assets/809acf5c-c99e-4afc-bb92-28752c96b6c8" />

#### 💡 Massive Privilege Escalation Vector Spotted!
Take a close look at our group memberships and privileges:

    BUILTIN\Backup Operators: Members of this group are trusted with full access to any file on the system, completely bypassing Access Control Lists (ACLs).

    SeBackupPrivilege & SeRestorePrivilege: These rights are explicitly Enabled. This means we have the authority to read and extract critical system files—including the entire Active Directory database (ntds.dit) or local registry hives (SAM, SYSTEM). This is our direct pathway to Domain Admin.

#### <ins>Decrypting the PSCredential Object</ins>
Before rushing into dumping the Active Directory database, let's explore our user's home directory for immediate artifacts or clues left behind.
We see a peculiar XML file named xyan1d3.xml sitting right in the root of our profile. Let's look inside.

The file contains an exported PowerShell PSCredential object. The user mockingly explicitly says "Nope your flag is not here", but check out the password field, it contains a long, hex-encoded string starting with 01000000....

This signature pattern confirms it is encrypted using the Windows Data Protection API (DPAPI). Since this credential block was encrypted by our current user context (xyan1d3), we possess the exact keys necessary to decrypt it inline using standard PowerShell functions.

Since we are currently logged in as `xyan1d3`, we have the exact security context required to decrypt it automatically. We can easily import the XML file back into a PowerShell variable using `Import-Clixml`, which handles the decryption seamlessly in the background, and then call the `GetNetworkCredential()` method to expose the plaintext.

Run the following commands within your Evil-WinRM session:
```
# 1. Import the XML file to reconstruct the credential object
$cred = Import-Clixml -Path C:\Users\xyan1d3\xyan1d3.xml

# 2. Extract and display the plaintext password string
$cred.GetNetworkCredential().Password
```

_Figure 10: Flag Captured_

<img width="624" height="347" alt="image-243" src="https://github.com/user-attachments/assets/4dbd7652-e1e3-469b-b3ae-b48a6075ea2e" />

## 🚨7: Privilege Escalation
Now that we have stable access as `xyan1d3` and know we possess `SeBackupPrivilege`, our immediate goal is to harvest password hashes to escalate our privileges further.
Our first step was to use Impacket's `secretsdump` remotely against the Domain Controller using the network-based DRSUAPI replication method (`-just-dc`). We attempted the following command from our Kali machine:
```
impacket-secretsdump "raz0rblack.thm/xyan1d3:password@$target" -just-dc
```

It failed because the -just-dc flag relies on specific replication privileges typically reserved for Domain Administrators (like the Replicating Directory Changes permission). Even though xyan1d3 is a member of the Backup Operators group, the account lacks the explicit Active Directory rights required to perform a remote network DCSync attack.

_Figure 11: Failed SecretsDump Attack_

<img width="623" height="251" alt="image-224" src="https://github.com/user-attachments/assets/735059f6-85e8-4369-8ce3-ee87b5d370ad" />

### The Workaround: Abusing SeBackupPrivilege Locally
Since the network route is blocked, we can pivot to an offline attack. Because SeBackupPrivilege is explicitly Enabled in our current session, we have the authority to read any sensitive file on the operating system, completely bypassing local Access Control Lists (ACLs).

Instead of targeting the live Active Directory database over the network, we can use the built-in Windows Registry utility to save copies of the local registry hives directly to disk.

Inside active Evil-WinRM session, execute the following commands to save the SAM and SYSTEM hives:
```
# 1. Export the local SAM registry hive
reg save hklm\sam sam.hive

# 2. Export the local SYSTEM registry hive (contains the boot key needed to decrypt the SAM)
reg save hklm\system system.hive
```

Once the hives are successfully saved to the target machine's filesystem, use Evil-WinRM's built-in download feature to pull them back to your local Kali workspace:
```
download sam.hive
download system.hive
```

_Figure 12: Save SAM & SYSTEM Hives_

<img width="616" height="497" alt="image-225" src="https://github.com/user-attachments/assets/968757b4-d41d-4d5e-b2f2-5b0e34d8c705" />

#### <ins>Parsing the Hives Offline</ins>
With the files safely downloaded onto our Kali Linux instance, we can now use Impacket's secretsdump locally to parse the database offline. This does not touch the target network at all, making it completely silent.
Run this command in the directory where your downloaded hive files are located:
```
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

_Figure 13: Dump the Hashes_

<img width="626" height="187" alt="image-226" src="https://github.com/user-attachments/assets/be93848a-1baa-40c3-a178-5f56f0cbf081" />

Impacket has cleanly decrypted and dumped the local NTLM password hashes stored in the SAM database.

We now have grabbed the crown jewel of local access: the **Administrator** account's NTLM hash.
```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9689931redacted...:::
```

<ins>We will now use a Pass-the-Hash (PtH) attack to log right back in via Evil-WinRM, but this time with absolute full **Local Administrator** authority.</ins>

Open a terminal on Kali and run Evil-WinRM using the `-H` flag to pass the NTLM hash:
```bash
evil-winrm -i $target -u Administrator -H 9689931bed40ca5redacted...
```

_Figure 14: Login as Administrator_

<img width="621" height="179" alt="image-227" src="https://github.com/user-attachments/assets/433b9d78-93ae-4e12-9bfc-331140dc8426" />

After utilizing the local registry dumps to compromise the system completely, we successfully authenticated as the local **Administrator** via Evil-WinRM. 
While exploring the Administrator's directories for our final objective, we stumbled upon another curious file named `root.xml`.

_Figure 15: root.xml_

<img width="629" height="653" alt="image-228" src="https://github.com/user-attachments/assets/2f7f531a-9432-4e8e-9967-f4d674288a37" />

Unlike traditional text flags, inspecting `root.xml` did not immediately yield a plaintext flag. Instead, it contained a massive string of raw hexadecimal characters.
To reveal the underlying message, we can copy this hex string back into our local Kali Linux machine and use the xxd command with the -r (reverse) and -p (plain continuous hex dump) flags to convert it back into human-readable ASCII text:
```
echo "44616d6e20796......" | xxd -r -p
```

_Figure 16: Decode the Flag_

<img width="618" height="289" alt="image-229" src="https://github.com/user-attachments/assets/0f24b591-0c22-484c-8f6c-494817f636cd" />

We have successfully retrieved the root flag!

Remember how our initial network-based `secretsdump` failed because `xyan1d3` lacked directory replication privileges. Now that we have the highest-privileged credential in the domain, we can perform a classic **DCSync attack**. By using a Pass-the-Hash (PtH) technique, we can password data for every user account in the Active Directory environment via the DRSUAPI method.

Let's execute the remote sync from our Kali machine using the Administrator's hash:
```
 impacket-secretsdump "raz0rblack.thm/Administrator@$target" -hashes :9689931bed40ca5.....redacted -just-dc
```

_Figure 17: SecretsDump_

<img width="627" height="656" alt="image-230" src="https://github.com/user-attachments/assets/93eba115-9db5-4182-8493-a4410a4dc487" />

This dump completely breaks wide open every account remaining on our checklist, we not have NTLM hash for every user and can login via Pass the Hash for every user!

## 🚩8: Finding Other Flags
With our admin capabilities active across multiple user accounts, we decided to explore the home directory of Tyson Williams (`twilliams`) . 
Upon checking the root of his user profile, a standard directory listing revealed a highly suspicious entry:
The room creator left a massive, ironically named file: definitely_definitely_..._not_a_flag.exe. Despite the .exe extension, its tiny file size (80 bytes) strongly suggests it is just a text file in disguise. This can be opened with command:
```
type .\definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_not_a_flag.exe
```

_Figure 18: Sus File_

<img width="622" height="371" alt="image-231" src="https://github.com/user-attachments/assets/c422213e-7c6a-408b-bf15-5c9181044b23" />

This yielded our flag successfully!

<ins>Next question was about the content of Top-Secret.</ins>
"What is the complete top secret?" Rather than clicking blindly through every directory on the system, we can utilize a recursive PowerShell search to scan the entire C:\ drive for any file or directory containing the word "secret".
```
Get-ChildItem -Path C:\ -Filter "*secret*" -Recurse -ErrorAction SilentlyContinue
```
💡 Tip: Appending -ErrorAction SilentlyContinue is crucial here; it forces the scanner to skip past system folders that our current user doesn't have permission to read, preventing our terminal from flooding with access errors.

_Figure 19: Find Secret_

<img width="627" height="253" alt="image-232" src="https://github.com/user-attachments/assets/15a0753b-3d3a-42d7-9783-521b00886b04" />

The search query hits perfectly, identifying an internal hidden folder path: C:\Program Files\Top Secret\top_secret.png

Now that we know where the target image is located, we need to bring it back to our Kali Linux environment to view and analyze it properly.
Navigate to the target directory and use Evil-WinRM's built-in download wrapper to pull the asset down across our management stream:
```
cd "C:\Program Files\Top Secret"
dir
download top_secret.png
```

_Figure 20: Download and open secret file_

<img width="624" height="578" alt="image-233" src="https://github.com/user-attachments/assets/3cdb2373-5753-43b9-bf34-0d8eb4a75292" />

_Figure 21: Open Secret.png_

<img width="593" height="587" alt="image-234" src="https://github.com/user-attachments/assets/ae631de7-2e3f-47de-a373-8949cf52737e" />

<ins>Opening up the exfiltrated top_secret.png on our Kali machine reveals a well-known community meme with the answer.</ins>

---

With the information gathered from our previous domain-wide compromises, we pivot to expanding our footprint on internal file shares. We targeted Steven Bradley's account (`sbradley`) using his credentials via SMB.

When attempting to connect to the target's SMB service using `impacket-smbclient`, we hit a standard active directory protection mechanism:
```
impacket-smbclient "raz0rblack.thm/sbradley@$target" -hashes aad3b435b51404eeaad3b435b51404ee:351c839c5e0....redacted
```
The server blocks direct authentication because the account flag enforces a mandatory password reset.

_Figure 22: Change Password_

<img width="625" height="111" alt="image-235" src="https://github.com/user-attachments/assets/c63d6065-3fab-44df-976d-e20bad2cbd0f" />

#### <ins>The Workaround: Administrative Password Reset</ins>
Since we already maintain an active Evil-WinRM session as the local Administrator, we can easily override this requirement. By using the native Windows network utility, we force a password update across the domain for Steven's account:
```
net user sbradley "Password123!" /domain
```

_Figure 23: Password changed_

<img width="628" height="74" alt="image-236" src="https://github.com/user-attachments/assets/6ad44724-0431-422c-9237-9ad542466098" />

Now that the account password is updated and active, we re-authenticate via impacket-smbclient using our newly assigned plaintext credentials:
```
impacket-smbclient "raz0rblack.thm/sbradley@$target"
```
Once inside the interactive SMB prompt, listing the available shares reveals a non-standard custom directory named trash. We choose to access it and recursively pull down all contained files: `mget *`

_Figure 24: Download all trash files_

<img width="612" height="400" alt="image-237" src="https://github.com/user-attachments/assets/c2e61839-7a01-4b2f-abf4-c12a7fe9cf7b" />

With the loot successfully exfiltrated to our local Kali machine, we begin parsing the contents starting with the plaintext chat log.

_Figure 25: Chat Log_

<img width="619" height="304" alt="image-238" src="https://github.com/user-attachments/assets/ed128511-f68c-4f7b-bfe8-6aa697d44cf9" />

This confirms that experiment_gone_wrong.zip holds a full backup of the Active Directory database created during an earlier exploit scenario.
Then we attempt to run a standard unzipping operation on the zip file confirming that the archive is heavily password protected.

_Figure 26: Archive Locked_

<img width="623" height="229" alt="image-240" src="https://github.com/user-attachments/assets/105d7bbc-e5c3-4099-a61b-96cde18bc95e" />

To break past the encryption layer, we process the zip file structure into an offline cracking format compatible with John the Ripper using zip2john:
```
zip2john experiment_gone_wrong.zip > johnpls
```

Next, we launch our brute-force attack mapping the extracted hash signature against the standard rockyou.txt wordlist:
```
john johnpls --wordlist=/home/kali/rockyou.txt
```

_Figure 27: Password Cracking_

<img width="629" height="155" alt="image-239" src="https://github.com/user-attachments/assets/970f9905-0a7b-4b0e-8944-a57024e240dd" />

The tool successfully identifies the matching plaintext string, allowing us to decrypt the archive contents and cross off What is the zip file's password? on our objectives board.

---

Now that we have proclaimed all the flags except 1, we shift focus toward towards the last one - Ljudmila Vetrova (lvetrova).
Instead of guessing a password, we utilize her raw NTLM hash (f220d3988deb3f516c....) extracted from our active database dumps to spawn a direct remote session using a classic Pass-the-Hash (PtH) technique:
```
evil-winrm -i $target -u lvetrova -H f220d3988deb3f516c73f........
```

_Figure 28: Login as lvetrova_

<img width="624" height="678" alt="image-241" src="https://github.com/user-attachments/assets/b85b0669-edf3-4037-9e35-b46dec9619f1" />

Just like our experience with prior user endpoints, we locate a custom XML object blueprint stored directly under lvetrova.xml.
We look at the serialized construction parameters;
The user configuration notes explicitly indicate that the solution string resides directly within the credential object wrapper.
Because we are executed entirely within her valid DPAPI validation environment, we deserialize the configuration inline to print our clean plaintext reward:
```
$credential = Import-Clixml -Path "C:\Users\lvetrova\lvetrova.xml"
$credential.GetNetworkCredential().Password
```

_Figure 29: Final Flag!_

<img width="622" height="599" alt="image-242" src="https://github.com/user-attachments/assets/1cea68db-0fb6-47e6-b3b4-de047eaa82bb" />

The structural decryption loop successfully processes the flag string, completing our final user account milestone for the Razorblack deployment!
