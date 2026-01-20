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
- Configure recieving on port 9997
- Create some indexes
- Install Splunk Add-on for Microsoft Windows

## Installation

We're ready to stand up Splunk on the VM we just created. Before we begin, make sure the system time is correct using `timedatectl`.

Next, download Splunk Enterprise. Go [here](https://www.splunk.com/en_us/download/splunk-enterprise.html) and create an account. You should be able to download it straight onto your Ubuntu server using the wget URL they provide. The one below will not work for you.

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

Once Splunk is runnning, we should be able to access it via web browser on port 8000 at `http://<splunk-IP>:8000`. You should see a login page.

![](/img/blueteamlab/partone/Pasted image 20260119215706.png)

Log in with the admin credentials you created during the installation. Once you are in, you need to configure a reciever port to receive forwarded logs. By default, this is TCP port 9997.

To do this, go to Settings → Forwarding and Receiving → Configure Receiving → Add New Port: 9997. 

![](img/blueteamlab/partone/Pasted image 20260119220327.png)

You can verify that the receiver is running with `ss -ano | grep 9997` in your SSH session.

```
splunker@Splunk:~$ ss -ano | grep 9997
tcp   LISTEN    0      128                                                  0.0.0.0:9997              0.0.0.0:*       
```
Next, we'll configure a couple of indexes. Indexes in Splunk are logical containers that hold the events that we forward. Later on, we can assign certain log events to go to certain indexes, which will make them easier for us to search.

For now, I'll just add 2 indexes, `wineventlog` and `sysmon`. 

To do this, go to Settings → Indexes → New Index. 

The New Index screen is intimidating, but all you need to do right now is fill in the Index Name field.

![](img/blueteamlab/partone/Pasted image 20260119221213.png)

Your new indexes will get added onto the system defaults at the bottom. You can also see event counts here later.

![](img/blueteamlab/partone/Pasted image 20260119221505.png)

Next, install Splunk Add-On for Microsoft Windows. You can do this from the main Splunk Enterprise page -> Apps -> Find More Apps. 

This isn't an app exactly, but it improves the way that Splunk handles Windows events. It will automatically extract important fields and also normalizes events to map them to Splunk's CIM (Common Information Model). This makes it easier to correlate events from different sources later.

![](img/blueteamlab/partone/Pasted image 20260119222905.png)

That's the barebones config done. Now to get the AD domain up so we can start forwarding some logs.

# Domain Controller - DC01

For the DC, we'll need to do the following:

- Install Active Directory Domain Services, promote the server to a DC
- Enable Advanced Auditing via group policy
- Install and configure Sysmon, deploy via group policy
- Install and configure Spunk Universal Forwarder
- Create some test users and groups

I'm starting from the assumption that you have an RDP connection to the server as an admin user. To start an AD domain and turn this server into a Domain Controller, install Active Directory Domain Services from either the Server Manager GUI or PowerShell.


# User Workstation - WKST-01

For the user workstation, we'll need to do the following:

- Join it to our domain
- Verify that our group policies work
- Install Splunk Universal Forwarder

# Attacker Simulation - Pentest

For the attack simulation machine, we'll need to do the following:

- Ensure we can reach targets
- Test that attack activity reaches Splunk
