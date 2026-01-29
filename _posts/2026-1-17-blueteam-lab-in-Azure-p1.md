---
layout: single
classes: wide
title:  "BlueTeam Lab - Part One: The Build"
date:   2026-01-17 20:26:59 -0500
categories: homelab blueteam
---

This is the first in a series of posts that will cover building out a VM lab network to practice blue team concepts. I recently did the BTL1 certification course from Security Blue Team and wanted to build out my own lab and configure a SIEM myself, and also to test and tune custom detection and alerts. The lab network will use a Splunk server for SIEM, which will be monitoring a very minimal AD domain containing 1 DC and 1 Workstation. The network will also have an Ubuntu VM that will serve as a machine to launch attacks from. I'll be testing various enumeration, persistence, and lateral movement techniques from here, which I'll write detections for in Splunk.

I'm doing this in Azure, but you can take these concepts and run with them using VMs hosted wherever you like. If you haven't used Azure and don't have somewhere to host them, you can get a free subscription that gets you 4 regional vCPUs and $200.00 credit to use within 30 days. Even without that, it'll be pretty cheap if you stick to B-series VMs and spin down the lab when you're done.

Speaking of subscriptions, I'm also going to be using the Splunk Enterprise trial, which gives you access to Splunk Enterprise for 60 days at a 500MB/day indexing limit. After the 60 days runs out, it just turns into Splunk Free, which removes a lot of the Enterprise features but keeps the 500MB/day limit.

Here's the rough plan for the lab.

```
Azure VNet (10.0.0.0/24)
│
├── DC01 - 10.0.0.4 (Windows Server 2025)
│   └── Active Directory + DNS
│
├── WKST-01 - 10.0.0.5 (Windows 10 Pro)
│   └── Domain-joined user endpoint
│
├── SPLUNK - 10.0.0.6 (Ubuntu Server 24.04 LTS)
│   └── SIEM (Splunk Enterprise)
│
└── PENTEST - 10.0.0.7 (Ubuntu Pro 24.04 LTS)
    └── Attacker simulation
```

# Azure Prep

We'll do some prep work in Azure to get started. If you're doing this in a self-hosted lab you can skip ahead. I'll just cover the process for creating the Splunk server VM. Azure is huge, but we're just creating 4 VMs on an isolated virtual network. I'll cover creation steps for the Splunk server in Azure Portal, which is a browser-based GUI for managing Azure resources, with the assumption you already have an account set up.

1. In Azure Portal, Create a Resource  -> Virtual Machines -> Create.
2. Under basics, create a new Resource group for our stuff: `rg-blueteamlab`.
3. Pick a hostname, region, and Image. For Splunk, select Ubuntu Server 24.04 LTS.
4. Select a size for the VM. You can stick to B-Series to save money, but you might find that they aren't available in the region you selected, so try using a different region if you have trouble. Just keep all the VMs in the same region.
5. Create an admin user, ideally with an SSH keypair. We'll grab the private key later.
6. Change Public inbound ports to None. We'll set this up later using a Network Security Group (NSG).
7. Go to the Networking tab. Create a Virtual Network `vnet-blueteamlab (10.0.0.0/16)` and a Subnet `(10.0.0.0/24)`. Our VMs will live on this subnet.
8. (Optional) Create a public IP for the VM. This will allow us to directly access our created VMs over the internet. I did this for Splunk because I wanted to access the web interface easily from my computer. We'll lock this down with a NSG in the next step. Public IPs have a nominal cost, like $0.005/hour for a Standard Static IPv4 address.
You'll need an entry point into the lab itself. Giving one of the VMs a public IP will let you use it as a jump box; letting you SSH/RDP into it and then pivoting from there to the machine you need on the internal network. 
For example, to get to the attack simulation machine (PENTEST), I can go `My Computer -> SSH into SPLUNK -> SSH into PENTEST`.
There are also other ways to do this. You can configure an Azure Bastion Host to do this without a public IP at all. Another way would be to configure a site-to-site VPN connection, but that's more complex and costly for what we're doing here.

9. Under Configure network security group, select Advanced. Create a new Network Security Group. We'll want to lock down outside access so that various internet scoundrels won't try to bruteforce us.

