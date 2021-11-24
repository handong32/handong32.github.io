# Building Custom Linux Appliances

## Motivation
Linux appliances are a relatively old idea (https://www.usenix.org/legacy/publications/library/proceedings/lisa2000/full_papers/shaffer/shaffer_html/index.html), often understood as a self-contained system image containing *just enough* software and operating systems support to run a single application. Typically, they are built by system administrators to reduce system maintenance and improve ease of deployment. However, these appliances are equally useful for anyone interested in benchmarking the Linux kernel while ensuring experimental stability with minimal system noise. 

As recently discussed in https://www.usenix.org/publications/loginonline/musings-operating-systems-research, almost all modern OS research is conducted on Linux; which typically implies a Linux flavor such as Ubuntu, Fedora, etc. However, it is also not always clear how much effort researchers have taken to remove as many extraneous processes and kernel modules as possible to ensure a clean and stable Linux environment. For example, my Asus Ultrabook running Ubuntu 20.04 with Linux-5.11.0 idles with **231 processes** and **123 kernel modules** loaded. On the same laptop which I got it to run a custom built Linux-5.5.0 kernel and a custom initramfs; idles with **80 processes** and **0 kernel modules** loaded.

Furthermore in a recent paper published in SOSP'19: "An Analysis of Performance Evolution of Linux’s Core Operations", the authors demonstrated how Linux's configuration settings can have significant performance impacts on an application. This also motivates the overall premise of how much of Linux's default configuration does a kernel really need to be *functional* and how much complexity can thus be stripped out to run a minimum set of binaries? For example, following suggestions in this paper, I was able to build and boot a Linux-5.4.0 kernel with a custom config file that is less than 50% the length of a default Linux configuration.

Beyond these, another side benefit of a custom Linux appliance is that the root filesystem is loaded as a RAM disk by default; which is another positive for researchers who want to minimize system noise from disk paging as all executables are now run out from memory. Obviously, other considerations are needed should the experiments require data from a external storage device.

## Goals
There are many tutorials online to build your own Linux kernel and initramfs, however, their usecases are typically too general and the steps involved can be quite complex. This tutorial will demonstrate that it is infact surprsingly easy to get a barebones Linux up and running that is ready to execute some simple binaries. The tutorial will cover the specific use-cases of how to build Linux appliances, how to boot them, and how this can help enable experimental automation, with a focus on performance and system stability. 

Concretely, here are the following steps in this tutorial towards this goal:

1. Steps to create an initramfs
2. Build a basic Linux kernel
3. How to get simple binaries to run
4. How to boot appliance via GRUB
5. How to get device drivers working
6. Modiyfing /init for experiments
7. Booting via PXEBOOT protocol in a local network

## Preparation
Before we embark on this journey, here's a general breakdown of what is required on your end. First, you should have a testing computer that you will be booting the Linux appliance on and you should have an existing Linux flavor already installed on the computer, this can be Ubuntu, Fedora, etc. Having a pre-existing OS on your testing machine is important because you want to make sure you are in a state where the binary you intend to run is infact runnable and it can also be used to install additional packages that you may need. The implication here is also that the kernel used for the Linux appliance should be simiarly versioned as the existing OS to ensure system library compatibility, details of this will be covered in the Linux kernel section below.

## Steps to create an initramfs
The steps below are mainly draw from the work done by https://www.linuxfromscratch.org/lfs/view/stable/. We will be skipping all the steps to get the cross compilers and systems packages installed and running, instead, this tutorial demonstrates how binaries can be made runnable simply by copying over needed system libraries from the already installed OS. 

### 1. Setup working environment
Open up a terminal and log in to the `root` user by running `sudo -s`. Next, export the name for the initramfs for where you will be creating the directory by running `export LFS=~/initfs`. Check that it has been setup correctly by running `echo $LFS`. You might also want to put the `export LFS=~/initfs` command into the `.bashrc` file of the `root` user so that it gets set automatically.

### 2. Create directory structure for `$LFS`

These directories are typically where default system libraries are placed, details are found at [here](https://www.linuxfromscratch.org/lfs/view/stable/chapter04/creatingminlayout.html).
```
mkdir -pv $LFS
mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}

for i in bin lib sbin; do
  ln -sv usr/$i $LFS/$i
done

case $(uname -m) in
  x86_64) mkdir -pv $LFS/lib64 ;;
esac
```

Setup virtual kernel filesystems -- which are used to typically communicate with the kernel, details are found [here](https://www.linuxfromscratch.org/lfs/view/stable/chapter07/kernfs.html). **Note:** Remember to umount.
```
mkdir -pv $LFS/{dev,proc,sys,run}

mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3

mount -v --bind /dev $LFS/dev

mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```

Create the rest of the directories needed, details [here](https://www.linuxfromscratch.org/lfs/view/stable-systemd/chapter07/creatingdirs.html).
```
mkdir -pv $LFS/{boot,home,mnt,opt,srv}

mkdir -pv $LFS/etc/{opt,sysconfig}
mkdir -pv $LFS/lib/firmware
mkdir -pv $LFS/media/{floppy,cdrom}
mkdir -pv $LFS/usr/{,local/}{include,src}
mkdir -pv $LFS/usr/local/{bin,lib,sbin}
mkdir -pv $LFS/usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -pv $LFS/usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -pv $LFS/usr/{,local/}share/man/man{1..8}
mkdir -pv $LFS/var/{cache,local,log,mail,opt,spool}
mkdir -pv $LFS/var/lib/{color,misc,locate}

ln -sfv $LFS/run $LFS/var/run
ln -sfv $LFS/run/lock $LFS/var/lock

install -dv -m 0750 $LFS/root
install -dv -m 1777 $LFS/tmp $LFS/var/tmp

ln -sv $LFS/proc/self/mounts $LFS/etc/mtab

touch $LFS/var/log/{btmp,lastlog,faillog,wtmp}
chgrp -v utmp $LFS/var/log/lastlog
chmod -v 664  $LFS/var/log/lastlog
chmod -v 600  $LFS/var/log/btmp
```

Set up root user and groups, details [here](https://www.linuxfromscratch.org/lfs/view/stable-systemd/chapter07/createfiles.html).
```
cat > $LFS/etc/hosts << EOF
127.0.0.1  localhost lfs
::1        localhost
EOF

cat > $LFS/etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/bin/false
systemd-bus-proxy:x:72:72:systemd Bus Proxy:/:/bin/false
systemd-journal-gateway:x:73:73:systemd Journal Gateway:/:/bin/false
systemd-journal-remote:x:74:74:systemd Journal Remote:/:/bin/false
systemd-journal-upload:x:75:75:systemd Journal Upload:/:/bin/false
systemd-network:x:76:76:systemd Network Management:/:/bin/false
systemd-resolve:x:77:77:systemd Resolver:/:/bin/false
systemd-timesync:x:78:78:systemd Time Synchronization:/:/bin/false
systemd-coredump:x:79:79:systemd Core Dumper:/:/bin/false
uuidd:x:80:80:UUID Generation Daemon User:/dev/null:/bin/false
systemd-oom:x:81:81:systemd Out Of Memory Daemon:/:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
EOF

cat > $LFS/etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
systemd-journal:x:23:
input:x:24:
mail:x:34:
kvm:x:61:
systemd-bus-proxy:x:72:
systemd-journal-gateway:x:73:
systemd-journal-remote:x:74:
systemd-journal-upload:x:75:
systemd-network:x:76:
systemd-resolve:x:77:
systemd-timesync:x:78:
systemd-coredump:x:79:
uuidd:x:80:
systemd-oom:x:81:81:
wheel:x:97:
nogroup:x:99:
users:x:999:
EOF
```

## Getting binaries to run

After you've created the directory structure above, the `chroot` program can then be used to test out `$LFS` filesystem. However, as the filesystem itself is *bare* without any binaries in it, we see the following error when running `chroot`:
```
[root@ ~]# chroot $LFS
chroot: failed to run command ‘/bin/bash’: No such file or directory
```

### 1. Getting `/bin/bash` to run 

To get `chroot` working, we need to copy over the `/bin/bash` binary to `$LFS/bin/`:
```
## figure out where binary lives
[root@ ~]# which bash
/usr/bin/bash

## copy bash to correct directory in $LFS
ubuntu:~$ cp /usr/bin/bash $LFS/usr/bin/
```

Now we figure out the library dependencies and copy them over as well:
```
## ldd can tell us the system libraries that bash depends upon
[root@ ~]# ldd /usr/bin/bash
	linux-vdso.so.1 (0x00007ffd6ffa6000)
	libtinfo.so.6 => /lib64/libtinfo.so.6 (0x00007f8e3751b000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f8e37514000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f8e37345000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f8e376bc000)

## Basically, we have to make sure we copy the correct libraries to the correct location in the new initramfs
## We can ignore linux-vdso.so.1 as the kernel provides it automatically: https://man7.org/linux/man-pages/man7/vdso.7.html.
[root@ ~]# cp /lib64/libtinfo.so.6 $LFS/lib64/
[root@ ~]# cp /lib64/libdl.so.2 $LFS/lib64/
[root@ ~]# cp /lib64/libc.so.6 $LFS/lib64/
[root@ ~]# cp /lib64/ld-linux-x86-64.so.2 $LFS/lib64/

```
After the steps above, running `chroot $LFS`, you should be greeted with a bash prompt. 
```
[root@ ~]# chroot $LFS
bash-5.1# 
```

### 2. Getting other programs to run
However, in order to run other programs we will also need to follow the same steps above. For that purpose, below is a simple script `copy_bin_libs` to automate the entire process. Example usage: `MYINIT=$LFS BINS="ls echo env nproc ln mkdir mknod mount pwd grep umount cat head tail wc ps ip lsblk lspci lsmod poweroff reboot df du" ./copy_bin_libs`

`copy_bin_libs`:
```
#!/bin/bash

export MYINIT=${MYINIT:=''}
export BINS=${BINS:=''}

## all we do is copy the files into the proper directory in the initrd
for bins in ${BINS}; do

    echo $bins
    ## figure out where binary lives
    bins_loc=$(which $bins)
    bins_dir_loc=$(which $bins | xargs -I '{}' dirname '{}')    

    echo $bins_loc
    echo $bins_dir_loc
    
    ## set up directory tree and copy bins here
    mkdir -p "${MYINIT}/${bins_dir_loc}"
    cp $bins_loc "${MYINIT}/${bins_dir_loc}"
    
    ## figure out library dependencies of $bins
    libs_loc=$(ldd $bins_loc | grep "=> /" | awk '{print $3}')

    ## set up directory tree and copy libs here
    for libs in ${libs_loc}; do
	libs_dir=$(dirname $libs)
	
	echo $libs
	echo $libs_dir
	
	mkdir -p "${MYINIT}/${libs_dir}"
	cp $libs "${MYINIT}/${libs_dir}"
    done
    echo "+++++++++++++"
done
```

## Create startup script for Linux
After following the previous steps to ensure a basic set of programs are runnable in your appliance, the next step is to create the startup file that is essentially `PID=1`, the program that Linux runs to initiate the rest of the system. Modern systems have typically migrated to use [systemd](https://systemd.io/) as the bootstrapping program due to its comprehensive set of tools. However, we will instead use the older [`init`](https://en.wikipedia.org/wiki/Init) script as it is simpler to edit and enables greater control to automate experiments. Run the steps below to create a simple `init` file that sets up a standard Linux filesystem on the booted system:

```
cat > $LFS/init <<EOF
#!/bin/bash

set -x

export HOME=/root
export LOGNAME=root
export TERM=vt100
export PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin
export ENV="HOME=\$HOME LOGNAME=\$LOGNAME TERM=\$TERM PATH=\$PATH"

# setup standard file system view
mount -t proc /proc /proc
mount -t sysfs /sys /sys

# take care of /dev 1) try using devtmpfs and devpts or 2) dev of exiting fs and devpts
if ! mount -t devtmpfs -o mode=0755 udev /dev; then
  # failed to mount devtmpfs
  # so add a few useful things if not already in file system
  echo "W: devtmpfs not available, falling back to tmpfs for /dev"
  [[ -e /dev/console ]] || mknod -m 0600 /dev/console c 5 1
  [[ -e /dev/null ]] || mknod /dev/null c 1 3
  [[ -e /dev/zero ]] || mknod /dev/zero c 1 5
  [[ -e /dev/random ]] || mknod /dev/random c 1 8
  [[ -e /dev/urandom ]] || mknod /dev/urandom c 1 9
fi 

[[ -e /dev/pts ]] || mkdir /dev/pts

mount -t devpts devpts /dev/pts

# Some things don't work properly without /etc/mtab.
ln -sf /proc/mounts /etc/mtab

# if we get here then we might as well start a shell :-) 
/bin/bash

# if bash fails, shuts off machine
poweroff -f
EOF
```

Set permissions: `chmod 755 $LFS/init`. Next, `cd $LFS` to be inside the `$LFS` directory and compress the initramfs into a cpio format by running `find . | cpio -o -H newc > ../myinitfs.cpio`


## External Resources
* https://www.linuxjournal.com/content/diy-build-custom-minimal-linux-distribution-source
* https://wiki.debian.org/initramfs/
* https://wiki.gentoo.org/wiki/Custom_Initramfs
* https://lyngvaer.no/log/create-linux-initramfs
* https://www.kernel.org/doc/html/latest/admin-guide/initrd.html
* https://www.linuxfromscratch.org/blfs/view/cvs/postlfs/initramfs.html
* https://ibug.io/blog/2019/04/os-lab-1/
