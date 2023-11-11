# Ubuntu Autoinstall
# Updates
* I tested the playbok when Nocloud_net is not enabled. When I be sure about NoCloud_net, get an update aboute it(2023-11-11) 
# Overview

The project goal is to provide a flexible and as simple as possible configuration for unattended OS installation via ISO file for Ubuntu linux distribution. 
It was designed to prepare Ubuntu Autoinstall file(cloud-init) and can be used by professional system administrators or for development/testing/education purposes.
The Ansible playbooks contain ready to use installation templates for various methods.

# Features

Templates for unattended installation included in the playbook:
* Creating user-data and meta-data file for each target-server on "/cdrom/autoinstall/" path. (when Nocloud_net is not Deployed)
* Creating user-data and meta-data file for each target-server on "www/" path.(when Nocloud_net is Deployed)
* Auto-detecting EFI and BIOS system.
* Creating an approprate partition table based on EFI or BIOS requirement.
* Assigning desired IP address when target server will boot to fetch user-data and meta-data file from http server.(Static_Boot_CFG).
* Detecting block device name such as sda or vda automatically.
* Auto-detecting active(up) interface and assign the ip to first interface. (it is tested on a physical server with multiple interfaces.)
* Logical volume customization.
* Creating multiple users either as sudo or normal.

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
  locale: en_US                                                 # locale
  keyboard:                                                     # Keyboard section
    layout: us                                                  # keyboard layout
    variant: intl                                               # keyboard variant
  apt:                                                          # Apt section
    fallback: continue-anyway                                   # it does not interrupt the autoinstall procidure if there is no any internet connection.
  network:                                                      # network section. The following section write exactly on netplan file.
    network:                                                    # network configuration
      version: 2   
      ethernets:      
        nicname:                                                # It will be changed with early-commands that handeled by early-commands
          addresses: [{{ item.Network.Ip }}]                    # ip address of the target system (it depends on your variable file.)
          gateway4: {{ item.Network.Gateway }}                  # ip address of gateway (it depends on your variable file.)
          nameservers:                                          # DNS section
            addresses:                                          # name server addresses
            {% if item.Network.DNS  | type_debug == 'list' -%}  # it check the DNS in variable file is setting with multiple address.
            {% for nsaddr in item.Network.DNS -%}               # iterate on the DNS list
              - {{ nsaddr }}                                    # set each item of the list
            {% endfor -%}                                       # end for
            {% else -%}                                         # if DNS in variable file is setting with a single address.
             - {{ item.Network.DNS }}                           # set the DNS value.
            {% endif -%}                                        # end if  {% if item.Network.DNS  | type_debug == 'list' -%} 
#
  ssh:                                                         # ssh configuration section
    install-server: yes                                        # install ssh server(sshd.service)
  {% if ( (Package_CFG.Deploy == true) and                     # Check that the Package_CFG is selected to deploy.
        (Package_CFG.Package_List | length > 0) ) -%}
  packages:                                                    # package configuration section.
                                                               # Note: this section requires the Internet connection and
                                                               # if is selected to deploy, but there is no any internet connection
                                                               # then the installation porcess will be failed.
  {% for pkgs in Package_CFG.Package_List %}                   # iterate on the Package_List to set the items.
  - {{ pkgs }}
  {% endfor -%}                                                # end for 
  {% endif -%}                                                 # end if ({% if ( (Package_CFG.Deploy == true) and)
