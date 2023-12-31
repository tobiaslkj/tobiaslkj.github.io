---
layout: post
title:  "Virtual Home Lab with Hyper-V (Part 2)"
categories: [homelab]
tags: [hyper-v, pfsense]
---

# Part 2 of 3 - Configuring pfSense for VLANs

This guide is part 2 of a 3 part series for setting up a virtualised home lab with Hyper-V.

In this part, we will be setting up the VLANs for the components we configured in part [one]({% post_url 2023-12-27-home-lab-part-1 %}).

Virtual Local Area Network(VLAN) allow us to create logical networks on top of a physical one. This would allow only hosts within the same VLAN to talk to each other. For hosts to communicate cross VLAN, a router would be required.

`Lab Private` switch will be connecting all VLANs and the firewall together. The VMs will be sitting in two different VLANs. Thus, we need to setup 2 VLANs. 

![Desktop View](/images/homelab/5016ab07-d76f-437f-9f72-6044240972f4.png)

## Creating VLANs in pfsense
To create the VLANs at pfSense, we will use pfsense GUI webconfigurator.
1. Use any of the VM set up previously to access pfSense GUI web configurator via http://192.168.1.1. At the login page, use the following default credentials to log in: `admin:pfsense`.
2. We can safely ignore the default password warning for now. But if you would like to change it now, do remember your password!
3. At the dashboard, go to ***Interfaces > Assignments***. Then go to the ***VLANs*** tab and click *Add*. Use the settings below for creating the VLAN.

	**VLAN 10**:
	```
	Parent Interface: hn1 
	VLAN Tag: 10
	VLAN Priority: 0
	Description: VLAN 10
	```
	**VLAN 20**:
	```
	Parent Interface: hn1 
	VLAN Tag: 20
	VLAN Priority: 0
	Description: VLAN 20
	```

4. Once both VLANs are created, you will see something like below.
	![Desktop View](/images/homelab/0a43f44c-0e89-4a08-a64d-8c5259bdc350.png)

5. Go back to the ***Interface Assignments*** tab. You would see that there is an option to add inteface assignments for available network ports. 
6. Click on ***Add*** twice to add both the VLANs as an interface. You will see something like below after adding both VLANs as an interface. 
![Desktop View](/images/homelab/342df963-4094-4979-a431-3a678c0f8ebe.png)
7. Note that the OPT1 interface has been mapped to VLAN10 and OPT2 interface has been mapped to VLAN20.

## Configuring DHCP Service for VLANs
Next, we will configure DHCP service to issue IP address for each VLAN. To do so, we will need to configure the VLAN interfaces and enable DHCP for each of them.

For each interface OPT1 and OPT2, follow the steps below. I will use OPT1 and VLAN10 as an example.

**Configuring Interface**
1. Go to ***Interfaces > OPT1***
2. Under General Configuration,
	1. Checked ***Enable Interface***
	2. Set *IPv4 Configuration Type* to ***Static IPv4***
	3. Set *IPv6 Configuration Type* to ***None***
3. Under Static IPv4 Configuration,
	1. Set *IPv4 Address* to `192.168.10.1` for OPT1, `192.168.20.1` for OPT2
	2. At the right, set the value (subnet mask) to `/24`
4. Click *Save*
5. Click on *Apply Changes*. Repeat steps 1-4 for OPT2 before proceeding to the next section.

**Enabling DHCP Service**
1. Go to ***Services > DHCP Service***.
2. Click on *OPT1* Tab
3. Under General DHCP Options
	1. Checked ***Enable DHCP Server on OPT1 Interface***
4. Under Primary Address Pool
	1. Set ***Address Pool Range***
		1. *From*: `192.168.10.100` (OPT1) `192.168.20.100` (OPT2)
		2. To: `192.168.10.150` (OPT1) `192.168.20.150` (OPT2)
5. Scroll down and click `Save`.
6. Click `Apply Changes`.

You can choose the IP range for but do take note that the first address `192.168.x.1` was used as the default interface IP address.

## Adding firewall rule for access to webconfigurator

This section is not related for configuration of VLANs. But for convenience we would also need to allow one of our VLAN to be able to access the firewall for configuration. This is to prevent VMs from getting locked out once we switch our VM to use VLAN access. In this case, I will only set VLAN 10 to be able to access.

1. Go to `Firewall > Rules`. Click on OPT1 tab.
2. Click on any of the Add button to add a firewall rule. The setting for the rule is as follows:
	```
	Action: Pass
	Interface: OPT1
	Source: OPT1 Subnets/OPT1 Net
	Destination: OPT1 address
	Destination Port Range: Choose "HTTPS (443)" in "From" dropdown.
	```
