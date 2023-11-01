# Ubuntu Autoinstall

# Languages
* persian: [Persian_README](https://github.com/msh-8/autoinstall/blob/master/per-README.MD)


# Overview

The project goal is to provide a flexible and as simple as possible configuration for unattended OS installation via ISO file for Ubuntu linux distribution. 
It was designed to prepare Ubuntu Autoinstall file(cloud-init) and can be used by professional system administrators or for development/testing/education purposes.
The Ansible playbooks contain ready to use installation templates for various methods.

# Features

Templates for unattended installation included in the playbook:
* Embeded cloud-init file on the ISO file.(make ISO file per each change.)
* Cloud-init file over http web server.(make ISO once and change cloud-init file on-demand.)

# Caveats

In the order to effectively use the playbook for your particular purposes you have to understand the principles of Linux network boot
and have a base knowledge about autoinstall files for the required OS: 
* Cloud-init documents: [Cloud-init Docs](https://cloudinit.readthedocs.io/en/latest/)
* Curtin documents: [curtin’s documentation](https://curtin.readthedocs.io/en/latest/) ( for storage, network, apt, etc, configurations )
* BOOTP/DHCP client for klibc: [klibc/klibc.git](https://git.kernel.org/pub/scm/libs/klibc/klibc.git/tree/usr/kinit/ipconfig/README.ipconfig) ( for setting static ip address in boot )

# Requirments

* Ansible version >= 2.2.1.0
* xorriso
* 7z
* python3

# Description

ongoing

# Aditional links
The following links could help you to understand the customization the Ubuntu with cloud-init:
* Ubuntu 22.04 Server Autoinstall ISO: [Puget Systems](https://www.pugetsystems.com/labs/hpc/ubuntu-22-04-server-autoinstall-iso/)
* Automate Ubuntu Server 20.04 Installation with Ansible: [Rutgerblom](https://rutgerblom.com/2020/07/27/automated-ubuntu-server-20-04-installation-with-ansible/)
* Customize cloud-init autoinstall (user-data) Ubuntu 20.04: [GoLinuxCloud.](https://www.golinuxcloud.com/customize-cloud-init-user-data-ubuntu/#2_Install_Ubuntu_with_GUI_GNOME_Desktop)
* Automating Custom ISO with Cloud-Init: [Dev.to](https://dev.to/otomato_io/automating-custom-iso-with-cloud-init-2lc2)
* How to automate a bare metal Ubuntu 22.04 LTS installation: [Jimangel.io](https://www.jimangel.io/posts/automate-ubuntu-22-04-lts-bare-metal/)
