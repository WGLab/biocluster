# Introduction

There are also many miscellaneous Linux configuration tasks that may not apply to most Rocks users. We will describe some of them here as reference.

## Install CUDA

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


## install tensorflow

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

The make process itself should take 60-120 minutes depending on your computer's configuration. It will be an extremely boring period of time. Make sure you do not lose internet connection otherwise you'll have to start over again.

Now tensorflow works in CentOS 6!

