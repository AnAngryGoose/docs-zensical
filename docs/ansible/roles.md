## Roles

Each role is a self-contained unit. Here's every role with complete task files.

### Role: base

Installs packages, hardens SSH, sets timezone, creates directory structure.

??? note "roles/base/tasks/main.yml"

    ```yaml
    ---
    - name: Set timezone
      community.general.timezone:
        name: "{{ timezone }}"

    - name: Set locale
      ansible.builtin.locale_gen:
        name: "{{ locale }}"
        state: present

    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install base packages
      ansible.builtin.apt:
        name: "{{ base_packages }}"
        state: present

    - name: Create directory structure
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
        mode: "0755"
      loop:
        - "{{ docker_compose_dir }}"
        - "{{ appdata_dir }}"
        - "{{ backup_mount }}"

    - name: Configure SSH daemon
      ansible.builtin.template:
        src: sshd_config.j2
        dest: /etc/ssh/sshd_config
        owner: root
        group: root
        mode: "0644"
        validate: "sshd -t -f %s"
      notify: restart sshd

    - name: Ensure passwordless sudo for main user
      ansible.builtin.copy:
        content: "{{ main_user }} ALL=(ALL) NOPASSWD:ALL\n"
        dest: "/etc/sudoers.d/{{ main_user }}"
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
    ```

??? note "roles/base/handlers/main.yml"

    ```yaml
    ---
    - name: restart sshd
      ansible.builtin.service:
        name: sshd
        state: restarted
    ```

??? note "roles/base/templates/sshd_config.j2"

    ```
    # Managed by Ansible -- do not edit manually
    Port 22
    AddressFamily any
    ListenAddress 0.0.0.0
    ListenAddress ::

    PermitRootLogin no
    PasswordAuthentication no
    PubkeyAuthentication yes
    AuthorizedKeysFile .ssh/authorized_keys

    ChallengeResponseAuthentication no
    UsePAM yes

    X11Forwarding no
    PrintMotd no

    AcceptEnv LANG LC_*

    Subsystem sftp /usr/lib/openssh/sftp-server
    ```

---

### Role: docker

Installs Docker CE from the official repo and configures the daemon.

??? note "roles/docker/tasks/main.yml"

    ```yaml
    ---
    - name: Remove old Docker packages
      ansible.builtin.apt:
        name:
          - docker
          - docker-engine
          - docker.io
          - containerd
          - runc
        state: absent

    - name: Install Docker prerequisites
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
        state: present

    - name: Create keyrings directory
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: "0755"

    - name: Add Docker GPG key
      ansible.builtin.shell: |
        curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        chmod a+r /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Add Docker apt repository
      ansible.builtin.copy:
        content: |
          deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian {{ ansible_distribution_release }} stable
        dest: /etc/apt/sources.list.d/docker.list
        mode: "0644"

    - name: Update apt cache after adding Docker repo
      ansible.builtin.apt:
        update_cache: yes

    - name: Install Docker CE
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        state: present

    - name: Add user to docker group
      ansible.builtin.user:
        name: "{{ main_user }}"
        groups: docker
        append: yes

    - name: Configure Docker daemon
      ansible.builtin.template:
        src: daemon.json.j2
        dest: /etc/docker/daemon.json
        owner: root
        group: root
        mode: "0644"
      notify: restart docker

    - name: Ensure Docker is running and enabled
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes
    ```

    Ref: https://docs.docker.com/engine/install/debian/

??? note "roles/docker/handlers/main.yml"

    ```yaml
    ---
    - name: restart docker
      ansible.builtin.service:
        name: docker
        state: restarted
    ```

??? note "roles/docker/templates/daemon.json.j2"

    ```json
    {
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }
    ```

    Ref: https://docs.docker.com/engine/logging/drivers/json-file/

---

### Role: borgmatic

Installs borgmatic and configures backups to the NAS.

??? note "roles/borgmatic/tasks/main.yml"

    ```yaml
    ---
    - name: Install borgmatic via pip
      ansible.builtin.pip:
        name: borgmatic
        extra_args: --break-system-packages
        state: present

    - name: Create borgmatic config directory
      ansible.builtin.file:
        path: /etc/borgmatic
        state: directory
        owner: root
        group: root
        mode: "0700"

    - name: Template borgmatic config
      ansible.builtin.template:
        src: borgmatic-config.yml.j2
        dest: /etc/borgmatic/config.yaml
        owner: root
        group: root
        mode: "0600"

    - name: Template borgmatic cron job
      ansible.builtin.template:
        src: borgmatic.cron.j2
        dest: /etc/cron.d/borgmatic
        owner: root
        group: root
        mode: "0644"

    - name: Check if borg repo is initialized
      ansible.builtin.stat:
        path: "{{ backup_mount }}/{{ inventory_hostname_short }}/README"
      register: borg_repo_check
      when: skip_nas_mount is not defined or not skip_nas_mount

    - name: Initialize borg repo
      ansible.builtin.command: >
        borg init --encryption=repokey
        {{ backup_mount }}/{{ inventory_hostname_short }}
      environment:
        BORG_PASSPHRASE: ""
      when:
        - skip_nas_mount is not defined or not skip_nas_mount
        - not borg_repo_check.stat.exists
    ```

