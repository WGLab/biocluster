## Introduction

There are also many miscellaneous Linux configuration tasks that applies to any CentOS system and are not necessarily Rocks specific. We will describe some of them here as reference.

## Install CUDA

For machines equipped with NVIDIA GPU, you may want to install CUDA for GPU computing. Below is a general procedure to do that in a Rocks 6.2 system (with Centos 6).

#install freeglut
[root@phoenix ~]# yum --enablerepo epel install libGL-devel
[root@phoenix ~]# yum --enablerepo epel install libGLU-devel
[root@phoenix ~]# wget ftp://rpmfind.net/linux/centos/6.7/os/x86_64/Packages/freeglut-2.6.0-1.el6.x86_64.rpm
[root@phoenix ~]# wget ftp://rpmfind.net/linux/centos/6.7/os/x86_64/Packages/freeglut-devel-2.6.0-1.el6.x86_64.rpm
[root@phoenix ~]# rpm -ivh freeglut-2.6.0-1.el6.x86_64.rpm 
[root@phoenix ~]# rpm -ivh freeglut-devel-2.6.0-1.el6.x86_64.rpm 

#install prerequisite for CUDA
[root@phoenix ~]# yum --enablerepo epel install libvdpau
[root@phoenix ~]# yum --enablerepo epel install dkms

#download CUDA from https://developer.nvidia.com/cuda-downloads
[root@phoenix ~]# rpm -i cuda-repo-rhel6-7-5-local-7.5-18.x86_64.rpm
[root@phoenix ~]# yum clean all
[root@phoenix ~]# yum install cuda


#download cudaCNN (registartion required)
[root@phoenix ~]# tar xvfz cudnn-7.0-linux-x64-v4.0-prod.tgz 
[root@phoenix ~]# cp cuda/include/cudnn.h /usr/local/cuda/include/
[root@phoenix ~]# cp cuda/lib64/libcudnn* /usr/local/cuda/lib64/



## install tensorflow

default isntruction does not work since Centos 6 does not support glibc2.15, which is required by tensorflow. (glibc2.15 is only supported in Centos 7, and it is almost impossible for even an intermediate Linux user to build glibc).

If you must use tensorflow, you have to install CentOS 7. There is no workaround that I can succeed in using.




