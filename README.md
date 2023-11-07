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
* Tasks are setting at [ansible/tasks/main.yml](ansible/tasks/main.yml)
* variables are defined at [ansible/defaults/main.yml](ansible/defaults/main.yml)
* Template for user-data file at [ansible/templates/user-data.j2](ansible/templates/user-data.j2)
* Template for grub file at [ansible/templates/grub-cfg.j2](ansible/templates/grub-cfg.j2)

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
* **Users**           : List of users
    * **Username**    : Username
    * **Sudo**        : Make a user as a sudoer(Boolean=> True/False)
    * **Password**    : Hashed_password(openssl passwd -6) 

### Package_CFG:
The list of packages supposed to be intalled on target server(s).<br>
Note: the usage of this parameter need an Internet connection to get packages from repositories.
* If there is no any Internet connection please change value of **Deploy** to false.
* **Package_List**      : The list of packages.
     ```
     - htop              : The name of package.
     - traceroute        : The name of package.

### Host_CFG:
To configure OS of the target server use this section. Each list item has a **Deploy** variable to define whether the target ISO will be created for this host(target server) or not.
* **OS**                  : The general parameter like hostaname, keyboard layout and Locale.
   * **Host_Name**        : Host name for target server.
   * **Locale**           : Locale
   * **Keyboardlayout**   : Keyboard layout
   * **Keyboardvariant**  : Keyboard variant
* **Network**             : Network Parameters
   * **Ip**               : IP address/CIDR (192.168.1.1/24)
   * **Gateway**          : gateway ip
   * **DNS**              : address or addresses of nameservers. ( **List** )
     ```
       - 8.8.8.8
       - 4.2.2.1