??? note "roles/borgmatic/templates/borgmatic-config.yml.j2"

    ```yaml
    # Managed by Ansible
    source_directories:
    {% for dir in borg_backup_targets %}
      - {{ dir }}
    {% endfor %}

    repositories:
      - path: {{ backup_mount }}/{{ inventory_hostname_short }}
        label: nas

    exclude_patterns:
    {% for pattern in borg_excludes %}
      - {{ pattern }}
    {% endfor %}

    encryption_passphrase: ""

    retention:
      keep_daily: 7
      keep_weekly: 4
      keep_monthly: 6

    consistency:
      checks:
        - repository
        - archives
      check_last: 3
    ```

??? note "roles/borgmatic/templates/borgmatic.cron.j2"

    ```
    # Managed by Ansible
    0 3 * * * root PATH=$PATH:/usr/bin:/usr/local/bin borgmatic --verbosity -1 --syslog-verbosity 1 --list --stats
    ```

---

### Role: tailscale

Installs Tailscale but does not start it.

??? note "roles/tailscale/tasks/main.yml"

    ```yaml
    ---
    - name: Add Tailscale GPG key
      ansible.builtin.shell: |
        curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg \
          -o /usr/share/keyrings/tailscale-archive-keyring.gpg
      args:
        creates: /usr/share/keyrings/tailscale-archive-keyring.gpg

    - name: Add Tailscale apt repository
      ansible.builtin.copy:
        content: |
          deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] https://pkgs.tailscale.com/stable/debian bookworm main
        dest: /etc/apt/sources.list.d/tailscale.list
        mode: "0644"

    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Install Tailscale
      ansible.builtin.apt:
        name: tailscale
        state: present

    - name: Ensure Tailscale is stopped and disabled
      ansible.builtin.service:
        name: tailscaled
        state: stopped
        enabled: no
    ```

    Ref: https://tailscale.com/kb/1174/install-debian-bookworm

---

### Role: mounts

Mounts the NAS backup share via CIFS.

??? note "roles/mounts/tasks/main.yml"

    ```yaml
    ---
    - name: Create samba credentials directory
      ansible.builtin.file:
        path: /etc/samba
        state: directory
        owner: root
        group: root
        mode: "0700"
      when: skip_nas_mount is not defined or not skip_nas_mount

    - name: Template NAS credentials file
      ansible.builtin.template:
        src: nas-creds.j2
        dest: "{{ nas_creds_file }}"
        owner: root
        group: root
        mode: "0600"
      when: skip_nas_mount is not defined or not skip_nas_mount

    - name: Mount NAS backup share
      ansible.posix.mount:
        path: "{{ backup_mount }}"
        src: "//{{ nas_backup_host }}/{{ nas_backup_share }}"
        fstype: cifs
        opts: "credentials={{ nas_creds_file }},uid=1000,gid=1000,iocharset=utf8"
        state: mounted
      when: skip_nas_mount is not defined or not skip_nas_mount
    ```

??? note "roles/mounts/templates/nas-creds.j2"

    ```
    # Managed by Ansible
    username={{ nas_samba_user }}
    password={{ vault_nas_samba_password }}
    ```

---



### Role: stacks_infra

Deploys Docker Compose stacks to ops-01 (infra node).

