# Building Custom Linux Appliances

## Motivation
Linux appliances are a relatively old [idea](https://www.usenix.org/legacy/publications/library/proceedings/lisa2000/full_papers/shaffer/shaffer_html/index.html), often understood as a self-contained system image containing *just enough* software and operating systems support to run a single application. Typically, they are built by system administrators to reduce system maintenance and improve ease of deployment. However, these appliances are equally useful for anyone interested in benchmarking the Linux kernel while ensuring experimental stability with minimal system noise. 

As a motivating example, my Asus Ultrabook running Ubuntu 20.04 with Linux-5.11.0 idles with **231 processes** and **123 kernel modules** loaded. On the same laptop which I got it to run a custom built Linux-5.5.0 kernel and a custom initramfs; idles with **80 processes** and **0 kernel modules** loaded.

## Goals
