---
layout: post
title: Configure OpenStack Neutron flat network with ml2 plugin
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Ring Building</p>

OS:RHEL6.4
OpenStack version: Havana
Controller node : KVM
Compute node :  ESXI

# Create flat network
Overall networking configurations:

- Edit conf files `/etc/neutron/neutron.conf` in Controller node:

    [DEFAULT]
    core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
    # If needs neutron l3 agent work(router/NAT) 
    service_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
    [database]
    connection = mysql://neutron:neutron@xianghui-10-9-1-141.sce....

- Edit conf files `/etc/neutron/plugins/ml2/ml2_conf.ini` in Controller node:

    [ml2]
    type_drivers = flat
    tenant_network_types = flat
    mechanism_drivers = openvswitch
    [ml2_type_flat]
    flat_networks = physnet1

- Edit conf files `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini` in Controller node:

    [ovs]
    bridge_mappings = physnet1:br-eth0

# Using openvswitch to create a virtual switch and bind a nic
(choose eth1 as the data network, eth0 as the management network)

*It's better to have two nics, or the connection may be lost.*

    ovs-vsctl add-br br-eth0
    ovs-vsctl add-port br-eth0 eth1
    /usr/bin/python /usr/bin/neutron-server --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --log-file /var/log/neutron/server.log &
    service neutron-openvswitch-agent restart
    service neutron-dhcp-agent restart

    [root@xianghui-10-9-1-141 SDK]# ovs-vsctl show
    c68671ea-d3d2-4f04-b806-7be10e65936e
    Bridge "br-eth0"
        Port "phy-br-eth0"
            Interface "phy-br-eth0"
        Port "eth1"
            Interface "eth1"
        Port "br-eth0"
            Interface "br-eth0"
                type: internal
    Bridge br-int
        Port "tap922fa636-15"
            tag: 4095
            Interface "tap922fa636-15"
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port "tapc6ae5432-66"
            tag: 4095
            Interface "tapc6ae5432-66"
                type: internal
        Port "tap909c5f0b-61"
            tag: 1
            Interface "tap909c5f0b-61"
                type: internal
        Port "int-br-eth0"
            Interface "int-br-eth0"
    ovs_version: "1.10.0"

# create a port group named "br-int" in ESXI host


# Create flat network(flat-50)  ip range 50.0.0.0/24
    [root@xianghui-10-9-1-141 ˜]# neutron net-create flat-50 --provider:network_type flat --provider:physical_network physnet1
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | id                        | fffe6c04-1e09-4cb3-886b-6404c0729c1a |
    | name                      | flat-50                              |
    | provider:network_type     | flat                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  |                                      |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tenant_id                 | 57f4a09b4bfb4c97bf9b216e6157ef12     |
    +---------------------------+--------------------------------------+
    [root@xianghui-10-9-1-141 ˜]# neutron subnet-create flat-50 50.0.0.0/24
    Created a new subnet:
    +------------------+--------------------------------------------+
    | Field            | Value                                      |
    +------------------+--------------------------------------------+
    | allocation_pools | {"start": "50.0.0.2", "end": "50.0.0.254"} |
    | cidr             | 50.0.0.0/24                                |
    | dns_nameservers  |                                            |
    | enable_dhcp      | True                                       |
    | gateway_ip       | 50.0.0.1                                   |
    | host_routes      |                                            |
    | id               | 19a19f9c-81b6-453a-9efc-541e1985b9d7       |
    | ip_version       | 4                                          |
    | name             |                                            |
    | network_id       | fffe6c04-1e09-4cb3-886b-6404c0729c1a       |
    | tenant_id        | 57f4a09b4bfb4c97bf9b216e6157ef12           |
    +------------------+--------------------------------------------+

    [root@xianghui-10-9-1-141 ˜]# neutron net-list
    +--------------------------------------+---------+--------------------------------------------------+
    | id                                   | name    | subnets                                          |
    +--------------------------------------+---------+--------------------------------------------------+
    | fffe6c04-1e09-4cb3-886b-6404c0729c1a | flat-50 | 19a19f9c-81b6-453a-9efc-541e1985b9d7 50.0.0.0/24 |
    +--------------------------------------+---------+--------------------------------------------------+

*Make sure there is only one dnsmasq process*

    [root@xianghui-10-9-1-141 ˜]# ps -ef|grep dnsmasq
    root     22034 15050  0 03:42 pts/3    00:00:00 grep dnsmasq
    nobody   24702     1  0 01:39 ?        00:00:00 /usr/sbin/dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=tap909c5f0b-61 --except-interface=lo --pid-file=/var/lib/neutron/dhcp/fffe6c04-1e09-4cb3-886b-6404c0729c1a/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/fffe6c04-1e09-4cb3-886b-6404c0729c1a/host --dhcp-optsfile=/var/lib/neutron/dhcp/fffe6c04-1e09-4cb3-886b-6404c0729c1a/opts --leasefile-ro --dhcp-range=set:tag0,50.0.0.0,static,86400s --dhcp-lease-max=256 --conf-file= --domain=openstacklocal

