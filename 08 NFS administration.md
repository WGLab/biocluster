## Reference

The main reference for NFS can be found [here](http://nfs.sourceforge.net/nfs-howto/).

The command below can be used to check the version of NFS:

```
[root@biocluster /]$ /sbin/mount.nfs -V
mount.nfs (linux nfs-utils 1.2.3)
```

## NFS tuning

### checking NFS version installed (2, 3, 4 are all installed)

```
[root@compute-0-0 ~]# rpcinfo -p nas-0-0 
[root@nas-0-0 ~]# rpcinfo -p localhost

[kaiwang@nas-0-0 ~]$ cat /proc/fs/nfsd/portlist
rdma 20049
tcp 2049
udp 2049
 
[kaiwang@biocluster ~]$ cat /proc/fs/nfsd/portlist
tcp 2049
udp 2049
```

check NFS status to see if retransmission happens a lot since last reboot. Usually we want to see retrans to be 0 if it is high, increase the number of NFS daemons (see below).
```
[yunfeiguo@biocluster2 partition1]$ nfsstat -rc
Client rpc stats:
calls      retrans    authrefrsh
98184      0          98188
```

### change number of NFS daemons

To change number of instances of NFS daemon in the NFS server (nas-0-0), in /etc/init.d/nfs, the line below specifies the default number of threads (8)

```
        [ -z "$RPCNFSDCOUNT" ] && RPCNFSDCOUNT=8 
```

type `service nfs reload` to reload the change, without restarting server.

I found that RPCNFSDCOUNT=64 needs to be changed in `/etc/sysconfig/nfs` instead of the `/etc/init.d/nfs`, since the first line of the `init.d/nfs` file asks to run the sysconfig/nfs file instead.

Typically, the nas-0-0 will have 6, 12 or more cores, so it is totally fine to set the value to 64 or more, to accormodate busy NFS requests from clients, especially when the number of nodes in the cluster is high.

### Improving NFS performance

Some of the following points can be used to improve performance:

1. Add 'RPCNFSDCOUNT=64' in `/etc/sysconfig/nfs` as mentioned above
2. In head node, Modify `/etc/auto.master`. In the line `/home auto.home`. Remove `--timeout 1200`, add `rsize=32768,wsize=32768,hard,intr`. Since IB traffic is used for NFS, here we set up very large rsize and wsize values.
3. Reload config info (`service autofs reload`)
4. You may want to tune LSI RAID card parameters as well to improve read/write performance. The following command ensures write-back, read-ahead, and use cache even if a bad battery is encountered (the latest generation of card no longer use battery though).

```
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp WB -LALL -aALL
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp CachedBadBBU -Lall -aAll
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp RA -LALL -aALL
```

### Setting up RDMA mount

The following instructions are outdated, as we are not using RDMA now. However, it may be helpful to some readers:

Regardless of how the client was built (module or built-in), use this command to mount the NFS/RDMA server:

```
$ mount -o rdma,port=20049 <IPoIB-server-name-or-address>:/<export> /mnt
```

To verify that the mount is using RDMA, run "cat /proc/mounts" and check the "proto" field for the given mount. 

### Read Ahead and IO scheduler tuning

```
[root@nas-0-0 /home/kaiwang]$ blockdev --report
RO    RA   SSZ   BSZ   StartSec            Size   Device
…...
rw   256   512  4096          0  45000519843840   /dev/sdd
rw   256   512   512         34  45000519809536   /dev/sdd1
rw   256   512   512          0 154289712398336   /dev/dm-0
```

To change the 'Read-Ahead' value to 8 MB (16384 times of 512 bytes blocks).

```
# blockdev --setra 16384 /dev/mapper/nas--0--0-export
```

More about IO scheduler: http://www.redhat.com/magazine/008jun05/features/schedulers/

* deadline scheduler: this is a cyclic elevator but with a twist: requests are given a deadline by which they get served. When a request starts to look like it's going to expire, the kernel will skip intermediate sectors and move to that request, thus giving some realtime behaviour.
* AS scheduler: a cyclic elevator with waiting policy: after you service a request that looks like it might have future requests coming nearby, you pause even if there's more sectors in your work queue. The anticipatory scheduler literally anticipates more requests to follow on this track or very close by. How AS decides whether to anticipate is basically just lot of guesswork based on typical access patterns.
* cfq scheduler: different sort of stab at fairness. It's different from either of these two, and doesn't use cyclic elevator and has realtime guarantees and aggressively avoids starvation. It could be a good scheduler for multiuser systems.
* noop scheduler: just service next request in the queue without any algorithm to prefer this or that request.

To make it permanent upon system reboot, just add this command entry in `/etc/rc.local` on nas-0-0

```
echo anticipatory > /sys/block/sdb/queue/scheduler
echo anticipatory > /sys/block/sdc/queue/scheduler
blockdev --setra 16384 /dev/mapper/nas--0--0-export
```

### Fixing "owned by nobody" problem in NFS

Once after I upgrade Rocks system, I found that when I `ls -l /home/kaiwang` it shows that it is owned by “nobody” (it only happens in head node, not compute node). This is due to `/home/kaiwang` mounted by NFS v4. Therefore, I checked http://www.softpanorama.org/Net/Application_layer/NFS/Troubleshooting/nfsv4_mounts_files_as_nobody.shtml, and added “Domain=local” in the `/etc/idmapd.conf` file in biocluster, and did `service rpcidmapd restart` in both biocluster and in nas-0-0, and everything works fine now.

## Some NFS observations:

(1) rsize/wsize: For NFSv2 or NFSv3, the default values for both parameters is set to 8192. For NFSv4, the default values for both parameters is set to 32768. See http://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-nfs-client-config-options.html.

There are no real performance differences when increasing the size to rsize=1048576,wsize=1048576. 

(2) readahead: default is 256 but can be set to 16384 for sdb, sdc, sdd, sde, dm-0. There are no realistic differences in terms of performance

(3) IO scheduler: No realistic differences in terms of performance, so cfq (completely fair queue) as default would be fine.



## Monitor file system IO

We created a cron process with “*/15 * * * * ssh nas-0-0 ./iomonitor.script”, which contains
```
[user@biocluster ~]$ cat iomonitor.script
#!/bin/bash
date >> iostat
iostat -d -x >> iostat
```
It shows that “dm-0” can utilize as high as 57% CPU when running some disk-heavy programs, so the file system load can be really high and is probably why it is unstable.

I think “iostat -n -m -x” may be more informative. 

```
[root@nas-0-0 ~]# iostat -n -m -x
Linux 2.6.32-504.16.2.el6.x86_64 (nas-0-0.local)        04/28/2016      _x86_64_        (16 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
sdb               0.50     3.20  174.83  118.76    19.89    31.42   357.93     2.06    7.01   0.26   7.73
sda              18.60   439.56    1.78   16.14     0.09     1.78   213.54     0.06    3.18   0.34   0.60
sdd               0.00     0.00    0.00    0.00     0.00     0.00     8.73     0.00    5.69   5.69   0.00
sdc               0.00     0.00    0.00    0.77     0.00     0.02    62.48     0.00    0.47   0.42   0.03
dm-0              0.00     0.00  175.33  122.73    19.89    31.45   352.72     2.08    6.96   0.26   7.75

Filesystem:               rMB_nor/s    wMB_nor/s    rMB_dir/s    wMB_dir/s    rMB_svr/s    wMB_svr/s     ops/s    rops/s    wops/s
```

To further diagnose which processes are responsible for high wait time in CPU usage, you can do a `yum install iotop`, and then use the iotop program to check the IO usage of various programs.

## Fix autofs issues

When I cannot umount a NFS directory, use “umount -l” will always work.

When I cannot stop a service such as autofs, just do `ps aux | fgrep auto` and `kill -9` the process.
