# IPMI management

Most servers today allows remote IPMI management, which basically allows one to run command line tools to remotely control a server, such as starting up, shutting down and resetting.

Additionally, almost all manufacturer will allow remote KVM (keyboard-video-mouse) access to server, though some charges for using this feature.

I have used SuperMicro server and Dell server, which both support remote KVM for free. The biocluster uses IBM server, which requires a "media key" to use the remote KVM feature. Typically it cost >$100, but these keys can be found in ebay for around $5. Similarly, HP servers require something similar that one needs to pay to use these KVM features.

The instructions below are about SuperMicro KVM access. It provides web access via a JVM, but the Rocks system is just too old to use the feature. 

To access the KVM in a node in the cluster, I first use vncserver to connect to head node, then use web interface to visit nas-0-0-ipmi, then select 'remote access', and a dialogue will pop out asking how to handle the *.jnlp file. Select "open", then navigate to the java directory. But Java will say something like "jnlp: illegal instruction". How to solve it?

To solve this problem, I need to download Java 1.8 (64bit binary) and install in local directory by unpacking it. Then run the "jcontrol" program and add "https://nas-0-0-ipmi" into the exception list for security settings (since the java program from the IPMI is self-signed). Then do the same thing above, but navigate to the new Java and use the javaws to open this jnlp file. Then everything works just fine.

