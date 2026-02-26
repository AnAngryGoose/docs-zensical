# **Install Docker Engine on Debian**

[Docker Docs :simple-docker: ](https://docs.docker.com/engine/install/debian/)

---

##  **Uninstall old versions**

---

Before you can install Docker Engine, you need to uninstall any conflicting packages.

Your Linux distribution may provide unofficial Docker packages, which may conflict with the official packages provided by Docker. You must uninstall these packages before you install the official version of Docker Engine.

```bash
$ sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-doc podman-docker containerd runc | cut -f1)
```

## **Installation Methods**

---

### **Install using the apt repository**

1. Set up the `apt` repository.

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```
2. Install the Docker packages.

To install latest version:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

!!!Note
  The Docker service starts automatically after installation. To verify that Docker is running, use:

  `sudo systemctl status docker`

  Some systems may have this behavior disabled and will require a manual start:

  `sudo systemctl start docker`

3. Verify that the installation is successful by running the hello-world image:

```bash
sudo docker run hello-world
```
This command downloads a test image and runs it in a container. When the container runs, it prints a confirmation message and exits.


### **Install using the convenience script**

---

Docker provides a convenience script at [https://get.docker.com/](https://get.docker.com/) to install Docker into development environments non-interactively. The convenience script isn't recommended for production environments, but it's useful for creating a provisioning script tailored to your needs. Also refer to the [install using the repository](https://docs.docker.com/engine/install/debian/#install-using-the-repository) steps to learn about installation steps to install using the package repository. The source code for the script is open source, and you can find it in the [`docker-install` repository on GitHub](https://github.com/docker/docker-install).

Always examine scripts downloaded from the internet before running them locally. Before installing, make yourself familiar with potential risks and limitations of the convenience script:

-   The script requires `root` or `sudo` privileges to run.
-   The script attempts to detect your Linux distribution and version and configure your package management system for you.
-   The script doesn't allow you to customize most installation parameters.
-   The script installs dependencies and recommendations without asking for confirmation. This may install a large number of packages, depending on the current configuration of your host machine.
-   By default, the script installs the latest stable release of Docker, containerd, and runc. When using this script to provision a machine, this may result in unexpected major version upgrades of Docker. Always test upgrades in a test environment before deploying to your production systems.
-   The script isn't designed to upgrade an existing Docker installation. When using the script to update an existing installation, dependencies may not be updated to the expected version, resulting in outdated versions.

!!!Tip
Preview script steps before running. You can run the script with the `--dry-run` option to learn what steps the script will run when invoked:

    $ curl -fsSL https://get.docker.com -o get-docker.sh
    $ sudo sh ./get-docker.sh --dry-run
     

This example downloads the script from [https://get.docker.com/](https://get.docker.com/) and runs it to install the latest stable release of Docker on Linux:

    $ curl -fsSL https://get.docker.com -o get-docker.sh
    $ sudo sh get-docker.sh
    Executing docker install script, commit: 7cae5f8b0decc17d6571f9f52eb840fbc13b2737
    <...>
    

You have now successfully installed and started Docker Engine. The `docker` service starts automatically on Debian based distributions. On `RPM` based distributions, such as CentOS, Fedora or RHEL, you need to start it manually using the appropriate `systemctl` or `service` command. As the message indicates, non-root users can't run Docker commands by default.

> **Use Docker as a non-privileged user, or install in rootless mode?**
> 
> The installation script requires `root` or `sudo` privileges to install and use Docker. If you want to grant non-root users access to Docker, refer to the [post-installation steps for Linux](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user). You can also install Docker without `root` privileges, or configured to run in rootless mode. For instructions on running Docker in rootless mode, refer to [run the Docker daemon as a non-root user (rootless mode)](https://docs.docker.com/engine/security/rootless/).

#### [Install pre-releases](https://docs.docker.com/engine/install/debian/#install-pre-releases)

Docker also provides a convenience script at [https://test.docker.com/](https://test.docker.com/) to install pre-releases of Docker on Linux. This script is equal to the script at `get.docker.com`, but configures your package manager to use the test channel of the Docker package repository. The test channel includes both stable and pre-releases (beta versions, release-candidates) of Docker. Use this script to get early access to new releases, and to evaluate them in a testing environment before they're released as stable.

To install the latest version of Docker on Linux from the test channel, run:

    $ curl -fsSL https://test.docker.com -o test-docker.sh
    $ sudo sh test-docker.sh
    

#### [Upgrade Docker after using the convenience script](https://docs.docker.com/engine/install/debian/#upgrade-docker-after-using-the-convenience-script)

If you installed Docker using the convenience script, you should upgrade Docker using your package manager directly. There's no advantage to re-running the convenience script. Re-running it can cause issues if it attempts to re-install repositories which already exist on the host machine.

## [Uninstall Docker Engine](https://docs.docker.com/engine/install/debian/#uninstall-docker-engine)

1.  Uninstall the Docker Engine, CLI, containerd, and Docker Compose packages:
    
        $ sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
        
    
2.  Images, containers, volumes, or custom configuration files on your host aren't automatically removed. To delete all images, containers, and volumes:
    
        $ sudo rm -rf /var/lib/docker
        $ sudo rm -rf /var/lib/containerd
        
    
3.  Remove source list and keyrings
    
        $ sudo rm /etc/apt/sources.list.d/docker.sources
        $ sudo rm /etc/apt/keyrings/docker.asc
        
    

You have to delete any edited configuration files manually.