3. Click the `Save` button at the bottom and apply the changes.

# Configuring Hyper-V for VLANs assignment
VLAN is achieved by having a tag to each ethernet frame. Extracted from [Wikipedia](https://en.wikipedia.org/wiki/Ethernet_frame) below, we can see that there is a *802.1Q tag* with 4 octets for an optional VLAN tag. Thus, we will need to configure Hyper-V with the correct VLAN ID tagging. We would also need to set the "port" of the switch connected by the VM to be either ***Access*** or ***Trunk*** mode.
![Desktop View](/images/homelab/a4b0869f-5bf2-4eda-9d3c-0a9e04564b60.png)

## Configuring Windows 10 Client VM settings for VLAN
Start powershell as an administrator on the management host for the next few steps.

Get our VM names with the following command. Note the names of the windows client. (If you follow the previous guide, it will be  `Windows 10 (vlan10)` and `Windows 10 (vlan20)`)

```
Get-VM
```

For each of the windows 10 client, we will need to configure the virtual network adapter to be ***Access*** mode and the respective VLAN ID.

1. Configuring VLAN access mode and VLAN ID for `Windows 10 (vlan10)`
	```
	Set-VMNetworkAdapterVlan -VMName "Windows 10 (vlan10)" -Access -VlanId 10
	```

2. Configuring VLAN access mode and VLAN ID for `Windows 10 (vlan20)`
	```
	Set-VMNetworkAdapterVlan -VMName "Windows 10 (vlan20)" -Access -VlanId 20
	```

3. Confirm both has been set correctly with the following command.

	```
	Get-VMNetworkAdapterVlan -VMName "Windows 10 (vlan10)","Windows 10 (vlan20)"
	```

![Desktop View](/images/homelab/a462e143-788d-4443-a712-aa4aa38e937d.png)

## Configuring pfsense VM settings for VLAN
Next, we need to set the LAN interface of the firewall to be ***Trunk*** mode. A Trunk interface allows multiple VLAN traffic to pass through.

1. Get get the network adapters of our firewall with the follow command. 
	```
	Get-VMNetworkAdapter -VMName "Firewall"
	```
	![Desktop View](/images/homelab/3e0ed4e9-61df-485c-b021-0dbbf70d20da.png)
2. Take note of the MAC Address of network adapter connected to `Lab Private`. For mine, it would be `00155DF80122`
3. Run the following command after replacing the MAC Address extracted previously. 
	```
	Get-VMNetworkAdapter -VMName "Firewall" | Where-Object {$_.MacAddress -eq "<MacAddress>"} | Set-VMNetworkAdapterVlan -Trunk -AllowedVlanIdList "10,20" -NativeVlanId 1
	```
4. Then run the following command to ensure that it has been set.
	```
	Get-VMNetworkAdapter -VMName "Firewall" | Where-Object {$_.MacAddress -eq "<MacAddress>"} | Get-VMNetworkAdapterVlan
	```
	![Desktop View](/images/homelab/e7760d82-2276-4078-a91d-6a4a60aabf8e.png)


## Testing 
Let us test if the VLANs and the DHCP service is working. Go to each VM and start command prompt as administrator. 

1. Run the following command to get a new IP.
	```
	ipconfig /release
	ipconfig /renew
	```

2. Verify that the IP assigned is correct for the respective VLAN. 192.168.10.x for VLAN 10 and 192.168.20.x for VLAN 20.

	**windows 10 (vlan10)**
	![Desktop View](/images/homelab/94c40987-4d88-4ab9-86fe-bc3a90788923.png)

	**windows 10 (vlan20)**
	![Desktop View](/images/homelab/85a7c6b6-8338-434d-a787-dc778b1d6320.png)

	> Note: If you see that the Default Gateway is not set, go back to the DHCP server for each VLAN interface via ***Services > DHCP Server***  and set the ***Gateway*** under *Other DHCP Options* as the IP of the interface e.g `192.168.10.1` for OPT1 and `192.168.20.1` for OPT2. This settings tell the DHCP clients that any destination address not within the local network should be routed to the default gateway. 

3. Next go to the client Windows 10 (vlan10), we will try to access the web configurator from here. Go to [https://192.168.10.1](https://192.168.10.1). You will be able to see the login page.
![Desktop View](/images/homelab/f54e22e5-3347-49b7-8060-229b79a04b15.png)

If we try to access the web configurator from the client `Windows 10 (vlan20)`, we would not able to access the web configurator as there are no firewall rules that allows it.

Now that we have configured both our VLANs. In the last part, we will configure our pfsense firewall such that clients in VLAN 10 are able to access the internet, but not for the clients in VLAN20. 

See you in the next part!