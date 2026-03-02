# VLANs - OPNSense & Omada Controller

[OPNSense Docs :simple-opnsense: ](https://docs.opnsense.org/manual/how-tos/vlan_and_lagg.html)

[Zenarmor Tutorial :material-web: ](https://www.zenarmor.com/docs/network-security-tutorials/how-to-configure-vlan-on-opnsense)

[Home Network Guy Youtube :material-youtube: ](https://www.youtube.com/watch?v=fPP4UE6IuRc)

---

I am only using IPv4 for this guide. Not familiar enough with IPv6, and I don't feel like dealing with for now. I'll come around to it (maybe).

This is made for DNSMasq and Unbound DNS specifically, but same ideas apply if using something else. Same for omada controller (for the most part) 

---

## VLAN Structure

I currently have my network setup as this:

You can set this however works for you. 

* **VLAN 10:** Management (Mgmt) - 10.10.10.x

    Devices: Omada Switches, WAPs, Omada Controller, Proxmox Web GUI, iDRAC/IPMI.

    **Rules: strict access; only your main PC can reach this.**

* **VLAN 20:** Trusted LAN - 10.10.20.x

    Devices: Personal, known devices. 

* **VLAN 30:** Lab / Server - 10.10.30.x

    Devices: VMs, Docker Containers. 

* **VLAN 50:** IoT - 10.10.50.x

    Devices: Smart plugs, Alexa, Thermostats, random IoT stuff. 

    Rules: Can access Internet, CANNOT access other VLANs.

* **VLAN 99:** Guest - 10.10.99.x

    Devices: Random, new, unkown devices. 


## VLAN Creation in OPNSense 

---

### WAN Settings 

**Go to Interfaces** > [WAN]

* IPv4 Configuration Type = DHCP

This will allow your OPNSense router to be able to hand out DHCP addresses to the rest of the network - assuming your ISP allows this.



### Create a New VLAN

Go to **Interfaces > Devices > VLAN**. Configure each VLAN as:

!!!note
    
    I am using the management interface for the examples, same ideas apply for the others. 

* `Device`: [blank - will auto create name, can be custom if wanted.]

* `Parent`: igb0 (12.34.56.mac.address) (physical interface that will carry traffic)
  
* `VLAN tag`: 10 

* `VLAN Priority:` Best effort (0, default) 

* `Description:` MANAGEMENT (or whatever you want)


* **Repeat this for all other VLANs**

### Interface Assignment 

 Go to **Interface > Assignments**

* Assign each new VLAN (device).

* Match the description name 



### VLAN IP Configuration Settings

Go to each new interface - **[MANAGMENT], [TRSUTED]. etc.**

* Enable each interface

* Prevent interface removal 

* IPv4 Configuration Type = **Static IPv4**

* Static IPv4 Configuration - this will assign the gateway IP address for your LAN. 

    * IPv4 Address = 10.10.10.1/24 (be sure to change this to /24).

* (Usually for LAN/VLANs) leave **Block private networks** and **Block bogon networks unchecke**d. Enable them only on WAN-like interfaces.


## DHCP / DNS Settings 

---

### DHCP

**1**. Go to **Services >  Dnsmasq DNS & DHCP > General**

!!!important

    Make sure the VLAN interfaces are selected uunder **Interface**.

    
**2**. Go to **Services >  Dnsmasq DNS & DHCP > DHCP Ranges**

* Press the + button 

* Select desired interface

* Start Address = 10.10.10.100

* End Address = 10.10.10.200

* **Repeat this for all other interfaces, changing 3rd octect.**

    * MANAGEMENT = 10.10.10.100 - 10.10.10.200

    * TRUSTED = 10.10.20.100 - 10.10.20.200

    * ETC.

### DNS  

For client configuration see: [Client DHCP and DNS configuration](../vlandhcp)

**1.** Go to **Services > Unbound DNS > General** 

* Listen port: 53

* Enable DNSSEC Support: Enabled (if upstream DNS supports)

* Register DHCP Static Mappings: Enabled

    * This will register all static DHCP reservations. This will auto create a DNS hostname. Handy.

* Do not register A/AAAA records: Enabled.

    * By default, OPNSense will register all IPs for all interfaces to routers hostname (e.g. OPNSense.internal). Hostname lookup will return all IPs, exposing all IPs and gets in the way.

    * Enabling this also allows you to access router web interface more easily from seperate VLAN if using a DNS override to lookup hostname. 

* Flush DNS Cache during reload: Enabled 

    * Allows changes to apply immediately after reload, instead of waiting for new lease. 


## Firewall Rules

---

!!!tip

    Basic setup for the rules is: all VLANs can access internet, but not eachother unless specified.


**1**. Go to **Firewall > Aliases**

Aliases will group together IPs and networks, much easier to create rules

* Create new Alias by pressing + button. Configure as: 

``` 
Name: PrivateNetworks

Type: Network(s) 

Content: 10.0.0./8 172.16.0.0/12 192.168.0.0/16 
    # All private IP ranges

Description: All private IP addresses
```
**2**. Go to **Firewall > Groups**

* Configure as:

```
Name: private
    # This will generate a network name as "private net" like other interfaces, easier to manange. 

Members: LAN and all VLANs - NOT WAN

(no) GUI Groups: Enabled 
    # Will not group them under Interfaces page.

Description: All private interfaces. 
```

**3**. Go to **Firewall > Rules** 

This will create 2 rules. One to allow internet access without other VLAN access. The other will 

* Select MANAGEMENT interface

!!!note 

    By default on a new interface, all access is blocked. Need to make rules to allow. 

* Configure first rule as: 

```
Action: Pass

Interface: MANAGMENT (should auto fill)

TCP/IP: IPv4 

Protocol: any

Source: MANAGEMENT net

Destination: private net

Destination / Invert: ENABLED # IMPORTANT 
    # This will allow access to internet, but blocking all other private IPs. Inverting the rule above. 

Destination Port Range: any | any (can only select if other protocols selected)

Description: Internet access - block local network. 
```
* Because first rule blocks all private networks, the MANAGEMENT interface's IP address (10.10.10.1), the second rule will correct that for selected interface only. 

* Configure second rule as:

```
Action: Pass

Interface: MANAGMENT (should auto fill)

TCP/IP: IPv4 

Protocol: UDP

Source: MANAGEMENT net

Destination: MANAGMENT address 

Destination / Invert: disabled

Destination Port Range: DNS | DNS 

Description: Allow DNS on MANAGMENT network 
```

!!!important 

    **Repeat this for all other interfaces. Ensuring to change the selected net and addresses for each. For first rule, "private net" should be destination on all. You can clone the rules and make it faster.** 

## Switch Configuration - Omada Controller 

### VLAN Profile Creation

* Open the Omada Controller WebUI 

* Go to **Settings > Wired and Wireless Networks > LAN**

* Select Create New LAN 

* Configure as:
    * `Name`: VLAN Name 
    * `Purpose`: VLAN
    * `VLAN`: Enter the tag defined in your OPNSense configuration above
    *  Other options availabe as needed for different configs are listed. 

### VLAN assignment

- Select **Devices** on sidebar
- Select **Ports**
    - Click the edit icon on the port you want to configure
    - Set the profile to the VLAN profile created in Omada. 
- This should be working correctly now. 

## Testing

---

Testing confirms whether the VLAN works as expected before deploying it widely.

1.  Connect a device (e.g., laptop or VM) to a switch port assigned to the VLAN.
2.  Ensure it receives an IP address from the correct VLAN subnet.
3.  Test connectivity (e.g., ping the gateway or browse the internet).
4.  Verify that firewall rules are functioning as intended.
This helps identify and fix any misconfigurations early.

Linux - `ip a` should show newly assigned, correct, subnet

Windows - `ipconfig` 





