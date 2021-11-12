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

## 1. Create an initial initramfs
```
    ## set up directory structure
    mkdir -pv ${MYINIT}
    mkdir -pv ${MYINIT}/{bin,boot,dev,{etc/,}opt,home,lib/{firmware,modules},lib64,mnt}
    mkdir -pv ${MYINIT}/{proc,media/{floppy,cdrom},sbin,srv,sys}
    mkdir -pv ${MYINIT}/var/{lock,log,mail,run,spool}
    mkdir -pv ${MYINIT}/var/{opt,cache,lib/{misc,locate},local}
    install -dv -m 0750 ${MYINIT}/root
    install -dv -m 1777 ${MYINIT}{/var,}/tmp
    install -dv ${MYINIT}/etc/init.d
    mkdir -pv ${MYINIT}/usr/{,local/}{bin,include,lib{,64},sbin,src}
    mkdir -pv ${MYINIT}/usr/{,local/}share/{doc,info,locale,man}
    mkdir -pv ${MYINIT}/usr/{,local/}share/{misc,terminfo,zoneinfo}
    mkdir -pv ${MYINIT}/usr/{,local/}share/man/man{1,2,3,4,5,6,7,8}
    for dir in ${MYINIT}/usr{,/local}; do
	ln -sv share/{man,doc,info} ${dir}
    done
```

## External Resources
* https://www.linuxjournal.com/content/diy-build-custom-minimal-linux-distribution-source
* https://wiki.debian.org/initramfs/
* https://wiki.gentoo.org/wiki/Custom_Initramfs
* https://lyngvaer.no/log/create-linux-initramfs
* https://www.kernel.org/doc/html/latest/admin-guide/initrd.html
* https://www.linuxfromscratch.org/blfs/view/cvs/postlfs/initramfs.html
* https://ibug.io/blog/2019/04/os-lab-1/
