# Introduction

There are also many miscellaneous Linux configuration tasks that may not apply to most Rocks users. We will describe some of them here as reference.

## hardware information

When you get access to a new computer server, you may want to run some commands to figure out the hardware configurations. Below is a number of commands that you can use:

- dmidecode ( lists hardware model, serial number, cpu, memory. This command was handy when i needed to find serial number without a visit to data center)
- lshal (also list hardware model and so on)
- lspci ( lists pci device)
- lsusb ( lists usb device)
- lsscsi ( lists scsi device)
- systool
- fdisk -l ( lists hard drive)
- cat /proc/cpuinfo ( more on cpu)
- cat /proc/meminfo ( more on memory)
- dmesg | egrep ‘(SCSI|scsi0|ide0|hda|sda|serio|mice|eth0|eth1)’



## Install CUDA in Rocks 6 (CentOS 6)

Update: it is much more straightforward in Rocks 7 to use CUDA and tensorflow and I suggest that you update to Rocks 7 if you use a lot of deep learning.

For machines equipped with NVIDIA GPU, you may want to install CUDA for GPU computing. Below is a general procedure to do that in a Rocks 6.2 system (with Centos 6).

- install freeglut
```
[root@phoenix ~]# yum --enablerepo epel install libGL-devel
[root@phoenix ~]# yum --enablerepo epel install libGLU-devel
[root@phoenix ~]# wget ftp://rpmfind.net/linux/centos/6.7/os/x86_64/Packages/freeglut-2.6.0-1.el6.x86_64.rpm
[root@phoenix ~]# wget ftp://rpmfind.net/linux/centos/6.7/os/x86_64/Packages/freeglut-devel-2.6.0-1.el6.x86_64.rpm
[root@phoenix ~]# rpm -ivh freeglut-2.6.0-1.el6.x86_64.rpm 
[root@phoenix ~]# rpm -ivh freeglut-devel-2.6.0-1.el6.x86_64.rpm 
```

- install prerequisite for CUDA
```
[root@phoenix ~]# yum --enablerepo epel install libvdpau
[root@phoenix ~]# yum --enablerepo epel install dkms
```

- download CUDA from https://developer.nvidia.com/cuda-downloads
```
[root@phoenix ~]# rpm -i cuda-repo-rhel6-7-5-local-7.5-18.x86_64.rpm
[root@phoenix ~]# yum clean all
[root@phoenix ~]# yum install cuda
```

- download cudaCNN from https://developer.nvidia.com/cudnn (registartion required)
```
[root@phoenix ~]# tar xvfz cudnn-7.0-linux-x64-v4.0-prod.tgz 
[root@phoenix ~]# cp cuda/include/cudnn.h /usr/local/cuda/include/
[root@phoenix ~]# cp cuda/lib64/libcudnn* /usr/local/cuda/lib64/
```


## install tensorflow in Rocks 6 (CentOS 6)

Tensorflow is useful for deep learning, and in our experience, the user interface and usability is far better than Torch or Theano. The default instruction does not work since Centos 6 does not support glibc2.15, which is required by tensorflow. (glibc2.15 rpm is only available in Centos 7). See below for the typical error message.