*Check whether dhcp interface rx/tx is normal*

    [root@xianghui-10-9-1-141 ˜]# ifconfig tap909c5f0b-61
tap909c5f0b-61 Link encap:Ethernet  HWaddr FA:16:3E:24:92:CE
          inet addr:50.0.0.2  Bcast:50.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::dc69:1fff:fe61:c9e4/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:36074 errors:0 dropped:0 overruns:0 frame:0
          TX packets:465 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2823032 (2.6 MiB)  TX bytes:20362 (19.8 KiB)

# Create virtual machine（compute node is ESXI）
    [root@xianghui-10-9-1-141 SDK]# glance index
    ID                                   Name                           Disk Format          Container Format     Size
    ------------------------------------ ------------------------------ -------------------- -------------------- --------------
    94afb726-5984-4eb7-9d7d-d055437577d0 trend-thin                     vmdk                 bare                      268435456
    [root@xianghui-10-9-1-141 SDK]# neutron net-list
    +--------------------------------------+---------+--------------------------------------------------+
    | id                                   | name    | subnets                                          |
    +--------------------------------------+---------+--------------------------------------------------+
    | fffe6c04-1e09-4cb3-886b-6404c0729c1a | flat-50 | 19a19f9c-81b6-453a-9efc-541e1985b9d7 50.0.0.0/24 |
    +--------------------------------------+---------+--------------------------------------------------+
    [root@xianghui-10-9-1-141 SDK]# nova boot --image 94afb726-5984-4eb7-9d7d-d055437577d0 --flavor 2 --nic net-id=fffe6c04-1e09-4cb3-886b-6                                404c0729c1a test_esxi_1
    +--------------------------------------+--------------------------------------+
    | Property                             | Value                                |
    +--------------------------------------+--------------------------------------+
    | OS-EXT-STS:task_state                | scheduling                           |
    | image                                | trend-thin                           |
    | OS-EXT-STS:vm_state                  | building                             |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000001                    |
    | OS-SRV-USG:launched_at               | None                                 |
    | flavor                               | m1.small                             |
    | id                                   | cbdcd834-de07-4ea2-b34c-1c1c1cccd27f |
    | security_groups                      | [{u'name': u'default'}]              |
    | user_id                              | e37a1e89b1bf4f3d9a14aa7306f5b4de     |
    | OS-DCF:diskConfig                    | MANUAL                               |
    | accessIPv4                           |                                      |
    | accessIPv6                           |                                      |
    | progress                             | 0                                    |
    | OS-EXT-STS:power_state               | 0                                    |
    | OS-EXT-AZ:availability_zone          | nova                                 |
    | config_drive                         |                                      |
    | status                               | BUILD                                |
    | updated                              | 2013-11-08T09:06:36Z                 |
    | hostId                               |                                      |
    | OS-EXT-SRV-ATTR:host                 | None                                 |
    | OS-SRV-USG:terminated_at             | None                                 |
    | key_name                             | None                                 |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | None                                 |
    | name                                 | test_esxi_1                          |
    | adminPass                            | oVXwsp2AkdZ7                         |
    | tenant_id                            | 57f4a09b4bfb4c97bf9b216e6157ef12     |
    | created                              | 2013-11-08T09:06:36Z                 |
    | os-extended-volumes:volumes_attached | []                                   |
    | metadata                             | {}                                   |
    +--------------------------------------+--------------------------------------+
    [root@xianghui-10-9-1-141 SDK]# nova list
    +--------------------------------------+-------------+--------+------------+-------------+------------------+
    | ID                                   | Name        | Status | Task State | Power State | Networks         |
    +--------------------------------------+-------------+--------+------------+-------------+------------------+
    | cbdcd834-de07-4ea2-b34c-1c1c1cccd27f | test_esxi_1 | BUILD  | spawning   | NOSTATE     | flat-50=50.0.0.3 |
    +--------------------------------------+-------------+--------+------------+-------------+------------------+
    [root@xianghui-10-9-1-141 SDK]# nova list
    +--------------------------------------+-------------+--------+------------+-------------+------------------+
    | ID                                   | Name        | Status | Task State | Power State | Networks         |
    +--------------------------------------+-------------+--------+------------+-------------+------------------+
    | cbdcd834-de07-4ea2-b34c-1c1c1cccd27f | test_esxi_1 | ACTIVE | None       | Running     | flat-50=50.0.0.3 |
    +--------------------------------------+-------------+--------+------------+-------------+------------------+

    [root@xianghui-10-9-1-141 ˜]# ping 50.0.0.3
    PING 50.0.0.3 (50.0.0.3) 56(84) bytes of data.
    64 bytes from 50.0.0.3: icmp_seq=1 ttl=64 time=1.69 ms

    --- 50.0.0.3 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 957ms
    rtt min/avg/max/mdev = 1.691/1.691/1.691/0.000 ms

