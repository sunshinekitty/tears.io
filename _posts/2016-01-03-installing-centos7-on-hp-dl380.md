---
layout: post
comments: true
title: Installing Centos7 on an HP G5 DL380
preview: Starting with RHEL/EL7 the drivers for the raid controller card, the P400i, are no longer loaded in by default.  When you go to install Centos 7 on one of these you will likely see that it doesn't detect your LD.
---

Starting with RHEL/EL7 the drivers for the raid controller card, the P400i, are no longer loaded in by default.  When you go to install Centos 7 on one of these you will likely see that it doesn't detect your LD.

When you boot up and go into the install, press `tab` to get to your install options.  From here append `hpsa.hpsa_simple_mode=1 hpsa.hpsa_allow_any=1` to your boot options.

The install should now pick up on your LD and you should be able to install as normal.  After you reboot however you'll have to make this more permanent.  At the grub screen press `e` to get back to your boot options.  From here append `hpsa.hpsa_simple_mode=1 hpsa.hpsa_allow_any=1` again so that you can get into the machine.

Once you're logged in open up `/etc/default/grub` and add these options to the file.  The file should now look something like:

```
GRUB_TIMEOUT=5
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root rd.lvm.lv=centos/swap crashkernel=auto rhgb quiet hpsa.hpsa_simple_mode=1 hpsa.hpsa_allow_any=1"
GRUB_DISABLE_RECOVERY="true"
```

Now run `grub2-mkconfig -o /boot/grub2/grub.cfg` and reboot the machine.  It should be detected from now on each time you boot.

**Success!**  
*note: hpssacli installed separately*

```
[root@localhost ~]# hpssacli ctrl all show config


Smart Array P400 in Slot 1                (sn: P61620E9SV69YJ)


   Port Name: 1I

   Port Name: 2I

   Internal Drive Cage at Port 1I, Box 1, OK

   Internal Drive Cage at Port 2I, Box 1, OK
   array A (SAS, Unused Space: 0  MB)


      logicaldrive 1 (546.8 GB, RAID 6, OK)

      physicaldrive 1I:1:5 (port 1I:box 1:bay 5, SAS, 146 GB, OK)
      physicaldrive 1I:1:6 (port 1I:box 1:bay 6, SAS, 146 GB, OK)
      physicaldrive 2I:1:1 (port 2I:box 1:bay 1, SAS, 146 GB, OK)
      physicaldrive 2I:1:2 (port 2I:box 1:bay 2, SAS, 146 GB, OK)
      physicaldrive 2I:1:3 (port 2I:box 1:bay 3, SAS, 146 GB, OK)
      physicaldrive 2I:1:4 (port 2I:box 1:bay 4, SAS, 146 GB, OK)
```
