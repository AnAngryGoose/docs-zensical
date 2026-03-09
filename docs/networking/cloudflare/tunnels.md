# Cloudflare Tunnels

[Cloudflare docs :simple-cloudflare:](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)

---

Cloudflare Tunnel provides a secure way to connect resources to Cloudflare without a publicly routable IP address. Instead of sending traffic to an external IP, `cloudflared` creates outbound-only connections to Cloudflare's global network — no open ports, no exposed home IP.

For this setup, I use one tunnel per host. One `cloudflared` container routes all services on that host through a shared external Docker network. Services join the network — no extra containers, no extra tokens.

```
Cloudflare Edge
    │
    ├─ tunnel: machine1 ──── cloudflared
    │                               ├─ immich.domain.me   → immich-server:2283
    │                               └─ ha.domain.me       → homeassistant:8123
    │
    └─ tunnel: machine2 ─────── cloudflared
                                    ├─ seerr.domain.me    → seerr:5055
                                    └─ files.domain.me    → filebrowser:80
```

!!! note
    Cloudflare terminates HTTPS and manages certificates automatically. No ports need to be opened on OPNsense, and no SSL certs need to be managed manually.

---

## 1. Create the Tunnel

1. Log in to the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)

2. Navigate to **Networks > Connectors > Cloudflare Tunnels**

3. Click **Create a tunnel**, choose **Cloudflared** as the connector type, and select **Next**

4. Enter a name for the tunnel — use the host name (e.g. `prod-deb-01`)

5. Click **Save tunnel**

6. On the **Install connector** page, select **Docker** and copy the token from the provided command — it's the long string after `--token`

---

## 2. Create the Shared Tunnel Network

On the host, create a persistent external Docker network. This must exist before any stack tries to join it.

```bash
docker network create tunnel-net
```

This network persists independently of any compose stack. Only create it once per host.

---

## 3. Deploy the Connector

### One Tunnel per Host

!!! success "Recommended"

Create a standalone compose stack for `cloudflared`. This is the only place the tunnel token lives.

`/opt/docker/cloudflared/compose.yaml`

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    command: tunnel --no-autoupdate run
    networks:
      - tunnel-net

networks:
  tunnel-net:
    external: true
```

`/opt/docker/cloudflared/.env`

```
TUNNEL_TOKEN=your_token_here
```

Start it:

```bash
cd /opt/docker/cloudflared
docker compose up -d
```

The connector will appear in the Zero Trust dashboard. Once running, click **Next** in the dashboard to proceed. The tunnel should show as **Healthy** under **Networks > Connectors**.

### 3a. One Tunnel Per Service

Instead of one tunnel per host, it is possible to run a separate `cloudflared` container per service — colocated in the same compose stack.

!!! warning "Not Recommended"
    Each `cloudflared` container establishes four outbound connections to Cloudflare's edge. With 10+ services, that's 40+ persistent connections and 10+ containers doing nothing but tunneling — more compose files to maintain, more tokens to track, more things to break. Per the [Cloudflare community](https://community.cloudflare.com/t/can-i-use-cloudflared-in-a-docker-compose-yml/407168), one tunnel per host is the recommended approach.

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    environment:
      - TUNNEL_TOKEN=your_token_here
    command: tunnel --no-autoupdate run
    networks:
      - tunnel-net

  my-app:
    image: your-app-image:latest
    container_name: my-app
    networks:
      - tunnel-net
    # No ports need to be exposed to the host!

networks:
  tunnel-net:
    name: tunnel-net
```

---

## 4. Expose a Service

For any service you want to expose externally, add it to the shared `tunnel-net` network. No `cloudflared` container needed in the service's compose file.

```yaml
services:
  my-app:
    image: your-app-image:latest
    container_name: my-app
    restart: unless-stopped
    networks:
      - tunnel-net      # joins the shared tunnel network
      - internal-net    # for DB, cache, etc — isolated from tunnel

networks:
  tunnel-net:
    external: true      # must already exist on the host
  internal-net:
    internal: true      # no external access
```

The service URL used in Cloudflare is the **container name** and **internal port** — not the host port.

```
container_name: bentopdf      →  http://bentopdf:8080
container_name: immich-server →  http://immich-server:2283
```

!!! warning
    Do not expose `ports:` to the host for services routed through the tunnel. Traffic enters via Cloudflare only. Services not going through the tunnel may still use `ports:` for internal LAN access — this is intentional.

---

## 5. Add a Published Application Route

Tell Cloudflare where to route traffic for this service. Per the [official docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/#2a-publish-an-application), this is done immediately after the connector is running — before or alongside creating the Access application.

1. Navigate to **Networks > Connectors > Cloudflare Tunnels**

2. Click your tunnel → **Published application routes** tab → **Add a published application route**

3. Fill in the details:

    | Field | Value |
    |---|---|
    | Subdomain | e.g. `immich` |
    | Domain | `domain.me` |
    | Path | leave empty |
    | Service Type | `HTTP` |
    | URL | `immich-server:2283` (container name + internal port) |

4. Under **Additional application settings**, enable **Protect with Access** — this instructs `cloudflared` to validate the Access JWT on every request, rejecting anything that bypasses the Access layer.

5. Select **Save**. Cloudflare automatically creates the CNAME DNS record.

!!! warning
    The route is publicly reachable as soon as it's saved. Proceed immediately to step 6 to lock it down with an Access application.

!!! note
    Docker's internal DNS resolves container names across the shared `tunnel-net` network — use the container name, not an IP address.

---

## 6. Create an Access Application

Per the [official docs](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/), creating an Access application is what controls who can reach the published route. All Access applications are deny-by-default — a user must match an Allow policy to be granted access.

1. Navigate to **Access controls > Applications**

2. Click **Add an application** → **Self-hosted**

3. Configure:

    | Field | Value |
    |---|---|
    | Application name | e.g. `Immich` |
    | Session Duration | `30 days` (recommended for personal use) |
    | Application domain | Match exactly what was set in the tunnel route (e.g. `immich.domain.me`) |

4. Click **Policies** tab
   

### Create an Access Policy

!!! note "Access Policies"
    There are many different options regarding the access policy. Email is the easiest for this guide to show the process, however you can include, exclude or require various things including names, countries, etc. 

1. Configure the policy:

    | Field | Value |
    |---|---|
    | Policy Name | e.g. `Allow Me` |
    | Action | `Allow` |

2. Under **Configure rules**:

    - **Include:** Select `Emails` → add each allowed email address
    - Or use **Email ending in** for a whole domain
  



3. Click **Next** through Setup, then **Add application**

!!! tip
    For multiple users, create a reusable group under **Reusable components > Lists**, add emails there, then reference the group in each Access policy instead of listing emails individually.



!!! note "Native apps and Access"
    Cloudflare Access breaks native app authentication — mobile apps cannot complete the browser-based OTP flow. For services accessed via a native app (e.g. Immich mobile), skip Access and rely on the app's own built-in auth instead. Ensure the app's auth is properly configured before exposing it.

---
