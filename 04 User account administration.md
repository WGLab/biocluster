## Introduction

Rocks is basically a set of Python programs that controlls and facilitates cluster management. For user account management, once you make changes in the head node, there is a specific command `rocks sync users` that propagates all user information across all compute nodes. In recent versions of Rocks, now host-based authentification is used, so that any user can log into a compute node without typing password (previously, SSH key-based authentification is used, but occasionally it has problems for various reasons).

## Setting up home directory

For the setup of my cluster, the only place that I need to change Rocks source code is for user account administration. The reason is that I wanted to use a common storage server called nas-0-0, rather than head node, to store all user files, yet Rocks does not have an easy way to do this, unless I change source code. Fortunately, it is written as Python scripts, so changing them is straightforward.

In `/opt/rocks/lib/python2.6/site-packages/rocks/commands/sync/users/plugin_fixnewusers.py`, just change “default_dir” to “/mnt/nas-0-0/home”. That's the only necessary change to set up user home directory correctly.

Under the scene, this is what happened:

1. when typing “useradd kaiwang5”, the /etc/password is changed and a new directory /mnt/nas-0-0/kaiwang5 is created (depending on setting in /etc/default/useradd). It is important to note that /mnt/nas-0-0 is automounted by autofs, and in fact, the only time it is needed is when people use `useradd`.
2. next when typing “rocks sync users”, the /etc/password file is changed from /mnt/nas-0-0/kaiwang5 to /home/kaiwang5, and the /etc/auto.home is added (the parameter are determined by `Info_HomeDirSrv` and `Info_HomeDirLoc` system parameters, or if not set, determined by the  /opt/rocks/lib/python2.6/site-packages/rocks/commands/sync/users/plugin_fixnewusers.py command).
3. this is the reason to modify  /opt/rocks/lib/python2.6/site-packages/rocks/commands/sync/users/plugin_fixnewusers.py, change “default_dir”.
4. we should automount each user, not the whole `/home`. Therefore, autofs is used.
5. when I cannot umount a NFS directory, use “umount -l” will always work. When I cannot stop a service such as autofs, just do “ps aux | fgrep auto” and “kill -9” the process.
6. rocks list attr will print out all Rocks variables.

## Standard procedure for adding a new user

Standard procedure for adding a new user
1. login by “su -”
2. useradd -c “Kai Wang” -g “col” kaiwang
3. rocks sync users
4. passwd kaiwang
5. log in, make sure that everything is fine

the current gid and uid information for all users can be accessed in the /etc/passwd file.

> Troubleshooting: once I add the new account panda, the home directory cannot be accessed, and the /etc/passwd file shows "panda:x:516:518:Pandiyan:/mnt/nas-0-0/home/panda:/bin/bash". I suspect that there is an issue with automount, so I did "service restart autofs", then do "usermod -d /home/panda panda", and then everything returns normal.

## Useful commands for account changes

- add a group
```
[root@biocluster ~]# groupadd wanglab
```

- delete account
```
[root@biocluster ~]# userdel -r tumorim
```

Note that the -r will delete this user cleanly; otherwise mailbox and other things may still exist)
```
[root@biocluster ~]# rocks sync users
```
 
- change a user group (only after first log in)

`usermod -g wanglab kaiwang`
 
- change user home directory to group home

`chgrp -c -R wanglab /home/tumorim`

## disable and lock a user

```
usermod -L -e 1 guest
```

The -L option lock user's password by putting a ! in from of the the encrypted password. To disable user account set expire date to 1.

Use chage command to see current status of the user account: `chage -l guest`

Use below to re-enable the user account: `chage -E -1 shuangchaoma`

### UID and GID

To find a user's UID or GID in Unix, use the id command. To find a specific user's UID, at the Unix prompt, enter: `id -u username`
 
Replace username with the appropriate user's username. To find a user's GID, at the Unix prompt, enter: `id -g username`
 
If you wish to find out all the groups a user belongs to, instead enter: `id -G username`
 
If you wish to see the UID and all groups associated with a user, enter id without any options, as follows: `id username`

### Kill all user process

```
 echo `ps -fu $User | awk 'NR != 1 {print $2}'`
```

Once you're sure that you're eching the right stuff, replace the echo with kill.

Alternatively, just do a `killall -u <user>`.
