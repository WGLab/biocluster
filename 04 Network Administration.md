# Network Administration

## Useful command line

Find all network devices in the network that connects to this machine:

```
[root@biocluster ~]# arp -a| sort
```

Typical configuration files

```
[kaiwang@biocluster ~]$ cat /etc/resolv.conf 
search local med.usc.edu
nameserver 127.0.0.1
nameserver 128.125.253.143
nameserver 128.125.7.23
```

The user-defined hosts are stored in /etc/hosts.local file, and ‘rocks sync config’ and ‘rocks sync network’ will copy this to all cluster.

```
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
```

Change MTU for IB card for the cluster

```
rocks set network mtu ipoib 65520
```

this should set the MTU for all hosts in the system to 65520



Examine cable connection

```
ethtool -p eth0
```

to blink the eth0 port, to help identify which port is eth0. This is important, since when inserting a new ethernet card into the system, the new card may be treated as eth0/1 instead, so the built-in ethernet ports are eth2/3.

turn on httpd on start

```
[root@bioinform2 ~]# chkconfig httpd --list
httpd           0:off   1:off   2:off   3:off   4:off   5:off   6:off
[root@bioinform2 ~]# chkconfig httpd on
[root@bioinform2 ~]# chkconfig httpd --list
httpd           0:off   1:off   2:on    3:on    4:on    5:on    6:off
```

Or, alternatively,

```
[root@bioinform2 ~]# chkconfig --level 35 httpd on
```

iptables

turning on firewall

add `-A INPUT -i eth1 -j ACCEPT` in /etc/sysconfig/iptables

Measure network bandwidth and latency

First run `qperf` without argument in `nas-0-0`, then run

```
[kaiwang@biocluster ~]$ qperf nas-0-0-ib tcp_bw tcp_lat udp_bw udp_lat ud_bw ud_lat rc_bi_bw sdp_bw
[kaiwang@biocluster ~]$ qperf nas-0-0 tcp_bw tcp_lat udp_bw udp_lat ud_bw ud_lat rc_bi_bw sdp_bw
```

There is a clear different in tcp_bw (from 1.49GB/s to 119MB/s), but the latency is not great for IB.

Measure infiniband status

First run `ibv_rc_pingpong` in biocluster, then run 

```
[kaiwang@compute-0-10 ~]$ ibv_rc_pingpong biocluster   
  local address:  LID 0x000e, QPN 0x48004b, PSN 0x74f5ba, GID ::
  remote address: LID 0x0003, QPN 0x74004b, PSN 0xbaa1c2, GID ::
8192000 bytes in 0.01 seconds = 5671.66 Mbit/sec
1000 iters in 0.01 seconds = 11.56 usec/iter
```

Similarly, ibv_uc_pingpong,ibv_ud_pingpong, ibv_srq_pingpongcan be used. The ibv_srq_pingpong is very fast as it presumably use 16 different queue pairs by default.

##ibping

first run ibstat in biocluster to find the port GUID
then run "ibping -S" in biocluster as root.
then run "ibping -G 0x0002c903000ecfffa" in compute-0-0

using ibstate, I found that compute-0-6 and compute-0-4 are 40, but others have 20.

## DenyHosts to prevent unauthorized brute force attack

```
yum install denyhosts
service denyhosts restart
chkconfig denyhosts on
```

Edit the /etc/denyhosts.conf file for configuation changes



## iptables

Some useful guideline include [this](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Security_Guide/s1-firewall-ipt-fwd.html).


## Port forwarding

This is useful when you set up a web service in one compute node (such as compute-0-0), but want to access this service from frontend from the outside world using a specific port such as 8010.

The following two commands can do this on frontend:

```
iptables -t nat -A PREROUTING -p tcp -i eth1 --dport 8010 -j DNAT --to-destination 10.1.1.253:8010
iptables -A FORWARD -p tcp -d 10.1.1.253 --dport 8010 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

## Allow node to access outside world

To allow LAN nodes with private IP addresses to communicate with external public networks, configure the firewall for IP masquerading, which masks requests from LAN nodes with the IP address of the firewall's external device (in this case, eth1):

```
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE 
```

The rule uses the NAT packet matching table (`-t nat`) and specifies the built-in POSTROUTING chain for NAT (`-A POSTROUTING`) on the firewall's external networking device (`-o eth0`).

## list of all hosts in local network

Use the `nmap -sP 192.168.1.*` command, which basically ping all hosts.

## Adding a ethernet adapter on a compute node to access Internet

It could be as simple as this:

```
[root@biocluster /]$ rocks add host interface compute-0-0 iface=eth1 subnet=public2 name=compute-0-0-public ip=128.125.248.230
```

However, in my case, since the new IP is not in the same subnet as the IP of the frontend, I have to create a new subnet to make this work:

```
[root@biocluster /home/kaiwang]$ rocks add network public2 128.125.248.0 255.255.254.0 
[root@biocluster /home/kaiwang]$ rocks list network
NETWORK  SUBNET         NETMASK         MTU    DNSZONE     SERVEDNS
ipoib:   192.168.1.0    255.255.255.0   65520  ipoib       True    
private: 10.1.1.0       255.255.255.0   1500   local       True    
public:  68.181.163.128 255.255.255.128 1500   med.usc.edu False   
public2: 128.125.248.0  255.255.254.0   1500   public2     False   
[root@biocluster /home/kaiwang]$ rocks remove host interface compute-0-0 eth1
[root@biocluster /home/kaiwang]$ rocks add host interface compute-0-0 iface=eth1 subnet=public2 name=compute-0-0-public ip=128.125.248.230
```

A further complication is that typically default gateway for any network is the head node. However, head node may have iptables that disable any other node to access the internet. In these cases, one need to change the default gateway for eth1 to something else.

Use `rocks list host route compute-0-0` to check what's the current gateway. It is clear that head node (10.1.1.1) is the default.

```
bioinform2: 68.181.163.131  255.255.255.255 10.1.1.1 G     
```

Now do this:
```
[root@biocluster /]$ rocks add host route compute-0-0 0.0.0.0 128.125.249.254 netmask=0.0.0.0
[root@biocluster /]$ rocks sync host network compute-0-0
```

Now compute-0-0 can access internet directly through eth1.