## Templates
### user-data.j2
This file will create the user-data file include required cloud-init configuration.
Note: For some limitation in template language I do not write any comment on this file. If you intend to know about comments you should view the [user-data-with-comment.j2](ansible/templates/user-data-with-comment.j2). (**Do not use this file.**)
```
#cloud-config                                      # Type of file
autoinstall:                                       # Subiquity realize that it is autoinstall configuration
  version: 1                                       # Version of autoinstall configuration
  early-commands:                                  # put any command that should be ran before starting autoinstall.
    - |                                            # Array of commands
      cat << EOF >> /test.sh                       # Create an script in "/" path.
      #!/bin/bash                                  # Shebang
      if [ -e "/sys/firmware/efi"  ]; then         # Check the target server boot is EFI or BIOS
       python3 /grub.py                            # run grub.py file in "/" path if system is EFI
      else                                         # Otherwize:
        python3 /bios.py                           # run bios.py file in "/" path if system is BIOS
      fi                                           # End if.
      ip=\$(grep  -E '.*\/[0-9]{1,2}' /autoinstall.yaml  | awk -F'-' '{print\$2}')            # find the ip address in the autoinstall.yaml file.
      gateway=\$(grep -E 'gateway4:.[0-9]{1,3}\.' /autoinstall.yaml | awk '{print\$2}')         #  find the gatewat address in the autoinstall.yaml file.

# Note: The following codes for target servers have multiple interfaces.
# The script will find the active(up) interfaces and set the ip address that given in autoinstall file on each interface then test connection with the gateway.
# Finally it finds the exact interface that is up and can connect to the network.#
 
      for interface in \$(ls /sys/class/net); do               # find the insterfaces on the target server.
        if [ \${interface} = 'lo' ]; then                      # exclude the loopback interface.
          continue
        elif [ \$(cat /sys/class/net/\${interface}/operstate) = 'up' ]; then   # check the interface is in up state.
          ip addr flush dev \${interface}                                      # clear the ip address of the interface.
          ip addr add \${ip} dev \${interface}                                 # set the ip address
          ping -c2 \$(echo \${gateway}) &>/dev/null                            # ping the gateway
          status_code=\$?                                                      # keep ping status code
          if [ \${status_code} -eq 0 ]; then                                   # check the ping status code.
            nicname=\${interface}                                              # if statu code == 0 ,select the interface and, 
            ip addr flush dev \${interface}                                    # then clear the ip address and,
            break                                                              # break the loop
          else                                                                 # otherwize
            ip addr flush dev \${interface}                                    # clear ip address to set it on the next interface.
            nicname=en1                                                        # set en1 as default interface name if all interface check was failed.
          fi                                                                   # end if
      	fi                                                                    # end if
      done                                                                     # end loop 
      sed -i "s/nicname/\$nicname/" /autoinstall.yaml                          # change nicname to selected interface in /autoinstall.yaml file
      EOF                                                                      # End of the cat.
 #############################################################################################################################
#                                                    NOTE:                                                                    #
# The following scripts will create a file as grub.py name in the "/" paht.                                                   #
# The python code is used here to edit and configure storage section of/autoinstall.yaml.                                     #
# The storage configuration for EFI system will be handeled with the following python codes.                                  #
# The code will select block device name such as sda or vda by its own automatically.                                         #
 #############################################################################################################################
    - |                                                                        # start second early command in autoinstall.yaml
      cat << EOF > /grub.py                                                    # create grub.py in "/" path.
      #!/usr/bin/python3                                                       # Shebang
      import yaml                                                              # import yaml module
      import subprocess                                                        # import subprocess module
      diskname = subprocess.getoutput("lsblk -o NAME | grep -E 'sda$|vda$'")   # find disk name
      #######################################################################################################
     #                                       storage section configuration                                   #
      #######################################################################################################
      # Create gpt partition table for sda or vda disk.
      disk_dic = {'id': diskname, 'type': 'disk', 'wipe': 'superblock', 'ptable': 'gpt', 'path': str('/dev/'+diskname), 'grub_device': False}
      # Create 800M for efi bootloader partition (/boot/efi)
      part1_dic = {'id': str(diskname+'1'), 'type': 'partition', 'size': '800M', 'device': diskname, 'flag': 'boot', 'wipe': 'superblock', 'grub_device': True, 'number': 1}
      # Create filesystem format(fat32) for efi partition (/boot/efi)
      fmt1_dic = {'id': str(diskname+'1-format'), 'type': 'format', 'fstype': 'fat32', 'label': 'BOOTEFI', 'volume': str(diskname+'1')}
      # Create 2G partition for boot.(/boot)
      part2_dic = {'id': str(diskname+'2'), 'type': 'partition', 'size': '2G', 'device': diskname, 'name': 'boot-partition', 'wipe': 'superblock', 'number': 2}
      # Create filesystem format(ext4) for boot partition(/boot)
      fmt2_dic = {'id': str(diskname+'2-format'), 'type': 'format', 'fstype': 'ext4', 'label': 'BOOT', 'volume': str(diskname+'2')}
      # Create lvm partition
      part3_dic = {'id': str(diskname+'3'), 'type': 'partition', 'size': -1, 'device': diskname, 'name': 'root-partition', 'wipe': 'superblock', 'number': 3}
      # Create volume group(vg)
      volgp_dic = {'id': 'vg0', 'devices': [str(diskname+'3')], 'type': 'lvm_volgroup', 'name': 'vg0'}
      # Create root lv. (it depends on your variable setting)
      lvm_part_rootlv_dic = {'id': 'rootlv', 'type': 'lvm_partition', 'volgroup': 'vg0', 'name': 'rootlv', 'size': '10G', 'wipe': 'superblock'}
      # Create varlv.   (it depends on your variable setting)
      lvm_part_varlv_dic = {'id': 'varlv', 'type': 'lvm_partition', 'volgroup': 'vg0', 'name': 'varlv', 'size': '4G', 'wipe': 'superblock'}
      # Create homelv. (it depends on your variable setting)
      lvm_part_homelv_dic = {'id': 'homelv', 'type': 'lvm_partition', 'volgroup': 'vg0', 'name': 'homelv', 'size': '8G', 'wipe': 'superblock'}
      # Create filesystem format(ext4) for rootlv(it depends on your variable setting)
      lvm_fmt_rootlv_dic = {'id': 'rootlv-format', 'type': 'format', 'fstype': 'ext4', 'volume': 'rootlv', 'preserve': False}
      # Create filesystem format(ext4) for varlv(it depends on your variable setting)
      lvm_fmt_varlv_dic = {'id': 'varlv-format', 'type': 'format', 'fstype': 'ext4', 'volume': 'varlv', 'preserve': False}
      # Create filesystem format(ext4) for homelv(it depends on your variable setting)
      lvm_fmt_homelv_dic = {'id': 'homelv-format', 'type': 'format', 'fstype': 'ext4', 'volume': 'homelv', 'preserve': False}
      # Create mount point for /boot/efi.
      mnt1_dic = {'id': str(diskname+'1-mount'), 'type': 'mount', 'path': '/boot/efi', 'device': str(diskname+'1-format')}
      # Create mount point for /boot.
      mnt2_dic = {'id': str(diskname+'2-mount'), 'type': 'mount', 'path': '/boot', 'device': str(diskname+'2-format')}
      # Create mount point for "/".(it depends on your variable setting)
      lvm_mnt_rootlv_dic = {'id': 'rootlv-mnt', 'type': 'mount', 'path': '/', 'device': 'rootlv-format'}
      # Create mount point for "/var".(it depends on your variable setting)
      lvm_mnt_varlv_dic = {'id': 'varlv-mnt', 'type': 'mount', 'path': '/var', 'device': 'varlv-format'}
      # Create mount point for "/home".(it depends on your variable setting)
      lvm_mnt_homelv_dic = {'id': 'homelv-mnt', 'type': 'mount', 'path': '/home', 'device': 'homelv-format'}
      # Create a list from above dictionaries to configure storage section in autoinstall.yaml file.
      disk_cfg_lst = [disk_dic, part1_dic, fmt1_dic, part2_dic, fmt2_dic, part3_dic, volgp_dic, lvm_part_rootlv_dic, lvm_part_varlv_dic, lvm_part_homelv_dic, lvm_fmt_rootlv_dic, lvm_fmt_varlv_dic, lvm_fmt_homelv_dic, mnt1_dic, mnt2_dic, lvm_mnt_rootlv_dic, lvm_mnt_varlv_dic, lvm_mnt_homelv_dic]
      with open('/autoinstall.yaml', 'r') as reader:                  # load the /autoinstall.yaml file
         autoinstall = yaml.safe_load(reader)                     
      autoinstall['storage'].clear()                                 # clear the previous storage section from yaml file.
      autoinstall['storage'].update({'config' : disk_cfg_lst})       # append disk_cfg_list contains above dictionaries in the autoinstall.    
      with open('/autoinstall.yaml', 'w') as writer:                 # write and close the /autoinstall.yaml
        yaml.dump(autoinstall, writer)                               
      EOF
    - |                                                              # the third early-command
#############################################################################################################################
#                                                    NOTE:                                                                    #
# The following scripts will create a file as bios.py name in the "/" paht.                                                   #
# The python code is used here to edit and configure storage section of/autoinstall.yaml.                                     #
# The storage configuration for BIOS system will be handeled with the following python codes.                                 #
# The code will select block device name such as sda or vda by its own automatically.                                         #
# The following code descrition is same as grub.py code with only different partition table                                   #
 #############################################################################################################################
      cat << EOF > /bios.py
      #!/usr/bin/python3
      import yaml
      import subprocess
      diskname = subprocess.getoutput("lsblk -o NAME | grep -E 'sda$|vda$'")
      disk_dic = {'id': diskname, 'type': 'disk', 'wipe': 'superblock', 'ptable': 'gpt', 'path': str('/dev/'+diskname), 'grub_device': True}
      part1_dic = {'id': str(diskname+'1'), 'type': 'partition', 'size': '1M', 'device': diskname, 'flag': 'bios_grub', 'wipe': 'superblock', 'number': 1}
      part2_dic = {'id': str(diskname+'2'), 'type': 'partition', 'size': '2G', 'device': diskname, 'name': 'boot-partition', 'wipe': 'superblock', 'number': 2}
      fmt2_dic = {'id': str(diskname+'2-format'), 'type': 'format', 'fstype': 'ext4', 'label': 'BOOT', 'volume': str(diskname+'2')}
      part3_dic = {'id': str(diskname+'3'), 'type': 'partition', 'size': -1, 'device': diskname, 'name': 'root-partition', 'wipe': 'superblock', 'number': 3}
      volgp_dic = {'id': 'vg0', 'devices': [str(diskname+'3')], 'type': 'lvm_volgroup', 'name': 'vg0'}
      lvm_part_rootlv_dic = {'id': 'rootlv', 'type': 'lvm_partition', 'volgroup': 'vg0', 'name': 'rootlv', 'size': '10G', 'wipe': 'superblock'}
      lvm_part_varlv_dic = {'id': 'varlv', 'type': 'lvm_partition', 'volgroup': 'vg0', 'name': 'varlv', 'size': '4G', 'wipe': 'superblock'}
      lvm_part_homelv_dic = {'id': 'homelv', 'type': 'lvm_partition', 'volgroup': 'vg0', 'name': 'homelv', 'size': '8G', 'wipe': 'superblock'}
      lvm_fmt_rootlv_dic = {'id': 'rootlv-format', 'type': 'format', 'fstype': 'ext4', 'volume': 'rootlv', 'preserve': False}
      lvm_fmt_varlv_dic = {'id': 'varlv-format', 'type': 'format', 'fstype': 'ext4', 'volume': 'varlv', 'preserve': False}
      lvm_fmt_homelv_dic = {'id': 'homelv-format', 'type': 'format', 'fstype': 'ext4', 'volume': 'homelv', 'preserve': False}
      mnt2_dic = {'id': str(diskname+'2-mount'), 'type': 'mount', 'path': '/boot', 'device': str(diskname+'2-format')}
      lvm_mnt_rootlv_dic = {'id': 'rootlv-mnt', 'type': 'mount', 'path': '/', 'device': 'rootlv-format'}
      lvm_mnt_varlv_dic = {'id': 'varlv-mnt', 'type': 'mount', 'path': '/var', 'device': 'varlv-format'}
      lvm_mnt_homelv_dic = {'id': 'homelv-mnt', 'type': 'mount', 'path': '/home', 'device': 'homelv-format'}
      disk_cfg_lst = [disk_dic, part1_dic, part2_dic, fmt2_dic, part3_dic, volgp_dic, lvm_part_rootlv_dic, lvm_part_varlv_dic, lvm_part_homelv_dic, lvm_fmt_rootlv_dic, lvm_fmt_varlv_dic, lvm_fmt_homelv_dic, mnt2_dic, lvm_mnt_rootlv_dic, lvm_mnt_varlv_dic, lvm_mnt_homelv_dic]
      with open('/autoinstall.yaml', 'r') as reader:
        autoinstall = yaml.safe_load(reader)
      autoinstall['storage'].clear()
      autoinstall['storage'].update({'config' : disk_cfg_lst})
      with open('/autoinstall.yaml', 'w') as writer:
        yaml.dump(autoinstall, writer)
      EOF                                                       # end of the cat
    - chmod +x /grub.py                                         # make executable /grub.py file.
    - chmod +x /bios.py                                         # make executable /bios.py file.
    - chmod +x /test.sh                                         # make executable /test.sh file.
    - bash /test.sh                                             # execute the test.sh file.

```

### grub-cfg.j2
This file will create grub.cfg file on the "/cdrom/bood/grub/" path. It create the boot requirements on ISO.
Note: For some limitation in template language I do not write any comment on this file. If you intend to know about comments you should view the [grub-cfg-with-comment.j2](ansible/templates/grub-cfg-with-comment.j2) (**Do not use this file.**)
```
comment
```

# Usage

    ansible-playbook -i hosts site.yaml



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
