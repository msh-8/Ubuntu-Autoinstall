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

## Playbook structure
* Global configuration variables defined at [ansible/defaults/main.yml](ansible/defaults/main.yml)
* Template for user-data file at [ansible/templates/user-data.j2](ansible/templates/user-data.j2)
* Template for grub file at [ansible/templates/grub-cfg.j2]

## Variables
The main key of variables is "Deploy". If you want to use any variable in templates, you have to change the Deploy to true boolean value.
### Global_CFG:
* **Working_DIR**     : All files will be located here
* **Ubuntu_ISO**      : Ubuntu ISO name (it will be downloaded.)
* **Ubuntu_ISO_URL**  : The url of the ISO file.

### Nocloud_Net:
Configuration of nocloud_net should be defined here. If you want to target server get user-data config file over http, configure parameters.

* **Http_Ip**      : IP of the http server that publish the user-data and meta-data file.
* **Http_Port**    : the port of the web server

### Static_Boot_IP:
If you want to set ip static on the target server when system is booting, please configure the parameters.
If **Deploy** set to false use DHCP.

* **IP**      : IP address of target system over boot procedure.
* **Subnet**  : Subnet of IP address.
* **Gateway** : Gateway to find the route of http web.

### Disk_CFG:
The customization of block, filesystem and mount point placed here. If you want to change disk layer such as root, boot, home, etc, you will be able to set the following parameters.
If you set **Deploy** to false, the default parameters will be setting.
* **Partition_customization**  : Partition customization section.
    * **Pv0_Size**                 : Size of the pv(physical volume) to use on LVM.
* **LV_Customization**         : The list of the mount points(LVM).
    * **Name**                 : Name of the LV in the target server.
    * **Path**                 : Mount point path of LV.   
    * **Size**                 : Size of LV \
          Note: If you want to use the whole size of the disk for an LV, you can set -1 for size. For example:<br>
            Size: -1<br>
          Otherwize:<br>
            Size: 10**G**<br>

### Users_CFG:
The desired users and their sudo permission will be confiured on **Users_CFG** section.
* **Users**         : List of users
    * **Username**    : username
    * **

# Usage

    ansible-playbook -i hosts site.yaml

### See also example [hosts](hosts) & [site_vars.yml](site_vars.yml)

### An example to setup network boot services for CentOS 7

    # configure epel release
    yum install http://mirror.yandex.ru/epel/7/x86_64/e/epel-release-7-10.noarch.rpm

# Aditional links
The following links could help you to understand the customization the Ubuntu with cloud-init:
* Ubuntu 22.04 Server Autoinstall ISO: [Puget Systems](https://www.pugetsystems.com/labs/hpc/ubuntu-22-04-server-autoinstall-iso/)
* Automate Ubuntu Server 20.04 Installation with Ansible: [Rutgerblom](https://rutgerblom.com/2020/07/27/automated-ubuntu-server-20-04-installation-with-ansible/)
* Customize cloud-init autoinstall (user-data) Ubuntu 20.04: [GoLinuxCloud.](https://www.golinuxcloud.com/customize-cloud-init-user-data-ubuntu/#2_Install_Ubuntu_with_GUI_GNOME_Desktop)
* Automating Custom ISO with Cloud-Init: [Dev.to](https://dev.to/otomato_io/automating-custom-iso-with-cloud-init-2lc2)
* How to automate a bare metal Ubuntu 22.04 LTS installation: [Jimangel.io](https://www.jimangel.io/posts/automate-ubuntu-22-04-lts-bare-metal/)
