## Installation

### Overview
The cluster operating system that I use is called Rocks. It is one of several clusterware that is widely used, and is well maintained and regularly updated. You first need to download the ISO images, write it to a DVD, 

### head node
use an external DVD-ROM (or internal one if your head node has one) to start the computer.

Just follow the instructions in the screen to install Rocks. Nothing tricky here. The only thing that need customization are:

- Fully-qualified host name: you have to specify a host name for the server. For example, you can register a domain name such as mydomain.org, and then point a subdomain called mycluster.mydomain.org to the IP address of your cluster. Then the host name for the cluster is mycluster.mydomain.org. Depending on the friendliness of your employer, you may be able to register a subdomain such as mycluster.myuniversity.edu with your IT department, and associate an IP address with it.

- IP address: Contact your IT department to obtain a static IP address. This is extremely important! Usually they will ask for your MAC address, and make sure to provide MAC address for eth1 (not eth0!) to them, since eth1 is the port that is connected to Internet. The gateway and netmask and DNS server information are provided to you together with IP address.

- packages to install: select everything except kvm, bio and condor. KVM (kernel virtual machine) is probably not something that you really need. bio (bioinformatics tools) has very limited and typically outdated tools which you can install yourself easily. condor is another cluster management system that you do not need if you have SGE installed in the cluster. 

### Storage node
The next step is to install storage node. You can certainly install compute node next as well, it does not really matter. 

### Compute node
1. Install the node into the rack
2. Connect the power cord for the node to the power supply.
3. Connect the ‘eth0’ Gigabit Ethernet controller to the Gigabit Ethernet Switch in the rack.
4. Log into the frontend node (BIOCLUSTER) and log in as ‘root’.
5. Run the program ‘insert-ethers’ command:
# insert-ethers
6. The screen below will appear, select ‘Compute’ and hit ‘OK’
7. Then, the following screen will appear:
8. This indicates that ‘insert-ethers’ is waiting for new compute nodes. At this point you can
now power on the new node or nodes. The new nodes boot order should be set to PXE
boot first.
9. When the frontend node (BIOCLUSTER) receives the DHCP request, something similar
to the following will be seen in the ‘insert-ethers’ window:
This indicates that ‘insert-ethers’ has received the DHCP request from the compute node,
inserted it into the database and updated all configuration files (e.g. /etc/hosts,
/etc/dhcpd.conf, DNS and batch system (SGE) files).
10. The above screen will display for a few seconds and then the following will be seen:
The “()” next to ‘compute-0-x’ indicates that the node has not yet requested a kickstart
file. The above screen will be seen for each node that is successfully identified by ‘insertethers’.
11. Once the node successfully requests the kickstart file from the head node
(BIOCLUSTER), the “()” will change to “(*)”:
If there are no more nodes to boot, then a person can quit ‘insert-ethers’ by pressing either ‘F10’ or ‘F11’. Kickstart files are retrieved via https. If there was an error during
transmission, the error code will be visible instead of “*”.

12. Once the install completes, the node(s) should be ready to use. For more detailed
instructions on adding nodes, review the Rocks documentation. http://localhost/














