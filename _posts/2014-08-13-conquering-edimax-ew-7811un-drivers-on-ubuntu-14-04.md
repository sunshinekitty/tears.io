---
layout: post
title: Conquering Edimax EW-7811Un drivers on Ubuntu 14.04 and 14.10
comments: true
preview: With the default drivers I initially was getting 56KB/s with this adapter and had to reset it every 3-5 minutes.  I made some (crappy) tweaks and got up to 2MB/s. Which was enough to install vim and fix the problem.
---

<div class="message">
  Last updated 03/23/2015, tested working on Ubuntu 14.04 and 14.10 with kernel 3.19
</div>

I will admit that I spent way too much of my time cursing at my computer to fix this issue.

With the default drivers I initially was getting 56KB/s with this adapter, and you have to reset it once every 3-5 minutes. I made some (crappy) tweaks and got up to 2MB/s. Which was enough to install vim and fix the problem.

With Edimax’s drivers they provide when installing on Ubuntu 14.04 you will get something like this error:

```
/EW7811Un_Linux_driver_v1.0.0.5/RTL8188C_8192C_USB_linux_v4.0.2_9000.20130911/driver/rtl8188C_8192C_usb_linux_v4.0.2_9000.20130911/os_dep/linux/os_intfs.c: In function ‘rtw_proc_init_one’:
```

Great, for those who care that function the drivers depends on is deprecated; hence the problem.

There was a fork made of these drivers which is almost functional [here](https://github.com/dz0ny/rt8192cu).

First install the dependencies of these drivers, then download the drivers:

```bash
sudo apt-get update && sudo apt-get install git build-essential linux-headers-generic dkms
git clone https://github.com/dz0ny/rt8192cu.git --depth 1
```

Currently (as of 03/23/2015) there is a bug when installing this with GCC 4.9 or newer. To fix this bug remove line 1580 from `rt8192cu/os_dep/linux/usb_intf.c`. This line should read:

```c
DBG_871X("build time: %s %s\n", __DATE__, __TIME__);
```

If line does not exist then it may have been fixed since my last update. After this line doesn’t exist finish up by executing the following:

```bash
cd rt8192cu
sudo make dkms
```

If it completes without errors then reboot and you will be set!
