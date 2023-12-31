---
layout: post
title:  "Virtual Home Lab with Hyper-V (Part 3)"
categories: [homelab]
tags: [hyper-v, pfsense]
---

# Part 3 of 3 - Allowing Internet connections for specifc VLANs 

This guide is part 3 of a 3 part series for setting up a virtualised home lab with Hyper-V.

In this part, we will be covering the following:
- Adding firewall rules to allow clients in VLAN10 to reach clients in VLAN20
	- Traffic allowed from VLAN 10 to VLAN 20 can only be HTTP/HTTPS and ICMP traffic
- Configuring NAT on our management host to translate internal traffic
- Adding firewall rules to allow clients in specific VLANs (VLAN 10) can access the internet
- Restricting access to anywhere for VLAN20 clients


## Adding firewall rules to allow clients in VLAN10 to reach clients in VLAN20

Before we start, let's allow ICMP ECHO on the Windows Defender firewall on the Windows client in VLAN20. You may choose to skip this step, but it would be difficult to troubleshoot in case you meet with any error.

> If you are on a *nix based client or you have used a VM that allows ICMP ECHO from another network, you may choose to skip this part and proceed to the .

### Allowing ICMP ECHO on Windows Client
1. On the VLAN20 windows client, go to ***Windows Defender Firewall > Advanced Settings***. 
2. Right click on the ***Inbound Rules***  at the right panel and select ***New Rule...***. 
3. Use the following setting to create the rule to allow ICMP ping. For settings not listed, you can use the defaults.

	```
	Rule Type: Custom
	Protocol Type: ICMPv4
	Name: Allow Ping For Now
	```

4. Let's check if the ping is working from our Firewall. Go to the Firewall VM, choose option 8 and you will get a command line input.
![Desktop View](/images/homelab/038af1ef-0126-4cd0-8acd-74570bf05a35.png)

5. From the command line input, run a ping test to ensure that you can ping it. You should be able to ping it.
```
ping -c 2 192.168.20.100
```
> Do note that the IP above may be different in your case.

### Adding of firewall rules on pfsense 
In the previous section, we did a ping test from `Firewall` to the `Windows 10 (vlan20)` client. If we try to ping `Windows 10 (vlan20)`  from our `Windows 10 (vlan10)`, we will not be able to do so. This is because there are no rules in  `Firewall` currently to allow network traffic in/out for both VLAN10 and VLAN20. (With the exception of the rule we created in part 2 to allow VLAN10 to access web configurator.)

VLAN10 Client pinging VLAN20	

![Desktop View](/images/homelab/fce06b3e-383c-43dc-ad1e-8673df88962b.png)

Thus, we will now create a rule to allow any clients in VLAN10 to ping VLAN20. 
1. Access web configurator from `Windows 10 (vlan10)` and go to ***Firewall > Rules > OPT1***. 
2. Click on any of the ***Add*** button to add a rule with the following settings.
	```
	Address Family: IPv4
	Protocol: ICMP
	ICMP Subtypes: any
	Source: OPT1 Subnets
	Destination: OPT2 Subnets
	```
	> Note: The rule above allows any types of ICMP traffic from OPT1 subnets to OPT2 subnets only.
3. Save the rule and apply changes.
4. Now go back to `Windows 10 (vlan20)`, we will issue the ping command again. We are able to ping the client in VLAN20 now.

	![Desktop View](/images/homelab/1809825b-b57c-4dd4-af3b-98a0a954a3f6.png)
