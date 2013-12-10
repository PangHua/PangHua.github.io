---
layout: post
title: Configure OpenStack Neutron gre network with ml2 plugin
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Home</p>

OS:RHEL6.4

OpenStack version:Havana

Controller node: xianghui-controller 10.7.0.170

Compute node: xianghui-compute 10.7.0.176

# Prepare yum and packages

Adding [RDO yum source][rdo] to support gre

[rdo]:  http://repos.fedorapeople.org/repos/openstack/openstack-havana/epel-6/

Downloading below packages

- kernel-2.6.32-358.114.1.openstack.el6.gre.2.x86_64.rpm
- kernel-firmware-2.6.32-358.114.1.openstack.el6.gre.2.noarch.rpm

    [root@xianghui-controller ~]# rpm -ivh kernel-2.6.32-358.114.1.openstack.el6.gre.2.x86_64.rpm kernel-firmware-2.6.32-358.114.1.openstack.el6.gre.2.noarch.rpm
    warning: kernel-2.6.32-358.114.1.openstack.el6.gre.2.x86_64.rpm: Header V4 RSA/SHA1 Signature, key ID 2bc7c801: NOKEY
    Preparing...                ########################################### [100%]
    1:kernel-firmware        ########################################### [ 50%]
    2:kernel                 ########################################### [100%]

Installed openvswitch version

    [root@xianghui-controller ~]# rpm -qa|grep openvswitch
    openvswitch-1.10.0-3.ibm.x86_64

# Configure Neutron conf files:
Edit `/etc/neutron/neutron.conf` in controller node

    [default]
    #core_plugin = neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2
    core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
    [database]
    connection = mysql://neutron:neutron@10.7.0.170/ovs_neutron

Edit `/etc/neutron/plugins/ml2/ml2_conf.ini` in controller node

    [ml2]
    type_drivers = gre
    tenant_network_types = gre
    [ml2_type_gre]
    tunnel_id_ranges = 15:1000

Edit `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini` in controller node
    
    enable_tunneling = True
    local_ip = 10.7.0.170

Edit `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini` in compute node
    
    enable_tunneling = True
    local_ip = 10.7.0.176

# Restart Neutron services

    [root@xianghui-controller ~]# service neutron-dhcp-agent restart
    [root@xianghui-controller ~]# service neutron-openvswitch-agent restart
    [root@xianghui-controller ~]# /usr/bin/python /usr/bin/neutron-server --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --log-file /var/log/neutron/server.log &
    [root@xianghui-compute ~]# service neutron-openvswitch-agent restart

