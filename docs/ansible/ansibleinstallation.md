# Ansible Installation

---

Official Installation instructions can be found [here](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html). For OS specific installation, go [here](https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html)

This will show installation on Ubuntu.

## **Installing Ansible on Ubuntu**

Ubuntu provides Ansible packages through a Personal Package Archive (PPA) that contains more recent versions than the standard repositories.

Ubuntu builds are available [in a PPA here](https://launchpad.net/~ansible/+archive/ubuntu/ansible).

Configure the PPA on your system and install Ansible:

```bash
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo add-apt-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```

!!!note
    On older Ubuntu distributions, “software-properties-common” is called “python-software-properties”. You may want to use `apt-get` rather than `apt` in older versions. Also, only newer distributions (18.04, 18.10, and later) have a `-u` or `--update` flag. Adjust your script as needed.