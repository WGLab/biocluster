# Overview

Once installation of the cluster is done, you typically want to make some customization to the cluster. This section describes several common tasks.

## Install R

R is a statistical computing language that is commonly used in bioinformatics. However, Rocks package repository does not include R, so you need to install it manually using [EPEL](https://fedoraproject.org/wiki/EPEL), which stands for Extra Package for Enterprise Linux. The detailed procedure is described below.

1. log in head node as root.
2. Install EPEL if you have not done so: `rpm -Uvh https://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm`. You may need to change the version number, depending on your Rocks/CentOS version.
3. Install R by `yum install R`. Note that yum should automatically search EPEL for R packages, but if not, you can manually force yum to do that by `yum --enablerepo=epel install R`.
4. Go to `/usr/cache/yum/epel/packages` to know what packages are just installed. Typically, 5-10 packages are installed to have full functionality of R in the system.
5. Now copy all these \*.rpm files to `/export/rocks/install/contrib/6.1.1/x86_64/RPMS` (change the path to suite your own system!). 
6. Now go to `/export/rocks/install/site_profile/6.1.1/nodes`, edit `extend-compute.xml`, and add in the name of the packages (see an example below).
7. This is totally optional and I do not actually use it. If you want to use graphical interface in compute node (which does not make sense to me), since X11 is required, use `rocks add host attr compute-0-0 X11` to enable this.
8. Then `cd /export/rocks/install`, type `rocks create distro`, then re-install all compute nodes. (Hint: you can do `rocks set host boot compute action=install` so all compute nodes re-install themselves after rebooting).

> Example: this is what I need to add in my `extend-compute.xml` file:
```
<package>R</package>
<package>R-core</package>
<package>R-core-devel</package>
<package>R-devel</package>
<package>R-java</package>
<package>R-java-devel</package>
<package>blas</package>
<package>blas-devel</package>
<package>ecj</package>
<package>gcc-java</package>
<package>lapack</package>
<package>lapack-devel</package>
<package>libgcj-devel</package>
<package>libgcj-src</package>
<package>libicu</package>
<package>libicu-devel</package>
<package>texinfo</package>
<package>texinfo-tex</package>
<package>xz-devel</package>
<package>cmake</package>
<package>bzip2-devel</package>
<package>gcc-c++</package>
<package>libart_lgpl</package>
<package>libgcj</package>
<package>libstdc++-devel</package>
```


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

or you can manually update: 

```
date --set "5 Aug 2012 12:54 IST"
```

Obviously in the above command you must set your date, time and time zone correctly.

Now as root, synchronize the hardware clock to the current system time as local time.

```
hwclock --systohc --localtime
```

Now the hardware clock is re-adjusted to the system time and both now point to the local time.

