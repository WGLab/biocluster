# Overview

Once installation of the cluster is done, you typically want to make some customization to the cluster. This section describes several common tasks.

## Setting up system path for Perl and Python and Java

If you are installing an updated major version (such as 6.2) rather than initial major version (such as 6.0) of Rocks, the default Perl and Python will be at `/usr/bin/perl` and `/usr/bin/python` respectively. However, you may have selected to install Perl/Python roll during the installation, and these are actually installed in `/opt/perl/bin` and `/opt/python/bin` respectively. These directories are NOT in default PATH, so your users cannot access it by default (in other word, if you type `perl -v` or `python -v`, you will access the default lower version of perl and python).

To address this, you need to change system path globally. The correct way to do this is to add a script file into `/etc/profile.d`. There are many other ways to do this (such as adding PATH into `/etc/profile` or `/etc/environment` and so on), but they are not perfect and may not survive a system update that overwrite `/etc/profile` or `/etc/environment`.

For example, after installing Rocks, you can check what's the system path:

```
[root@biocluster admin]# echo $PATH
/opt/openmpi/bin:/usr/lib64/qt-3.3/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/opt/ganglia/bin:/opt/ganglia/sbin:/usr/java/latest/bin:/opt/maven/bin:/opt/pdsh/bin:/opt/rocks/bin:/opt/rocks/sbin:/opt/gridengine/bin/linux-x64:/root/bin
```

Now we need to add perl and python into the PATH and make sure that they are before the `/usr/bin` path:

```
# echo 'pathmunge /opt/perl/bin' > /etc/profile.d/perl.sh
# chmod +x /etc/profile.d/perl.sh
# echo 'pathmunge /opt/python/bin' > /etc/profile.d/python.sh
# chmod +x /etc/profile.d/python.sh
```

The function pathmunge is defined in the `/etc/profile` file. It takes two arguments (including an optional `after` argument). By default, the new path is added at the front of the PATH.

Now we can log out and log in again, and check the new PATH:

```
[root@biocluster ~]# echo $PATH
/opt/openmpi/bin:/usr/lib64/qt-3.3/bin:/opt/python/bin:/opt/perl/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/opt/ganglia/bin:/opt/ganglia/sbin:/usr/java/latest/bin:/opt/maven/bin:/opt/pdsh/bin:/opt/rocks/bin:/opt/rocks/sbin:/opt/gridengine/bin/linux-x64:/root/bin
```

For Java, the java.sh file already exists, but you need to add `pathmunge /usr/java/latest/bin` to the first line to make sure that Rocks uses the correct version of Java. 

```
[root@biocluster admin]# ll /usr/java/
total 4
lrwxrwxrwx 1 root root   16 Feb 10 05:49 default -> /usr/java/latest
drwxr-xr-x 8 root root 4096 May  7  2015 jdk1.7.0_51
lrwxrwxrwx 1 root root   21 Feb 10 05:49 latest -> /usr/java/jdk1.7.0_51
```

By default, `/usr/bin/java` is before `/usr/java/latest/bin` in PATH so it will be used, which is version 1.5.

## Adjust system time

In case you did not change BIOS to set up the correct time, you can use a few commands to make sure that the time is set up correctly.

First check the hardware clock with the following command.

```
[root@biocluster ~]# date
Wed Feb 10 11:45:21 PST 2016
[root@biocluster ~]# hwclock --show
Wed 10 Feb 2016 11:45:40 AM PST  -0.578490 seconds
```

I am running the commands in the afternoon, so this time is wrong and we need to change it.

By default the ntp package is installed and we can do:

```
[root@biocluster ~]# ntpdate pool.ntp.org
10 Feb 16:40:51 ntpdate[9205]: step time server 65.182.224.39 offset 17684.036678 sec

[root@biocluster ~]# date
Wed Feb 10 16:41:01 PST 2016
```

or you can manually update: 

```
date --set "10 Feb 2015 16:45 PST"
```

Obviously in the above command you must set your date, time and time zone correctly.

Now as root, synchronize the hardware clock to the current system time as local time.

```
hwclock --systohc --localtime
```

Now the hardware clock is re-adjusted to the system time and both now point to the local time (which was synchronized to NTP server).

