## Introduction

Sun Grid Engine is a job scheduling system that is widely used in computing clusters today. Users should use use `qmon` (configure SGE by GUI) and `qconf` (configure SEG by command line) to configure SGE.

## Detailed Reference

The more authorative and comprehensive reference is http://gridscheduler.sourceforge.net/htmlman/htmlman1/qstat.html and http://gridscheduler.sourceforge.net/htmlman/htmlman1/qconf.html

A simpler configuration tutorial is given at http://ait.web.psi.ch/services/linux/hpc/merlin3/sge/admin/sge_queues.html

## SGE cheat sheet

```
qstat -f -u "*"                             all user jobs
qstat -g c                               show available queue and load (total/available cores)
qstat -f                                 detailed list of machines and job state 
qstat -explain c -j <job-id>               specific job status
qdel job-id                              delete job
qsub -l h_vmem=### job.sh                mem limit, see queue_conf(5) RESOURCE LIMITS


qconf -mc		change the complex configuration (very important command!)

Command 		Description
qconf -sp pename 	Show the configuration for the specified parallel environment.
qconf -spl 		Show a list of all currently configured parallel environments.
qconf -ap pename 	Add a new parallel environment.
qconf -Ap filename 	Add a parallel environment from file filename.
qconf -mp pename 	Modify the specified parallel environment using an editor.
qconf -Mp filename 	Modify a parallel environment from file filename.
qconf -dp pename 	Delete the specified parallel environment.
qconf -sql 		list of currently defined queues
```

To view the basic cluster and host configuration use the qconf -sconf command:

```
   qconf -sconf  [global]
   qconf -sconf  host_name
```

To modify the basic cluster or host specific configuration use the following commands:

```
   qconf -mconf  global
   qconf -mconf  host_name
```

To view and change the setup of hosts use qconf -XY [file] with the following options:

```
   =================================================================
	                 HOST   execution   admin     submit  host_group 
	                   Y      e          h          s
	ACTION        X 
   ----------------------------------------------------------------- 
   add (edit)     a           *          *          *         *
   add (file)     A           *                               *
   delete         d           *          *          *         *
   modify (edit)  m           *                               *
   modify (file)  M           *                               *
   show           s           *          *          *         *
   show list      sYl         *                               *
   ================================================================
```

Great reference on SGE configuration
http://arc.liv.ac.uk/SGE/howto/sge-configs.html

## Configuration of h_vmem for any new cluster

This is extremely important to set up for any cluster, to enable to limit the amount of memory that each job can request to avoid memory issues for user jobs. The key is to make h_vmem requestable.

Use `qconf -mc` to change the line to 
```
h_vmem              h_vmem     MEMORY      <=    YES         YES        4G       0
```
So that h_vmem is consumable with default value of 4G.

Now run the example command below 20 times to confirm:
`'echo "sleep 120" | qsub -cwd -V -l hostname=compute-0-0`
If default is 4G, and if the resource is 48G at compute-0-0, then only 12 jobs can be run, and 8 jobs will be put into waiting list.

## Configuration of parallel environment

check parameters of a pe
```
[kaiwang@biocluster ~/]$ qconf -sp mpi
pe_name            mpi
slots              9999
user_lists         NONE
xuser_lists        NONE
start_proc_args    /opt/gridengine/mpi/startmpi.sh $pe_hostfile
stop_proc_args     /opt/gridengine/mpi/stopmpi.sh
allocation_rule    $fill_up
control_slaves     FALSE
job_is_first_task  TRUE
urgency_slots      min
accounting_summary FALSE
```

To add a new pe such as smp
first edit a file `pe.txt`

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

Then run `qconf -Ap pe.txt` as root. Now smp can be used as a PE in the qsub argument. 

Next thing is to add the new PE smp into the all.q queue by `qconf -mq all.q`, and add `smp` to the `pe_list` line in the file.


## SGE complex value for host

```
qconf -me <HOSTNAME>
```
Then use "complex_values   h_vmem=48G", to set 48G for a particular host as h_vmem value.

> Extremely important: whenever adding a new host, one must use "qconf -me" to set up complex_values. In combination with `-l h_vmem=xG` in the `qsub` command, this will eliminate the possibility of running out of memory when multiple jobs in the same host all request large chunks of memory at the same time.

## Temporarily enable or disable a host

```
qmod -d all.q@compute-0-4
qmod -e all.q@compute-0-4
```

`qmod -d all.q@compute-*` should disable all the queue instances


## job status

'au' simply means that Grid Engine is likely not running on the node. The "a" means 'alarm' and the "u" means unheard/unreachable. The combination of the two more often than not means that SGE is not running on the compute node.

E is a worse state to see. It means that there was a major problem on the compute node (with the system or the job itself). SGE intentionally marked the queue as state "E" so that other jobs would not run into the same bad problem.

