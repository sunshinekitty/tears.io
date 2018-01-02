---
layout: post
comments: true
title: Creating a floating IP on Openstack Neutron
preview: I was running a couple of database servers on Nova VM's and found it was a little more complicated to get floating IP's working.  This guide will walk you through the creation of a floating IP.
---

I was running a couple of database servers on Nova VM's and found it was a little more complicated to get floating IP's working.  First you'll want to source your Keystone ENV file for the respective tenant user (must not be admin).

First create a floating port inside of the tenant vlan.

```
neutron port-create --security-group d90c139c-1cd7-4581-a9d6-ee2cf979409f --security-group=default --security-group=mysql tenant
```

Create a floating IP on our home vlan to the port you just made.

```
neutron floatingip-create --port-id=$ID homelan
```

Find the port ID's which are mapped to the two VM's you want to share an IP between, eg 10.0.0.43 and 10.0.0.45

```
[root@hv1 ~(keystone_alex)]# neutron port-list | grep "10.0.0.43\|10.0.0.45"
| 008c2c0a-8a20-4e71-9daf-7def6fbc7985 |      | fa:16:3e:da:f4:ff | {"subnet_id": "1298b7c1-1277-4926-aed0-05d021e5ada3", "ip_address": "10.0.0.43"} |
| f321d184-ba80-47d3-95ad-7105e454ca3d |      | fa:16:3e:4c:d6:aa | {"subnet_id": "1298b7c1-1277-4926-aed0-05d021e5ada3", "ip_address": "10.0.0.45"} |
```

Allow the private floating port's IP address on these two servers ports.  10.0.0.50 is the private floating port's IP here.  There's a chance you'll need to be an Openstack admin in order to run the following commands based off your Nova config.

```
neutron port-update 008c2c0a-8a20-4e71-9daf-7def6fbc7985 --allowed_address_pairs list=true type=dict ip_address=10.0.0.50
neutron port-update f321d184-ba80-47d3-95ad-7105e454ca3d --allowed_address_pairs list=true type=dict ip_address=10.0.0.50
```

We can now confirm these ports have the floating IP in their allowed_address_pair's list and can use this floating IP.

```
[root@hv1 ~(keystone_admin)]# neutron port-show f321d184-ba80-47d3-95ad-7105e454ca3d
+-----------------------+-----------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                               |
+-----------------------+-----------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                |
| allowed_address_pairs | {"ip_address": "10.0.0.50", "mac_address": "fa:16:3e:4c:d6:aa"}                                     |
| binding:host_id       | hv1.tears.io                                                                                        |
| binding:profile       | {}                                                                                                  |
| binding:vif_details   | {"port_filter": true, "ovs_hybrid_plug": true}                                                      |
| binding:vif_type      | ovs                                                                                                 |
| binding:vnic_type     | normal                                                                                              |
| device_id             | 0683e37b-5a55-437b-9783-7c4015750dbf                                                                |
| device_owner          | compute:nova
```