Sometimes you may want to change time zone, such as from EDT to PDT. In this case, edit /etc/sysconfig/clock , and change `ZONE="America/New_York"` to `ZONE="America/Los_Angeles"`. Then do `cd /etc; rm localtime; ln -s /usr/share/zoneinfo/US/Pacific localtime`. Then `date` will show the correct time. For compute nodes, the following command listed their timezone:

```
[root@biocluster etc]# rocks list attr | fgrep zone
Kickstart_Timezone:                America/Los_Angeles        
```

## Chagne timezone for the entire cluster

Technically you need to rebuild the whole cluster. However, as a lazy person, this is a workaround. First do this:

```
[root@biocluster /]$ rocks run host compute 'cd /etc; rm localtime; ln -s /usr/share/zoneinfo/America/New_York localtime'
```

Also make sure to do this for NAS devices, since they are not counted as compute. Also do this on head node of course.

Next do this: 

```
rocks set attr attr=Kickstart_Timezone value=America/New_York
```

This ensures that current and future nodes will be updated to New York time.



## Change limit on the number of open files

By default, Linux has a limit of 1024 files that can be opened simultaneously. For bioinformatics applications, this is way too small. A simple variant calling (such as by GATK) or a simple allele frequency calculation (such as by PennCNV) can easily exceed this limit and result in mysterious failure of the software tools.

You can follow this procedure to change this limit:

First, run a `ulimit -a`, which confirms that limit on open files is “1024” which is too few.  Next, run the following command (if the `extend-compute.xml` file does NOT exist):

```
# cd /export/rocks/install/site-profiles/6.1/nodes/
# cp skeleton.xml extend-compute.xml
```

Now modify the “\<post\>” section in extend-compute.xml with:

```
echo '* soft nofile 20000' >> /etc/security/limits.conf
echo '* hard nofile 20000' >> /etc/security/limits.conf
```

The difference is: the soft limit may be changed later, up to the hard limit value, by the process running with these limits and hard limit can only be lowered. Here I set them as identical.

To apply your customized configuration scripts to compute nodes, rebuild the distribution:
```
# cd /export/rocks/install
# rocks create distro
```

Now re-install all nodes.

## Setting up parallel environment

By default, Rocks does not provide a smp parallel environment for software tools that can leverage multiple cores in the same machine. I do not understand why, but this is how to address this problem:

First, edit a `pe.txt` file:

```
pe_name           smp
slots             9999
user_lists        NONE
xuser_lists       NONE
start_proc_args   /bin/true
stop_proc_args    /bin/true
allocation_rule   $pe_slots
control_slaves    FALSE
job_is_first_task TRUE
urgency_slots     min
accounting_summary FALSE
```

Then run `qconf -Ap pe.txt` as root. Now smp can be used as a PE in the qsub argument (`qsub -pe smp`). Alternatively, I found that it is easier to just type `qconf -ap smp` to edit a default file directly and then save this file.

Next thing is to add the new PE smp into the `all.q` queue. Please do the following as root on the head node:

```
qconf -mq all.q
```

This will open an interactive vi session. Look for the `pe_list` line, and add `smp` to that line. Save the file and exit vi. You should now be able to use the smp PE.

## Change updatedb schedule

If you have a large number of files in the system, you should change the updatedb schedule. Otherwise, the  `updatedb` program can consume a significant amount of system resources and occasionally makes the system extremely slow. You just need to edit  `/etc/updatedb.conf` and add `/export` and `/home` below:
```
PRUNEPATHS = "/afs /media /net /sfs /tmp /udev /var/spool/cups /var/spool/squid /var/tmp /export"
```

so that the `/export` (in nas-0-0) and `/home` (in biocluster) directory is not used in the updatedb operation.

Alternatively, you can simply just disable updatedb in cron or remove it completely. 

To do this, first find out where the program is located on disk.
```
$ type updatedb
updatedb is /usr/bin/updatedb
```
Next find out what package provides updatedb.
```
$ rpm -qf /usr/bin/updatedb
mlocate-0.26-3.fc19.x86_64
```
See if anything requires mlocate.
```
$ rpm -q --whatrequires mlocate
no package requires mlocate
```
Nothing requires it so you can remove the package.
```
$ yum remove mlocate
```
It will show that it is removed successfully, with a warning that `warning: /etc/updatedb.conf saved as /etc/updatedb.conf.rpmsave`. Note that this should be done in both head node and nas-0-0. I personally do not recommend removing mlocate though.

