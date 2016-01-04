## Organization of the cluster

Biocluster runs the [Rocks cluster management system](http://www.rocksclusters.org), which is based on [CentOS](https://www.centos.org/). A typical Rocks cluster is composed of a head node (by default, it is called frontend), a storage node (by default, the storage node is the head node itself), and multiple compute nodes. For genomics/bioinformatics applications, it is important to set up a dedicated storage node with a lot of drives, rather than relying on the head node for storage. See below for a simple diagram.

![cluster](https://cloud.githubusercontent.com/assets/5926328/12101852/c97c8e00-b2eb-11e5-8e21-949d4960fb46.png)

These eth0 port in these six machines are connected by an ethernet network switch, which is called network-0-0. The head node is also connected to Internet via eth1.

Users log into head node from eth1, and submit jobs from the head node by [Sun Grid Engine](https://en.wikipedia.org/wiki/Oracle_Grid_Engine), which is a job queueing system. These jobs are executed in compute nodes. All user data and files are stored in storage node, and can be accessed by compute nodes or head node by NFS.

## Adding high-speed internal network

Since all nodes in biocluster used NFS to access a shared file system in a storage node, I need to add in an extra level of high-speed network for the internal network. Around 2010, I decided to use [infiniband (IB)](https://en.wikipedia.org/wiki/InfiniBand), since 10G ethernet still has latency issues and is as expensive as IB network (more details below). I use ip-over-ib so that the IB communication is hidden from user's perspective (in other word, you access the IB port in another host simply by using its IP address).

This basically means that each compute node has an eth0 port and an ib0 port, and they need to be in different subnet. By default, the eth0 is in the 10.1.1.\* subnet called "private", and the ib0 is in the 192.168.1.0 subnet called "ipoib". Furthermore, to allow remote control of compute nodes by IPMI, I share the IPMI port with eth0 in all compute nodes and the storage node, so the IPMI traffic goes through the "private" network as well with 10.1.1.\* IP addresses.

Installtion of IB card is straightforward: it is merely an extra card that you insert into a PCIExpress slot. For servers, most likely you will need a PCI riser card, so that you insert IB card into the riser card and insert the riser card into the motherboard. 

See below for an updated network diagram. The actual implementation is a bit more complicated than drafted below, but the set up below is already sufficient to build a high-performance cluster for a bioinformatics lab.

![cluster](https://cloud.githubusercontent.com/assets/5926328/12102015/f7d6a3ac-b2ec-11e5-8dbc-2be442c69eb6.png)

## Choosing hardware components

Every major computer manufacture (Dell, HP, IBM, etc) offers rackmount server boxes, and there are also many smaller companies that specialize in server boxes with their own brands and SuperMicro components. Typically, you may want to pick a 1U head node, one or multiple 2U compute node box (that houses 4-socket nodes), and one 4U storage node. 

For a typical low-cost cluster, you can pick a compute node that contains two 12/8/6-core Intel/AMD server CPUs, 2GB or 4GB server memory per CPU core, and 1TB hard drive per node. Examples are IBM DX360 M3 and Dell PowerEdge R820. For head node, it can be a simple 1U lower-end machine, since its main purpose is not to execute jobs but to handle user log in and job submission. For storage node, its main purpose is to store files with many hard enterprise-level hard drives, and typical set up are 12 drives (for 2U servers) or 36 drivers (for 4U servers). The network switch should be gigabit ethernet switch, and typical set up can be 12-port, 24-port or 48-port, depending no how many compute node you may want to have in the cluster.

As mentioned above, for my own cluster, I also set up a second network using 40Gbps infinibnd connection. The reason is that for genomics applications, gigabit ethernet connection has a maximum speed of ~120MB/second for file transmission, and it may be too slow for data analysis (especially considering that multiple users or multiple jobs may be using the same cluster at the same time). With infiniband and a typical storage box, the file reading/writing rate can easily reach >800MB/second, which is limited by hard drive array itself, not the infiniband. Infiniband also allows fast MPI computation. Typical infiniband switch and adapter providers include Menllanox and Intel (which purchased Qlogic infiniband), and I have used both. However, 40Gbps/100Gbpx ethernet cards are gaining more and more popularity nowadays and you may consider using them as well.

I built biocluster in 2010, using mostly IBM DX360 M3 as compute nodes, a SuperMicro storage server with with 36 drives as storage node, which was connected to a 45-drive JBOD (Just a Box of Drives, which expands the storage capacity). In addition, about half of the machines are plugged into a 4U Dell 354P 4200 Watt UPS, so that these machines can continue work with unintended power interruption. For complicated reasons, these IBM boxes are not high-density ones that I would have hoped for, so I can only fit two compute nodes per 2U space. Finally, these machines were housed in a Dell 4820 48U Rack, even though they were designed for a completely different IBM iDataPlex half-depth rack, so it has been difficult to set up the cabling well enough. Despite the ugly appearance, these machines have worked well so far after so many years.

A second cluster that I helped build quite a while ago include Dell c1100 as head node, many Dell C6100 as compute nodes, Dell C2100 as storage node, plus an external JBOD. The components are connected by Netgear gigabit ethernet switch and Mellanox 56Gbps infiniband switch.

