# Storage administration

MegaRaid Hardware information
Biocluster head node has 9260-8i. nas-0-0 has 9260-4i. bionas2 has 9271-4i.
System status information
http://mycusthelp.info/LSI/_cs/AnswerDetail.aspx?sSessionID=&aid=8264
use “./lsiget.sh” to generate the tar.gz file and send to SM.

MegaCli
MegaCli installation
Download the program from http://www.lsi.com/support/Pages/download-results.aspx?keyword=megacli
By default, MegaCli is installed at /opt/MegaRAID/MegaCli/MegaCli64

MegaCli command line reference
Use the following command to print complete help message:
/opt/MegaRAID/MegaCli/MegaCli64 -h

The following link has more detailed description:
http://mycusthelp.info/LSI/_cs/AnswerPreview.aspx?sSessionID=&inc=8040

MegaCli cheat sheet
http://erikimh.com/megacli-cheatsheet/
http://genuxation.com/wiki/index.php/MegaCLI
https://supportforums.cisco.com/docs/DOC-16309
lsi.sh script https://calomel.org/megacli_lsi_commands.html 

Essential commands to use in a new system
[root@nas-0-1 ~]$ /opt/MegaRAID/MegaCli/MegaCli64  -AdpAllInfo -aALL | fgrep Failed
Security Key Failed              : No
  Failed Disks    : 2 
Deny Force Failed                       : No

This immediately tells us that two LD has failed!!!
To solve this, I cleared foreign configuration, and then manually stopped an ongoing ‘Rebuild’, then assigned the two ‘bad drives’ to ‘global hot spare’, then the rebuild starts automatically. I stopped an ongoing ‘Rebuild’ because it is rebuilt from a different enclosure.




show system informatoin:
MegaCli -AdpAllInfo -aALL
MegaCli -CfgDsply -aALL
MegaCli -adpeventlog -getevents -f lsi-events.log -a0 -nolog


Show adapter information:
/opt/MegaRAID/MegaCli/MegaCli64 -AdpAllInfo -a0

show enclosure information:
/opt/MegaRAID/MegaCli/MegaCli64 -EncInfo -aALL

show virtual dive information:
/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -aALL

show physical drive information
/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL

/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL | fgrep S.M.A.R.T
#Identify discs that may fail in the future

show BBU infomration:
/opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -aALL

Manual relearn battery
/opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -BbuLearn -a0 

use WB(write-back) even if BBU is bad
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp CachedBadBBU -Lall -aAll

Reverse above procedure
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp NoCachedBadBBU -Lall -aAll

Set virtual drive to writeback
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp WB -LALL -aALL

Set virtual drive to readahead (ReadAdaptive)
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp RA -LALL -aALL

Identify all hot spares
/opt/MegaRAID/MegaCli/MegaCli64 -PDlist -a0 | fgrep Hotspare -B 10

set global hot spare
/opt/MegaRAID/MegaCli/MegaCli64 -PDHSP -Set -PhysDrv [E:S] -aN


Rebuild a drive

Perform diagnosis on adapter
/opt/MegaRAID/MegaCli/MegaCli64 -AdpDiag -a0

Start, stop, suspend, resume, and show the progress of a consistency check operation
/opt/MegaRAID/MegaCli/MegaCli64 -LDCC -Start|-Abort|-Suspend|-Resume|-ShowProg|-ProgDsply -Lx|-L0,1,2|-LALL -aN|-a0,1,2|-aALL

MegaRaid Storage Manager

Install: 
Download the package, then “./install.csh -a” for complete installation
However, every time I install it, I got the following message. It works for a while but after a while, the nas-0-0 can no longer be accessed from biocluster.
Can not find snmptrap in /usr/bin
Can not continue installation of LSI SAS-IR Agent
Please install the net-snmp agent
warning: %post(sas_ir_snmp-13.08-0401.x86_64) scriptlet failed, exit status 1


I solved the problem by installing net-snmp-utils (first search for packages that provides snmptrap, then install it):
[root@nas-0-0 disk]# yum provides */snmptrap  
[root@nas-0-0 disk]# yum install net-snmp-utils

Uninstall: 
/usr/local/MegaRAID\ Storage\ Manager/uninstaller.sh
or
/usr/local/MegaRAID\ Storage\ Manager/__uninstaller.sh if the former does not work (such as in nas-0-1)





