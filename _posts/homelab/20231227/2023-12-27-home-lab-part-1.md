---
layout: post
title:  "Virtual Home Lab with Hyper-V (Part 1)"
categories: [homelab]
tags: [hyper-v, pfsense]
---

Hey Readers!

This is my first time writing a tech guide post. So I hope you guys would spare me on my grammars or if the content is not properly organized.

Let's start!

---

# Part 1 of 3 - Creating the Hyper-V components
This series of post serves to help the future me or anyone else who would like to set up a virtualised home lab. 

This guide is part 1 of 3 for setting up a virtualised home lab with Hyper-V.

## Prerequisites 
- Windows 10 PC with Hyper-V Enabled

For going through this lab, I would recommended to have at least 8 virtual cores, 16GB RAM and 50GB disk storage.

## Why?
A virtualised home lab would be useful in the event that one need to download vulnerable images for practicing pentest skills. At the same time provide a level of isolation for the vulnerable images. We can choose to use cloud technologies for this but it would incur some costs. Thus, setting up in a virtualised home lab environment would help to save some of the costs incurred.

## The idea
The objective of this setup is to isolate the virtual machines created in our lab from our host. We will only allow web traffic to leave the network isolated network. In the event of a malware infection, we want to reduce the chance of worms propagating from our VMs to our host. However, this setup would not block be able to block a malware communicating back to the C2 server via HTTP/HTTPS (yet). But we can add additional controls to reduce the risk of it happening. 

So now let us first take a look at the end state of our network.
![Desktop View](/images/homelab/5016ab07-d76f-437f-9f72-6044240972f4.png)

This would allow us to:
- Add additional VLANs
- Add additional VMs into a VLAN
- Allow/Deny traffic between the VLANs
- Internal machines are able to access the internet
- Blocking of ingress/egress traffic using the pfSense firewall

A short explanation for the above setup:
- `192.168.10.0/24` network is a trusted network. We can put our VMs such as a kali/parrot box here.
- `192.168.20.0/24` network is a "sandbox" network and untrusted. We can put VMs downloaded from Vulnhub here. This network will be restricted to access any networks.
- `10.0.0.0/24` is the network connecting the WAN interface of our firewall and the management host.
- `192.168.100.0/24` is the local network of the management host. This network should be able to access internet.

## The components
We will be creating the virtual components
- 3x VM
	- 1x router/firewall
	- 2x client
- 2x Hyper-V Switch

