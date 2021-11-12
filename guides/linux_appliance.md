# Building Custom Linux Appliances

## Motivation
Linux appliances are a relatively old idea (https://www.usenix.org/legacy/publications/library/proceedings/lisa2000/full_papers/shaffer/shaffer_html/index.html), often understood as a self-contained system image containing *just enough* software and operating systems support to run a single application. Typically, they are built by system administrators to reduce system maintenance and improve ease of deployment. However, these appliances are equally useful for anyone interested in benchmarking the Linux kernel while ensuring experimental stability with minimal system noise. 

As recently discussed in https://www.usenix.org/publications/loginonline/musings-operating-systems-research, almost all modern OS research is conducted on Linux; which typically implies a Linux flavor such as Ubuntu, Fedora, etc. However, it is also not always clear how much effort researchers have taken to remove as many extraneous processes and kernel modules as possible to ensure a clean and stable Linux environment. For example, my Asus Ultrabook running Ubuntu 20.04 with Linux-5.11.0 idles with **231 processes** and **123 kernel modules** loaded. On the same laptop which I got it to run a custom built Linux-5.5.0 kernel and a custom initramfs; idles with **80 processes** and **0 kernel modules** loaded.

Furthermore in a recent paper published in SOSP'19: "An Analysis of Performance Evolution of Linux’s Core Operations", the authors demonstrated how Linux's configuration settings can have significant performance impacts on an application. This also motivates the overall premise of how much of Linux's default configuration does a kernel really need to be *functional* and how much complexity can thus be stripped out to run a minimum set of binaries? For example, following suggestions in this paper, I was able to build and boot a Linux-5.4.0 kernel with a custom config file that is less than 50% the length of a default Linux configuration.

Beyond these, another side benefit of a custom Linux appliance is that the root filesystem is loaded as a RAM disk by default; which is another positive for researchers who want to minimize system noise from disk paging as all executables are now run out from memory. Obviously, other considerations are needed should the experiments require data from a external storage device.

## Goals
There are many tutorials online to build your own Linux kernel and initramfs, however, their usecases are typically too general and the steps involved can be quite complex. This tutorial will demonstrate that it is infact surprsingly easy to get a barebones Linux up and running that is ready to execute some simple binaries. The tutorial will cover the specific use-case of how to build Linux appliances, how to boot them, and how this can help enable experimental automation, with a focus on performance and system stability. 

Concretely, here are the following steps in this tutorial towards this goal:

1. Create an initial initramfs
2. Build a basic Linux kernel
3. How to get simple binaries to run
4. How to boot appliance via GRUB
5. How to get device drivers working
6. Modiyfing /init for experiments
7. Booting via PXEBOOT protocol in a local network

## Preparation
Before we embark on this journey, here's a general breakdown of what is required on your end. First, you should have a testing computer that you will be booting the Linux appliance on and you should have an existing Linux flavor already installed on the computer, this can be Ubuntu, Fedora, etc. Having a pre-existing OS on your testing machine is important because you want to make sure you are in a state where the binary is runnable and it can also be used to install additional packages that you may need. Further, we will be relying on the existing system libraries of the OS to get basic Unix utilities such as Bash working. The implication here is also that the kernel used for the Linux appliance should be simiarly versioned as the exiting OS to ensure system library compatibility. 

## Steps to create an initramfs

First, create a name for your initramfs file
```
export MYINIT=initfs
```

Now, we set up the overall directory structure along with establishing root user
```
## set up directory structure for the initramfs
mkdir -pv ${MYINIT}
mkdir -pv ${MYINIT}/{bin,boot,dev,etcopt,home,lib/{firmware,modules},lib64,mnt}
mkdir -pv ${MYINIT}/{proc,media/{floppy},sbin,srv,sys}
mkdir -pv ${MYINIT}/var/{cache,lock,log,run,lib}
install -dv -m 0750 ${MYINIT}/root
install -dv -m 1777 ${MYINIT}{/var,}/tmp
mkdir -pv ${MYINIT}/usr/{,local/}{bin,lib{,64},sbin}
mkdir -pv ${MYINIT}/usr/{,local/}share/man
    
## set up root user info
cat > ${MYINIT}/etc/passwd <<EOF
root::0:0:root:/root:/bin/ash
EOF

cat > ${MYINIT}/etc/group <<EOF
root:x:0:
bin:x:1:
sys:x:2:
kmem:x:3:
tty:x:4:
daemon:x:6:
disk:x:8:
dialout:x:10:
video:x:12:
utmp:x:13:
usb:x:14:
EOF

## bash profile to get the environments set up correctly
cat > ${MYINIT}/etc/profile <<EOF
# /etc/profile: system-wide .profile file for the Bourne shell (sh(1))
# and Bourne compatible shells (bash(1), ksh(1), ash(1), ...).
if [ "`id -u`" -eq 0 ]; then
  PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
else
  PATH="/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games"
fi
export PATH
if [ "${PS1-}" ]; then
  if [ "${BASH-}" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
  else
    if [ "`id -u`" -eq 0 ]; then
      PS1='# '
    else
      PS1='$ '
    fi
  fi
fi
if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi
EOF
```

Now, we create the very important */init* file which will be used to eventually drive experiments
```
 ## create an ini file (note: not using systemd)
cat > ${MYINIT}/init <<EOF
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

## set permissions for init
chmod 755 ${MYINIT}/init
```

Next, we will compress the initramfs into a cpio format
```
## make sure you are inside the $MYINIT directory
cd ${MYINIT}
find . | cpio -o -H newc > ../${MYINIT}.cpio
```

## Getting binaries to run
The */init* created above is the first script is run to set up various Linux subsystems. Examining it, one can see that #!/bin/bash is the first program that is called, which provides an environment to run a basic shell. I will show an example of how to copy some utility binaries and its dependent libraries into the correct folders. The steps shown below will be completely manual but it can also be completely scriptable as well, however, that will be outside the scope of this tutorial.

```
## From /init file above, we see that the following programs will be needed: bash, mount, echo, mkdir, ln, poweroff

## figure out where binary lives
ubuntu:~$ export MYBIN=bash
ubuntu:~$ which $MYBIN
/usr/bin/bash
ubuntu:~$ which $MYBIN | xargs -I '{}' dirname '{}'
/usr/bin

## get full path
export MYBIN_LOC=$(which $MYBIN)

## copy bash to correct directory in initramfs
ubuntu:~$ cp $MYBIN_LOC ${MYINIT}/usr/bin/

## ldd can tell us the system libraries the binary depends upon
ubuntu:~$ ldd $MYBIN_LOC
	linux-vdso.so.1 (0x00007fff1a188000)
	libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x00007fb088faa000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fb088fa4000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fb088db2000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fb089119000)

## Basically, we have to make sure we copy the correct libraries at the correct location in the new initramfs
## We can ignore linux-vdso.so.1 as the kernel provides it automatically: https://man7.org/linux/man-pages/man7/vdso.7.html.

## simple scripting to get the necessary libraries
ubuntu:~$ ldd $MYBIN_LOC | grep "ld-linux" | awk '{print $1}'
/lib64/ld-linux-x86-64.so.2

ubuntu:~$ ldd $MYBIN_LOC | grep "=> /" | awk '{print $3}'
/lib/x86_64-linux-gnu/libtinfo.so.6
/lib/x86_64-linux-gnu/libdl.so.2
/lib/x86_64-linux-gnu/libc.so.6

## copy the libraries that bash depend on
ubuntu:~$ cp /lib64/ld-linux-x86-64.so.2 ${MYINIT}/lib64/
ubuntu:~$ cp /lib/x86_64-linux-gnu/libtinfo.so.6 ${MYINIT}/lib/x86_64-linux-gnu/
ubuntu:~$ cp /lib/x86_64-linux-gnu/libdl.so.2 ${MYINIT}/lib/x86_64-linux-gnu/
ubuntu:~$ cp /lib/x86_64-linux-gnu/libc.so.6 ${MYINIT}/lib/x86_64-linux-gnu/

```


## External Resources
* https://www.linuxjournal.com/content/diy-build-custom-minimal-linux-distribution-source
* https://wiki.debian.org/initramfs/
* https://wiki.gentoo.org/wiki/Custom_Initramfs
* https://lyngvaer.no/log/create-linux-initramfs
* https://www.kernel.org/doc/html/latest/admin-guide/initrd.html
* https://www.linuxfromscratch.org/blfs/view/cvs/postlfs/initramfs.html
* https://ibug.io/blog/2019/04/os-lab-1/
