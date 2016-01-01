# User Account Administration



## Useful command line

Standard procedure for adding a new user
1. login by “su -”
2. useradd -c “Kai Wang” -g “col” kaiwang
3. rocks sync users
4. passwd kaiwang
5. log in, make sure that everything is fine

the current gid and uid information for all users can be accessed in the /etc/passwd file.


* Troubleshooting: once I add the new account panda, the home directory cannot be accessed, and the /etc/passwd file shows "panda:x:516:518:Pandiyan:/mnt/nas-0-0/home/panda:/bin/bash". I suspect that there is an issue with automount, so I did "service restart autofs", then do "usermod -d /home/panda panda", and then everything returns normal.

- add a group
[root@biocluster ~]# groupadd wanglab
 
- delete account
[root@biocluster ~]# userdel -r tumorim
(the -r will delete this user cleanly; otherwise mailbox and other things may still exist)
[root@biocluster ~]# rocks sync users
 
change a user group (only after first log in)
[root@biocluster ~]# usermod -g wanglab kaiwang
 
change user home directory to group home
[root@biocluster ~]# chgrp -c -R wanglab /home/tumorim
#disable and lock a user
usermod -L -e 1 guest
The -L option lock user's password by putting a ! in from of the the encrypted password. To disable user account set expire date to 1.
Use chage command to see current status of the user account:
# chage -l guest
Use below to re-enable the user account
[root@biocluster /home/kaiwang]$ chage -E -1 shuangchaoma

UID and GID
To find a user's UID or GID in Unix, use the id command. To find a specific user's UID, at the Unix prompt, enter:
id -u username
 
Replace username with the appropriate user's username. To find a user's GID, at the Unix prompt, enter:
id -g username
 
If you wish to find out all the groups a user belongs to, instead enter:
id -G username
 
If you wish to see the UID and all groups associated with a user, enter id without any options, as follows:
id username 
list all groups in a system
[root@biocluster ~]# cat /etc/group |cut -d: -f1
[root@biocluster ~]# cat /etc/passwd | cut -d: -f1
Kill all user process
 echo `ps -fu $User | awk 'NR != 1 {print $2}'`

Once you're sure that you're eching the right stuff, replace the echo with kill.


