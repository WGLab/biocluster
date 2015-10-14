# NFS administration

NFS configuration
http://nfs.sourceforge.net/nfs-howto/
 
[root@biocluster /home/kaiwang]$ /sbin/mount.nfs -V
mount.nfs (linux nfs-utils 1.2.3)

 
#checking NFS version installed (2, 3, 4 are all installed)
[root@compute-0-0 ~]# rpcinfo -p nas-0-0 
[root@nas-0-0 ~]# rpcinfo -p localhost
 
[kaiwang@nas-0-0 ~]$ cat /proc/fs/nfsd/portlist
rdma 20049
tcp 2049
udp 2049
 
[kaiwang@biocluster ~]$ cat /proc/fs/nfsd/portlist
tcp 2049
udp 2049

check NFS status to see if retransmission happens a lot since last reboot. Usually we want to see retrans to be 0 if it is high, increase the number of nfsd.
[yunfeiguo@biocluster2 partition1]$ nfsstat -rc
Client rpc stats:
calls      retrans    authrefrsh
98184      0          98188


The following sentences are outdated as we are not using RDMA now:
Regardless of how the client was built (module or built-in), use this command to mount the NFS/RDMA server:
$ mount -o rdma,port=20049 <IPoIB-server-name-or-address>:/<export> /mnt
To verify that the mount is using RDMA, run "cat /proc/mounts" and check the "proto" field for the given mount. 
Setting up NFS server
change number of instances of NFS daemon in the server

in /etc/init.d/nfs, the line below specifies the default number of threads (8)
        [ -z "$RPCNFSDCOUNT" ] && RPCNFSDCOUNT=8 
 
type service nfs reload to reload the change, without restarting server.
I found that RPCNFSDCOUNT=32 needs to be changed in /etc/sysconfig/nfs instead of the /etc/init.d/nfs, since the first line of the init.d/nfs file asks to run the sysconfig/nfs file instead.

Read Ahead and IO scheduler tuning
[root@nas-0-0 /home/kaiwang]$ blockdev --report
RO    RA   SSZ   BSZ   StartSec            Size   Device
…...
rw   256   512  4096          0  45000519843840   /dev/sdd
rw   256   512   512         34  45000519809536   /dev/sdd1
rw   256   512   512          0 154289712398336   /dev/dm-0

To change the 'Read-Ahead' value to 8 MB (16384 times of 512 bytes blocks).
# blockdev --setra 16384 /dev/mapper/nas--0--0-export

More about IO scheduler: http://www.redhat.com/magazine/008jun05/features/schedulers/

deadline scheduler: this is a cyclic elevator but with a twist: requests are given a deadline by which they get served. When a request starts to look like it's going to expire, the kernel will skip intermediate sectors and move to that request, thus giving some realtime behaviour.
AS scheduler: a cyclic elevator with waiting policy: after you service a request that looks like it might have future requests coming nearby, you pause even if there's more sectors in your work queue. The anticipatory scheduler literally anticipates more requests to follow on this track or very close by. How AS decides whether to anticipate is basically just lot of guesswork based on typical access patterns.
cfq scheduler: different sort of stab at fairness. It's different from either of these two, and doesn't use cyclic elevator and has realtime guarantees and aggressively avoids starvation. It could be a good scheduler for multiuser systems.
noop scheduler: just service next request in the queue without any algorithm to prefer this or that request.

To make it permanent upon system reboot, just add this command entry in /etc/rc.local on nas-0-0

echo anticipatory > /sys/block/sdb/queue/scheduler
echo anticipatory > /sys/block/sdc/queue/scheduler
blockdev --setra 16384 /dev/mapper/nas--0--0-export




NFS benchmark

Local in nas-0-0
[kaiwang@nas-0-0 /export/home/kaiwang]$ time dd if=/dev/zero of=testfile bs=16k count=3276800; time dd if=testfile of=/dev/null bs=16k
53687091200 bytes (54 GB) copied, 42.4457 s, 1.3 GB/s
53687091200 bytes (54 GB) copied, 70.6557 s, 760 MB/s

NFS in nas-0-0
[kaiwang@nas-0-0 ~]$ time dd if=/dev/zero of=testfile bs=16k count=3276800; time dd if=testfile of=/dev/null bs=16k
53687091200 bytes (54 GB) copied, 42.3526 s, 1.3 GB/s
53687091200 bytes (54 GB) copied, 68.5094 s, 784 MB/s

