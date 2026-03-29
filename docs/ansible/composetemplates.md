# Compose Templates

---

These are the Docker Compose files as Jinja2 templates. Ansible fills in the variables and places them on the target host.

### roles/stacks_infra/templates/npm-compose.yml.j2

```yaml
# Managed by Ansible
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    dns:
      - {{ docker_dns }}
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - {{ appdata_dir }}/npm/data:/data
      - {{ appdata_dir }}/npm/letsencrypt:/etc/letsencrypt
```

### roles/stacks_infra/templates/monitoring-compose.yml.j2

```yaml
# Managed by Ansible
services:
  beszel-hub:
    image: henrygd/beszel
    container_name: beszel-hub
    restart: unless-stopped
    ports:
      - "8090:8090"
    volumes:
      - {{ appdata_dir }}/beszel/hub-data:/beszel_data

  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - {{ appdata_dir }}/uptime-kuma:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro

  autokuma:
    image: ghcr.io/bigboot/autokuma:latest
    container_name: autokuma
    restart: unless-stopped
    environment:
      AUTOKUMA__KUMA__URL: http://uptime-kuma:3001
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - {{ appdata_dir }}/autokuma:/data

  dozzle:
    image: amir20/dozzle:latest
    container_name: dozzle
    restart: unless-stopped
    ports:
      - "8081:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  glances:
    image: nicolargo/glances:latest
    container_name: glances
    restart: unless-stopped
    pid: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      GLANCES_OPT: "-w"
```

### roles/stacks_infra/templates/vaultwarden-compose.yml.j2

```yaml
# Managed by Ansible
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.domain.me"
      SIGNUPS_ALLOWED: "false"
      ADMIN_TOKEN: "{{ vault_vaultwarden_admin_token }}"
    ports:
      - "8222:80"
    volumes:
      - {{ appdata_dir }}/vaultwarden/data:/data
```

### roles/stacks_apps/templates/paperless-compose.yml.j2

```yaml
# Managed by Ansible
services:
  paperless-redis:
    image: redis:alpine
    container_name: paperless-redis
    restart: unless-stopped

  paperless-db:
    image: postgres:15-alpine
    container_name: paperless-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: paperless
    volumes:
      - {{ appdata_dir }}/paperless/pgdata:/var/lib/postgresql/data

  paperless:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    container_name: paperless
    restart: unless-stopped
    depends_on:
      - paperless-db
      - paperless-redis
    ports:
      - "8000:8000"
    environment:
      PAPERLESS_REDIS: redis://paperless-redis:6379
      PAPERLESS_DBHOST: paperless-db
      PAPERLESS_DBNAME: paperless
      PAPERLESS_DBUSER: paperless
      PAPERLESS_DBPASS: paperless
      PAPERLESS_OCR_LANGUAGE: eng
      PAPERLESS_TIME_ZONE: {{ timezone }}
      PAPERLESS_URL: "https://docs.domain.me"
    volumes:
      - {{ appdata_dir }}/paperless/data:/usr/src/paperless/data
      - {{ appdata_dir }}/paperless/media:/usr/src/paperless/media
      - {{ appdata_dir }}/paperless/export:/usr/src/paperless/export
      - {{ appdata_dir }}/paperless/consume:/usr/src/paperless/consume
```

### roles/stacks_apps/templates/forgejo-compose.yml.j2

```yaml
# Managed by Ansible
services:
  forgejo:
    image: codeberg.org/forgejo/forgejo:latest
    container_name: forgejo
    restart: unless-stopped
    environment:
      USER_UID: "1000"
      USER_GID: "1000"
      FORGEJO__security__SECRET_KEY: "{{ vault_forgejo_secret_key }}"
    ports:
      - "3000:3000"
      - "2222:22"
    volumes:
      - {{ appdata_dir }}/forgejo/data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
```

---