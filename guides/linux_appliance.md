# Building Custom Linux Appliances

## Motivation
Linux appliances are a relatively old idea (https://www.usenix.org/legacy/publications/library/proceedings/lisa2000/full_papers/shaffer/shaffer_html/index.html), often understood as a self-contained system image containing *just enough* software and operating systems support to run a single application. Typically, they are built by system administrators to reduce system maintenance and improve ease of deployment. However, these appliances are equally useful for anyone interested in benchmarking the Linux kernel while ensuring experimental stability with minimal system noise. 

As recently discussed in https://www.usenix.org/publications/loginonline/musings-operating-systems-research, almost all modern OS research is conducted on Linux; which typically implies a Linux flavor such as Ubuntu, Fedora, etc. However, it is also not always clear how much effort researchers have taken to remove as many extraneous processes and kernel modules as possible to ensure a clean and stable Linux environment. For example, my Asus Ultrabook running Ubuntu 20.04 with Linux-5.11.0 idles with **231 processes** and **123 kernel modules** loaded. On the same laptop which I got it to run a custom built Linux-5.5.0 kernel and a custom initramfs; idles with **80 processes** and **0 kernel modules** loaded.

Furthermore in a recent paper published in SOSP'19: "An Analysis of Performance Evolution of Linux’s Core Operations", the authors demonstrated how Linux's configuration settings can have significant performance impacts on an application. This also motivates the overall premise of how much of Linux's default configuration does a kernel really need to be *functional* and how much complexity can thus be stripped out to run a minimum set of binaries? For example, following suggestions in this paper, I was able to build and boot a Linux-5.4.0 kernel with a custom config file that is less than 50% the length of a default Linux configuration.

## Goals
There are many tutorials online to build your own Linux kernel and initramfs. However, their usecases are typically too general and the goal with this tutorial is to demonstrate the specific use-case of how to build Linux appliances, how to boot them, and how this can help enable experimental automation, with a focus on performance and system stability. Concretely, here are the following goals of this tutorial:

1.

In this tutorial, there will be code to show how to build the kernel, modules, initramfs, how to get binaries running in a minimal environment, and images of the boot process on my laptop and how to bootstrap a simple PXEBOOT between two computers
