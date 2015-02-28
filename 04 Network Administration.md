# Network Administration

## Useful command line

Find all network devices in the network that connects to this machine:
[root@biocluster ~]# arp -a| sort


Typical configuration files
[kaiwang@biocluster ~]$ cat /etc/resolv.conf 
search local med.usc.edu
nameserver 127.0.0.1
nameserver 128.125.253.143
nameserver 128.125.7.23

The user-defined hosts are stored in /etc/hosts.local file, and ‘rocks sync config’ and ‘rocks sync network’ will copy this to all cluster.

[root@biocluster ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0 
DEVICE=eth0
HWADDR=00:25:90:09:E6:66
IPADDR=10.1.1.1
NETMASK=255.255.255.0
BOOTPROTO=none
ONBOOT=yes
MTU=1500
[root@biocluster ~]# cat /etc/sysconfig/network-scripts/ifcfg-ib0  
DEVICE=ib0
HWADDR=80:00:00:48:FE:80:00:00:00:00:00:00:00:02:C9:03:00:0E:CF:FF
IPADDR=192.168.1.1
NETMASK=255.255.255.0
BOOTPROTO=none
ONBOOT=yes
MTU=65520


Change MTU for IB card for the cluster

rocks set network mtu ipoib 65520

this should set the MTU for all hosts in the system to 65520



Examine cable connection
ethtool -p eth0
to blink the eth0 port, to help identify which port is eth0. This is important, since when inserting a new ethernet card into the system, the new card may be treated as eth0/1 instead, so the built-in ethernet ports are eth2/3.

turn on httpd on start
[root@bioinform2 ~]# chkconfig httpd --list
httpd           0:off   1:off   2:off   3:off   4:off   5:off   6:off
[root@bioinform2 ~]# chkconfig httpd on
[root@bioinform2 ~]# chkconfig httpd --list
httpd           0:off   1:off   2:on    3:on    4:on    5:on    6:off
Or, alternatively,
[root@bioinform2 ~]# chkconfig --level 35 httpd on
iptables
setting up Apache server
/etc/httpd/conf/httpd.conf 
turning on firewall
add “-A INPUT -i eth1 -j ACCEPT” in /etc/sysconfig/iptables

Measure network bandwidth and latency
First run qperf without argument in nas-0-0, then run

[kaiwang@biocluster ~]$ qperf nas-0-0-ib tcp_bw tcp_lat udp_bw udp_lat ud_bw ud_lat rc_bi_bw sdp_bw
[kaiwang@biocluster ~]$ qperf nas-0-0 tcp_bw tcp_lat udp_bw udp_lat ud_bw ud_lat rc_bi_bw sdp_bw

There is a clear different in tcp_bw (from 1.49GB/s to 119MB/s), but the latency is not great for IB.

Measure infiniband status
First run ibv_rc_pingpong in biocluster, then run 

[kaiwang@compute-0-10 ~]$ ibv_rc_pingpong biocluster   
  local address:  LID 0x000e, QPN 0x48004b, PSN 0x74f5ba, GID ::
  remote address: LID 0x0003, QPN 0x74004b, PSN 0xbaa1c2, GID ::
8192000 bytes in 0.01 seconds = 5671.66 Mbit/sec
1000 iters in 0.01 seconds = 11.56 usec/iter

Similarly, ibv_uc_pingpong,ibv_ud_pingpong, ibv_srq_pingpongcan be used. The ibv_srq_pingpong is very fast as it presumably use 16 different queue pairs by default.

#ibping
first run ibstat in biocluster to find the port GUID
then run "ibping -S" in biocluster as root.
then run "ibping -G 0x0002c903000ecfffa" in compute-0-0

using ibstate, I found that compute-0-6 and compute-0-4 are 40, but others have 20.

DenyHosts to prevent unauthorized brute force attack

yum install denyhosts
service denyhosts restart
chkconfig denyhosts on

Edit the /etc/denyhosts.conf file for configuation changes

