---
layout: post
title: Configuring a 3600 series AP without a WLC
preview: I recently purchased a Cisco Aironet 3602i.  By default this series of access points are configured to be a light access point, or LAP for short.  For larger networks with many access points it makes sense to configure them all simultaneously with something called a wireless lan controller (WLC).  Me not knowing this before ordering my AP I found myself at a loss.
---

I recently purchased a Cisco Aironet 3602i.  By default this series of access points are configured to be a light access point, or LAP for short.  For larger networks with many access points it makes sense to configure them all simultaneously with something called a wireless lan controller (WLC).  

Me not knowing this before ordering my AP I found myself at a loss.  I only plan to have one AP for the foreseeable future, and so purchasing a WLC to do configuration with didn't make sense.

Luckily you can take this AP and turn it into what's called an autonomous AP, meaning you can manage the configuration on the AP itself.

## Converting the LAP to an AAP

### You have a Cisco account

You will want to download the AAP firmware from [here](https://software.cisco.com/download/navigator.html?mdfid=284000638&flowid=29982).  Select your AP model and then from the options list click on "Autonomous AP IOS Software".  I'd recommend downloading the latest stable release.

### You don't have a Cisco account

I find this aspect of Cisco **dumbfounding**, but software downloads are not available unless you're a paying, registered customer.  If you purchased through an authorized reseller or happen to have a business account then you can download their software.  If you're not, then their download site is of **no** use to you.

Lucky for you there is a small loophole.  Out of the *"kindness"* of their hearts, Cisco will provide you with their software, given there's a known security issue with the version you're running which does not have a workaround.  Also lucky you for, there are probably some of those for your hardware.  I recommend following along with [this article](https://damn.technology/free-cisco-ios-updates). Be sure to mention that you are running an autonomous AP and need the autonomous AP firmware.  I can confirm that this still works, however they do require you to have a Cisco account.  A free account (referred to as a guest) should suffice.  I will also mention that Cisco support first tried calling me to discuss the details, I ignored their call and less than 30 minutes later they emailed me with a temporary download link.

### You have the firmware
Great! Now you're ready to copy over your new firmware.  Without a WLC, a LAP doesn't have much functionality.  Unfortunately I wasn't able to find a way to backup your existing image before copying over the new one.  The following will be destructive and should be done at your own discretion.

#### Install a TFTP server
You'll need to have a TFTP server setup on a machine located in your network which has an IP in the same subnet as the AP in order to download the image.

There are various guides describing how to install and configure a TFP server, and so I invite you to Google around if this one doesn't suffice.

First, verify your machine has xinetd installed and running.  By default on Centos 7 this is the case.
```bash
[root@hv1 ~]# ps -alxww | grep /usr/sbin/xinet
0:00 /usr/sbin/xinetd -stayalive -pidfile /var/run/xinetd.pid
```

Now install `tftp-server`
```bash
yum install -y tftp-server tftp
```

Next we need to configure xinetd to run our tftp server.  Copy the following into `/etc/xinetd.d/ftp` if the file does not already exist.

```
# default: off
# description: The tftp server serves files using the Trivial File Transfer \
#    Protocol.  The tftp protocol is often used to boot diskless \
#    workstations, download configuration files to network-aware printers, \
#    and to start the installation process for some operating systems.
service tftp
{
    socket_type     = dgram
    protocol        = udp
    wait            = yes
    user            = root
    server          = /usr/sbin/in.tftpd
    server_args     = -s /var/lib/tftpboot
    disable         = yes
}
```

Now change `disable = yes` to `disable = no`, verify that `server_args` are set to `-s /var/lib/tftpboot` and reload the service.

```bash
systemctl reload xinetd
```

To verify that TFTP is working as expected make a test file inside of `/var/lib/tftpboot` to try downloading.

```bash
echo 'boop' > /var/lib/tftpboot/test.txt
```

Then try to download it

```bash
[root@hv1 ~]# tftp localhost
tftp> get test.txt
[root@hv1 ~]# cat test.txt
boop
```

If this works, you're ready to proceed.

### Re-image time

Copy over your firmware file to your TFTP directory.
So in my case, I ran:
```bash
cp ap3g2-k9w7-tar.153-3.JC.tar /var/lib/tftpboot/
```

Login to your AP and run the following
```
ap: en
ap# ping (your tftp server, verify ap can reach it)
ap# debug capwap console cli
ap# archive download tftp://(your tftp server)/(your filename)
```

It will probably take a little while to convert the AP, but after it finishes you'll be set!
