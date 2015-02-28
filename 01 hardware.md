# Hardware

## Organization of the cluster
A typical cluster is composed of a head node (by default, it is called frontend), a storage node (by default, the storage node is the head node), and multiple compute nodes. See below for a diagram.

![cluster](/img/cluster.png)

These eth0 port in these six machines are connected by a ethernet network switch, which is called network-0-0. The head node is also connected to Internet via eth1.

Users log into head node, and submit jobs from the head node. These jobs are executed in compute nodes. All user data are stored in storage node.

## Choosing the component
Every major computer manufacture (Dell, HP, IBM, etc) offers rackmount server boxes, and there are also many smaller companies that specialize in server boxes with their own brands. Typically, you may want to pick a 1U head node, one or multiple 2U compute node box (that houses 4-socket nodes), and one 2U or 4U storage node. 

For a typical low-cost cluster, you can pick a compute node that contains two 8-core or 6-core Intel/AMD server CPUs, 2GB or 4GB server memory per CPU core, and 1TB consumer hard drive per node. Examples are IBM DX360 and Dell PowerEdge R820. For head node, it can be a simple 1U lower-end machine, since its main purpose is not to execute jobs. For storage node, its main purpose is to store files with many hard enterprise-level hard drives, and typical set up are 12 drives (for 2U servers) or 36 drivers (for 4U servers). The network switch should be gigabit ethernet switch, and typical set up can be 12-port, 24-port or 48-port, depending no how many compute node you may want to have in the cluster.

For my own cluster, I also set up a second network using 40Gbps infinibnd connection. The reason is that for genomics applications, gigabit ethernet connection has a maximum speed of ~120MB/second for file transmission, and it may be too slow for data analysis (especially considering that multiple users or multiple jobs may be using the same cluster at the same time). With infiniband and a typical storage box, the file reading/writing rate can easily reach 800MB/second, which is limited by hard drive array itself, not the infiniband. Infiniband also allows fast MPI computation. Typical infiniband switch and adapter providers include Menllanox and Intel (which purchased Qlogic infiniband), and I have used both. However, 10Gbps/40Gbps/100Gbpx ethernet cards are entering the market and gaining more and more popularity and may become the mainstream in the future. 

Below is a picture of the cluster that I built in 2010, using mostly IBM DX360 M3 as compute nodes, Silicon Mechanics R346 as head node, Silicon Mechanics StorForm R518 with 36 drives as storage node, which was connected to a 45-drive JBOD (Just a Box of Drives, which expands the storage capacity). In addition, about half of the machines are plugged into a 4U Dell 354P 4200 Watt UPS, so that these machines can continue work with unintended power interruption. It may look ugly because I did not set up cabling well enough. In addition, these IBM boxes are not high-density ones, so I can only fit two compute nodes per 2U space. Finally, these machines were housed in a Dell 4820 48U Rack, even though they were designed for a completely different IBM iDataPlex half-depth rack, so it has been difficult to set up the cabling well enough. Despite the ugly appearance, these machines have worked well so far after so many years.

![cluster](/img/zni_cluster.jpg)

A second cluster that I helped build quite a while ago include Dell c1100 as head node, C6100 as compute nodes, Dell C2100 as storage node, plus an external JBOD. The components are connected by Netgear gigabit ethernet switch and Mellanox 56Gbps infiniband switch.












