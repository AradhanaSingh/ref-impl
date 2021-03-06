OSIC Deployment Process
=======================

Table of Contents
-----------------

* [Provisioning the Deployment Host](https://github.com/osic/ref-impl/blob/master/documents/bare_metal_provisioning.md#provisioning-the-deployment-host)
* [Download and Setup the osic-prep LXC Container](https://github.com/osic/ref-impl/blob/master/documents/bare_metal_provisioning.md#download-and-setup-the-osic-prep-lxc-container)
* [PXE Boot the Servers](https://github.com/osic/ref-impl/blob/master/documents/bare_metal_provisioning.md#pxe-boot-the-servers)
* [Bootstrapping the Servers](https://github.com/osic/ref-impl/blob/master/documents/bare_metal_provisioning.md#bootstrapping-the-servers)

Overview
---------

You have a number of bare metal servers and you want to build your own cloud on top of them. To achieve that goal, the first step is to have your bare metal servers provisioned with an Operating System, most likely Linux if you will later be using an Open Source platform to build your cloud. On a production deployment, the process of deploying all these servers starts by manually provisioning  the first of your servers. This host will become your deployment host and will be used later to provision the rest of the servers by booting them over Network. This mechanism is called PXE Booting where servers use their PXE-enabled Network Interface Cards to boot from a network hosted kernel.

Provisioning the Deployment Host
--------------------------------

### ILO overview

ILO or Integrated Lights-Out, is a card integrated to the motherboard in most HP ProLiant servers which allows users to remotely configure, monitor and connect to servers even though no Operating System is installed, usually called out-of-band management. ILO has its own network interface and is commonly used for:

* Power control of the server
* Mount physical CD/DVD drive or image remotely
* Request an Integrated Remote Console for the server
* Monitor server health


### Manually Provision the Deployment Host

First, download a [modified Ubuntu Server 14.04.3 ISO](http://23.253.105.87/ubuntu-14.04.3-server-i40e-hp-raid-x86_64.iso). The modified Ubuntu Server ISO contains i40e driver version 1.3.47 and HP iLO tools.

Boot the deployment host to this ISO using a USB drive, CD/DVD-ROM, iDRAC, or iLO. Whatever is easiest.

__NOTE:__ to deploy a host through ILO:

1. Open a web browser and browse to the host's ILO IP address.
2. Login with the ILO credentials
3. Request a __remote console__ from the GUI (.NET console for windows or Java console for other OSes).
4. To deploy the server, select the __Virtual Drives__ tab from the ILO console, press __Image File CD/DVD-ROM__ then select the Ubuntu image you downloaded to your local directory. Depending on your browser and OS, If you are using the Java console, you may need to allow your java plugin to run in unsafe mode, so that it can access the ubuntu image from your local directory.
5. Click the __Power Switch__ tab and select __Reset__ to reboot the host from the image.

__Before you move on, be sure you have unselected or removed the Ubuntu ISO from the ILO console (unselect Image File CD/DVD-ROM from the Virtual Drives tab)__ so that future server reboots do not continue to use it to boot.

Once the deployment host is booted to the ISO, follow these steps to begin installation:

1. Select __Language__

2. Hit __Fn + F6__

3. Dismiss the __Expert mode__ menu by hitting __Esc__.

4. Scroll to the beginning of the line and delete `file=/cdrom/preseed/ubuntu-server.seed`.

5. Type `preseed/url=http://23.253.105.87/osic.seed` in its place.

6. Hit __Enter__ to begin the install process. The console may appear to freeze for sometime.

7. You will (eventually) be prompted for the following menus:

   * Select a language
   * Select your location
   * Configure the keyboard
   * Configure the network

  DHCP detection will fail. You will need to manually select the proper network interface - typically __p1p1__ - and manually configure networking on the __PXE__ network (refer to your onboarding email to find the __PXE__ network information). When asked for name servers, type 8.8.8.8 8.8.4.4.

  You may see an error stating:
    "/dev/sda" contains GPT signatures, indicating that it had a GPT table... Is this a GPT partition table?
  If you encounter this error select "No" and continue.

Once networking is configured, the Preseed file will be downloaded. The remainder of the Ubuntu install will be unattended.

The Ubuntu install will be finished when the system reboots and a login prompt appears.

### Update Linux Kernel

Once the system boots, it can be SSH'd to using the IP address you manually assigned. Login with username __root__ and password __cobbler__.

You will need to update the Linux kernel on the deployment host in order to get an updated upstream i40e driver.

    apt-get update; apt-get install -y linux-generic-lts-xenial

When the update finishes running, __reboot__ the server and proceed with the rest of the guide.

Download and Setup the osic-prep LXC Container
----------------------------------------------

With the deployment host provisioning done, SSH to it.

Next, you will download a pre-packaged LXC container that contains a tool you need to PXE boot the rest of the servers called Cobbler

### Cobbler overview

There is a numerous tools that implement the PXE mechanism. However, we decided here to use Cobbler since it is powerful, easy to use and handy when it comes to quickly setting up network installation environments. Cobbler is a Linux based provisioning system which lets you, among other things, configure Network installation for each server from its MAC address, manage DNS and serve DHCP requests, etc.

### Setup LXC Linux Bridge

In order to use the LXC container, a new bridge will need to be created: __br-pxe__.

__NOTE: Follow these instructions very carefully.__

First, install the necessary packages:

    apt-get install vlan bridge-utils

Reconfigure the network interface file __/etc/network/interfaces__ to match the following (your IP addresses and ports will most likely be different):

    # The loopback network interface
    auto lo
    iface lo inet loopback

    auto p1p1
    iface p1p1 inet manual

    # Container Bridge
    auto br-pxe
    iface br-pxe inet static
    address 172.22.0.21
    netmask 255.255.252.0
    gateway 172.22.0.1
    dns-nameservers 8.8.8.8 8.8.4.4
    bridge_ports p1p1
    bridge_stp off
    bridge_waitport 0
    bridge_fd 0

Bring up __br-pxe__. I recommend you have access to the iLO in case the following commands fail and you lose network connectivity:

    ifdown p1p1; ifup br-pxe

### Install LXC and Configure LXC Container

Install the necessary LXC package:

    apt-get install lxc

Change into root's home directory:

    cd /root

Download the LXC container to the deployment host:

    wget http://23.253.105.87/osic.tar.gz

Untar the LXC container:

    tar xvzf /root/osic.tar.gz

Move the LXC container directory into the proper directory:

    mv /root/osic-prep /var/lib/lxc/

Once moved, the LXC container should be stopped, verify by running `lxc-ls -f`. Before starting it, open __/var/lib/lxc/osic-prep/config__ and change __lxc.network.ipv4 = 172.22.0.22/22__ to a free IP address from the PXE network you are using. Do not forget to set the CIDR notation as well. If your PXE network already is __172.22.0.22/22__, you do not need to make further changes.

    lxc.network.type = veth
    lxc.network.name = eth1
    lxc.network.ipv4 = 172.22.0.22/22
    lxc.network.link = br-pxe
    lxc.network.hwaddr = 00:16:3e:xx:xx:xx
    lxc.network.flags = up
    lxc.network.mtu = 1500

Start the LXC container:

    lxc-start -d --name osic-prep

You should be able to ping the IP address you just set for the LXC container from the host.

### Configure LXC Container

There are a few configuration changes that need to be made to the pre-packaged LXC container for it to function on your network.

Start by attaching to the LXC container:

    lxc-attach --name osic-prep

If you had to change the IP address above, reconfigure the DHCP server by running the following sed commands. You will need to change __172.22.0.22__ to match the IP address you set above:

    sed -i '/^next_server: / s/ .*/ 172.22.0.22/' /etc/cobbler/settings

    sed -i '/^server: / s/ .*/ 172.22.0.22/' /etc/cobbler/settings

Open __/etc/cobbler/dhcp.template__ and reconfigure your DHCP settings. You will need to change the __subnet__, __netmask__, __option routers__, __option subnet-mask__, and __range dynamic-bootp__ parameters to match your network.

    subnet 172.22.0.0 netmask 255.255.252.0 {
         option routers             172.22.0.1;
         option domain-name-servers 8.8.8.8;
         option subnet-mask         255.255.252.0;
         range dynamic-bootp        172.22.0.23 172.22.0.200;
         default-lease-time         21600;
         max-lease-time             43200;
         next-server                $next_server;


Finally, restart Cobbler and sync it:

    service cobbler restart

    cobbler sync

At this point you can PXE boot any servers, but it is still a manual process. In order for it to be an automated process, a CSV file needs to be created.

PXE Boot the Servers
--------------------

### Gather MAC Addresses

Go to root home directory

    cd /root

You will need to obtain the MAC address of the network interface (e.g. p1p1) configured to PXE boot on every server. Be sure the MAC addresses are mapped to their respective hostname.

You can do this by logging into the LXC container and creating a CSV file named __ilo.csv__. __Each line should have a hostname that you wish to assign for the server, its ILO IP address, type of node you wish it to be (controller, logging, compute, cinder, swift).__ Please put hostnames that are meaningful to you like controller01, controller02, etc. Use the information from your onboarding email to create the CSV. It is recommended that you specify three hosts as your controllers and at least three swift nodes if you decide to deploy swift as well.

For example:

    729427-controller01,10.15.243.158,controller
    729426-controller02,10.15.243.157,controller
    729425-controller03,10.15.243.156,controller
    729424-logging01,10.15.243.155,logging
    729423-logging02,10.15.243.154,logging
    729422-logging03,10.15.243.153,logging
    729421-compute01,10.15.243.152,compute
    729420-compute02,10.15.243.151,compute
    729419-compute03,10.15.243.150,compute
    729418-compute04,10.15.243.149,compute
    729417-compute05,10.15.243.148,compute
    729416-compute06,10.15.243.147,compute
    729415-compute07,10.15.243.146,compute
    729414-compute08,10.15.243.145,compute
    729413-cinder01,10.15.243.144,cinder
    729412-cinder02,10.15.243.143,cinder
    729411-cinder03,10.15.243.142,cinder
    729410-swift01,10.15.243.141,swift
    729409-swift02,10.15.243.140,swift
    729408-swift03,10.15.243.139,swift

Be sure to remove any spaces in your CSV file. We also recommend removing the deployment host you manually provisioned from this CSV so you do not accidentally reboot the host you are working from.

Once this information is collected, it will be used to create another CSV file that will be the input for many different steps in the build process.

### Create Input CSV

Now, we will use a script to create a CSV named __input.csv__ in the following format.

    hostname,mac-address,host-ip,host-netmask,host-gateway,dns,pxe-interface,cobbler-profile

If this will be an openstack-ansible installation, it is recommended to order the rows in the CSV file in the following order, otherwise order the rows however you wish:

1. Controller nodes
2. Logging nodes
3. Compute nodes
4. Cinder nodes
5. Swift nodes

An example for openstack-ansible installations:

    744800-infra01.example.com,A0:36:9F:7F:70:C0,172.22.0.23,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-generic
    744819-infra02.example.com,A0:36:9F:7F:6A:C8,172.22.0.24,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-generic
    744820-infra03.example.com,A0:36:9F:82:8C:E8,172.22.0.25,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-generic
    744821-logging01.example.com,A0:36:9F:82:8C:E9,172.22.0.26,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-generic
    744822-compute01.example.com,A0:36:9F:82:8C:EA,172.22.0.27,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-generic
    744823-compute02.example.com,A0:36:9F:82:8C:EB,172.22.0.28,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-generic
    744824-cinder01.example.com,A0:36:9F:82:8C:EC,172.22.0.29,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-cinder
    744825-object01.example.com,A0:36:9F:7F:70:C1,172.22.0.30,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-swift
    744826-object02.example.com,A0:36:9F:7F:6A:C2,172.22.0.31,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-swift
    744827-object03.example.com,A0:36:9F:82:8C:E3,172.22.0.32,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,ubuntu-14.04.3-server-unattended-osic-swift

To do just that, the following script will loop through each iLO IP address in __ilo.csv__ to obtain the MAC address of the network interface configured to PXE boot and setup rest of information as well as shown above:

__NOTE:__ make sure to Set COUNT to the first usable address after deployment host and container (ex. If you use .2 and .3 for deployment and container, start with .4 controller1) and make sure to change __host-ip,host-netmask,host-gateway__ in the script (__172.22.0.$COUNT,255.255.252.0,172.22.0.1__) to match your PXE network configurations. If you later discover that you have configured the wrong ips here, you need to restart from this point.

```
COUNT=23
for i in $(cat ilo.csv)
do
    NAME=`echo $i | cut -d',' -f1`
    IP=`echo $i | cut -d',' -f2`
    TYPE=`echo $i | cut -d',' -f3`

    case "$TYPE" in
      cinder)
            SEED='ubuntu-14.04.3-server-unattended-osic-cinder'
            ;;
        swift)
            SEED='ubuntu-14.04.3-server-unattended-osic-swift'
            ;;
        *)
        SEED='ubuntu-14.04.3-server-unattended-osic-generic'
            ;;
    esac
    MAC=`sshpass -p calvincalvin ssh -o StrictHostKeyChecking=no root@$IP show /system1/network1/Integrated_NICs | grep Port1 | cut -d'=' -f2`
    #hostname,mac-address,host-ip,host-netmask,host-gateway,dns,pxe-interface,cobbler-profile
    echo "$NAME,${MAC//[$'\t\r\n ']},172.22.0.$COUNT,255.255.252.0,172.22.0.1,8.8.8.8,p1p1,$SEED" | tee -a input.csv

    (( COUNT++ ))
done
```

__NOTE:__ before you continue, make sure the generated script __input.csv__ has all the information as shown previously. In case you run into some missing information, you may need to paste the above command in a bash script and execute it.

 
### Assigning a Cobbler Profile

The last column in the CSV file specifies which Cobbler Profile to map the Cobbler System to. You have the following options:

* ubuntu-14.04.3-server-unattended-osic-generic
* ubuntu-14.04.3-server-unattended-osic-generic-ssd
* ubuntu-14.04.3-server-unattended-osic-cinder
* ubuntu-14.04.3-server-unattended-osic-cinder-ssd
* ubuntu-14.04.3-server-unattended-osic-swift
* ubuntu-14.04.3-server-unattended-osic-swift-ssd

Typically, you will use the __ubuntu-14.04.3-server-unattended-osic-generic__ Cobbler Profile. It will create one RAID10 raid group. The operating system will see this as __/dev/sda__.

The __ubuntu-14.04.3-server-unattended-osic-cinder__ Cobbler Profile will create one RAID1 raid group and a second RAID10 raid group. These will be seen by the operating system as __/dev/sda__ and __/dev/sdb__, respectively.

The __ubuntu-14.04.3-server-unattended-osic-swift__ Cobbler Profile will create one RAID1 raid group and 10 RAID0 raid groups each containing one disk. The HP Storage Controller will not present a disk to the operating system unless it is in a RAID group. Because Swift needs to deal with individual, non-RAIDed disks, the only way to do this is to put each disk in its own RAID0 raid group.

You will only use the __ssd__ Cobbler Profiles if the servers contain SSD drives.

### Generate Cobbler Systems

With this CSV file in place, run the __generate_cobbler_systems.py__ script to generate a __cobbler system__ command for each server and pipe the output to `bash` to actually add the __cobbler system__ to Cobbler:

    cd /root/rpc-prep-scripts

    python generate_cobbler_system.py /root/input.csv | bash

Verify the __cobbler system__ entries were added by running `cobbler system list`.

Once all of the __cobbler systems__ are setup, run `cobbler sync`.

### Begin PXE Booting

To begin PXE booting, Set the servers to boot from PXE on the next reboot and reboot all of the servers with the following command  (if the deployment host is the first controller, you will want to __remove__ it from the __ilo.csv__ file so you don't reboot the host running the LXC container):

__NOTE__: change root and calvincalvin below to your ILO username and password.

    for i in $(cat /root/ilo.csv)
    do
    NAME=$(echo $i | cut -d',' -f1)
    IP=$(echo $i | cut -d',' -f2)
    echo $NAME
    ipmitool -I lanplus -H $IP -U root -P calvincalvin chassis bootdev pxe
    sleep 1
    ipmitool -I lanplus -H $IP -U root -P calvincalvin power reset
    done

__NOTE:__ if the servers are already shut down, you might want to change __power reset__ with __power on__ in the above command.

As the servers finish PXE booting, a call will be made to the cobbler API to ensure the server does not PXE boot again.

To quickly see which servers are still set to PXE boot, run the following command:

    for i in $(cobbler system list)
    do
    NETBOOT=$(cobbler system report --name $i | awk '/^Netboot/ {print $NF}')
    if [[ ${NETBOOT} == True ]]; then
    echo -e "$i: netboot_enabled : ${NETBOOT}"
    fi
    done

Any server which returns __True__ has not yet PXE booted. Rerun last command until there is no output to make sure all your servers has finished pxebooting. Time to wait depends on the number of servers you are deploying. If somehow, one or two servers did not go through for a long time, you may want to investigate them with their ILO console. In most cases, this is due to rebooting those servers either fails or hangs, so you may need to reboot them manually with ILO.

__NOTE__: In case you want to re-pxeboot servers, make sure to clean old settings from cobbler with the following command:

    for i in `cobbler system list`; do cobbler system remove --name $i; done;


Bootstrapping the Servers
-------------------------

When all servers finish PXE booting, you will now need to bootstrap the servers.

### Generate Ansible Inventory

Start by running the `generate_ansible_hosts.py` Python script:

    cd /root/rpc-prep-scripts

    python generate_ansible_hosts.py /root/input.csv > /root/osic-prep-ansible/hosts

If this will be an openstack-ansible installation, organize the Ansible __hosts__ file into groups for __controller__, __logging__, __compute__, __cinder__, and __swift__, otherwise leave the Ansible __hosts__ file as it is and jump to the next section.

An example for openstack-ansible installation:

    [controller]
    744800-infra01.example.com ansible_ssh_host=10.240.0.51
    744819-infra02.example.com ansible_ssh_host=10.240.0.52
    744820-infra03.example.com ansible_ssh_host=10.240.0.53

    [logging]
    744821-logging01.example.com ansible_ssh_host=10.240.0.54

    [compute]
    744822-compute01.example.com ansible_ssh_host=10.240.0.55
    744823-compute02.example.com ansible_ssh_host=10.240.0.56

    [cinder]
    744824-cinder01.example.com ansible_ssh_host=10.240.0.57

    [swift]
    744825-object01.example.com ansible_ssh_host=10.240.0.58
    744826-object02.example.com ansible_ssh_host=10.240.0.59
    744827-object03.example.com ansible_ssh_host=10.240.0.60

### Verify Connectivity

The LXC container will not have all of the new server's SSH fingerprints in its __known_hosts__ file. This is needed to bypass prompts and create a silent login when SSHing to servers. Programatically add them by running the following command:

    for i in $(cat /root/osic-prep-ansible/hosts | awk /ansible_ssh_host/ | cut -d'=' -f2)
    do
    ssh-keygen -R $i
    ssh-keyscan -H $i >> /root/.ssh/known_hosts
    done

Verify Ansible can talk to every server (the password is __cobbler__):

    cd /root/osic-prep-ansible

    ansible -i hosts all -m shell -a "uptime" --ask-pass

### Setup SSH Public Keys

Generate an SSH key pair for the LXC container:

    ssh-keygen

Copy the LXC container's SSH public key to the __osic-prep-ansible__ directory:

    cp /root/.ssh/id_rsa.pub /root/osic-prep-ansible/playbooks/files/public_keys/osic-prep

### Bootstrap the Servers

Finally, run the bootstrap.yml Ansible Playbook (the password is again __cobbler__):

    cd /root/osic-prep-ansible

    ansible-playbook -i hosts playbooks/bootstrap.yml --ask-pass

### Clean Up LVM Logical Volumes

If this will be an openstack-ansible installation, you will need to clean up particular LVM Logical Volumes.

Each server is provisioned with a standard set of LVM Logical Volumes. Not all servers need all of the LVM Logical Volumes. Clean them up with the following steps.

Remove LVM Logical Volume __nova00__ from the Controller, Logging, Cinder, and Swift nodes:

    ansible-playbook -i hosts playbooks/remove-lvs-nova00.yml

Remove LVM Logical Volume __deleteme00__ from all nodes:

    ansible-playbook -i hosts playbooks/remove-lvs-deleteme00.yml

### Update Linux Kernel

Every server in the OSIC RAX Cluster is running two Intel X710 10 GbE NICs. These NICs have not been well tested in Ubuntu and as such the upstream i40e driver in the default 14.04.3 Linux kernel will begin showing issues when you setup VLAN tagged interfaces and bridges.

In order to get around this, you must install an updated Linux kernel.

You can do this by running the following commands:

    cd /root/osic-prep-ansible

    ansible -i hosts all -m shell -a "apt-get update; apt-get install -y linux-generic-lts-xenial" --forks 25

### Reboot Nodes

Finally, reboot all servers:

    ansible -i hosts all -m shell -a "reboot" --forks 25

Once all servers reboot, you can begin installing openstack-ansible.
