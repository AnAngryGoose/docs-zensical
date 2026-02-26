# OPNSense  

[OPNSense docs :simple-opnsense: ](https://docs.opnsense.org/)


---

OPNsense® is an open source, easy-to-use and easy-to-build FreeBSD based firewall and routing platform.

OPNsense includes most of the features available in expensive commercial firewalls, and more in many cases. It brings the rich feature set of commercial offerings with the benefits of open and verifiable sources.

## Installation 
---

For my installation I used an M920q. This would require 2 things in order to allow for further network connectivity. 

* Proprietary Lenovo PCIe Riser Card - P/N: 01AJ940

* INTEL I350T4 1GbE Quad Port Ethernet Server Adapter

For the software installation, the standard installation method can be followed [here](https://docs.opnsense.org/manual/install.html).

!!!note
    Pay attention to the inital network interface gievn after installation. This will be needed to access later. 

## Initial Configuration
---
After installation, the system will prompt you for the interface assignment. If you ignore this, then the default settings will be applied. Installation ends with the login prompt.

By default you have to log in to enter the console.

**Welcome message**

```
* * * Welcome to OPNsense [OPNsense 15.7.25 (amd64/OpenSSL) on OPNsense * * *

WAN (em1)     -> v4/DHCP4: 192.168.2.100/24
LAN (em0)     -> v4: 192.168.1.1/24

FreeBSD/10.1 (OPNsense.localdomain) (ttyv0)

login:
```
!!!tip
    A user can login to the console menu with his credentials. The default credentials after a fresh install are username “root” and password “opnsense”.

**VLANs and assigning interfaces**

If you choose to do manual interface assignment or when no config file can be found then you are asked to assign Interfaces and VLANs. VLANs are optional. If you do not need VLANs then choose no. You can always configure VLANs at a later time.

**LAN, WAN and optional interfaces**

The first interface is the LAN interface. Type the appropriate interface name, for example “em0”. The second interface is the WAN interface. Type the appropriate interface name, e.g. “em1” . Possible additional interfaces can be assigned as OPT interfaces. If you assigned all your interfaces you can press [ENTER] and confirm the settings. OPNsense will configure your system and present the login prompt when finished.

!!!important
    The interface is especially important to note as the WAN/LAN port configuration needs to be followed to correctly interface to router/network. 
Console

The console menu shows 13 options.

```
0)     Logout                              7)      Ping host
1)     Assign interfaces                   8)      Shell
2)     Set interface(s) IP address         9)      pfTop
3)     Reset the root password             10)     Filter logs
4)     Reset to factory defaults           11)     Restart web interface
5)     Reboot system                       12)     Upgrade from console
6)     Halt system                         13)     Restore a configuration
```


**opnsense-update**

OPNsense features a command line interface (CLI) tool “opnsense-update”. Via menu option *8) Shell*, the user can get to the shell and use opnsense-update.

For help, type man opnsense-update and press [Enter].

**Upgrade from console**

The other method to upgrade the system is via console option 12) Upgrade from console

**GUI**

An update can be done through the GUI via `System ‣ Firmware ‣ Updates.`


## Migration to OPNSense 
---

To allow for ease of transitioning to OPNSense from an existing router/firewall (in my case a TP-Link ER605), it is best to setup a few things ***before*** installtion the OPNSense router. This can be done with the setup wizard, as well as direct settings.

!!!note
    The default IP for access is 192.168.1.1. Wizard can be re-run under `System ‣ Configuration ‣ Wizard.

* 1) Under `Interfaces ‣ [WAN]`: Ensure your IPv4 configuration Type is set to `DHCP` (same for IPv6) [if applicable].

* 2) Under `Interfaces ‣ [LAN] ‣ Static IPv4 configuration`: Set the IPv4 address to the same IP as current router. 

* 3) Remove existing router, install new OPNSense router. May need flush DNS cache following. 

## Tailscale DNS migration
---

* Ensure DNS Nameserver is no longer set to pi-hole/previous DNS sinkhole. If using OPNSense as primary DNS, set the Global Nameserver under Tailscale Admin Console > DNS.

* Reconnect to tailscale using `sudo tailscale up --reset --accept-dns=false` - this will wipe all previous flags and allow use of the OPNSense DNS. 

**To install tailscale on OPNSense**

[Tailscale docs :simple-tailscale: ](https://tailscale.com/kb/1097/install-opnsense) (old)

* Go to System > Firmware > Plugins. (Enable community plugins)

* Install os-tailscale.

* Once installed, it will appear in VPN > Tailscale. 

* Generate an Auth Key (Tailscale admin > settings > keys)

* Paste the key, leave server blank 

* Advertise Exit Node - route internet traffic through OPNsense. 

* Accept Subnet Routes - LAN Access

* Advertised Routes tab > Add your subnet to advertise (e.g. 192.168.1.0/24)

* In the TS admin console, should see the new machine with the "subnets" flag. Hit edit and approve the route. 

* Advertise your LAN subnet (192.168.1.0/24) from OPNsense.