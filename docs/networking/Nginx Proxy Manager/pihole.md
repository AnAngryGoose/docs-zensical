# NPM with Pi-Hole

---

Turns out setting up Pi-Hole with NPM requires extra work and took me entirely too goddamn long to figure that out, so here is the fix. Found this on reddit thread.

---



Nginx Setup

On the nginx web gui, add a new Proxy Host.

    Details

        Domain names = pihole.mydomain.com

        Scheme = http

        Forward Hostname/IP = <pihole_ip>

        Forward Port = 80

    SSL

    Your Let's Encrypt cert. *.mydomain.com

    Force SSL = Yes

    HTTP/2 Support = Yes

In the custom confg (this is the magic)
```
 location / {
 proxy_pass http://<pihole_ip>:80/admin/;
 proxy_set_header Host $host;
 proxy_set_header X-Real-IP $remote_addr;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 proxy_hide_header X-Frame-Options;
 proxy_set_header X-Frame-Options "SAMEORIGIN";
 proxy_read_timeout 90;
 }

 location /admin/ {
 proxy_pass http://<pihole_ip>:80/admin/;
 proxy_set_header Host $host;
 proxy_set_header X-Real-IP $remote_addr;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 proxy_hide_header X-Frame-Options;
 proxy_set_header X-Frame-Options "SAMEORIGIN";
 proxy_read_timeout 90;
 }

 location /api/ {
 proxy_pass http://<pihole_ip>:80/api/;
 proxy_set_header Host $host;
 proxy_set_header X-Real-IP $remote_addr;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 proxy_hide_header X-Frame-Options;
 proxy_set_header X-Frame-Options "SAMEORIGIN";
 proxy_read_timeout 90;
 }
```
It seems you have to define /, /admin/, and /api/ locations with the full URL with no nginx variables.

With this setup, I am able to access Pihole via pihole.mydomain.com on my local network with the full dashboard and graphs working.
10


[Reddit link](https://www.reddit.com/r/pihole/comments/tttp7j/pihole_with_nginx_reverse_proxy_redirection_to/)