??? note "roles/stacks_infra/tasks/main.yml"

    ```yaml
    ---
    # Each stack gets its own directory and compose file.
    # Only a few key stacks shown here as examples.
    # Follow the same pattern for each additional stack.

    # --- NPM ---
    - name: Create NPM directory
      ansible.builtin.file:
        path: "{{ docker_compose_dir }}/npm"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"

    - name: Create NPM appdata directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      loop:
        - "{{ appdata_dir }}/npm/data"
        - "{{ appdata_dir }}/npm/letsencrypt"

    - name: Deploy NPM compose
      ansible.builtin.template:
        src: npm-compose.yml.j2
        dest: "{{ docker_compose_dir }}/npm/docker-compose.yml"
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      register: npm_compose

    - name: Start NPM stack
      ansible.builtin.command:
        cmd: docker compose up -d
        chdir: "{{ docker_compose_dir }}/npm"
      when: npm_compose.changed
      become_user: "{{ main_user }}"

    # --- Monitoring (Beszel Hub, Uptime Kuma, AutoKuma, Dozzle) ---
    - name: Create monitoring directory
      ansible.builtin.file:
        path: "{{ docker_compose_dir }}/monitoring"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"

    - name: Create monitoring appdata directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      loop:
        - "{{ appdata_dir }}/beszel/hub-data"
        - "{{ appdata_dir }}/uptime-kuma"
        - "{{ appdata_dir }}/autokuma"

    - name: Deploy monitoring compose
      ansible.builtin.template:
        src: monitoring-compose.yml.j2
        dest: "{{ docker_compose_dir }}/monitoring/docker-compose.yml"
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      register: monitoring_compose

    - name: Start monitoring stack
      ansible.builtin.command:
        cmd: docker compose up -d
        chdir: "{{ docker_compose_dir }}/monitoring"
      when: monitoring_compose.changed
      become_user: "{{ main_user }}"

    # --- Vaultwarden ---
    - name: Create Vaultwarden directory
      ansible.builtin.file:
        path: "{{ docker_compose_dir }}/vaultwarden"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"

    - name: Create Vaultwarden appdata directory
      ansible.builtin.file:
        path: "{{ appdata_dir }}/vaultwarden/data"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"

    - name: Deploy Vaultwarden compose
      ansible.builtin.template:
        src: vaultwarden-compose.yml.j2
        dest: "{{ docker_compose_dir }}/vaultwarden/docker-compose.yml"
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      register: vaultwarden_compose

    - name: Start Vaultwarden stack
      ansible.builtin.command:
        cmd: docker compose up -d
        chdir: "{{ docker_compose_dir }}/vaultwarden"
      when: vaultwarden_compose.changed
      become_user: "{{ main_user }}"

    # Repeat the same pattern for: cloudflared, homeassistant, homepage, code-server
    # The structure is identical: create dir, create appdata, template compose, start.
    ```

!!! info

    Every additional stack follows the exact same 4-step pattern:
    
    1. Create compose directory
   
    2. Create appdata directories
   
    3. Template the compose file
   
    4. Start the stack if changed

    ---

### Role: stacks_apps

Same pattern for prod-01 (appserver) stacks. Only showing a couple as examples since the pattern is identical.

??? note "roles/stacks_apps/tasks/main.yml"

    ```yaml
    ---
    # --- Paperless-ngx ---
    - name: Create Paperless directory
      ansible.builtin.file:
        path: "{{ docker_compose_dir }}/paperless"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"

    - name: Create Paperless appdata directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      loop:
        - "{{ appdata_dir }}/paperless/data"
        - "{{ appdata_dir }}/paperless/media"
        - "{{ appdata_dir }}/paperless/export"
        - "{{ appdata_dir }}/paperless/consume"
        - "{{ appdata_dir }}/paperless/pgdata"

    - name: Deploy Paperless compose
      ansible.builtin.template:
        src: paperless-compose.yml.j2
        dest: "{{ docker_compose_dir }}/paperless/docker-compose.yml"
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      register: paperless_compose

    - name: Start Paperless stack
      ansible.builtin.command:
        cmd: docker compose up -d
        chdir: "{{ docker_compose_dir }}/paperless"
      when: paperless_compose.changed
      become_user: "{{ main_user }}"

    # --- Forgejo ---
    - name: Create Forgejo directory
      ansible.builtin.file:
        path: "{{ docker_compose_dir }}/forgejo"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"

    - name: Create Forgejo appdata directory
      ansible.builtin.file:
        path: "{{ appdata_dir }}/forgejo/data"
        state: directory
        owner: "{{ main_user }}"
        group: "{{ main_user }}"

    - name: Deploy Forgejo compose
      ansible.builtin.template:
        src: forgejo-compose.yml.j2
        dest: "{{ docker_compose_dir }}/forgejo/docker-compose.yml"
        owner: "{{ main_user }}"
        group: "{{ main_user }}"
      register: forgejo_compose

    - name: Start Forgejo stack
      ansible.builtin.command:
        cmd: docker compose up -d
        chdir: "{{ docker_compose_dir }}/forgejo"
      when: forgejo_compose.changed
      become_user: "{{ main_user }}"

    # Repeat for: affine, vikunja, trilium, utilities, omada, socket-proxy
    ```

---