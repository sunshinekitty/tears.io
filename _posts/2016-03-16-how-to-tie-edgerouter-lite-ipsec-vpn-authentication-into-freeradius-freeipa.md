---
layout: post
title: How to tie Edgerouter Lite IPSEC VPN authentication into FreeRADIUS (FreeIPA)
preview: I hate managing user accounts, and so should you! That's why I try and get all that I can tied into FreeIPA's ldap so that they're only managed in one place.  I'm going to show you how to setup an ipsec vpn on an Edgerouter Lite which authenticates against a local freeipa instance.
---

<div class="message">
    This is a very lengthy post, despite explanations being brief.  <b>FreeIPA and FreeRADIUS are running on Centos 7.</b>
</div>

I hate managing user accounts, and so should you! That's why I try and get all that I can tied into FreeIPA's ldap so that they're only managed in one place.  I'm going to show you how to setup an ipsec vpn on an Edgerouter Lite which authenticates against a local freeipa instance.

The Edgerouter Lite (and Vayatta) come with support to authenticate against a RADIUS server.  You can configure your RADIUS server to then authenticate against your LDAP instance.  For this guide I'm going to assume that you already have FreeIPA up and running.

This server requires the following setup, optionally you can run RADIUS on the same host as FreeIPA but some configuration settings will be different.

![Logical layout](https://i.imgur.com/ZNa9yI5.png)

I'll be referring to these servers using these hostnames in this guide.  I'd like to point out that this guide is going to add in support for mschap in order to support Windows devices.  For an explanation on possible security issues with this I'll defer to [here](https://www.redhat.com/archives/freeipa-users/2015-December/msg00170.html).

## Setting up FreeIPA

In order to authenticate with mschap it's required that we hand over NTLM hashes from our FreeIPA instance for FreeRADIUS to use.  This is where security becomes a problem, these hashes can be used to brute-force encrypted passwords in FreeIPA.  Make sure your RADIUS instance is on a private network and well secured.

Follow along with generating the NTLM hashes below:
```bash
# WARNING the following may try and run `ipa-server-upgrade`
[root@freeipa ~]# yum -y install ipa-server-trust-ad
[root@freeipa ~]# ipa-adtrust-install

The log file for this installation can be found in /var/log/ipaserver-install.log
==============================================================================
This program will setup components needed to establish trust to AD domains for
the IPA Server.

This includes:
  * Configure Samba
  * Add trust related objects to IPA LDAP server

To accept the default shown in brackets, press the Enter key.

WARNING: The smb.conf already exists. Running ipa-adtrust-install will break your existing samba configuration.


Do you wish to continue? [no]: yes
Do you want to enable support for trusted domains in Schema Compatibility plugin?
This will allow clients older than SSSD 1.9 and non-Linux clients to work with trusted users.

Enable trusted domains support in slapi-nis? [no]: yes

Configuring cross-realm trusts for IPA server requires password for user 'admin'.
This user is a regular system account used for IPA server administration.

admin password:

Enter the NetBIOS name for the IPA domain.
Only up to 15 uppercase ASCII letters and digits are allowed.
Example: EXAMPLE.

NetBIOS domain name [TEARS]: TEARS

WARNING: 14 existing users or groups do not have a SID identifier assigned.
Installer can run a task to have ipa-sidgen Directory Server plugin generate
the SID identifier for all these users. Please note, the in case of a high
number of users and groups, the operation might lead to high replication
traffic and performance degradation. Refer to ipa-adtrust-install(1) man page
for details.

Do you want to run the ipa-sidgen task? [no]: yes

The following operations may take some minutes to complete.
Please wait until the prompt is returned.

Configuring CIFS
  [1/23]: stopping smbd
  [2/23]: creating samba domain object
  [3/23]: creating samba config registry
  [4/23]: writing samba config file
  [5/23]: adding cifs Kerberos principal
  [6/23]: adding cifs and host Kerberos principals to the adtrust agents group
  [7/23]: check for cifs services defined on other replicas
  [8/23]: adding cifs principal to S4U2Proxy targets
  [9/23]: adding admin(group) SIDs
  [10/23]: adding RID bases
  [11/23]: updating Kerberos config
'dns_lookup_kdc' already set to 'true', nothing to do.
  [12/23]: activating CLDAP plugin
  [13/23]: activating sidgen task
  [14/23]: configuring smbd to start on boot
  [15/23]: adding special DNS service records
  [16/23]: enabling trusted domains support for older clients via Schema Compatibility plugin
  [17/23]: restarting Directory Server to take MS PAC and LDAP plugins changes into account
  [18/23]: adding fallback group
  [19/23]: adding Default Trust View
  [20/23]: setting SELinux booleans
  [21/23]: enabling oddjobd
  [22/23]: starting CIFS services
  [23/23]: adding SIDs to existing users and groups
Done configuring CIFS.

=============================================================================
Setup complete

You must make sure these network ports are open:
        TCP Ports:
          * 138: netbios-dgm
          * 139: netbios-ssn
          * 445: microsoft-ds
        UDP Ports:
          * 138: netbios-dgm
          * 139: netbios-ssn
          * 389: (C)LDAP
          * 445: microsoft-ds

=============================================================================

[root@freeipa ~]#
```
Obviously make sure that these ports are open (HOW-TO not covered in this post).

Now create a user for FreeRADIUS to access FreeIPA using the Web UI. We'll see we made it with username "freeradius" with password of "replacethispassword". I typically like to add any service accounts under a group of "service" to keep things organized.

## Setting up FreeRADIUS

```bash
[root@radius ~]# yum -y update
[root@radius ~]# yum -y install freeradius freeradius-utils freeradius-ldap
[root@radius ~]# echo "192.168.0.55 radius.tears.io" >> /etc/hosts # replace IP with your own
```

Next open up `/etc/raddb/clients.conf` and add in the following:
<em>Note: Replace the subnet with your own local subnet, replace $ECRET with a better one ;-)</em>
```
client 192.168.0.0/24 {
        secret          = $ECRET
        require_message_authenticator = no
        nas_type     = other
}
```

