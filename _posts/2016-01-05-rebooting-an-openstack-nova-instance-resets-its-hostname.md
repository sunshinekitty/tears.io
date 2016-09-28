---
layout: post
comments: true
title: Rebooting an Openstack Nova instance resets it's hostname
preview: Recently when setting up a FreeIPA server I ran into the issue of the server resetting it's hostname every time it rebooted.  I had configured FreeIPA to use a different hostname than what the server was booted up with, and it was important that it kept that way otherwise FreeIPA would not start on boot properly.
---

Recently when setting up a FreeIPA server I ran into the issue of the server resetting it's hostname every time it rebooted.  I had configured FreeIPA to use a different hostname than what the server was booted up with, and it was important that it kept that way otherwise FreeIPA would not start on boot properly.

After checking in the logs via journalctl (and re-installing FreeIPA 3-4 times) I found that the cause was cloud-init! This service apparently runs each time the server is booted, and one of the params in the default Centos 7 config file is to set the hostname.  The sub-domain is set to the name you gave the server, and the domain is set to what's configured in nova inside of `/etc/nova/nova.conf` at `dhcp_domain=novalocal`.

There are a few ways you can go about fixing this.

### Remove cloud-init

```bash
yum remove cloud-init
```

### Update the cloud-init config

Remove set\_hostname and update_hostname from the cloud-init config

```bash
vim /etc/cloud/cloud.cfg
```

```
<...>
 - set_hostname
 - update_hostname
<...>
```

### Update nova.conf

This will change what the domain of your servers is set to, if you're okay with this format then this should be preferred as it doesn't require any changes be done on the guest VM.

```bash
vim /etc/nova/nova.conf
```

```
<...>
dhcp_domain=novalocal
<...>
```

```bash
systemctl restart openstack-nova-api.service
```

I hope this helps anyone that runs into this! The only relevant article I found was [this one](https://access.redhat.com/solutions/1269643), which is behind a paywall.