# Storage configuration section
# Note: The following configuration is seeting only as a sample configuration. It will be changed with grub.py or bios.py scripts after running early-commands.
  storage:
     config:
      - {ptable: gpt, path: /dev/sda, preserve: false, name: '', id: disk-sda, type: disk, grub_device: false}
      - {device: disk-sda, size: 800M, wipe: superblock, flag: boot, number: 1, preserve: false, grub_device: true, id: partition-0, type: partition}
      - {fstype: fat32, volume: partition-0, preserve: false, id: format-0, type: format}
      - {device: disk-sda, size: 2G, wipe: superblock, number: 2, preserve: false, offset: 801112064, id: partition-1, type: partition}
      - {fstype: ext4, volume: partition-1, preserve: false, id: format-1, type: format}
      - {device: disk-sda, size: -1, wipe: superblock, number: 3, preserve: false, id: partition-2, type: partition}
      - {name: ubuntu-vg, devices: [partition-2], preserve: false, id: lvm_volgroup-0, type: lvm_volgroup}
      - {name: ubuntu-lv, volgroup: lvm_volgroup-0, size: 10G, wipe: superblock, preserve: false, path: /dev/ubuntu-vg/ubuntu-lv, id: lvm_partition-0, type: lvm_partition}
      - {fstype: ext4, volume: lvm_partition-0, preserve: false, id: format-2, type: format}
      - {path: /, device: format-2, id: mount-2, type: mount}
      - {path: /boot, device: format-1, id: mount-1, type: mount}
      - {path: /boot/efi, device: format-0, id: mount-0, type: mount}
  {% if ( (Users_CFG.Deploy == true) and        # check the Users.CFG.Deploy is selected to deplpy.
     (Users_CFG.Users | length > 0) ) -%}
#
  late-commands:                                # this section will run the commands when autoinstall finished. and commands will execute on the target server.
  {% for sudoer in Users_CFG.Users -%}          # iterate on the User_CFG.Users list to check sudo value for each user. 
  {% if sudoer.Sudo == true -%}                 # check the user is selected as sudoer.
  - echo '{{ sudoer.Username }} ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/{{ sudoer.Username }} # Allow user to run sudo without password
  {% endif -%}                                  # end if ({% if sudoer.Sudo == true -%})
  {% endfor -%}                                 # end for
  user-data:                                    # user-data section. This section is used here to setup users.
    disable_root: false                         # enable root user
    hostname: "{{ item.OS.Host_Name }}"         # set hostname for target server.
    users:                                      # users section
      {% for user in Users_CFG.Users -%}        # iterate on the users lists to select items.   
      - name: {{ user.Username }}               # set username
        passwd: {{ user.Password }}             # set password (hashed password)
        shell: /bin/bash                        # set shell for each user.
        lock_passwd: false                      # disable lock password
      {% endfor -%}                             # end for {% for user in Users_CFG.Users -%}
  {% endif -%}                                  # end if   (**{% if ( (Users_CFG.Deploy == true) and ...) 
```

### grub-cfg.j2
This file will create grub.cfg file on the "/cdrom/bood/grub/" path. It create the boot requirements on ISO.
Note: For some limitation in template language I do not write any comment on this file. If you intend to know about comments you should view the [grub-cfg-with-comment.j2](ansible/templates/grub-cfg-with-comment.j2) (**Do not use this file.**)
```
set timeout=10                                                               # set time out for boot page

loadfont unicode                                                             # font setting