## Creating the Hyper-V Components 
For the end devices, feel free to use any ISO image. I will be using Windows 10 image. If you already have existing Windows VM, you can use them too.
For the router/firewall, we will be using pfsense 2.7.1 (latest at the time of writing). However, you can use the updated image from [here](https://www.pfsense.org/download/) and extract the ISO image.

### Creating the Virtual Switches
We will start by creating 1 internal and 1 private switch. 

1. Launch Hyper-V Manager.
2. Select your Hostname on the left panel. An actions panel will appear on the right of the window. 
3. Click on ***Virtual Switch Manager*** on the right panel. In the popup window, select ***Internal*** and click on ***Create Virtual Switch***. 
4. For the name of the switch, we will call it `Lab Internal`. Click on *Ok* to create the switch. You will see something like below once the switch has been created.

	![Desktop View](/images/homelab/dbaf3a43-941e-4293-a276-d0ea40a6b0b0.png)
5. We will create another private switch. To do so, repeat the same step 3 and 4	. But this time, select the ***Private*** option instead. Name the switch as `Lab Private`.
6. Ensure you have the 2 switches that you created and the settings are correct.

![Desktop View](/images/homelab/3df67de1-6dc6-4ddc-af72-54c231ff8e49.png)

### Creating the pfsense firewall
Next, we will create the pfsense firewall. 
1. Select ***New > Virtual Machine*** in the *Actions* panel at the right. 
2. In the *Specify Name and Location* window, we will name it as `Firewall`. Feel free to leave the location as default unless you want to store the VM in a different location. 
3. Use the below settings to proceed with the creation of the VM. 

	**Hyper-V VM Settings for pfsense firewall**
	```
	Generation: 1
	Memory: 2048 MB
	Connection: Not Connected
	Create a Virtual Hard Disk > Size 10GB
	Installation Option: Install an Operating System from a bootable image file > Browse > (Select the PFSense ISO downloaded previously)
	```
4. Once your firewall VM has been created, start the firewall by double clicking on it in the Virtual Machines list and clicking *Start* on the window appeared. 
5. At the Welcome page, select *Install*. In the partitioning page, choose your partitioning choice. I selected Auto (ZFS). 
6. In the Configure Options, select Install.
7. In the page to choose a disk, use Spacebar to select the option and click enter to proceed.

	![Desktop View](/images/homelab/3de9cc97-e4a8-462e-baa3-b445c1afa4d6.png)

8. Select yes and proceed with the installation. Reboot after the installation has completed.

**Adding a network interface**

After the reboot, we will need to shut down the Firewall to add an extra network interface.

1. In the Hyper-V manager window, turn off the firewall by right clicking on the firewall and click ***Turn Off***. 
2. Open the settings window by right clicking on the firewall and click ***Settings***.
3. In the popup window, select ***Add Hardware*** and in the right panel, select ***Network Adapter*** and click *Add*. 
4. We should have 2 Network Adapter as shown below after addition of a network adapter.

	![Desktop View](/images/homelab/33eb289e-b16d-4907-bab6-f9fe4c9be9c6.png)

5. For the first Network Adaptor, set the *Virtual Switch* to ***Lab Internal***. For the second Network Adaptor, set it to ***Lab Private***.

	![Desktop View](/images/homelab/413bd471-578e-48d6-9e2e-b469aed69ac5.png)


We will power up our firewall next, but first eject the ISO disc by selecting ***IDE Controller 1 > DVD Drive*** in the *Hardware Panel* and choose ***None*** in the Media option. Click on *Ok* to close the settings window and power up the firewall.

**Configuring the firewall**

After the firewall has been started, it will ask for a series of questions to configure the firewall network interfaces. Use keyboard to input the options provided.
- When prompted to setup vlans, we will not configure it for now so hit ***n*** and enter.
- When prompted to select **WAN** interface name, use ***hn0***.
- When prompted to select **LAN** interface name, use ***hn1***
- In the confirmation of interfaces, ensure WAN is point to HN0 and LAN is pointing to hn1 before proceeding.

It will take awhile for the pfSense to configure its firewall, but upon completion, we will be greeted with a page to select options.

![Desktop View](/images/homelab/27cfe1a8-c1de-47d1-9226-1dad6567a769.png)

We will keep this VM running while we proceed to create the other clients.

### Creating the Windows Clients

For the client OS, it can be of any OS of your choice. Thus, I will briefly cover through the installation for the Windows VM.

We will create 2 VMs with the name `Windows 10 (vlan10)` and `Windows 10 (vlan20)`.  Download the Windows 10 ISO image from Microsoft and proceed with the installation of the OS. For the creation of the VM in Hyper-V, use the settings below or feel free to change the any of the settings if needed. 

**VM 1 - Windows 10 (vlan20)**
```
Name: Windows 10 (vlan10)
Generation: 1
Memory: 4096MB 
Connection: Lab Private
Virtual Hard Disk Size: 20GB
Installation Source: Windows 10 ISO image
```

**VM 2 - Windows 10 (vlan20)**
```
Name: Windows 10 (vlan20)
Generation: 1
Memory: 4096MB
Connection: Lab Private
Virtual Hard Disk Size: 20GB
Installation Source: Windows 10 ISO image
```

> If you already have a VM running, add a network adapter to the VM via the VM's settings window and connect it to `Lab Private`. Just remember which client is the one that you want it to be in either of the VLANs!
{: .prompt-tip }

Once both Windows VMs have been set up, you can see that we do not have any Internet but we have been assigned an IP address. This is because the DHCP service is enabled by default in pfsense and it has issued the VMs IP addresses.

![Desktop View](/images/homelab/8361c67a-1bb1-484b-a3d6-73f087fd8ea2.png)

We can see that the other VM also has connectivity to the pfsense firewall.

![Desktop View](/images/homelab/0189f3e1-a3aa-45b5-bac6-355bb1e98bc3.png)

We have finish setting up the components for our network. In the next part of this guide, we will proceed to set up the VLANs. 