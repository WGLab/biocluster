## Overview

The cluster operating system that I use is called [Rocks](http://www.rocksclusters.org/wordpress/). It is one of several clusterware that is widely used, and is well maintained and regularly updated. You first need to download the ISO images, write it to a DVD, use an external DVD-ROM (or internal one if your head node has one) to start the computer.

## Head node installation

After starting up head node by the DVD drive, just follow the instructions shown in the screen to install Rocks. Nothing tricky here. The only thing that need customization are:

- Fully-qualified host name: you have to specify a host name for the server. For example, you can register a domain name such as mydomain.org, and then point a subdomain called mycluster.mydomain.org to the IP address of your cluster. Then the host name for the cluster is mycluster.mydomain.org. Depending on the friendliness of your employer, you may be able to register a subdomain such as mycluster.myuniversity.edu with your IT department, and associate an IP address with it.

- IP address: Contact your IT department to obtain a static IP address. This is extremely important! Usually they will ask for your MAC address, and make sure to provide MAC address for eth1 (not eth0!!!) to them, since eth1 is the port that is connected to Internet. The gateway and netmask and DNS server information are provided to you together with IP address.

- packages to install: select everything except kvm, bio and condor. KVM (kernel virtual machine) is probably not something that you really need. bio (bioinformatics tools) has very limited and typically outdated tools which you can install yourself easily. condor is another cluster management system that you do not need if you have SGE installed in the cluster. 

## Storage node

The next step is to install storage node. You can certainly install compute node next as well, it does not really matter much. You can follow the instructions for compute nodes below (except to select "NAS" as the "Appliance").

## Compute nodes

1. Place the node into the rack
2. Connect the power cord for the node to the power supply.
3. Connect the ‘eth0’ Gigabit Ethernet controller to the Gigabit Ethernet Switch in the rack.
4. Log into the frontend node and log in as ‘root’.
5. Run the program ‘insert-ethers’ command: `# insert-ethers`.
6. A blue screen will appear, select ‘Compute’ as the appliance and hit ‘OK’.
7. Then, the program will wait for DHCP requests from new compute node.
8. At this point you can now power on the new compute node. The new nodes boot order should be set to PXE boot first.
9. When the frontend node receives the DHCP request, it will print out something like 'compute-0-x' in the blue screen. This indicates that ‘insert-ethers’ has received the DHCP request from the compute node, inserted it into the database and updated all configuration files (e.g. `/etc/hosts`, `/etc/dhcpd.conf`, DNS and batch system (SGE) files).
10. The above screen will display for a few seconds and then the “()” next to ‘compute-0-x’ indicates that the node has not yet requested a kickstart file. The above screen will be seen for each node that is successfully identified by `insert-ethers`.
11. Once the node successfully requests the kickstart file from the head node, the “()” will change to “(\*)”. If there are no more nodes to boot, then a person can quit ‘insert-ethers’ by pressing either ‘F10’ or ‘F11’. Kickstart files are retrieved via https. If there was an error during transmission, the error code will be visible instead of “\*”.
12. Once the install completes, the node(s) should be ready to use.

> Technical note: It is extremely important that the compute node is set to PXE boot first, since the head node is expecting a DHCP request from the compute node and is sending the installation package to the compute node. It is also very important that you set up a static IP address (rather than DHCP) for IPMI port in compute node (more details [here](09 Remote management.md), if you share IPMI port with eth0; otherwise, IPMI will hijack your installation by issuing a DHCP request before eth0 does.

## Re-install nodes

When a node is shutdown suddenly, it will usually be re-installed automatically when it was started again. When a node is in error state (for example, E is shown in SGE qstat command) for unknown reason, you may want to have a fresh re-install the node manually. When you want to install new packages or updates to the cluster, you also need to re-install all compute nodes.

Use 

```
rocks list host boot
```

to check the boot action of all compute nodes. Generally, it should be "os" for all nodes.

Then use

```
rocks set host boot compute-0-0 action=install
```

to change the action to re-installation of Rocks system for the specific node.

Then if the machine is accessible, log into the machine and type `reboot`. If not accessible, just restart the machine via IPMI (see [this section](09 Remote management.md)) or manually by hand.

Alternatively, if the machine is accessible, you can just login and use `/boot/kickstart/cluster-kickstart` to reboot and re-install the node.

> Troubleshooting: When re-installation does not work

> Sometimes a node fails, and you specify to re-install the node and forced to restart the computer. However, the computer displays various error messages, such as "DHCP -/|\" hanging, or "hard drive parameter not found", or "no hard drive found". Sometimes, during Rocks install, the message "loading vmlinz...................." shows up and then freezes, or the "waiting for hardware to initialize" message that freezes. I found that all these problems were due to hard drive issues. After switching hard drive, all the issues were resolved.