Checking `br-int` `br-tun`
<pre><code>
[root@xianghui-controller ~]# ovs-vsctl show
    Bridge br-int
        Port "tap6ea55329-d8"
            tag: 1
            Interface "tap6ea55329-d8"
                type: internal
        Port "qvobedc859e-83"
            tag: 1
            Interface "qvobedc859e-83"
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
    Bridge br-tun
        Port br-tun
            Interface br-tun
                type: internal
        Port "gre-10.7.0.176"
            Interface "gre-10.7.0.176"
                type: gre
                options: {in_key=flow, local_ip="10.7.0.170", out_key=flow, remote_ip="10.7.0.176"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    ovs_version: "1.10.0"
</code></pre>

Checking dhcp server
<pre><code>
[root@xianghui-compute01 ~]# ps -ef|grep dnsmasq
nobody    7011     1  0 02:42 ?        00:00:00 /usr/sbin/dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=tap6ea55329-d8 --except-interface=lo --pid-file=/var/lib/neutron/dhcp/58b10b02-4e79-4c4f-ab5f-68f53b544d82/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/58b10b02-4e79-4c4f-ab5f-68f53b544d82/host --dhcp-optsfile=/var/lib/neutron/dhcp/58b10b02-4e79-4c4f-ab5f-68f53b544d82/opts --leasefile-ro --dhcp-range=set:tag0,80.0.80.0,static,86400s --dhcp-lease-max=256 --conf-file= --domain=openstacklocal
root     10401 23847  0 02:50 pts/0    00:00:00 grep dnsmasq

[root@xianghui-compute01 ~]# ifconfig tap6ea55329-d8
tap6ea55329-d8 Link encap:Ethernet  HWaddr FA:16:3E:AA:44:18
          inet addr:80.0.80.2  Bcast:80.0.80.255  Mask:255.255.255.0
          inet6 addr: fe80::9485:86ff:febd:71b3/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:185 errors:0 dropped:0 overruns:0 frame:0
          TX packets:107 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:11966 (11.6 KiB)  TX bytes:8346 (8.1 KiB)
</code></pre>

# Create gre network
<pre><code>
[root@xianghui-controller ~]# neutron net-create net_gre --provider:network_type gre --provider:segmentation_id 500
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 58b10b02-4e79-4c4f-ab5f-68f53b544d82 |
| name                      | net_gre                              |
| provider:network_type     | gre                                  |
| provider:physical_network |                                      |
| provider:segmentation_id  | 500                                 |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | b4e6e40f-fa4d-4ed5-afcd-8fff5fa6a28b |
| tenant_id                 | e73608fb6764404294624317905297c1     |
+---------------------------+--------------------------------------+
</code></pre>
<pre><code>
[root@xianghui-controller ~]# neutron subnet-create --ip_version 4 net_gre 80.0.80.0/24
+------------------+----------------------------------------------+
| Field            | Value                                        |
+------------------+----------------------------------------------+
| allocation_pools | {"start": "80.0.80.2", "end": "80.0.80.254"} |
| cidr             | 80.0.80.0/24                                 |
| dns_nameservers  |                                              |
| enable_dhcp      | True                                         |
| gateway_ip       | 80.0.80.1                                    |
| host_routes      |                                              |
| id               | b4e6e40f-fa4d-4ed5-afcd-8fff5fa6a28b         |
| ip_version       | 4                                            |
| name             |                                              |
| network_id       | 58b10b02-4e79-4c4f-ab5f-68f53b544d82         |
| tenant_id        | e73608fb6764404294624317905297c1             |
+------------------+----------------------------------------------+
</code></pre>
<pre><code>
[root@xianghui-controller ~]# neutron net-list
+--------------------------------------+---------+---------------------------------------------------+
| id                                   | name    | subnets                                           |
+--------------------------------------+---------+---------------------------------------------------+
| 58b10b02-4e79-4c4f-ab5f-68f53b544d82 | net_gre | b4e6e40f-fa4d-4ed5-afcd-8fff5fa6a28b 80.0.80.0/24 |
+--------------------------------------+---------+---------------------------------------------------+
</code></pre>

# Create a vm
*generate a user key to login without password*

    [root@xianghui-controller ~]# nova keypair-add --pub_key ~/.ssh/id_rsa.pub user_key

<pre><code>
[root@xianghui-controller ~]# nova boot --image b2e468ff-57fb-44e6-bd28-ba1ead8d7fb8 --flavor 2 --nic net-id=58b10b02-4e79-4c4f-ab5f-
68f53b544d82 --key-name user_key --availability-zone nova:xianghui-compute test_gre_1
[root@xianghui-controller ~]# nova list
+--------------------------------------+------------+--------+------------+-------------+-------------------+
| ID                                   | Name       | Status | Task State | Power State | Networks          |
+--------------------------------------+------------+--------+------------+-------------+-------------------+
| 683472cd-586d-44b5-93e9-a5dc62bdd2aa | test_gre_1 | ACTIVE | None       | Running     | net_gre=80.0.80.7 |
+--------------------------------------+------------+--------+------------+-------------+-------------------+
</code></pre>

    [root@xianghui-controller ~]# ping 80.0.80.7
    PING 80.0.80.7 (80.0.80.7) 56(84) bytes of data.
    64 bytes from 80.0.80.7: icmp_seq=1 ttl=64 time=1.79 ms
    64 bytes from 80.0.80.7: icmp_seq=2 ttl=64 time=0.401 ms

# Capture packets processed by dhcp client
dhcp client discover

[![dhcp-request](/images/tech/dhcp-request.jpg)](/images/tech/dhcp-request.jpg)

dhcp server reply packets, dhcpserver(80.0.80.2) approve client 80.0.80.8

[![dhcp-reply](/images/tech/dhcp-reply.jpg)](/images/tech/dhcp-reply.jpg)

Details:
<pre><code>
Details:
No.     Time        Source                Destination           Protocol Info
   2070 66.738018   80.0.80.2             80.0.80.8             DHCP     DHCP ACK      - Transaction ID 0xffdb3753

Frame 2070 (408 bytes on wire, 408 bytes captured)
Ethernet II, Src: RealtekU_07:00:aa (52:54:00:07:00:aa), Dst: RealtekU_07:00:b0 (52:54:00:07:00:b0)
    Destination: RealtekU_07:00:b0 (52:54:00:07:00:b0)
        Address: RealtekU_07:00:b0 (52:54:00:07:00:b0)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
    Source: RealtekU_07:00:aa (52:54:00:07:00:aa)
        Address: RealtekU_07:00:aa (52:54:00:07:00:aa)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
    Type: IP (0x0800)
Internet Protocol, Src: 10.7.0.170 (10.7.0.170), Dst: 10.7.0.176 (10.7.0.176)
    Version: 4
    Header length: 20 bytes
    Differentiated Services Field: 0x00 (DSCP 0x00: Default; ECN: 0x00)
        0000 00.. = Differentiated Services Codepoint: Default (0x00)
        .... ..0. = ECN-Capable Transport (ECT): 0
        .... ...0 = ECN-CE: 0
    Total Length: 394
    Identification: 0xcae0 (51936)
    Flags: 0x02 (Don't Fragment)
        0.. = Reserved bit: Not Set
        .1. = Don't fragment: Set
        ..0 = More fragments: Not Set
    Fragment offset: 0
    Time to live: 64
    Protocol: GRE (0x2f)
    Header checksum: 0x58fd [correct]
        [Good: True]
        [Bad : False]
    Source: 10.7.0.170 (10.7.0.170)
    Destination: 10.7.0.176 (10.7.0.176)
Generic Routing Encapsulation (Transparent Ethernet bridging)
    Flags and version: 0x2000
    Protocol Type: Transparent Ethernet bridging (0x6558)
    GRE Key: 0x000001f4
Ethernet II, Src: fa:16:3e:87:e3:12 (fa:16:3e:87:e3:12), Dst: fa:16:3e:14:ff:3d (fa:16:3e:14:ff:3d)
    Destination: fa:16:3e:14:ff:3d (fa:16:3e:14:ff:3d)
        Address: fa:16:3e:14:ff:3d (fa:16:3e:14:ff:3d)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
    Source: fa:16:3e:87:e3:12 (fa:16:3e:87:e3:12)
        Address: fa:16:3e:87:e3:12 (fa:16:3e:87:e3:12)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
    Type: IP (0x0800)
Internet Protocol, Src: 80.0.80.2 (80.0.80.2), Dst: 80.0.80.8 (80.0.80.8)
    Version: 4
    Header length: 20 bytes
    Differentiated Services Field: 0xc0 (DSCP 0x30: Class Selector 6; ECN: 0x00)
        1100 00.. = Differentiated Services Codepoint: Class Selector 6 (0x30)
        .... ..0. = ECN-Capable Transport (ECT): 0
        .... ...0 = ECN-CE: 0
    Total Length: 352
    Identification: 0xcae0 (51936)
    Flags: 0x00
        0.. = Reserved bit: Not Set
        .0. = Don't fragment: Not Set
        ..0 = More fragments: Not Set
    Fragment offset: 0
    Time to live: 64
    Protocol: UDP (0x11)
    Header checksum: 0x6de2 [correct]
        [Good: True]
        [Bad : False]
    Source: 80.0.80.2 (80.0.80.2)
    Destination: 80.0.80.8 (80.0.80.8)
User Datagram Protocol, Src Port: bootps (67), Dst Port: bootpc (68)
    Source port: bootps (67)
    Destination port: bootpc (68)
    Length: 332
    Checksum: 0x6ac4 [validation disabled]
        [Good Checksum: False]
        [Bad Checksum: False]
Bootstrap Protocol
    Message type: Boot Reply (2)
    Hardware type: Ethernet
    Hardware address length: 6
    Hops: 0
    Transaction ID: 0xffdb3753
    Seconds elapsed: 0
    Bootp flags: 0x0000 (Unicast)
    Client IP address: 0.0.0.0 (0.0.0.0)
    Your (client) IP address: 80.0.80.8 (80.0.80.8)
    Next server IP address: 80.0.80.2 (80.0.80.2)
    Relay agent IP address: 0.0.0.0 (0.0.0.0)
    Client MAC address: fa:16:3e:14:ff:3d (fa:16:3e:14:ff:3d)
    Client hardware address padding: 00000000000000000000
    Server host name not given
    Boot file name not given
    Magic cookie: (OK)
    Option: (t=53,l=1) DHCP Message Type = DHCP ACK
        Option: (53) DHCP Message Type
        Length: 1
        Value: 05
    Option: (t=54,l=4) DHCP Server Identifier = 80.0.80.2
        Option: (54) DHCP Server Identifier
        Length: 4
        Value: 50005002
    Option: (t=51,l=4) IP Address Lease Time = 1 day
        Option: (51) IP Address Lease Time
        Length: 4
        Value: 00015180
    Option: (t=58,l=4) Renewal Time Value = 12 hours
        Option: (58) Renewal Time Value
        Length: 4
        Value: 0000A8C0
    Option: (t=59,l=4) Rebinding Time Value = 21 hours
        Option: (59) Rebinding Time Value
        Length: 4
        Value: 00012750
    Option: (t=1,l=4) Subnet Mask = 255.255.255.0
        Option: (1) Subnet Mask
        Length: 4
        Value: FFFFFF00
    Option: (t=28,l=4) Broadcast Address = 80.0.80.255
        Option: (28) Broadcast Address
        Length: 4
        Value: 500050FF
    Option: (t=6,l=4) Domain Name Server = 80.0.80.2
        Option: (6) Domain Name Server
        Length: 4
        Value: 50005002
    Option: (t=15,l=14) Domain Name = "openstacklocal"
        Option: (15) Domain Name
        Length: 14
        Value: 6F70656E737461636B6C6F63616C
    Option: (t=12,l=14) Host Name = "host-80-0-80-8"
        Option: (12) Host Name
        Length: 14
        Value: 686F73742D38302D302D38302D38
    Option: (t=3,l=4) Router = 80.0.80.1
        Option: (3) Router
        Length: 4
        Value: 50005001
    End Option
</code></pre>