Optionally you can configure a `localhost` client so that you can test from your localhost, the config would look the same but with `client localhost {<...>}`.

Append the following to `/etc/raddb/users`:
```
DEFAULT Auth-Type := LDAP
```
Add the following to the end of `/etc/raddb/dictionary`:
```
VALUE           Auth-Type               Local                   0
VALUE           Auth-Type               System                  1
VALUE           Auth-Type               SecurID                 2
VALUE           Auth-Type               Crypt-Local             3
VALUE           Auth-Type               Reject                  4
VALUE           Auth-Type               LDAP                    5
```

Create `/etc/raddb/sites-enabled/default` and add:
```
server default {
listen {
        type = auth
        ipaddr = *
        port = 0
        limit {
              max_connections = 16
              lifetime = 0
              idle_timeout = 30
        }
}

listen {
        ipaddr = *
        port = 0
        type = acct

        limit {
        }
}

listen {
        type = auth
        ipv6addr = ::   # any.  ::1 == localhost
        port = 0
        limit {
              max_connections = 16
              lifetime = 0
              idle_timeout = 30
        }
}

listen {
        ipv6addr = ::
        port = 0
        type = acct

        limit {
        }
}

authorize {
        filter_username
        preprocess
        suffix
        eap {
                ok = return
        }
        files
        ldap
        expiration
        logintime
}

authenticate {
        Auth-Type LDAP {
                ldap
        }
}


preacct {
        preprocess
        acct_unique
        suffix
        files
}

accounting {
        detail
        unix
        -sql
        exec
        attr_filter.accounting_response
}

session {
}

post-auth {
        -sql
        exec
        remove_reply_message_if_eap
        Post-Auth-Type REJECT {
                -sql
                attr_filter.access_reject
                eap
                remove_reply_message_if_eap
        }
}

pre-proxy {
}

post-proxy {
        eap
}
}
```
Create `/etc/raddb/mods-enabled/ldap` and add:
```
ldap {
        server = "freeipa.tears.io"
        identity = "uid=freeradius,cn=users,cn=accounts,dc=tears,dc=io"
        password = replacethispassword
        base_dn = "cn=users,cn=accounts,dc=tears,dc=io"
        password_attribute = ipaNTHashipaNTHash

        update {
        }

        user {
                base_dn = "${..base_dn}"
{% raw  %}
                filter = "(uid=%{%{Stripped-User-Name}:-%{User-Name}})"
{% endraw %}
        }

        group {
                base_dn = "${..base_dn}"
                filter = "(objectClass=posixGroup)"
                membership_attribute = "memberOf"
        }

        profile {
        }

        client {
                base_dn = "${..base_dn}"
                filter = '(objectClass=frClient)'
                attribute {
                        identifier                      = 'radiusClientIdentifier'
                        secret                          = 'radiusClientSecret'
                }
        }

        accounting {
                reference = "%{tolower:type.%{Acct-Status-Type}}"

                type {
                        start {
                                update {
                                        description := "Online at %S"
                                }
                        }

                        interim-update {
                                update {
                                        description := "Last seen at %S"
                                }
                        }

                        stop {
                                update {
                                        description := "Offline at %S"
                                }
                        }
                }
        }

        post-auth {
                update {
                        description := "Authenticated at %S"
                }
        }

        options {
                chase_referrals = yes
                rebind = yes
                timeout = 10
                timelimit = 3
                net_timeout = 1
                idle = 60
                probes = 3
                interval = 3
                ldap_debug = 0x0028
        }

        tls {
        }

        pool {
                start = 5
                min = 4
                max = ${thread[pool].max_servers}
                spare = 3
                uses = 0
                lifetime = 0
                idle_timeout = 60
        }
}
```
Finally:
```bash
systemctl start radiusd
systemctl enable radiusd
```
If you run into issues starting radiusd you can run `radiusd -X` to help debug

