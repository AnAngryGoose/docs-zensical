# Cloudflare Tunnels

[Cloudflare docs :simple-cloudflare:](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)

---
This covers creating a cloudflare tunnel in docker, serving a container with it, and securing it using an access policy. This is assuming you already have a domain and CNAME DNS record setup with cloudflare. 

## Creating A Tunnel 

---

First, you need to generate a tunnel and get its Tunnel Token.

1. Login to the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
   
2. Navigate to Networks > Tunnels.
   
3. Click Create a tunnel, select cloudflared, and give it a name.
   
4. On the "Install connector" page, select Docker.
   
5. Copy the Token from the provided command (it's the long string after --token).

## Deploy the Connector 

---

There are multiple ways to deploy a tunnel. This is using a docker `compose.yaml` file. This allows you to easily manage the container as well as the tunnel together or seperately. 

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    environment:
      - TUNNEL_TOKEN=your_token_here  # Paste your token from step 1
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

NOTE: **The `cloudflared` container must be on the same Docker network as the service you want to expose.**


## Route Traffic (Public Hostname)

---

Now that the connector is running, tell Cloudflare where to send the traffic:

1. Back in the Cloudflare Dashboard, go to the Public Hostname tab for your tunnel.

2. Click Add a public hostname.

3. Fill in your details:

        Subdomain: app

        Domain: yourdomain.com

        Service Type: HTTP

        URL: my-app:80 (Use the Docker container name and internal port).

4. Save the configuration.

## Secure Tunnel

---

Using an access policy, you can secure the initial access to the tunnel. Multiple options are available including email, Indent providers (github, google, etc) and more. Im using an email for simplicity here. 

### Create the Access Application 

1. In the Zero Trust Dashboard, navigate to Access > Applications.

2. lick Add an application and select Self-hosted.

3. Application Configuration:

    Application name: (e.g., My App Protection)

    Session Duration: How long a user stays logged in.

    Application domain: Enter the Subdomain and Domain that matches exactly what you set up in your Tunnel (e.g., app.yourdomain.com).

4. Click Next.

### Create Access Policy


1. This defines who is allowed to get through the lock.

    Policy Name: (e.g., Allow Casey Only)

    Action: Ensure this is set to Allow.

    Assign a group (Optional): You can skip this for a simple rule.

    Configure rules:

        Include: Select Emails or Email ending in.

        Require: Select Emails. 

        Value: Enter your specific email address.

2. Click Next through the Setup page and then click Add application.

Your app, served at app.domain.com should now be forwarded via a cloudflare tunnel, and secure with an inital check by cloudflare. 