5. For allowing HTTP/HTTPS traffic from VLAN10 to VLAN20, it will be covered in the [next section](#adding-firewall-rules-to-allow-clients-in-specific-vlans-vlan-10-to-access-the-internet)


## Configuring NAT on our management host to translate internal traffic 

Let us do a stocktake of what we have did so far. Referencing the network diagram below, we have configured the interface of the firewall that connects to `Lab Private`. For the interface to `Lab Internal` and beyond, it is not configured yet. We will need to configure NAT on our managemnet host to allow the internal network to access the Internet.

![Desktop View](/images/homelab/5016ab07-d76f-437f-9f72-6044240972f4.png)
> For the network `192.168.100.0/24` network, it is the home network of our management host


As you can see, we have not configured the `10.0.0.0/24` network between the firewall and management host. So we will start with this step.

### Set IP address on management host's interface to Lab Internal
We will first assign a static IP address on the inteface to `Firewall` on our management host. 
1. On the management host, start a powershell as administrator.
2. Run the following command to retrieve the interface index of the network adapter connected to the `Lab Internal` switch created in Part 1. Note the value of `ifIndex`.
	```
	Get-NetAdapter -Name "vEthernet (Lab Internal)" | select ifindex
	```
	![Desktop View](/images/homelab/1060cdb6-eb02-44d7-a62e-da3bced3c45d.png)

3. Next, assign a static IP address to this interface with the following command after replacing the `ifIndex` retrieved from step 2. In my case, the value is `22`.
	```
	New-NetIPAddress -InterfaceIndex <ifIndex> -IPAddress 10.0.0.1 -PrefixLength 24
	```
![Desktop View](/images/homelab/3f3af70c-f0e4-44ed-9813-afa11c9f291c.png)
4. Run `ipconfig` command and we can see that the IP address has been set.
![Desktop View](/images/homelab/f3cf208a-1e23-4567-be3a-79fbe3bb7826.png)

### Set IP address on firewall's interface to Lab Internal
We will now set an IP address on the `Firewall` interface to Lab Internal. This interface is also the WAN port on our `Firewall`.  
1. Back to the `Windows 10 (vlan10)` client, access the web configurator and go to ***Interfaces > WAN***. 
2. Modify the configurations to be as the settings below:
	1. Enable: Check ***Enable Interface***
	2. IPv4 Configuration Type: ***Static IPv4***
	3. IPv6 Configuration Type: ***None***
	4. IPv4 Address: ***10.0.0.3/24***
	5. IPv4 Upstream Gateway: Add a new gateway with settings below.
		```
		Default: Checked "Default Gateway"
		Gateway Name: WANGW
		Gateway IPv4: 10.0.0.1
		```
	6. Click **Add** to add the gateway followed by **Save** to update the interface. 
	7. Click on **Apply** to apply the configurations.

### Disabling Gateway Monitoring
pfsense allow gateway monitoring to switch between gateways in the event that a gateway has failed. However, we do not need this in our homelab setup. Thus, we will be disabling this feature. This configuration would prevent the firewall from continuously pinging our management host.
1. Go to ***System > Routing***. 
2. In the **Gateways** tab, look for the gateway you created previously and click on the edit icon on the last column. 
3. Scroll down to `Gateway Monitoring` and check the `Disable Gateway Monitoring`.  
4. Save and apply the configurations. 
![Desktop View](/images/homelab/e3eaa25d-2def-4531-b545-52ae77a6af78.png)

### Adding NAT on management host
Next, we will add a NAT on the host so that traffic coming from the firewall can be translated and share the management host IP address. This would allow internal clients to access Internet. Do note that the management host should already have Internet access.

1. Open a powershell as administrator and run the following command.
	```
	New-NetNat -Name "NAT for Lab Internal Switch" -InternalIPInterfaceAddressPrefix "10.0.0.1/32"
	```
	![Desktop View](/images/homelab/fe76f0b5-6c6a-4fc1-8fb7-f3fe8e563d1e.png)
2. Then, go back to `Firewall` console and try to ping www.google.com. You should be able to do it.  If not, wait for awhile and try again.
	
	![Desktop View](/images/homelab/d3130d13-04d2-4ac4-a981-f5672e2f8933.png)

## Adding firewall rules to allow clients in specific VLANs (VLAN 10) to access the internet
Next, we will add firewall rules to allow VLAN10 clients to access Internet via HTTP/HTTPS traffic. To access a website on the Internet e.g. www.google.com, we would need to allow clients in VLAN10 to make DNS query to `Firewall` and allowing HTTP/HTTPS traffic out.  

To do so, we will add 1 alias and 2 rules to allow HTTP/HTTPS to anywhere and DNS traffic to only the local IP.

1. In the pfsense web configurator, go to ***Firewall > Aliases***. Click on **Add**. Create an alias with the following settings
	```
	Name: web_traffic
	Description: alias for web traffic
	Type: Port(s)
	Port(s): 80, 443
	```
2. Save and apply the changes.
3. Go to ***Firewall > Rules > OPT1***. 
4. Click any of the **Add** button to add a firewall rule with the following settings to allow HTTP/HTTPS to anywhere
	```
	Action: Pass
	Interface: OPT1
	Protocol: TCP
	Source: OPT1 Subnets
	Destination: Any
	Destination Port Range: Fill in `web_traffic` in the first `Custom` text field
	```
	> Note: As we are allowing web traffic to OPT2 network, the above rule would also allow web traffic to OPT2 network. Do also note that this rule would allow HTTP traffic to `Firewall` and any traffic reachable by the management OS.  Since this rule allows web traffic to `Firewall`, we can also delete the previously created rule for accessing the webconfigurator by clicking on the trash icon on the right.
![Desktop View](/images/homelab/cf823e15-9351-4ee0-a2da-86a5a4c5f695.png) 
5. Click on **Save** to save to add the rule.
6. Click any of the **Add** button to add a firewall rule with the following settings to allow DNS traffic to `Firewall`.
	```
	Action: Pass
	Interface: OPT1
	Protocol: TCP/UDP
	Source: OPT1 Subnets
	Destination: OPT1 Address
	Destination Port Range: Select DNS(53) from dropdown
	```
7. Click on **Save** to add the rule.

8. Apply the configurations. You should be able to access websites on the internet now from the `Windows 10 (vlan10)` client or any clients from VLAN10 right now. 

## Restricting access to anywhere for VLAN20 clients
If you have guessed that VLAN20 clients were not able to access anywhere, you are correct! Thus, there would be nothing to configure in this section.

This is due to the implicit deny behaviour in pfsense firewall. If there were no rules defined for permitting traffic to flow through, it would be implicitly deny. (Deny by default) 

## Conclusion
We have finish configuring a basic setup for our virtual home lab with Hyper-V. 

Just to recap, the way this network was design is so that we can spin up new VMs in the VLAN20 for untrusted VMs. In This can be VM images downloaded from [VulnHub](https://www.vulnhub.com), or even create a FlareVM instance for performing malware analysis in VLAN20. In VLAN10, we can create trusted VM such as a Kali/Parrot VM that can be use to reach VLAN10. At the same time, allow HTTP/HTTPS connection to the internet.

That's it! If you have read through all the way till now, thank you for your time and attention! :smile: I hope you had a great time here!

Do remember to change your default password on your firewall!