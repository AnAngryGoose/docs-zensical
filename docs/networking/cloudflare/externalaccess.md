# External Access with Cloudflare Tunnel & NPM 

---

In order to externally access your services from outside your network, you need to do a few things in order do it securely.

This setup below uses NPM to point services to Cloudflare Tunnel. Using a cloudflare wildcard DNS record, the flow in the end is: Set a service as public within NPM, and it immediately appears as public at the desired domain. No need to create individual records. 

## Cloudflare Tunnel 

### 1. Create the Tunnel

1. Log in to the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)

2. Navigate to **Networks > Connectors > Cloudflare Tunnels**

3. Click **Create a tunnel**, choose **Cloudflared** as the connector type, and select **Next**

4. Enter a name for the tunnel — use the host name (e.g. `prod-deb-01`)

5. Click **Save tunnel**

6. On the **Install connector** page, select **Docker** and copy the token from the provided command — it's the long string after `--token`

### 2. Deploy the connector 

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

## Nginx Proxy Manager 

### 1. Create the container

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    dns:
      - 10.10.30.1
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - /mnt/appdata/npm/data:/data
      - /mnt/appdata/npm/letsencrypt:/etc/letsencrypt
    networks: 
      - tunnel-net

networks:
  tunnel-net:
    external: true
```
## Zero Trust Application

In order to have your published sites secured behind Cloudflare access, you need to create a Access Control Application and Access Policy.

Per the [official docs](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/), creating an Access application is what controls who can reach the published route. All Access applications are deny-by-default — a user must match an Allow policy to be granted access.

1. Navigate to **Access controls > Applications**

2. Click **Add an application** → **Self-hosted**

3. Configure:

    | Field | Value |
    |---|---|
    | Application name | e.g. `Immich` |
    | Session Duration | `30 days` (recommended for personal use) |
    | Application domain | `*.domain.me` |

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

## DNS Records

### Creation of DNS Record

1. Tell Cloudflare where to route traffic for this service. Per the [official docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/#2a-publish-an-application), this is done immediately after the connector is running — before or alongside creating the Access application.

1. Navigate to **Networks > Connectors > Cloudflare Tunnels**

2. Click your tunnel → **Published application routes** tab → **Add a published application route**

3. Fill in the details:

    | Field | Value |
    |---|---|
    | Subdomain | `*` |
    | Domain | `domain.me` |
    | Path | leave empty |
    | Service Type | `HTTP` |
    | URL | `npm:80` (container name + internal port) |

4. Under **Additional application settings**, enable **Protect with Access** — this instructs `cloudflared` to validate the Access JWT on every request, rejecting anything that bypasses the Access layer.

5. Select **Save**. Cloudflare automatically creates the CNAME DNS record.

## SSL Certificate

Before publishing any sites, set up a wildcard SSL certificate in NPM so every subdomain is covered automatically.

### 1. Create a Cloudflare API Token

1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click **Create Token** → use the **Edit zone DNS** template
3. Under **Zone Resources**, select your domain
4. Click **Continue to summary** → **Create Token** and copy it

### 2. Request the Wildcard Certificate in NPM

1. In NPM, go to **SSL Certificates** → **Add SSL Certificate** → **Let's Encrypt**

2. Fill in the details:

    | Field | Value |
    |---|---|
    | Domain Names | `*.domain.me`, `domain.me` |
    | Email Address | your email |
    | Use a DNS Challenge | ✅ enabled |
    | DNS Provider | `Cloudflare` |
    | Credentials File Content | paste your API token |

3. Agree to the Let's Encrypt ToS and click **Save**

This single certificate covers all subdomains — no per-service cert needed.

## Publish a Site

Within Nginx Proxy Manager:

1. Select **Proxy Hosts** → **Add Proxy Host**

2. On the **Details** tab, fill in:

    | Field | Value |
    |---|---|
    | Domain Names | `service.domain.me` |
    | Scheme | `http` |
    | Forward Hostname / IP | container name or internal IP of the service |
    | Forward Port | the service's port |
    | Websockets Support | enable if the service requires it |

3. On the **SSL** tab:

    | Field | Value |
    |---|---|
    | SSL Certificate | select the `*.domain.me` wildcard cert |
    | Force SSL | ✅ enabled |

4. Click **Save**

That's it. Because the wildcard tunnel route already points `*.domain.me → npm:80`, the new subdomain is immediately reachable externally through Cloudflare — no additional DNS records needed.

## Internal Access

The same NPM proxy host also serves traffic from inside your network, without going through Cloudflare at all. To enable this, add a wildcard DNS record in your local resolver (AdGuard Home, Pi-hole, etc.) pointing to NPM's LAN IP:

| Type | Name | Value |
|---|---|---|
| A | `*.domain.me` | NPM's local IP (e.g. `10.10.30.x`) |

With this in place, internal clients resolve `service.domain.me` directly to NPM, while external clients go through the Cloudflare tunnel — both hitting the same proxy host entry.

```
External:  browser → Cloudflare → tunnel → NPM → service
Internal:  browser → local DNS → NPM → service
```

No duplicate entries. To make a service accessible externally, just add the proxy host in NPM. To keep it internal-only, don't add it — it simply won't be reachable through the tunnel.