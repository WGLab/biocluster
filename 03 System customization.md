# Overview

Once installation of the cluster is done, you typically want to make some customization to the cluster. This section describes several common tasks.

## Setting up system path for Perl and Python

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

For Java, Rocks automatically set up the version so you do not have to worry about it. To check this, 

```
[root@biocluster admin]# ll /usr/java/
total 4
lrwxrwxrwx 1 root root   16 Feb 10 05:49 default -> /usr/java/latest
drwxr-xr-x 8 root root 4096 May  7  2015 jdk1.7.0_51
lrwxrwxrwx 1 root root   21 Feb 10 05:49 latest -> /usr/java/jdk1.7.0_51
```

So the `/usr/java/lastest/bin` in the PATH already point to version 1.7 correctly.


## Install R

R is a statistical computing language that is commonly used in bioinformatics. However, Rocks package repository does not include R, so you need to install it manually using [EPEL](https://fedoraproject.org/wiki/EPEL), which stands for Extra Package for Enterprise Linux. The detailed procedure is described below.

1. log in head node as root.
2. Install EPEL if you have not done so: `rpm -Uvh https://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm`. You may need to change the version number, depending on your Rocks/CentOS version.
3. Install R by `yum install R`. Note that yum should automatically search EPEL for R packages, but if not, you can manually force yum to do that by `yum --enablerepo=epel install R`.
4. Go to `/usr/cache/yum/epel/packages` to know what packages are just installed. Typically, 5-10 packages are installed to have full functionality of R in the system.
5. Now copy all these \*.rpm files to `/export/rocks/install/contrib/6.1.1/x86_64/RPMS` (change the path to suite your own system!). 
6. Now go to `/export/rocks/install/site_profile/6.1.1/nodes`, edit `extend-compute.xml`, and add in the name of the packages (see an example below).
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
<package>ecj</package>
<package>gcc-java</package>
<package>lapack</package>
<package>lapack-devel</package>
<package>libgcj-devel</package>
<package>libgcj-src</package>
<package>libicu</package>
<package>libicu-devel</package>
<package>texinfo</package>
<package>texinfo-tex</package>
<package>xz-devel</package>
<package>cmake</package>
<package>bzip2-devel</package>
<package>gcc-c++</package>
<package>libart_lgpl</package>
<package>libgcj</package>
<package>libstdc++-devel</package>
```

## Set up infiniband network

Many infiniband switches have built-in subnet manager, which can manage the fabric automatically and ensure proper communications between the IB hosts. Just in case the IB switch does not have this functionality, you need to install a subnet manager in one of the nodes in the network. I recommend that you should install it in the storage node `nas-0-0`, because it is typically the most stable host in the entire network. Just do a `rocks add host attr nas-0-0 subnetmanager true` and then `rocks sync config`.

After installation of compute nodes, you need to manually set up the network for infiniband. Note that you should not edit the typical files such as `/etc/sysconfig/network-scripts/ifcfg-ib0`; it may work to configure the network, but it will not survive the next re-installation of the compute node. More explanation is given in [this section](05 Network administration.md). To ensure that the network information is stored within the central MySQL database in head node, you must use Rocks system command. An example is given below:

```
rocks add host interface compute-0-4 ib0
rocks set host interface ip compute-0-4 iface=ib0 ip=192.168.1.249
rocks set host interface subnet compute-0-4 iface=ib0 subnet=ipoib
rocks set host interface name compute-0-4 iface=ib0 name=compute-0-4-ib
rocks sync config
rocks sync host network compute-0-4
```

## Change limit on the number of open files

By default, Linux has a limit of 1024 files that can be opened simultaneously. For bioinformatics applications, this is way too small. A simple variant calling (such as by GATK) or a simple allele frequency calculation (such as by PennCNV) can easily exceed this limit and result in mysterious failure of the software tools.

You can follow this procedure to change this limit:

First, run a `ulimit -a`, which confirms that limit on open files is “1024” which is too few.  Next, run the following command (if the `extend-compute.xml` file does NOT exist):

```
# cd /export/rocks/install/site-profiles/6.1/nodes/
# cp skeleton.xml extend-compute.xml
```

Now modify the “<post>” section in extend-compute.xml with:

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

## Adjust system time

First check the hardware clock with the following command.

```
hwclock --show    
```

If your hardware clock is not set to your local time, then you must set the system time to local time as root.

If you installed the ntp package you can: 

```
ntpdate pool.ntp.org
```

or you can manually update: 

```
date --set "5 Aug 2012 12:54 IST"
```

Obviously in the above command you must set your date, time and time zone correctly.

Now as root, synchronize the hardware clock to the current system time as local time.

```
hwclock --systohc --localtime
```

Now the hardware clock is re-adjusted to the system time and both now point to the local time.

## Change updatedb schedule

If you have a large number of files in the system, you should change the updatedb schedule. Otherwise, the  `updatedb` program can consume a significant amount of system resources and occasionally makes the system extremely slow. You just need to edit  `/etc/updatedb.conf` and add `/export` and `/home` below:
```
PRUNEPATHS = "/afs /media /net /sfs /tmp /udev /var/spool/cups /var/spool/squid /var/tmp /export"
```

so that the `/export` (in nas-0-0) and `/home` (in biocluster) directory is not used in the updatedb operation.

# head node restrictions

Head node should be used for user login and job submission and external data download only. You may want to restrict the amount of memory used in the head node by each user, to prevent somebody from running a job with large chunks of memory (when swap is used, the head node becomes extremely slow). For example, adding the following

```
* soft as 24000000
* hard as 20000000
```
to `/etc/security/limits.conf` file, so no process uses more than 24GB memory.
