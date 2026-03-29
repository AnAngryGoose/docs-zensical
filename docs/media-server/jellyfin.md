---
icon: simple/jellyfin
---

# Jellyfin

Jellyfin is a Free Software Media System that puts you in control of managing and streaming your media. It is an alternative to the proprietary Emby and Plex, to provide media from a dedicated server to end-user devices via multiple apps.

[Jellyfin Docs](https://jellyfin.org/docs/)

---

## Installation

### Docker 

!!! success "Recommended"

Official container image: `jellyfin/jellyfin` 
This image is also published on the GitHub Container Registry: `ghcr.io/jellyfin/jellyfin`.

LinuxServer.io image: `linuxserver/jellyfin` 

hotio image: `ghcr.io/hotio/jellyfin`.

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    # Optional - specify the uid and gid you would like Jellyfin to use instead of root
    user: uid:gid
    ports:
      - 8096:8096/tcp
      - 7359:7359/udp
    volumes:
      - /path/to/config:/config
      - /path/to/cache:/cache
      - type: bind
        source: /path/to/media
        target: /media
      - type: bind
        source: /path/to/media2
        target: /media2
        read_only: true
      # Optional - extra fonts to be used during transcoding with subtitle burn-in
      - type: bind
        source: /path/to/fonts
        target: /usr/local/share/fonts/custom
        read_only: true
    restart: 'unless-stopped'
    # Optional - alternative address used for autodiscovery
    environment:
      - JELLYFIN_PublishedServerUrl=http://example.com
    # Optional - may be necessary for docker healthcheck to pass if running in host network mode
    extra_hosts:
      - 'host.docker.internal:host-gateway'
```

Then while in the same folder as the docker-compose.yml run:

`docker compose up -d`

### Linux 

#### Debian / Ubuntu and derivatives[​](https://jellyfin.org/docs/general/installation/linux#debian--ubuntu-and-derivatives "Direct link to Debian / Ubuntu and derivatives")

To simplify deployment and help automate this for as many users as possible, we provide a BASH script to handle repo installation as well as installing Jellyfin on Debian / Ubuntu and derivatives. All you need to do is run this command on your system (requires `curl`, or substitute `curl` with `wget -O-`):

```
curl https://repo.jellyfin.org/install-debuntu.sh | sudo bash
```

note

You can verify the script download integrity with (requires `sha256sum`):

```
diff <( curl -s https://repo.jellyfin.org/install-debuntu.sh -o install-debuntu.sh; sha256sum install-debuntu.sh ) <( curl -s https://repo.jellyfin.org/install-debuntu.sh.sha256sum )
```

An empty output means everything is correct. Then you can inspect the script to see what it does (optional but recommended) and execute it with:

```
less install-debuntu.shsudo bash install-debuntu.sh
```

note

The script tries to handle as many common derivatives as possible, including, at least, Linux Mint (Ubuntu and Debian editions), Raspbian/Raspberry Pi OS, and KDE Neon. We welcome PRs [to the script](https://github.com/jellyfin/jellyfin-repo-helper-scripts/blob/master/install-debuntu.sh) for any other common derivatives.

If you do not want to execute a script with superuser permissions, you can also install the Jellyfin software repository manually (using either [extrepo](https://jellyfin.org/docs/general/installation/advanced/manual/#debian-using-extrepo) or the [fully manual method](https://jellyfin.org/docs/general/installation/advanced/manual/#official-linux-repository-manual)).

#### Other Distributions[​](https://jellyfin.org/docs/general/installation/linux#other-distributions "Direct link to Other Distributions")

For other distributions, [containers](https://jellyfin.org/docs/general/installation/container) are the recommended way to install Jellyfin. There are also [community-maintained packages](https://jellyfin.org/docs/general/installation/advanced/community) provided by 3rd parties if you would like to use them instead.

## Networking

As a server software, Jellyfin offers different services over the network. Specifically Jellyfin supports the streaming of content and comes packed with a web-Client. - This will work purely over the HTTP(S) ports.

Additionally, in local networks, Jellyfin offers various Auto-Discovery services. These will not work outside your local subnet.

As a fully self-hosted software, Jellyfin runs independently from the Internet. You do not have to make your server accessible through the internet. Neither does Jellyfin require an internet connection to run; however you should note that it will load metadata from various Providers, which will not work without an Internet connection.

### Port Bindings[​](https://jellyfin.org/docs/general/post-install/networking/#port-bindings "Direct link to Port Bindings")

This section aims to provide an administrator with knowledge on what ports Jellyfin binds to and what purpose they serve.

| Port | Protocol | Configurable | Description |
| --- | --- | --- | --- |
| 8096 | TCP | ✔️  | Default HTTP |
| 8920 | TCP | ✔️  | Default HTTPS |
| 7359 | UDP | ❌   | Client Discovery |

### Accessing Jellyfin[​](https://jellyfin.org/docs/general/post-install/networking/#accessing-jellyfin "Direct link to Accessing Jellyfin")

This section focusses on how to make Jellyfin Available within Networks. Here you will find descriptions on how to make Jellyfin accessible both only locally and through the Internet.

In general, Jellyfin will be available locally on the specified port over the host-ip - e.g. `http://10.0.0.2:8096`. However its also possible to create a local DNS entry that will point to your Jellyfin-Server - e.g. `http://jellyfin.local:8096`.

#### Firewall / Port Forwarding[​](https://jellyfin.org/docs/general/post-install/networking/#firewall--port-forwarding "Direct link to Firewall / Port Forwarding")

Networks are usually divided from each other by firewalls. These block all incoming traffic and are meant to protect the network. To access Jellyfin through these boundaries, its ports need to be forwarded / opened in the respective firewalls.

Note that opening a port gives full access to that port to the next higher Network. Opening a port directly to the Internet is therefore insecure and not recommended.

There are different layers where a firewall can be placed:

| Layer | Example | Description |
| --- | --- | --- |
| Local | Docker, VM | Open ports at this layer to allow traffic from the Host to enter the Application |
| Host | physical machine, operating system | Open ports at this layer to allow traffic from the Network to enter the Host device |
| Network | Router | Open ports at this layer to allow traffic from the Internet to enter the Local Network |

Port forwarding vs. opening a Port

Whilst Routers often allow you to forward a port, firewalls typically only allow you to open one. The difference is within the Target. Opening a Port essentially just means that traffic on this Port will go through. Forwarding a Port you typically do in NAT scenarios - traffic is coming in on your public IP Address, what device inside your network should receive it. Sometimes, port forwarding also lets you map an external port to a different internal port.

How to open a Port

How exactly a port will be opened depends on your firewall software and its UI. Here is linked below how to open ports for:

-   [Windows Firewall](https://learn.microsoft.com/en-us/sql/reporting-services/report-server/configure-a-firewall-for-report-server-access?view=sql-server-ver16#open-ports-in-windows-firewall)
-   [firewalld](https://firewalld.org/documentation/howto/open-a-port-or-service.html)
-   [Uncomplicated Firewall](https://wiki.ubuntu.com/UncomplicatedFirewall#Basic_Usage) (ufw)
-   [nftables](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)

#### External Access[​](https://jellyfin.org/docs/general/post-install/networking/#external-access "Direct link to External Access")

Since Jellyfin is entirely self-hosted, you must manually expose it to the internet. To do so, you need a method to access the HTTP(S) ports remotely. Automatic discovery only works locally and should not be exposed externally.

To access a server remotely there will need to be a way to find it or its network on the internet. This can be done through the public IP Address of the Device or for IPv6 the Server's directly.

To store the IP Address, the easiest option would be to use a Domain and rely on DNS to resolve it. This can also be used to store the 'current IP Address' in the case of a dynamic public IP Address. However its not mandatory to use a Domain.

There are multiple ways of exposing Jellyfin to the outside - the most common ones are:

-   forwarding its Ports directly to the internet (not recommended!)
-   forwarding through a Reverse Proxy
-   using a VPN connection to enter the Network
-   use a VPS to Reverse Proxy to your home network

Learn more about reverse proxies in our dedicated [Reverse Proxy guide](https://jellyfin.org/docs/general/post-install/networking/reverse-proxy/).

#### SSL / https[​](https://jellyfin.org/docs/general/post-install/networking/#ssl--https "Direct link to SSL / https")

Using https to access the Server is recommended. By default, HTTPS is disabled because it requires an SSL certificate.

SSL Certificates are usually issued by a third party and verify that the Server and URL are assigned to another. Please use a trusted certificate authority such as [Let's Encrypt](https://jellyfin.org/docs/general/post-install/networking/advanced/letsencrypt) when using https.

caution

Self-signed certificates pose security and compatibility issues and are strongly discouraged.

While Jellyfin supports HTTPS, it is strongly recommended to handle HTTPS termination separately on a reverse proxy. You can find more info on how to set this up on our [Reverse Proxy](https://jellyfin.org/docs/general/post-install/networking/reverse-proxy/) page.

**It's strongly recommend that you check your SSL strength and server security at [SSLLabs](https://www.ssllabs.com/ssltest/analyze.html) if you are exposing these services to the internet.**

#### Base URL[​](https://jellyfin.org/docs/general/post-install/networking/#base-url "Direct link to Base URL")

Running Jellyfin with a path (e.g. `https://example.com/jellyfin`) is supported.

caution

Base URL is known to break HDHomeRun, the [DLNA plugin](https://jellyfin.org/docs/general/post-install/networking/dlna), Sonarr, Radarr, and MrMC.

The Base URL setting is a setting used to specify the URL prefix that your Jellyfin instance can be accessed at. In effect, it adds this URL fragment to the start of any URL path. For instance, if you have a Jellyfin server at `http://myserver` and access its main page `http://myserver/web/index.html`, setting a Base URL of `/jellyfin` will alter this main page to `http://myserver/jellyfin/web/index.html`. This can be useful if administrators want to access multiple Jellyfin instances under a single domain name, or if the Jellyfin instance lives only at a subpath to another domain with other services listening on `/`.

The entered value on the configuration page will be normalized to include a leading `/` if this is missing.

This setting requires a server restart to change, in order to avoid invalidating existing paths until the administrator is ready.

There are three main caveats to this setting.

1.  When setting a new Base URL (i.e. from `/` to `/baseurl`) or changing a Base URL (i.e. from `/baseurl` to `/newbaseurl`), the Jellyfin web server will automatically handle redirects to avoid displaying users invalid pages. For instance, accessing a server with a Base URL of `/jellyfin` on the `/` path will automatically append the `/jellyfin` Base URL. However, entirely removing a Base URL (i.e. from `/baseurl` to `/`, an empty value in the configuration) will not - all URLs with the old Base URL path will become invalid and throw 404 errors. This should be kept in mind when removing an existing Base URL.
2.  Client applications generally, for now, do not handle the Base URL redirects implicitly. Therefore, for instance in the Android TV app, the `Host` setting _must_ include the BaseURL as well (e.g. `http://myserver:8096/baseurl`), or the connection will fail.
3.  Any reverse proxy configurations must be updated to handle a new Base URL. Generally, passing `/` back to the Jellyfin instance will work fine in all cases and the paths will be normalized, and this is the standard configuration in our examples. Keep this in mind however when doing more advanced routing.