```
[kaiwang@phoenix ~]$ python
Python 2.7.9 (default, May  7 2015, 17:54:07) 
[GCC 4.4.7 20120313 (Red Hat 4.4.7-11)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/__init__.py", line 23, in <module>
    from tensorflow.python import *
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/__init__.py", line 45, in <module>
    from tensorflow.python import pywrap_tensorflow
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/pywrap_tensorflow.py", line 28, in <module>
    _pywrap_tensorflow = swig_import_helper()
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/pywrap_tensorflow.py", line 24, in swig_import_helper
    _mod = imp.load_module('_pywrap_tensorflow', fp, pathname, description)
ImportError: /lib64/libc.so.6: version `GLIBC_2.15' not found (required by /home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/_pywrap_tensorflow.so)
>>> 
```

There are two potential solutions. One is to rebuild tensorflow. However, tensorflow uses the Bazel builder developed by Google, which in turn requires Java 8 and high versions of GCC, which the system does not have. The other solution is to rebuild GLIBC and GCC.

So to address this, we should rebuild glibc (for C library) and gcc (for C++ library) from scratch. However, after I install glibc2.15, tensorflow begins to complain "version `GLIBC_2.17` not found". So it seems that the minimal version of glibc should have been 2.17 for tensorflow to work.

```
wget http://ftp.gnu.org/gnu/glibc/glibc-2.17.0.tar.gz
tar -xvzf glibc-2.17.0.tar.gz
cd glibc-2.17.0
mkdir glibc-build
cd glibc-build
../configure --prefix='/usr'
```

There is a bug in the make file though, so that make will not work correctly. To fix this, , edit the `../scripts/test-installation.pl` file, and change line 179 to `if (/\Q$ld_so_name\E/)`. Then type `make` and `make install`. Now glibc is installed correctly. Please note that the prefix should be `/usr` not `/usr/local` otherwise the system will be unusable for whatever reason that I do not understand.

Next we run tensorflow and the new message pops up:

```
[kaiwang@phoenix ~]$ python
Python 2.7.9 (default, May  7 2015, 17:54:07) 
[GCC 4.4.7 20120313 (Red Hat 4.4.7-11)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/__init__.py", line 23, in <module>
    from tensorflow.python import *
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/__init__.py", line 45, in <module>
    from tensorflow.python import pywrap_tensorflow
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/pywrap_tensorflow.py", line 28, in <module>
    _pywrap_tensorflow = swig_import_helper()
  File "/home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/pywrap_tensorflow.py", line 24, in swig_import_helper
    _mod = imp.load_module('_pywrap_tensorflow', fp, pathname, description)
ImportError: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.19' not found (required by /home/kaiwang/.local/lib/python2.7/site-packages/tensorflow/python/_pywrap_tensorflow.so)
>>> 
```

Let's check this ourselves:

```
[kaiwang@phoenix ~]$ strings /usr/lib64/libstdc++.so.6 | fgrep GLIBC
GLIBCXX_3.4
GLIBCXX_3.4.1
GLIBCXX_3.4.2
GLIBCXX_3.4.3
GLIBCXX_3.4.4
GLIBCXX_3.4.5
GLIBCXX_3.4.6
GLIBCXX_3.4.7
GLIBCXX_3.4.8
GLIBCXX_3.4.9
GLIBCXX_3.4.10
GLIBCXX_3.4.11
GLIBCXX_3.4.12
GLIBCXX_3.4.13
GLIBC_2.2.5
GLIBC_2.3
GLIBC_2.14
GLIBC_2.3.2
GLIBCXX_FORCE_NEW
GLIBCXX_DEBUG_MESSAGE_LENGTH
```

So we also need to reinstall C++ from GCC. 

First, need to `yum install glibc-devel.i686`, because it fixed the "Linux gnu/stubs-32.h: No such file or directory" error during gcc compilation. Apparently GCC compilation wants both 32 and 64 bit libraries.

Then download 4.9.3. Do NOT download 4.6.x  because it only supports GLIBCXX 3.4.16 yet we need 3.4.19.

```
wget http://ftp.gnu.org/gnu/gcc/gcc-4.9.3/gcc-4.9.3.tar.gz
tar -xvzf gcc-4.9.3.tar.gz
cd gcc-4.9.3
./contrib/download_prerequisites
mkdir build
cd build
../configure --prefix=/opt/gcc-4.9.3
make
make install
mv /usr/lib64/libstdc++.so.6 /usr/lib64/libstdc++.so.6.bak
cp /opt/gcc-4.9.3/lib64/libstdc++.so.6 /usr/lib64/libstdc++.so.6
cp /opt/gcc-4.9.3/lib64/libstdc++.so.6.0.20 /usr/lib64/libstdc++.so.6.0.20
```

The make process itself should take 60-120 minutes depending on your computer's configuration. It will be an extremely boring period of time; but it is also a good time for you to take a rest, look at the computer screen and start thinking about the true meaning of life. Make sure you do not lose internet connection otherwise you'll have to start over again.

In the end, make sure that it is compiled correctly:

```
[kaiwang@phoenix ~/]$ gcc -v -x c -E -
Using built-in specs.
COLLECT_GCC=gcc
Target: x86_64-unknown-linux-gnu
Configured with: ../configure --prefix=/opt/gcc-4.9.3
Thread model: posix
gcc version 4.9.3 (GCC) 
COLLECT_GCC_OPTIONS='-v' '-E' '-mtune=generic' '-march=x86-64'
 /opt/gcc-4.9.3/libexec/gcc/x86_64-unknown-linux-gnu/4.9.3/cc1 -E -quiet -v - -mtune=generic -march=x86-64
ignoring nonexistent directory "/opt/gcc-4.9.3/lib/gcc/x86_64-unknown-linux-gnu/4.9.3/../../../../x86_64-unknown-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /opt/gcc-4.9.3/lib/gcc/x86_64-unknown-linux-gnu/4.9.3/include
 /usr/local/include
 /opt/gcc-4.9.3/include
 /opt/gcc-4.9.3/lib/gcc/x86_64-unknown-linux-gnu/4.9.3/include-fixed
 /usr/include
End of search list.
```

Now tensorflow works in CentOS 6!

# Improved approach to install GCC and G++

It turns out that gcc 4.9.2 is indeed available from [SCL](https://wiki.centos.org/AdditionalResources/Repositories/SCL) for Centos 6! So technically we can just use yum to install it. The procedure is described below and tested by myself.

First, run the following commands to add SCL to repo list, and then install gcc and g++:

```
yum install centos-release-scl-rh
yum install devtoolset-3-gcc devtoolset-3-gcc-c++
```

Next, you have to activate the scl shell before you can use the latest gcc/g++:

```
scl enable devtoolset-3 bash
```

Now run `gcc --version` and you can see that the version is 4.9.2 now.

The obvious drawback of this approach is that you will always have to activate the scl shell first.

# continue tackle CentOS 6

For CentOS 6 users (including but not limited to Rocks users), we need to deal with a lot of issues related to installing higher version of specific software tools. The above sections give some examples, and below I will describe more examples on installing some of the most widely used packages for Linux.

1. php

    It used to be the case that only PHP4 is available in CentOS 6, but recently php5 is available from epel, so that one can directly use yum to install them (from the epel repository). I consider this a solved problem. However, when combined with httpd, it will no longer work (see more explanations below).
    
2. httpd

    The latest on CentOS 6 will be 2.2.15. This will cause a lot of practical problems, because a lot of people (so-called security experts) believe that there are vulnerability with apache and will force you to shut down your server unless you update to 2.4. I followed instructions [here](http://unix.stackexchange.com/questions/138899/centos-install-using-yum-apache-2-4) and was able to install httpd 2.4 successfully. The key challenge/complication here is that one must re-write all the configuration files, since we are not technically updating a httpd package, but installing a new package called http24.
    
    In short, these are the steps:

```
cd /etc/yum.repos.d/
wget http://repos.fedorapeople.org/repos/jkaluza/httpd24/epel-httpd24.repo
yum install httpd24
/opt/rh/httpd24/root/usr/sbin/httpd -version
yum install php-fpm
/etc/init.d/php-fpm start
```

Now need to make sure that httpd24 and php-fpm automatically run when the system starts.


```
chkconfig --level 345 httpd off
chkconfig --level 345 httpd24-httpd on
chkconfig --level 345 httpd24-htcacheclean on
```

The problem is that php web pages no longer works (i.e., PHP scripts no longer executes). The reason is that in httpd 2.2, I have `php.conf` in the `/etc/httpd/conf.d` directory that automatically loads the modules/libphp5.so module. However, this module is not available in httpd24. This is why we want to install php-fpm in the above steps.

In general, we can just follow the instructions for [php-fpm](https://wiki.apache.org/httpd/PHP-FPM) to set up handler of php scripts. Essentially php-fpm works on 127.0.0.1 port 9000 and listen to requests and execute php as if it is a cgi script. I prefer the ProxyPassMatch method below. This statement needs to be added into every conf file for every virtual server that uses php. Unfortunately I do not see a way to add it to the main httpd.conf file so that it applies to every virtual server in the website.

```
ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://127.0.0.1:9000/path/to/your/documentroot/$1
DirectoryIndex /index.php index.php index.html index.htm
````


# install google chrome

The procedure below has been tested on Rocks 7 and it works well.

First, edit `/etc/yum.repos.d/google-chrome.repo` and add following lines:

```
[google-chrome]
name=google-chrome
baseurl=http://dl.google.com/linux/chrome/rpm/stable/$basearch
enabled=1
gpgcheck=1
gpgkey=https://dl-ssl.google.com/linux/linux_signing_key.pub
```

Then run `yum info google-chrome-stable` to make sure that the repo can be found. Then do

```
[root@biocluster nodes]# yum --enablerepo=epel install libappindicator-gtk3
[root@biocluster nodes]# yum install google-chrome-stable
```

Type `google-chrome` to launch the browser.

# install MongoDB in Rocks 7 (CentOS 7) to work with PHP

The procedure below describes my journey to set up PHP-based web server under Rocks 7 that uses MongoDB. It is not straightforward and I hope that I remember everything that I have done, but I may have missed something below.

First, install MongoDB. Just follow standard instructions at https://docs.mongodb.com/v3.2/tutorial/install-mongodb-on-red-hat/. Remember CentoOS is based on RedHat, so the same procedure works. Note that by default "SELINUX=disabled" in Rocks, but it may not be the case in typical CentOS. Make sure to run `service mongod start` to start the service. Unlike previous versions of CentOS, the `chkconfig` command only works for SysV services now, and you need to use `systemctl` for other services. But anyway, if you just run `chkconfig mongod on`, it will automatically re-direct the command to `systemctl enable mongod.service` to make sure that mongod service runs when the computer restarts. 

Next, install PHP driver for MongoDB. If you google, you will see tons of articles about how to do this. But unfortunately I tried to follow them and cannot get things work. For example, in https://github.com/mongodb/mongo-php-library, they mentioned that it is simply a `pecl install mongodb` command that would work. However, this does not work for a few reasons: (1) pecl is not even available in rocks (2) even if it is available, it shows "pecl/mongodb requires PHP (version >= 5.5.0)" message yet Rocks 7 only has PHP 5.4. Other sites suggested doing `yum install php-pecl-mongo` which does not work for me since this package is not found anywhere. Official PHP site (http://php.net/manual/en/mongodb.installation.pecl.php) also suggested similar method but again it does not work in Rocks 7.

Finally, I realized that I need to install mongo, not mongodb, based on https://secure.php.net/manual/en/mongo.installation.php. After PHP 5.6, the developers decided to change the name from mongo to mongodb, which is why when you google, you only find information about mongodb instead of mongo. This is incredibly confusing.

The way I solved this problem is to install mongo, rather than mongodb. First, install epel and remi repository (`yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm`), since many things require these two repositories. Then, `yum --enablerepo=remi install php-pear`, which basically installs the pecl command into the system. Then, `yum --enablerepo=remi install php-devel` and `pecl install mongo`.

Then add the two `[MongoDB] extension=mongo.so` lines to `/etc/php.ini`. Again, note that we are not doing `extension=mongodb.so`, but `extension=mongo.so`.

Finally, do a `service httpd restart`. Then check a random page that has `<?php phpinfo() ?>` in it, to see if mongo is now loaded to PHP.


























