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

-or- 

Manual update: 

```
date --set "5 Aug 2012 12:54 IST"
```

Obviously in the above command you must set your date, time and time zone correctly.

Now as root, synchronize the hardware clock to the current system time as local time.

```
hwclock --systohc --localtime
```

Now the hardware clock is re-adjusted to the system time and both now point to the local time.