set menu_color_normal=white/black                                            # color of boot page.
set menu_color_highlight=black/light-gray                                    # color of boot page.
{% if Nocloud_Net.Deploy  == true %}                                         # check the Nocloud_net is selected in the variables.
{% if Static_Boot_Ip.Deploy  == true %}                                      # check the Static_Boot_IP is selected in the variables.
menuentry "Autoinstall Ubuntu Server" {                                      # menu entry for autoinstall
    set gfxpayload=keep
   # The following line will be configure nocloud-net setting for autoinstall.(it depend on your variables settings.)
   # The live os will get user-data and meta-data files of the your web server.
   # The live os will not search CD-Rom to find user-data and meta-data files.
   # The following setting is required a web server that serve the related files(user-data and meta-data).
   # The live os need an established connection to the web server. It means that you need set static ip address at system boot to connect to the web server. 
   # For example:
          # linux   /casper/vmlinuz ip=192.168.1.100::192.168.1.1:255.255.255.0 quiet autoinstall ds=nocloud-net\;s=http://192.169.1.10:8080/  ---
    linux   /casper/vmlinuz ip={{ Static_Boot_Ip.Ip }}::{{ Static_Boot_Ip.Gateway }}:{{ Static_Boot_Ip.Subnet }} quiet autoinstall ds=nocloud-net\;s=http://{{ Nocloud_Net.Http_Ip }}:{{ Nocloud_Net.Http_Port }}/  ---
    initrd  /casper/initrd
}
{% else %}                                                                  # otherwize
menuentry "Autoinstall Ubuntu Server" {                                     # if the static IP is not selected bu nocloud_net is selected, DHCP will set as follows:
    set gfxpayload=keep
   # The following line will be configure nocloud-net setting for autoinstall.(it depend on your variables settings.)
   # The live os will get user-data and meta-data files of the your web server.
   # The live os will not search CD-Rom to find user-data and meta-data files.
   # The following setting is required a web server that serve the related files(user-data and meta-data).
   # The live os need an established connection to the web server. It means that you need DHCP-server to assign ip address to system boot. 
   # For example:
    linux   /casper/vmlinuz ip=dhcp quiet autoinstall ds=nocloud-net\;s=http://{{ Nocloud_Net.Http_Ip }}:{{ Nocloud_Net.Http_Port }}/  ---
    initrd  /casper/initrd
}
{% endif %}                                                                   # endif ({% if Static_Boot_Ip.Deploy  == true %})
{% else %}                                                                    # if neither boot_ip nor nocloud_net is selected to deploy.
menuentry "Autoinstall Ubuntu Server" {
    set gfxpayload=keep
    linux   /casper/vmlinuz quiet autoinstall ds=nocloud\;s=/cdrom/autoinstall/  ---
    initrd  /casper/initrd
}
{% endif %}
menuentry "Try or Install Ubuntu Server" {
        set gfxpayload=keep
        linux   /casper/vmlinuz  ---
        initrd  /casper/initrd
}
menuentry "Ubuntu Server with the HWE kernel" {
        set gfxpayload=keep
        linux   /casper/hwe-vmlinuz  ---
        initrd  /casper/hwe-initrd
}
grub_platform
if [ "$grub_platform" = "efi" ]; then
menuentry 'Boot from next volume' {
        exit 1
}
menuentry 'UEFI Firmware Settings' {
        fwsetup
}
else
menuentry 'Test memory' {
        linux16 /boot/memtest86+.bin
}
fi                  # end if ({% if Nocloud_Net.Deploy  == true %})

```

# Usage
### Install requirements
      sudo aptg-et upgrade
      sudo apt-get install git 
      sudo apt-get install p7zip-full
      sudo apt-get install xorriso
      sudo apt-get install python3
      sudo apt-get install python3-pip
      sudo python3 -m pip install ansible
      git clone https://github.com/msh-8/autoinstall.git
      cd Ubuntu-Autoinstall
      
### Run playbook
**First of all, please change the variables as your requirements.**
```
ansible-playbook autoinstall.yaml
```

# Aditional links
The following links could help you to understand the customization the Ubuntu with cloud-init:
* Ubuntu 22.04 Server Autoinstall ISO: [Puget Systems](https://www.pugetsystems.com/labs/hpc/ubuntu-22-04-server-autoinstall-iso/)
* Automate Ubuntu Server 20.04 Installation with Ansible: [Rutgerblom](https://rutgerblom.com/2020/07/27/automated-ubuntu-server-20-04-installation-with-ansible/)
* Customize cloud-init autoinstall (user-data) Ubuntu 20.04: [GoLinuxCloud.](https://www.golinuxcloud.com/customize-cloud-init-user-data-ubuntu/#2_Install_Ubuntu_with_GUI_GNOME_Desktop)
* Automating Custom ISO with Cloud-Init: [Dev.to](https://dev.to/otomato_io/automating-custom-iso-with-cloud-init-2lc2)
* How to automate a bare metal Ubuntu 22.04 LTS installation: [Jimangel.io](https://www.jimangel.io/posts/automate-ubuntu-22-04-lts-bare-metal/)