You can test pap works with:
```bash
radtest {username} {password} {hostname} 10 {radius_secret}
```

And mschap with:
```bash
radtest -t mschap {username} {password} {hostname} 10 {radius_secret}
```

## Setting up Edgerouter Lite

First create the ipsec VPN.
```bash
alex@router:~$ configure
[edit]
# Create an ipsec interface on your WAN port
set vpn ipsec ipsec-interfaces interface eth0 #replace eth0 with WAN port
# Enable nat traversal
set vpn ipsec nat-traversal enable
# Lock down which sub-net clients connect from, to allow all use 0.0.0.0/0
set vpn ipsec nat-networks allowed-network 0.0.0.0/0
```

Ipsec is a great VPN tunnel to use, however it's not so great at handling authentication.  Which is why we use L2TP to implement this.

```bash
# Bind to your WAN IP (that of your WAN port)
set vpn l2tp remote-access dhcp-interface eth0
# Set your pool of IP addresses VPN clients should use for dhcp
set vpn l2tp remote-access client-ip-pool start 192.168.0.10
set vpn l2tp remote-access client-ip-pool stop 192.168.0.20
# Now connect to your radius server using the creds you setup above
set vpn l2tp remote-access authentication radius-server RADIUS_SERVER_IP key $ECRET
# Set IPSec to use l2tp with the secret
set vpn l2tp remote-access ipsec-settings authentication mode pre-shared-secret
set vpn l2tp remote-access ipsec-settings authentication pre-shared-secret SUPER-DUPER-SECRET
# Finally commit, write, and quit
commit
save
exit
```

Lastly open the following ports on the 'local' firewall for your router:
```
IKE - UDP port 500
L2TP - UDP port 1701
ESP - protocol 50
NAT-T - UDP port 4500
```

If all went well you should now be able to connect to the VPN using both pap and mschap protocols.
