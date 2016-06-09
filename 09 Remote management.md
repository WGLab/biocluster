## Introduction

It is important to monitor cluster status remotely and control the servers remotely, to eliminate/reduce physical visits to the cluster room. For example, you may want to re-start the servers when needed, or understand the source of hardware failure to take appropriate actions to fix the problems.

There are multiple ways to monitor the cluster status remotely. If the system is running well, you can use VNCserver to remotely log into the cluster, or use Ganglia web services to check the status of the cluster, or use a remote KVM to send specific instructions to the head node (assuming that the KVM switch has this functionality to allow remote access and administration). If the server itself died or becomes non-responsive, then we need to rely on other measures such as IPMI for remote management. Note that traditionally, Serial Over LAN was commonly used in cluster management, but it is rarely used in new clusters today (though still can be found in clusters built many years ago, such as USC's high-performance computing cluster).

This chapter mostly describes IPMI management related issues. IPMI stands for Intelligent Platform Management Interface. Most servers today allows remote IPMI management, which basically allows one to run command line tools to remotely control a server, such as starting up, shutting down and resetting, or to monitor the hardware health status. For more details, check wikipedia page [here](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface).

## IPMI on head node

Some people consider it a security concern, though I believe that the benefits far outweight the potential security risks. Having said that, I did see a report that SuperMicro set up a default IPMI password that many people never change (or realize) which resulted in unintended consequences that the server can be secretely controlled by somebody else. So to enable remote access to head node, make sure to change password!!! This is like changing your wireless router's admin password, whenever you set up a new wireless network. It should be something very natural to do.

When you start up a head node, go to the BIOS, there should be a dedicated section on IPMI configuration. Set up the IP address and netmask there (you need to request this information from your network administrator, typically by providing the MAC address of the IPMI ethernet port). Typically, there is an option in BIOS about "dedicated" versus "shared" ethernet port; make sure to select "dedicated" since you need to connect Internet to the dedicated IPMI ethernet port. (Technically, the real reason is that typical motherboards share IPMI ports with eth0, but eth0 is for connection to internal network in Rocks cluster, not for Internet access, so you cannot access eth0 from the outside world to use IPMI). After changing BIOS, it will take a while (30 seconds or longer) for the changes to take effect.

Now there are two ways to access the head node from internet. You can do a command line such as

```
ipmitool -H 100.200.300.400 -U USERID -P PASSW0RD power status

```

To examine the power status (on or off) of the server. The 100.200.300.400 is the IP address that you set up in BIOS in the head node. Again, make sure to change the password of the default user account!!! Otherwise, somebody else can easily control your server remotely!!!

Alternatively, you can open web browser, visit 100.200.300.400, and log into the web interface, and control/monitor the server remotely. You can check things like voltage, fan speed, hardware status, or perform tasks such as resetting the server, or power cycle the server, and perform password changes or setting up new accounts for IPMI access, etc.

## IPMI on compute node

If a compute node becomes unresponsive for unknown reason, you can use IPMI to restart it, or just use KVM over IPMI (see below) to check the screen of the comptue node to understand the error.

In /etc/hosts.local file, make sure to add lines such as `10.1.1.11       compute-0-0-ipmi`, to set up the interface for each compute node. Then enter the BIOS of compute-0-0, and enter the IP address there, but make sure to use "shared" rather than "dedicated". The simple reason is that "shared" allows you to use one single ethernet port to serve two network, which significantly eliminate the number of ethernet cables in the cluster. When "shared" is selected, the eth0 will be used for IPMI as well. Remember that the IP address of compute-0-0 is 10.1.1.253, so its IPMI port is in the same subnet as well.

Now from head node, you can run the same `ipmitool` commands to control or monitor each compute node. However, I recommend that you do NOT change the password of IPMI in compute node, since they are not connected to the Internet and there is no security risk. Even if you insist to change the password, you should use the identical password for all compute nodes, to faciliate system administration in the future.

## KVM access to head node over IPMI

Almost all server manufacturers will allow remote KVM (keyboard-video-mouse) over IPMI to access to server, though some of them charges extra fees for using this feature. Basically, this feature allows you to run a Java program to use open a screen in your own computer that attach to KVM of the remote head node.

I have used SuperMicro server and Dell server, which both support remote KVM for free (that is, the functionality is integrated in their motherboards). The biocluster uses IBM DX360M3 server, which requires a "media key" to use the remote KVM feature. Typically it cost >$100, but these keys can be found in Ebay for under $5. Similarly, HP servers require something similar that one needs to pay to use these KVM features; they charge a few hundred dollars for that, though you can grab one in Ebay for a few dollars.

When you visit the IPMI from your web browser, there should be a menu option that says something like "remote KVM". Click that option, which will usually ask you to download a jnlp file. Open this file, which will start a Java webstart (accept and acknowledge all security warnings here, as the Java program downloaded from remote head node is self-signed), and you will then be able to access the head node remotely as if you are standing in front of the node.

## KVM access to compute node

To access the KVM in a compute node (for example, compute-0-0) in the cluster, I first need to use vncserver to connect to head node, open Firefox, then use web interface to visit compute-0-0-ipmi, then select 'remote access', and a dialogue will pop out asking how to handle the *.jnlp file. Basically just run the Java webstart, which will open a new window that allows one to see what is going on in the compute node.

> Troubleshooting: Soemtimes when you open the jnlp file, Java will say something like "jnlp: illegal instruction". How to solve it?

> To solve this problem, I need to download Java 1.8 (64bit binary) and install in local directory by unpacking it. (When I am writing this article, Java 1.8 is not available in Rocks yet). Then run the "jcontrol" program and add "https://compute-0-0-ipmi" into the exception list for security settings (since the java program from the IPMI is self-signed). Then do the same thing above, but navigate to the new Java and use the javaws to open this jnlp file. Then everything works just fine.

## Ganglia monitoring of cluster health

By default, Ganglia is included in Rocks, and if you open the public web access to the head node, you can use a web browser to check the status of the cluster.

Ganglia is composed of two parts: Ganglia Monitoring Daemon (gmond) that is installed on every machine you’d like to monitor, and the Ganglia Meta Daemon (gmetad) that collects data from other gmetad and gmond sources and stores their state to disk in indexed round-robin databases.

If you see weird things in Ganglia web page (like missing a machine or having extra machines), what you can do is to restart the gmond service in every machine and restart gmetad.

## Fixing compatibility problem for IPMI in Rocks

For some reason, in some versions of Rocks that I encountered, once a node is turned off, it cannot be turned on again, and the Rocks system freezes at “Starting IPMI” stage when starting up.

I fixed this problem by first “yum install OpenIPMI” in head node. It will install/update a few packages. Then just add

```
<package>OpenIPMI</package>
<package>OpenIPMI-devel</package>
<package>OpenIPMI-libs</package>
```

to `extend-compute.xml` and then reinstall the compute nodes. Then everything returns normal.

This is no longer necessary as of Rocks 6.1.1, since OpenIPMI is installed in compute node by default.

Later on I encountered the same problem in a Rocks 6.2 machine, where all three IPMI packages are installed. Essentially the machine hangs during loading the CentOS, and pressing "right/left" key can show the log message. It is possible that the machine langs because the IPMI interface was not connected to any network, but configured for DHCP. I have yet to check this possibility.

## Additional references

I found a very well written reference on IPMI [here](http://wiki.adamsweet.org/doku.php?id=ipmi_on_linux). Some configuration options and usage examples [here](https://lonesysadmin.net/2007/06/21/how-to-configure-ipmi-on-a-dell-poweredge-running-red-hat-enterprise-linux/).
