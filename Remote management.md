# IPMI management

IPMI stands for Intelligent Platform Management Interface. Most servers today allows remote IPMI management, which basically allows one to run command line tools to remotely control a server, such as starting up, shutting down and resetting. For more details, check wikipedia page [here](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface).

Additionally, almost all server manufacturers will allow remote KVM (keyboard-video-mouse) access to server, though some of them charges for using this feature.

Traditionally, Serial Over LAN was commonly used in cluster management, but it is rarely used in new clusters today (though still can be found in clusters built many years ago, such as USC's high-performance computing cluster). The discussion below focus on KVM redirection and on several useful IPMI commands.

## KVM redirection

I have used SuperMicro server and Dell server, which both support remote KVM for free (that is, the functionality is integrated in their motherboards). The biocluster uses IBM DX360M3 server, which requires a "media key" to use the remote KVM feature. Typically it cost >$100, but these keys can be found in ebay for under $5. Similarly, HP servers require something similar that one needs to pay to use these KVM features; they charge a few hundred dollars for that, though you can grab one in Ebay for a few dollars.

Below I describe how to access the head node from Internet and how to access specific compute node from the head node.

### Remote KVM on head node

Some people consider it a security concern, though I believe that the benefits far outweight the potential security risk. Having said that, I did see a report that SuperMicro set up a default IPMI password that many people never change (or realize) which resulted in unintended consequences that the server can be secretely controlled by somebody else. So to enable remote KVM access to head node, make sure to change password!!! This is like changing your wireless router's admin password, whether you set up a new wireless network. It should be something very natural to do.

When you start up a head node, go to the BIOS, there should be a dedicated section on IPMI configuration. Set up the IP address and netmask there (you need to request this information from your network administrator, typically by providing the MAC address of the IPMI ethernet port). Typically, there is an option in BIOS about "dedicated" versus "shared" ethernet port; make sure to select "dedicated" since you need to connect Internet to the dedicated IPMI ethernet port.


The instructions below focuses on SuperMicro KVM access. It provides web access via a Java Virtual Machine, but the Rocks system is just too old to use the feature. 

To access the KVM in a node in the cluster, I first use vncserver to connect to head node, then use web interface to visit nas-0-0-ipmi, then select 'remote access', and a dialogue will pop out asking how to handle the *.jnlp file. Select "open", then navigate to the java directory. But Java will say something like "jnlp: illegal instruction". How to solve it?

To solve this problem, I need to download Java 1.8 (64bit binary) and install in local directory by unpacking it. Then run the "jcontrol" program and add "https://nas-0-0-ipmi" into the exception list for security settings (since the java program from the IPMI is self-signed). Then do the same thing above, but navigate to the new Java and use the javaws to open this jnlp file. Then everything works just fine.

