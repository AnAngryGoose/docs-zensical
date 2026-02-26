# Tailscale & OPNsense Configuration

---

**This covers how to allow remote access to local network via Tailscale (Subnet Routing) and resolve local domains (Split DNS) via AdGuard Home/Unbound.**

# Fix IP Access (Subnet Routing)

* Makes local IPs (e.g., `192.168.0.2:3000`) accessible from tailscale remote clients 


### 1. Advertise the Route (OPNsense)
1. Navigate to **VPN > Tailscale > Settings**.
2. Check **Advertise Routes**.
3. Enter your LAN subnet (e.g., `192.168.0.0/24`).
4. Click **Save**.

!!! note "CLI Alternative"
    If you are using the CLI instead of the plugin, run:
    ```bash
    tailscale up --advertise-routes=192.168.0.0/24
    ```

### 2. Approve the Route (Tailscale Console)
1. Open the [Tailscale Admin Console](https://login.tailscale.com/admin/machines).
2. Find your OPNsense machine in the list.
3. Click the **Three Dots (...) > Edit Route Settings**.
4. **Toggle ON** the subnet route (`192.168.0.0/24`) you previously advertised.

### 3. Allow Traffic (OPNsense Firewall)
Without this step, OPNsense will drop packets originating from the Tailscale interface.

1. Go to **Firewall > Rules > Tailscale**.
    * *Note: This interface might be named differently if you manually assigned it.*
2. Add a new rule:
    * **Action:** Pass
    * **Source:** `Tailscale net` (or `any`)
    * **Destination:** `LAN net` (or `any`)
3. Click **Save** and **Apply Changes**.

---

# Configure Split DNS

* Make local domains (e.g., `service.example.com`) resolve to the internal Nginx Proxy Manager IP over VPN.

* Configuring the Unbound/AGH DNS as the global DNS will also block all ads/trackers etc across the whole tailnet. 

### 1. Get OPNsense Tailscale IP
1. In OPNsense, go to **Interfaces > Overview**.
2. Copy the **Tailscale IP address** (starts with `100.x.y.z`).

### 2. Configure Split DNS (Tailscale Console)
1. Go to **Tailscale Admin Console > DNS**.
2. Scroll to **Nameservers**.
3. Click **Add Nameserver > Custom**.
4. Configure the following:
    * **Nameserver:** Paste the OPNsense Tailscale IP (`100.x.y.z`) from Step 1.
    * **Restrict to domain:** Toggle **ON**.
    * **Domain:** Enter your local domain (e.g., `example.com`).
5. Click **Save**.

!!! success "Result"
    When you type `example.com`, Tailscale will query OPNsense. For all other traffic (e.g., google.com), it will use the client's default DNS.

### 3. Allow DNS Queries to NPM for local domains (**AdGuard Home)**

Your Adguard Home / Unbound DNS resolver will, by default and by design, look to upstream interent for domain resolution when requested. AdGuard needs to know that `*.example.com` should be redirected to your Nginx Proxy Manager (NPM) internal IP. For this you need a **DNS rewrite** in Adguard Home.



1. Open your AdGuard Home dashboard.

2. Go to **Filters > DNS Rewrites**.

3. Click **Add DNS rewrite**.

4. **Domain**: *.example.com (The * is a wildcard, so it covers service.example.com, nas.example.com, etc.)

!!!note
    If AdGuard complains about the wildcard, just add example.com and individual subdomains, but wildcards usually work.

5. **Enter**: 192.168.0.x (Enter the LAN IP address of the device running Nginx Proxy Manager).

6. Click **Save**.


### 3A. Allow DNS Queries to NPM for local domains (**Unbound**)

If using unbound to forward local requests: 

1. Go to Services > Unbound DNS > Overrides.

2. Click + to add a new override.

3. Domain: example.com

4. IP: 192.168.0.x (Nginx Proxy Manager IP).

5. Description: Local NPM. (or whatever)

6. Click Save and Apply.

!!!note
    You usually have to add separate entries for subdomains (e.g., service) unless you check "Host Overrides" settings for wildcard support. Adguard works better, found these steps while fixing something so I figured I'd add it here. 

---

# Interfaces & Rebinds

If it still isn't working, check these common configuration errors:

1. **Interface Binding:** Ensure AdGuard Home is actually listening on the Tailscale interface (or "All Interfaces").
2. **Bootstrap DNS:** In AdGuard Home (**Settings > DNS Settings**), set "Bootstrap DNS servers" to your OPNsense LAN IP or `127.0.0.1`.
3. **DNS Rebind Check:**
    * Go to **System > Settings > General**.
    * Ensure **DNS Rebind Check** is **unchecked**.
    * *Alternatively:* Add your specific domain to the whitelist.

!!! warning "DNS Rebind Protection"
    If "DNS Rebind Check" is active, OPNsense may block the DNS response because it resolves a public domain to a private RFC1918 IP address.

---

## Verification

To verify the setup is working:

1. Disconnect your phone from WiFi (switch to 5G/LTE).
2. Enable the **Tailscale VPN**.
3. Attempt to access a service via IP: `http://192.168.0.2:3000` (Should load).
4. Attempt to access a service via Domain: `http://service.example.com` (Should load).

```