NFS in compute-0-0
[kaiwang@compute-0-0 ~]$ time dd if=/dev/zero of=testfile bs=16k count=3276800; time dd if=testfile of=/dev/null bs=16k
53687091200 bytes (54 GB) copied, 70.4702 s, 762 MB/s
53687091200 bytes (54 GB) copied, 77.9739 s, 689 MB/s

NFS in biocluster
[kaiwang@biocluster ~]$ time dd if=/dev/zero of=testfile bs=16k count=3276800; time dd if=testfile of=/dev/null bs=16k
53687091200 bytes (54 GB) copied, 68.8767 s, 779 MB/s
53687091200 bytes (54 GB) copied, 78.6073 s, 683 MB/s

Local in biogene
[root@nas-0-0 home]# dd if=/dev/zero of=testfile bs=16k count=3276800; dd if=testfile of=/dev/null bs=16k
3276800+0 records in
3276800+0 records out
53687091200 bytes (54 GB) copied, 41.4704 s, 1.3 GB/s

3276800+0 records in
3276800+0 records out
53687091200 bytes (54 GB) copied, 54.0377 s, 994 MB/s

NFS in biogene (without IB)
[kaiwang@biogene ~]$ dd if=/dev/zero of=testfile bs=16k count=3276800; dd if=testfile of=/dev/null bs=16k
3276800+0 records in
3276800+0 records out
53687091200 bytes (54 GB) copied, 465.866 s, 115 MB/s

3276800+0 records in
3276800+0 records out
53687091200 bytes (54 GB) copied, 457.37 s, 117 MB/s

NFS in biogene (with IB)
[root@compute-0-0 temp]# dd if=/dev/zero of=testfile bs=16k count=3276800; dd if=testfile of=/dev/null bs=16k
3276800+0 records in
3276800+0 records out
53687091200 bytes (54 GB) copied, 56.1966 s, 955 MB/s
3276800+0 records in
3276800+0 records out
53687091200 bytes (54 GB) copied, 81.053 s, 662 MB/s

Local in biogene

-t: measures disk read performance without any cache
-T; measures read performance from memory without reading disk
[root@nas-0-0 ~]# hdparm -tT /dev/mapper/nas--0--0-export 

/dev/mapper/nas--0--0-export:
 Timing cached reads:   17570 MB in  2.00 seconds = 8794.49 MB/sec
 Timing buffered disk reads: 784 MB in  3.00 seconds = 260.93 MB/sec




Some observations:

(1) rsize/wsize: For NFSv2 or NFSv3, the default values for both parameters is set to 8192. For NFSv4, the default values for both parameters is set to 32768. See http://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-nfs-client-config-options.html.

There are no real performance differences when increasing the size to rsize=1048576,wsize=1048576. 

(2) readahead: default is 256 but can be set to 16384 for sdb, sdc, sdd, sde, dm-0. There are no realistic differences in terms of performance

(3) IO scheduler: No realistic differences in terms of performance, so cfq (completely fair queue) as default would be fine.


NFS in biocluster
[kaiwang@biocluster ~]$ admin/iozone3_373/src/current/iozone -l 5 -u 5 -r 16k -s 60g -F exfile1 exfile2 exfile3 exfile4 exfile5 -i 0 -i 1 -c
-R: geneate Excel report; -a: full automatic mode; -g: set max file size for auto mode; -i: write test/read test selection; -b: binary file name; -c: use close() in timing calculation

        Children see throughput for  5 initial writers  =  993666.23 KB/sec
        Children see throughput for  5 rewriters        = 1012060.58 KB/sec
        Children see throughput for  5 readers          =  556454.10 KB/sec
        Children see throughput for 5 re-readers        =  618451.55 KB/sec




Monitor file system IO
Yunfei created a cron process with “*/15 * * * * ssh nas-0-0 ./iomonitor.script”, which contains
[00:08 yunfeiguo@biocluster ~]$ cat iomonitor.script
#!/bin/bash
date >> iostat
iostat -d -x >> iostat
I think “iostat -n -m -x” would be more informative. But it shows that “dm-0” can utilize as high as 57% CPU when he runs MAKER, so the file system load can be really high and is probably why it is unstable.