For Splunk, create 2 rules:

- Allow inbound SSH on port 22 from your public IP.
- Allow inbound HTTP on port 8000 from your public IP.

This will allow you to directly connect to the VM via SSH and work with Splunk web from your computer (assuming you host on port 8000). It will only allow connections originating from your public IP. 

Your public IP won't be a great source to restrict inbound traffic on if you're on a college campus or something. Consider whether using an Azure Bastion Host might be better for your needs.

Splunk also creates some default rules it does not show you in the VM creation screen, but are visible under the network settings for the VM after is has been created.

![](/img/blueteamlab/partone/Pasted image 20260119163250.png)

These are documented [here.](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)

10. Under Management, enable Auto-Shutdown and specify a time, just in case you forget to spin down the lab after you're done.

11. Go to Review + create and create the VM.

12. Once the VM is created, grab the SSH private key or an RDP file from the VM options -> Connect -> Connect. 


# Splunk SIEM Server - Splunk

For the Splunk server, we'll need to do the following:

- Download and install Splunk Enterprise
- Configure receiving on port 9997
- Create some indexes
- Install Splunk Add-on for Microsoft Windows

## Installation

We're ready to stand up Splunk on the VM we just created. Before we begin, make sure the system time is correct using `timedatectl`.

Next, download Splunk Enterprise. Go [here](https://www.splunk.com/en_us/download/splunk-enterprise.html) and create an account. You should be able to download it straight onto your Ubuntu server using the wget URL they provide.

`splunker@Splunk:~$ wget -O splunk-10.0.2-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.0.2/linux/splunk-10.0.2-linux-amd64.deb"`

Install the Splunk package you just downloaded:

`splunker@Splunk:~$ sudo dpkg -i splunk-10.0.2-linux-amd64.deb`

It will ask you to create an admin user to access the web interface, do this.

I experienced an issue where the first-time setup was not able to write directories to /opt/splunk. To resolve this, I gave the splunk user that was created during the install ownership over the Spunk install directory and restarted the splunk process. 

```
splunker@Splunk:~$ sudo chown -R splunk:splunk /opt/splunk
splunker@Splunk:/opt/splunk/bin$ ./splunk start
splunker@Splunk:/opt/splunk/bin$ sudo /opt/splunk/bin/splunk enable boot-start -user splunk --accept-license
splunker@Splunk:/opt/splunk/bin$ sudo systemctl start splunk
```

Documentation for first-time setup, in case you run into trouble: https://help.splunk.com/en/splunk-enterprise/get-started/install-and-upgrade/9.3/start-using-splunk-enterprise/start-splunk-enterprise-for-the-first-time

## Barebones Configuration

Once Splunk is running, we should be able to access it via web browser on port 8000 at `http://<splunk-IP>:8000`. You should see a login page.

![](/img/blueteamlab/partone/Pasted image 20260119215706.png)

Log in with the admin credentials you created during the installation. Once you are in, you need to configure a receiver port to receive forwarded logs. By default, this is TCP port 9997.

To do this, go to Settings → Forwarding and Receiving → Configure Receiving → Add New Port: 9997. 

![](/img/blueteamlab/partone/Pasted image 20260119220327.png)

You can verify that the receiver is running with `ss -ano | grep 9997` in your SSH session.

```
splunker@Splunk:~$ ss -ano | grep 9997
tcp   LISTEN    0      128                                                  0.0.0.0:9997              0.0.0.0:*       
```
Next, we'll configure a couple of indexes. Indexes in Splunk are logical containers that hold the events that we forward. Later on, we can assign certain log events to go to certain indexes, which will make them easier for us to search.

For now, I'll just add 2 indexes, `wineventlog` and `sysmon`. 

To do this, go to Settings → Indexes → New Index. 

The New Index screen is intimidating, but all you need to do right now is fill in the Index Name field.

![](/img/blueteamlab/partone/Pasted image 20260119221213.png)

Your new indexes will get added onto the system defaults at the bottom. You can also see event counts here later.

![](/img/blueteamlab/partone/Pasted image 20260119221505.png)

Next, install Splunk Add-On for Microsoft Windows. You can do this from the main Splunk Enterprise page -> Apps -> Find More Apps. 

This isn't an app exactly, but it improves the way that Splunk handles Windows events. It will automatically extract important fields and also normalizes events to map them to Splunk's CIM (Common Information Model). This makes it easier to correlate events from different sources later.

![](/img/blueteamlab/partone/Pasted image 20260119222905.png)

That's the barebones config done. Now to get the AD domain up so we can start forwarding some logs.

# Domain Controller - DC01

For the DC, we'll need to do the following:

- Install Active Directory Domain Services, promote the server to a DC
- Enable Advanced Auditing via group policy
- Install and configure Sysmon, deploy via group policy
- Install and configure Spunk Universal Forwarder

## Domain Creation

I'm starting from the assumption that you have an RDP connection to the server as an admin user. To start an AD domain and turn this server into a Domain Controller, install Active Directory Domain Services and promote it. Make sure it has a static IP set. In the Server Manager GUI, Install the AD DS Role, and then click the yellow notification flag in Server Manager and promote it to a domain controller. Or do it through PowerShell:

```
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName btlab.local
```

I called this domain `btlab.local`.

## Advanced Auditing

The default Windows auditing policy is limited to 9 categories:

![](/img/blueteamlab/partone/Pasted image 20260125153714.png)

Advanced Auditing expands on this and is going to vastly improve the usefulness of logs that we can feed into Splunk. It lets us pick specific subcategories of events to log. For example, under the default policy, logon events would all get lumped under "Logon Events", but Advanced Auditing allows us to seperate them out into interactive logons, kerberos auth, etc. 

Dialing in the Advanced Auditing settings correctly is going to improve the signal to noise ratio of the events we injest.

We're going to set this up through Group Policy and apply it to the DC and any other workstations we add to `btlab.local`. I set up an OU for Workstations and made another version of the policy to apply to them, as we may want to prioritize different events for each.

![](/img/blueteamlab/partone/Pasted image 20260112224517.png)

To create the policy, open Group Policy Management Editor, create the policy, link it to either the Domain Controllers or Workstations OU, and Edit it. The Audit Policy settings are under Computer Configuration -> Windows Settings -> Security Settings -> Advanced Audit Policy Configuration -> Audit Policies.

![](/img/blueteamlab/partone/Pasted image 20260112224732.png)

For what we're actually going to enable, I used the baseline recommendations from Microsoft outlined [here](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations?tabs=winserver), plus `Audit Kerberos Authentication Service` and `Audit Kerberos Service Ticket Operations` under Account Logon. This page has tabs for Windows Client and Windows Server, which I used to configure the policies for the Workstations OU and Domain Controllers OU.

Further documentation about these subcategories can be found [here.](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn452415(v=ws.11))

Once you're done, verify the policies are applied:

`auditpol /get /category:*`

## Sysmon

The other way we will improve our logs is through Sysmon. Sysmon works alongside the default windows event logs and generates more detailed events regarding process creation, network connections, and file changes. It can also be configured to include/exclude different event types, which we can tweak to fit our needs.

I'm going to be using the community config maintained by SwiftOnSecurity, found [here.](https://github.com/SwiftOnSecurity/sysmon-config) There are other community configs like [this one](https://github.com/olafhartong/sysmon-modular) by Olaf Hartong.

I will make one change to this in order to exclude the Splunk Universal Forwarder Directory, which we will install in the next step. Add these lines in the XML config file under `!--SYSMON EVENT ID 1 : PROCESS CREATION [ProcessCreate]-->` -> `<ProcessCreate onmatch="exclude">` .

```
<Image condition="begin with">C:\Program Files\Splunk\bin\</Image>
<Image condition="begin with">C:\Program Files\Splunk\bin\</Image>
<!--SECTION: Splunk-->          
<ParentImage condition="is">C:\Program Files\Splunk\bin\splunkd.exe</ParentImage>
<ParentImage condition="is">C:\Program Files\Splunk\bin\splunk.exe</ParentImage>
<Image condition="begin with">C:\Program Files\SplunkUniversalForwarder\bin\</Image>
<Image condition="begin with">C:\Program Files\SplunkUniversalForwarder\bin\</Image>
<ParentImage condition="is">C:\Program Files\SplunkUniversalForwarder\bin\splunkd.exe</ParentImage>
<ParentImage condition="is">C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe</ParentImage>
```

I'm also going to set up a group policy to automatically install and configure this on our future workstations. I did this by creating a network share called `Software$` on the C:\ drive of DC01, and placing sysmon.exe, the modified sysmon config, and a PowerShell script named `Install-Sysmon.ps1` in a subfolder called `Sysmon`. 

![](/img/blueteamlab/partone/Pasted image 20260125173321.png)

The group policy will call the script from this share and install sysmon using the config file and executable in the share.

```
## Install-Sysmon.ps1
$SysmonExe = "\\DC01\Software$\Sysmon\Sysmon.exe"
$SysmonConfig = "\\DC01\Software$\Sysmon\sysmonconfig-export.xml"

# Check if Sysmon service exists
$service = Get-Service -Name Sysmon -ErrorAction SilentlyContinue

if (-not $service) {
    # Install Sysmon
    Start-Process -FilePath $SysmonExe `
        -ArgumentList "-accepteula -i `"$SysmonConfig`"" `
        -Wait
}
else {
    # Update config if already installed
    Start-Process -FilePath $SysmonExe `
        -ArgumentList "-c `"$SysmonConfig`"" `
        -Wait
}
```

I'll just install it manually on the DC.

![](/img/blueteamlab/partone/Pasted image 20260112224418.png)

Sysmon is now installed. Sysmon events can be seen in Event Viewer under `Applications and Services Logs -> Microsoft -> Windows -> Sysmon -> Operational`.

![](/img/blueteamlab/partone/Pasted image 20260128222721.png)

## Splunk Forwarder

Next, we'll need to install and configure Splunk Universal Forwarder. This is the piece that forwards our logs to the Splunk server. There are also other log forwarders that can be used, like WEF for Windows or NxLog for Linux.

Download SUF from the Splunk site: https://www.splunk.com/en_us/download/universal-forwarder.html

I'll deploy this manually; this can also be done through a Splunk Deployment server, group policy, or other third-party tools. 

You will need to specify the IP and receiving port of the Splunk server set up earlier.

![](img/blueteamlab/partone/Pasted image 20260114215912.png)

![](/img/blueteamlab/partone/Pasted image 20260114220455.png)

We also need to create an `inputs.conf` file, which determines which events get forwarded to Splunk. The following will forward Security, System, Application, and Sysmon logs.

```
[WinEventLog://Security]
index = wineventlog
disabled = false

[WinEventLog://System]
index = wineventlog
disabled = false

[WinEventLog://Application]
index = wineventlog
disabled = false

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
disabled = false

```

Copy this file to `C:\Program Files\SplunkUniversalForwarder\etc\system\local` and restart the Splunk service.

Logging into Splunk, you should start seeing events come in from the DC.

![](/img/blueteamlab/partone/Pasted image 20260128225446.png)

# User Workstation - WKST-01

For the user workstation, we'll need to do the following:

- Join it to our domain
- Verify that our group policies work
- Install Splunk Universal Forwarder

Follow standard steps to join a computer to an AD domain to join WKST-01 to `btlab.local`. Nothing out of the ordinary here. Make sure that DNS is pointed to DC01.

To apply the group policies for Advanced Audit Logging and Sysmon we created earlier, you can either reboot or run `gpupdate /force`. 

Test that sysmon is installed on the machine (check installed programs or event viewer) and that advanced audit logging is enabled (auditpol).

Since I didn't automate installing SUF, we'll need to install that manually using the same steps as DC01. Make sure to create `inputs.conf` and restart the Splunk service.

Check Splunk to make sure logs are getting there:

![](/img/blueteamlab/partone/Pasted image 20260128230415.png)

# Attacker Simulation - Pentest

For the attack simulation machine, we'll need to do the following:

- Ensure we can reach targets
- Test that attack activity reaches Splunk
