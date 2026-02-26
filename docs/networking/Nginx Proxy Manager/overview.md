# NGINX Proxy Mananger 

[Github :simple-github: ](https://github.com/NginxProxyManager/nginx-proxy-manager)
---

## Overview 
**NGINX Proxy Mananger** is a user-friendly, web-based interface for configuring Nginx as a reverse proxy, simplifying the management of multiple web services on a single IP address, especially for home labs and self-hosted applications, by offering easy setup for proxy hosts, Let's Encrypt SSL certificates, redirects, access controls, and stream proxying, all within Docker containers. It removes the need for manual Nginx configuration, allowing quick deployment and management of secure web access to internal services through a simple dashboard. 

Instead of remembering 8000 different `IP-ADDR:PORT` combos, you can use NPM to set a reverse proxy to `service.domain.me`. 

## Installation

### Docker Compose

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"     # HTTP
      - "81:81"     # Admin UI
      - "443:443"   # HTTPS
    environment:
      DB_SQLITE_FILE: "/data/database.sqlite"
    volumes:
      - ${APPDATA}/ingress/npm/data:/data
      - ${APPDATA}/ingress/npm/letsencrypt:/etc/letsencrypt

```

## Usage 

* Navigate to you `IP-ADDR:PORT` address, using the Admin UI port set in your `compose.yaml` above. 

* Login with:
    
    * EMAIL: EMAIL@example.com
    * Password: changeme

    (probably best to change the password immediately.)

* Go to SSL Certificates, "Add Certificates", select "Let's Encrypt". 

* Enter your DDNS address, valid email, enable "Use a DNS challenge", choose your DNS provider, add your DDNS token, accept TOS and save.

!!! note 
    You can get a DDNS from multiple providers including DuckDNS, cloudflare, or porkbun. 

**To set a Host:** 

* Select "Hosts" > "Proxy Hosts" 

* Add Proxy Host

* Under Domain Names input desired domain

    * service.domain.com

    * or whatever you want it to be called

* Scheme: HTTP (the reverse proxy w/ your SSL cert will forward to HTTPS).

* Forward your hostname / IP of the hosting machine/service. 

* Forward Port - use port defined in service. This would be the host port defined in a container. 

    * e.g. a `compose.yaml` with:
    ```yaml
    ports:
      - '3005:8080'
    ```
    Would use port 3005 as the forwarded IP, not 8080. 

* Choose your ACL rule. 

* Choose "Block COmmon Exploits" and "Websockets support" 

* Select the **SSL** tab at the top

    * Select the SSL cert made earlier. 

    * Force SSL and HTTP/2 support. 

* Save

Now you can access `service.domain.com` instead of `192.168.3.20:3005`.