E states do not go away automatically, even if you reboot the cluster. Once you think the cluster is fine you can use the "qmod" command to clear the E state.

##host status

- 'au' - Host is in alarm and unreachable,
- 'u' - Host is unreachable. Usually SGE is down or the machine is down. Check this out.
- 'a' - Host is in alarm. It is normal on if the state of the node is full, it means, if on the node is using most of its resources.
- 'aS' - Host is in alarm and Suspended. If the node is using most of its resources, SGE suspends this node to take any other job unless resources are available.
- 'd' - Host is disabled,
- 'E' - ERROR. This requires the command `qmod -c` to clear the error state.

## When job is in dr state

When deleting a job, sometimes a "dr" will show up, indicating that the job is not running correctly and cannot be easily deleted. In this case, log in as "su", then "qdel <jobid>" to delete the job forcefully. If it does not work, do "qdel -f <jobid>" to delete the job.

## when node is in E state

re-install the node, then clear the Error log:

```
[root@biocluster /home/kaiwang]$ qmod -c all.q@compute-0-2
root@biocluster.med.usc.edu changed state of "all.q@compute-0-2.local" (no error)
```
See more explanations here: http://www.gridengine.info/2008/01/20/understanding-queue-error-state-e/

When machine restarts yet nodes are still full, use `qstat -u "*"` to show whose jobs are in dr state, then qdel these jobs

## Check error messages from SGE

less /opt/gridengine/default/spool/qmaster/messages

## Restart SGE

You can find these in $SGE_ROOT/<cell>/common/

If your cell is the usual "default" then all you need to do is:

```
cd $SGE_ROOT/default/common/
./sgemaster start
./sgeexecd start
```

Before you restart the master, make sure you don't have any old  
sge_qmaster or sge_schedd processes hanging around.


## Fair share policy

There are two types of fair shares: share tree versus functional.
>>    1. Make 2 changes in the main SGE configuration ('qconf -mconf'):
>>           * enforce_user auto
>>           * auto_user_fshare 100
>>
>>    2. Make 1 change in the SGE scheduler configuration ('qconf -msconf'):
>>           * weight_tickets_functional 10000



## Very useful tricks

* Restart a failed job

If you job fails in a node (the node should show up as 'au' status in qstat), you can restart the job in a different node. First, alter the job to be restartable, then submit it again.

```
qalter -r y <jobid>
qmod -r <jobid>
```

You will see that the status of the job becomes "Rq", and soon it will be submitted to a different node.

* Clear error for a job

Sometimes you will see that a job is at "Eqw" state in qstat. This is due to errors in running the job, usually due to NFS error in the node in my experience. If you fixed the error, you can clear the error message by `qmod -cj <jobid>`, and the job will be submitted again.

* Change priority of a job

Use `qlater -p <priority> <jobid>` to change the priority of a job. The valid range is -1024 to 1023. Lower number means lower priority. Regular users can only lower the priority. This applies only to queued jobs, not running jobs.

## Adding a new queue

We want to add a new queue using all.q as the template:

```
[root@biocluster ~]# qconf -sq all.q > bigmem.q
```
Then edit the bigmem.q file (change `qname` to `bigmem`, change `hostlist` to `@bigmemhosts`, change `slots` to something like `1,[dragon.local=32]` where dragon.local is a host in bigmemhosts), then

```
[root@biocluster ~]# qconf -Aq bigmem.q 
```
to add this queue.

Later you can directly edit it `qconf -mq bigmem` and further change the `hostlist` and `slots` parameter there.

For example, to switch a host from the all.q to bigmem, we can do this: (1) first `qconf -mhgrp @allhosts` to remove it, then `qconf -ahgrp @bigmemhosts` to add it. Then add this hostgroup to the bigmem queue.

## default submission parameters

If you have multiple queues, it makes sense to set up a default queue, since SGE rand

Edit the `/opt/gridengine/default/common/sge_request` file, add `-q all.q` as the default queue.

From user's perspective, they can also use a default SGE specification file .sge_request in their home directory. If they do not specify the parameters in command line, these defaults from the file will be used.


## Change appliance type

Some times you may want to chagne a NAS to a job execution host. This can be done by changing appliance type.

```
[root@biocluster ~]# rocks list membership
MEMBERSHIP               APPLIANCE    DISTRIBUTION PUBLIC
Frontend:                frontend     rocks-dist   no    
Compute:                 compute      rocks-dist   yes   
NAS Appliance:           nas          rocks-dist   yes   
Ethernet Switch:         network      rocks-dist   yes   
Power Distribution Unit: power        rocks-dist   yes   
Development Appliance:   devel-server rocks-dist   yes   
Login:                   login        rocks-dist   yes   
```

shows all appliance types.

```
[root@biocluster ~]# rocks set host membership dragon membership="Compute"
```

change the appliance type. Then re-install the node, and it will show up in `qhost`.













