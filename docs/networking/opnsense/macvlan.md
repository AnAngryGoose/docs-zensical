# macvlan 
---

**Without macvlan — the default Docker bridge**

Normally Docker puts containers on an internal virtual network (docker0 bridge, 172.17.0.0/16). Containers share the host's IP address and reach the outside world via NAT — the same concept as your home router sharing one public IP. If Frigate needs port 554 accessible, you do `-p 554:554` which punches a hole through the host's NAT to the container.

The problem is the container doesn't look like a real device to your network. It hides behind the host. OPNsense sees traffic coming from your NAS's IP, not from Frigate specifically. You can't apply per-container firewall rules, DHCP reservations, or DNS entries in any clean way.

---

**With macvlan — the container becomes a real network citizen**

macvlan is a Linux kernel feature that creates a virtual network interface with its own unique MAC address, attached directly to your physical NIC. Docker then gives that interface to the container. The result is the container appears on your network as if it were a completely separate physical machine plugged into your switch.

```
Physical NIC (eth0) on NAS
        │
        ├── NAS host itself        → 192.168.1.10
        ├── Frigate container      → 192.168.1.50  (macvlan)
        ├── Immich container       → 192.168.1.51  (macvlan, if needed)
        └── ...
```

OPNsense sees 192.168.1.50 as a distinct device. You can give it a DHCP reservation, write firewall rules specifically for it, and it can talk directly to your IoT VLAN cameras over RTSP without any port forwarding or NAT gymnastics.

---

**Why it specifically matters for Frigate**

Frigate needs to pull RTSP streams from your cameras which are on a separate IoT VLAN. With bridge networking, that traffic goes: camera → OPNsense → NAS host IP → NAT → Frigate container. OPNsense can't distinguish "this is Frigate" from "this is Plex" — it just sees your NAS IP.

With macvlan, Frigate has its own IP. Your OPNsense firewall rule becomes explicit and clean:

```
Source: Frigate IP (192.168.1.50)
Destination: IoT VLAN (192.168.20.0/24)
Port: 554 (RTSP)
Action: Allow

Everything else from IoT VLAN → Trusted VLAN: Block
```

That's proper network segmentation. Frigate can reach the cameras. The cameras can't reach anything else. Nothing else on your trusted VLAN can reach the cameras unless you explicitly allow it.

---

**The one quirk of macvlan**

By default, a macvlan container cannot communicate with the host machine it's running on. The host and the container are on the same physical NIC but the kernel deliberately blocks that path. So if Frigate (macvlan IP 192.168.1.50) needs to talk to your MQTT broker running on the NAS host itself, it can't — it has to reach it via the network through OPNsense and back.

The fix is either running your MQTT broker in a container too (so it also has its own IP), or creating a macvlan interface on the host side as well (called a bridge macvlan or `macvlan` with a companion `macvlan` on the host) which creates a virtual link between the two. In your case this isn't an issue since your MQTT broker is on your appserver anyway — Frigate just talks to it over the network normally.

---

So yes — at its simplest, macvlan gives a container its own IP and MAC address so it looks and behaves like a real independent machine on your network. That's exactly what you want for Frigate.