## head node restrictions

Head node should be used for user login and job submission and external data download only. You may want to restrict the amount of memory used in the head node by each user, to prevent somebody from running a job with large chunks of memory (when swap is used, the head node becomes extremely slow). For example, adding the following

```
* soft as 8000000
* hard as 16000000
```
to `/etc/security/limits.conf` file, so no process uses more than 8GB memory.


## Install R

R is a statistical computing language that is commonly used in bioinformatics. However, Rocks package repository does not include R, so you need to install it manually using [EPEL](https://fedoraproject.org/wiki/EPEL), which stands for Extra Package for Enterprise Linux. The detailed procedure is described below.

1. log in head node as root.
2. Install EPEL if you have not done so: `rpm -Uvh https://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm`. You may need to change the version number, depending on your Rocks/CentOS version.
3. Install R by `yum install R`. Note that yum should automatically search EPEL for R packages, but if not, you can manually force yum to do that by `yum --enablerepo=epel install R`. (See troubleshooting information below if this command fails)
4. Go to `/usr/cache/yum/epel/packages` (actually, check `/etc/yum.conf` to know the exact location) to know what packages are just installed. Typically, 10-20 packages are installed to have full functionality of R in the system.
5. Now copy all these \*.rpm files to `/export/rocks/install/contrib/6.2/x86_64/RPMS` (change the path to suite your own system!). 
6. Now go to `/export/rocks/install/site_profile/6.2/nodes`, edit `extend-compute.xml`, and add in the name of the packages (see an example below).
7. This is totally optional and I do not actually use it. If you want to use graphical interface in compute node (which does not make sense to me), since X11 is required, use `rocks add host attr compute-0-0 X11` to enable this.
8. Then `cd /export/rocks/install`, type `rocks create distro`, then re-install all compute nodes. (Hint: you can do `rocks set host boot compute action=install` so all compute nodes re-install themselves after rebooting).

> Example: this is what I need to add in my `extend-compute.xml` file:
```
<package>R</package>
<package>R-core</package>
<package>R-core-devel</package>
<package>R-devel</package>
<package>R-java</package>
<package>R-java-devel</package>
<package>blas</package>
<package>blas-devel</package>
<package>lapack</package>
<package>lapack-devel</package>
<package>libicu</package>
<package>libicu-devel</package>
<package>libRmath</package>
<package>libRmath-devel</package>
<package>texinfo</package>
<package>texinfo-tex</package>
<package>xz-devel</package>
```

- Troubleshooting

    In recent versions of Rocks, I cannot use yum to install R, and typical error messages are below:
    ```
    Error: Package: R-core-3.2.3-1.el6.x86_64 (epel)
           Requires: libicuuc.so.42()(64bit)
Error: Package: R-core-devel-3.2.3-1.el6.x86_64 (epel)
           Requires: blas-devel >= 3.0
Error: Package: R-core-devel-3.2.3-1.el6.x86_64 (epel)
           Requires: lapack-devel
Error: Package: R-core-devel-3.2.3-1.el6.x86_64 (epel)
           Requires: texinfo-tex
Error: Package: R-core-devel-3.2.3-1.el6.x86_64 (epel)
           Requires: xz-devel
Error: Package: R-core-3.2.3-1.el6.x86_64 (epel)
           Requires: libicui18n.so.42()(64bit)
Error: Package: R-core-devel-3.2.3-1.el6.x86_64 (epel)
           Requires: libicu-devel
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
 ```
 
   To solve this, install each one manually and finally just do a manual "rpm -ivh --nodeps x.rpm" one by one and in the end it will work. Go to www.rpmfind.net/ to find each RPM for Centos 6:
   ```
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/libicu-4.2.1-12.el6.x86_64.rpm
   [root@biocluster ~]# rpm -ivh libicu-4.2.1-12.el6.x86_64.rpm 
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/libicu-devel-4.2.1-12.el6.x86_64.rpm
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/xz-devel-4.999.9-0.5.beta.20091007git.el6.x86_64.rpm
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/texinfo-4.13a-8.el6.x86_64.rpm
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/texinfo-tex-4.13a-8.el6.x86_64.rpm
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/lapack-3.2.1-4.el6.x86_64.rpm
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/lapack-devel-3.2.1-4.el6.x86_64.rpm
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/blas-3.2.1-4.el6.x86_64.rpm
   [root@biocluster ~]# wget ftp://195.220.108.108/linux/centos/6.7/os/x86_64/Packages/blas-devel-3.2.1-4.el6.x86_64.rpm
   ```

## Install optional packages

Other optional packages can be installed in a similar manner as described above. For example, a useful package is `screen`, which allows easy terminal management and a tutorial is [here](http://www.tecmint.com/screen-command-examples-to-manage-linux-terminals/).

## Change compute node drive partition

By default Rocks partition the drive in computer node to 16G /, 1G swap, 4G /var and others to /state/partitionX. Many of the programs that we use need to use large temporary files or space, so we need to re-define the partition table for compute nodes. Edit /export/rocks/install/site-profiles/6.2/nodes/replace-partition.xml (or “cp skeleton.xml replace-partition.xml“ if it does not exist). Add the section below to the file.

```

<pre>
echo "clearpart --all --initlabel --drives=sda
part /boot --size 1000 --ondisk sda
part / --size 1000000 --ondisk sda
part swap --size 16000 --ondisk sda
part /state/partition1 --size 1 --grow --ondisk sda" > /tmp/user_partition_info
</pre>
```

Then reinstall compute nodes. If the partition size does not change after reinstall, then you should follow instructions [here](http://central6.rocksclusters.org/roll-documentation/base/6.1.1/customization-partitioning.html), to `rocks remove partition` and use `nukeit.sh` to remove the file .rocks-release from the first partition of each disk on the compute node.

## Set up infiniband network

Subnet manager (OpenSM) assigns Local IDentifiers (LIDs) to each port connected to the InfiniBand fabric, and develops a routing table based off of the assigned LIDs. Many infiniband switches have built-in subnet manager, which can manage the fabric automatically and ensure proper communications between the IB hosts. Just in case the IB switch does not have this functionality, you need to install a subnet manager in one of the nodes in the network. By default, the head node already has opensm installed and enabled, if the server has an IB card during installation (check this by `service opensm status` in head node). Nevertheless, I recommend that you should install it in the storage node `nas-0-0`, because it is typically the most stable host in the entire network. (If the [OFED roll](https://github.com/sdsc/mlnx-ofed-roll) is installed, then based on [manual](http://neams.rpi.edu/roll-documentation/rocks+mlnx-ofed/5.4/setting-up-subnetmanager.html), just do a `rocks add host attr nas-0-0 subnetmanager true` and then `rocks sync config` and then `ssh nas-0-0 /boot/kickstart/cluster-kickstart`, but I was not able to make it work.) To set up this, do a `yum install opensm` and `chkconfig --level 345 opensm on` in nas-0-0. Then use `chkconfig --level 345 opensm off` in head node.

The next thing is to set up a new subnet specifically for IB traffic, and we can just use 192.168.0.0 subnet. To do this, use

```
[root@biocluster ~]# rocks list network
NETWORK  SUBNET        NETMASK       MTU   DNSZONE   SERVEDNS
private: 10.1.0.0      255.255.0.0   1500  local     True    
public:  128.125.248.0 255.255.254.0 1500  wglab.org False   
[root@biocluster ~]# rocks add network ipoib 192.168.0.0 255.255.0.0 mtu=65520 servedns=true
[root@biocluster ~]# rocks list network
NETWORK  SUBNET        NETMASK       MTU    DNSZONE   SERVEDNS
ipoib:   192.168.0.0   255.255.0.0   65520  ipoib     True    
private: 10.1.0.0      255.255.0.0   1500   local     True    
public:  128.125.248.0 255.255.254.0 1500   wglab.org False   
```

After installation of compute nodes, you need to manually set up the network for infiniband. Note that you should not edit the typical files such as `/etc/sysconfig/network-scripts/ifcfg-ib0`; it may work to configure the network, but it will not survive the next re-installation of the compute node. More explanation is given in [this section](05 Network administration.md). To ensure that the network information is stored within the central MySQL database in head node, you must use Rocks system command. 

We first set up the ib in head node:

```
[root@biocluster ~]# rocks list host interface biocluster
SUBNET  IFACE MAC                                                         IP             NETMASK       MODULE NAME       VLAN OPTIONS CHANNEL
private eth0  04:7D:7B:47:6A:04                                           10.1.1.1       255.255.0.0   ------ biocluster ---- ------- -------
public  eth1  04:7D:7B:47:6A:05                                           128.125.248.41 255.255.254.0 ------ biocluster ---- ------- -------
------- ib0   80:00:00:48:FE:80:00:00:00:00:00:00:00:02:C9:03:00:50:5E:95 -------------- ------------- ------ ---------- ---- ------- -------
[root@biocluster ~]# rocks remove host interface biocluster iface=ib0
[root@biocluster ~]# rocks add host interface biocluster iface=ib0 ip=192.168.1.1 mac=80:00:00:48:FE:80:00:00:00:00:00:00:00:02:C9:03:00:50:5E:95 subnet=ipoib name=biocluster-ib
[root@biocluster ~]# rocks list host interface biocluster
SUBNET  IFACE MAC                                                         IP             NETMASK       MODULE NAME          VLAN OPTIONS CHANNEL
private eth0  04:7D:7B:47:6A:04                                           10.1.1.1       255.255.0.0   ------ biocluster    ---- ------- -------
public  eth1  04:7D:7B:47:6A:05                                           128.125.248.41 255.255.254.0 ------ biocluster    ---- ------- -------
ipoib   ib0   80:00:00:48:FE:80:00:00:00:00:00:00:00:02:C9:03:00:50:5E:95 192.168.1.1    255.255.0.0   ------ biocluster-ib ---- ------- -------
[root@biocluster ~]# rocks sync host network biocluster
```

The last command is required to make the change effective immediately.

Next, we set up each compute node. An example is given below:

```
[root@biocluster ~]# rocks list host interface nas-0-0
SUBNET  IFACE MAC                                                         IP           NETMASK     MODULE NAME    VLAN OPTIONS CHANNEL
private eth0  00:25:90:85:d1:50                                           10.1.255.254 255.255.0.0 ------ nas-0-0 ---- ------- -------
------- ib0   80:00:00:48:fe:80:00:00:00:00:00:00:00:02:c9:03:00:0c:c1:73 ------------ ----------- ------ ------- ---- ------- -------
------- eth1  00:25:90:85:d1:51                                           ------------ ----------- ------ ------- ---- ------- -------
------- eth3  00:25:90:85:d1:53                                           ------------ ----------- ------ ------- ---- ------- -------
------- eth2  00:25:90:85:d1:52                                           ------------ ----------- ------ ------- ---- ------- -------
[root@biocluster ~]# rocks remove host interface nas-0-0 iface=ib0
[root@biocluster ~]# rocks add host interface nas-0-0 iface=ib0 ip=192.168.255.254 mac=80:00:00:48:fe:80:00:00:00:00:00:00:00:02:c9:03:00:0c:c1:73 subnet=ipoib name=nas-0-0-ib
[root@biocluster ~]# rocks sync host network nas-0-0   
```

Some people say that firewall should be additionally opened for inter-node IB communications. I was not exactly sure how necessary it is and what role it plays, but here is the command (for head node).
```
# Firewall
rocks add firewall host=localhost chain=INPUT protocol=all service=all action=ACCEPT network=ipoib iface=ib0 rulename="A80-IB0-PRIVATE"
rocks sync host firewall localhost
```

## Enable web access to frontend

The general procedure is described [here](http://central6.rocksclusters.org/roll-documentation/base/6.1.1/enable-www.html). Briefly, 

```
[root@biocluster ~]# rocks report host firewall localhost
<file name="/etc/sysconfig/iptables" perms="500">
*nat
#  MASQUERADE (host) : 
-A POSTROUTING -o eth1 -j MASQUERADE
COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
#  A10-REJECT-411-TCP (host) : 
-A INPUT -p tcp --dport 372 -j REJECT --sport 1024:65535
#  A10-REJECT-411-UDP (host) : 
-A INPUT -p udp --dport 372 -j REJECT --sport 1024:65535
#  A15-ALL-LOCAL (global) : 
-A INPUT -j ACCEPT -i lo
#  A20-ALL-PRIVATE (global) : 
-A INPUT -i eth0 -j ACCEPT
#  A20-SSH-PUBLIC (global) : 
-A INPUT -i eth1 -p tcp --dport ssh -j ACCEPT -m state --state NEW
#  A30-RELATED-PUBLIC (global) : 
-A INPUT -i eth1 -j ACCEPT -m state --state RELATED,ESTABLISHED
#  A40-HTTPS-PUBLIC-LAN (host) : 
-A INPUT -i eth1 -p tcp --dport https -j ACCEPT -m state --state NEW --source &Kickstart_PublicNetwork;/&Kickstart_PublicNetmask;
#  A40-WWW-PUBLIC-LAN (host) : 
-A INPUT -i eth1 -p tcp --dport www -j ACCEPT -m state --state NEW --source &Kickstart_PublicNetwork;/&Kickstart_PublicNetmask;
#  A50-FORWARD-RELATED (host) : 
-A FORWARD -i eth1 -o eth0 -j ACCEPT -m state --state RELATED,ESTABLISHED
#  A60-FORWARD (host) : 
-A FORWARD -i eth0 -j ACCEPT
#  R10-GANGLIA-UDP (host) : block ganglia traffic from non-private interfaces
-A INPUT -p udp --dport 8649 -j REJECT
#  R20-MYSQL-TCP (host) : block mysql traffic from non-private interfaces
-A INPUT -p tcp --dport 3306 -j REJECT
#  R30-FOUNDATION-MYSQL (host) : block foundation mysql traffic from non-private interfaces
-A INPUT -p tcp --dport 40000 -j REJECT
#  R900-PRIVILEGED-TCP (global) : 
-A INPUT -i eth1 -p tcp -j REJECT --dport 0:1023
#  R900-PRIVILEGED-UDP (global) : 
-A INPUT -i eth1 -p udp -j REJECT --dport 0:1023
COMMIT
</file>
[root@biocluster ~]# rocks remove firewall host=localhost rulename=A40-WWW-PUBLIC-LAN
host list: ['localhost']
checking for host biocluster
[root@biocluster ~]# rocks add firewall host=localhost network=public protocol=tcp service=www chain=INPUT action=ACCEPT flags="-m state --state NEW --source 0.0.0.0/0.0.0.0" rulename=A40-WWW-PUBLIC-NEW
host list: ['localhost']
checking for host biocluster
[root@biocluster ~]# rocks sync host firewall localhost
```

We delete the A40-WWW-PUPBLIC-LAN and create a new A40-WWW-PUBLIC-NEW rule above, which allows web traffic from any source to flow in and out of the frontend. By default, we only allow those local subnet (within the same netmask) to access the frontend.

You can now go to web browser and visit the cluster's webpage. The home page is a wordpress page, which includes links to Ganglia cluster status report.

## Configure message of the day (motd)

Add the following to `/etc/motd`:

```
          _______________________________________
 ________|                                       |_________
\        |        biocluster.xxx.yyy.zzz         |        /
 \       |                                       |       /
  \      |       "Bioinformatics Clusterr"       |      /
  /      |_______________________________________|      \
 /___________)                               (___________\
```

## Configure a host as job submission host

Occasionally you may want to make a specific host as job submission host (especially if you opened the eth1 in this host). This can be easily doen by:

```
rocks set host attr compute-0-0 submit_host true
```

Then reinstall compute-0-0.

## Configure storage node

This requires changing a few autofs configuration files, so that `home`, `apps`, `datasets` and so on are automatically mounted to nas-0-0, rather than the directory in `/export` in head node. For example, for `apps`, just edit `/etc/auto.share` to be

```
apps nas-0-0-ib.ipoib:/export/&
datasets        nas-0-0-ib.ipoib:/export/&
```

For `home` directory, it is a little more complicated, and I will describe the treatment in the following User Management article.

## Configure SGE

This topic will be discussed in detail in later articles. However, here I give some recommendations on how to perform an initial set up of SGE. I suggest to use `qconf -mc` and change h_vmem to be consumable with default value of 4G (or another value that you want). Then use `qconf -me <HOSTNAME>` to change the complex_value for each host. This ensures smooth execution of jobs and ensures stability of the cluster when multiple users are present and some of them are running weird jobs that may crash nodes.







