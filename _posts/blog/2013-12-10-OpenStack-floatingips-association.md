---
layout: post
title: OpenStack-floatingips-association 
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Home</p>

# Enable `neutron-l3-agent`

Edit file `/etc/neutron/neutron.conf`

    service_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin

Restart `neutron-server`

    [root@xianghui-10-9-1-141 ~]# service neutron-server restart
    Stopping neutron:                                          [  OK  ]
    Starting neutron:                                          [  OK  ]

# Create router

    [root@xianghui-10-9-1-141 ~]# neutron router-create router1
    [root@xianghui-10-9-1-141 ~]# neutron router-list
    +--------------------------------------+---------+-----------------------+
    | id                                   | name    | external_gateway_info |
    +--------------------------------------+---------+-----------------------+
    | c36b384e-b1f5-45e5-bb4f-c3ed32885142 | router1 | null                  |
    +--------------------------------------+---------+-----------------------+

# Configure vlan

Edit file `/etc/neutron/plugins/ml2/ml2_conf.ini`

    [ml2]
    type_drivers = flat,vlan
    tenant_network_types = flat,vlan
    mechanism_drivers = openvswitch

    [ml2_type_vlan]
    network_vlan_ranges = physnet1:1000:2999

Restart Neutron services

    [root@xianghui-10-9-1-141 ~]# service neutron-server restart
    Stopping neutron:                                          [  OK  ]
    Starting neutron:                                          [  OK  ]
    [root@xianghui-10-9-1-141 ~]# service neutron-openvswitch-agent restart
    Stopping neutron-openvswitch-agent:                        [  OK  ]
    Starting neutron-openvswitch-agent:                        [  OK  ]

# Create external network

    [root@xianghui-10-9-1-141 ~]# neutron net-create ext_net --router:external=True
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | id                        | e27e26b1-8b31-4957-8ec0-d9b0b16d6368 |
    | name                      | ext_net                              |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  | 1000                                 |
    | router:external           | True                                 |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tenant_id                 | adc4e7a4effa44ffa3c6e48dd5a8555a     |
    +---------------------------+--------------------------------------+

External network ip range: 192.168.12.10-192.168.12.50

    [root@xianghui-10-9-1-141 ~]# neutron subnet-create ext_net --allocation-pool start=192.168.12.10,end=192.168.12.50 --no-gateway 192.168.12.0/24 --disable-dhcp
    Created a new subnet:
    +------------------+----------------------------------------------------+
    | Field            | Value                                              |
    +------------------+----------------------------------------------------+
    | allocation_pools | {"start": "192.168.12.10", "end": "192.168.12.50"} |
    | cidr             | 192.168.12.0/24                                    |
    | dns_nameservers  |                                                    |
    | enable_dhcp      | False                                              |
    | gateway_ip       |                                                    |
    | host_routes      |                                                    |
    | id               | 4705cdf1-d3ac-4b5e-817b-d547d22c641b               |
    | ip_version       | 4                                                  |
    | name             |                                                    |
    | network_id       | e27e26b1-8b31-4957-8ec0-d9b0b16d6368               |
    | tenant_id        | adc4e7a4effa44ffa3c6e48dd5a8555a                   |
    +------------------+----------------------------------------------------+

List all networks, one is external network the other is internal network

    [root@xianghui-10-9-1-141 ~]# neutron net-list
    +--------------------------------------+---------+------------------------------------------------------+
    | id                                   | name    | subnets                                              |
    +--------------------------------------+---------+------------------------------------------------------+
    | c8c64b0b-db49-44e4-bcb4-54fd86873631 | flat-51 | 5c62752f-27ba-4d38-9702-2ca17ec2741d 51.0.0.0/24     |
    | e27e26b1-8b31-4957-8ec0-d9b0b16d6368 | ext_net | 4705cdf1-d3ac-4b5e-817b-d547d22c641b 192.168.12.0/24 |
    +--------------------------------------+---------+------------------------------------------------------+

# Binding networks to the router

    [root@xianghui-10-9-1-141 ~]# neutron router-list
    +--------------------------------------+---------+-----------------------+
    | id                                   | name    | external_gateway_info |
    +--------------------------------------+---------+-----------------------+
    | c36b384e-b1f5-45e5-bb4f-c3ed32885142 | router1 | null                  |
    +--------------------------------------+---------+-----------------------+
    [root@xianghui-10-9-1-141 ~]# neutron router-gateway-set c36b384e-b1f5-45e5-bb4f-c3ed32885142 e27e26b1-8b31-4957-8ec0-d9b0b16d6368
    Set gateway for router c36b384e-b1f5-45e5-bb4f-c3ed32885142
    [root@xianghui-10-9-1-141 ~]# neutron router-interface-add c36b384e-b1f5-45e5-bb4f-c3ed32885142 5c62752f-27ba-4d38-9702-2ca17ec2741d
    Added interface 0d06055b-2f31-4d8e-b8da-e048d76a07cc to router c36b384e-b1f5-45e5-bb4f-c3ed32885142.

# List router interfaces    
    
    [root@xianghui-10-9-1-141 ~]# neutron router-list
    +--------------------------------------+---------+-----------------------------------------------------------------------------+
    | id                                   | name    | external_gateway_info                                                       |
    +--------------------------------------+---------+-----------------------------------------------------------------------------+
    | c36b384e-b1f5-45e5-bb4f-c3ed32885142 | router1 | {"network_id": "e27e26b1-8b31-4957-8ec0-d9b0b16d6368", "enable_snat": true} |
    +--------------------------------------+---------+-----------------------------------------------------------------------------+

# Configure neutron-l3-agent
Edit file `/etc/neutron/l3_agent.ini`

    [DEFAULT]
    use_namespaces = False
    router_id = c36b384e-b1f5-45e5-bb4f-c3ed32885142
    external_network_bridge = br-phy
    enable_metadata_proxy = False

Restart neutron-l3-agent

    [root@xianghui-10-9-1-141 ~]# service neutron-l3-agent start
    Starting neutron-l3-agent:                                 [  OK  ]
    [root@xianghui-10-9-1-141 ~]# service neutron-l3-agent status
    neutron-l3-agent (pid  14337) is running...

Checking network interfaces are being updated

    [root@xianghui-10-9-1-141 ~]# ifconfig |grep ..
    qg-79663b18-dc Link encap:Ethernet  HWaddr FA:16:3E:46:66:3A
          inet addr:192.168.12.10  Bcast:192.168.12.255  Mask:255.255.255.0
          inet6 addr: fe80::74fb:ff:fe19:e459/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:89 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:6028 (5.8 KiB)  TX bytes:468 (468.0 b)

    qr-0d06055b-2f Link encap:Ethernet  HWaddr FA:16:3E:D7:F4:19
          inet addr:51.0.0.1  Bcast:51.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::4cea:e7ff:feb9:e6b8/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:92 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:6266 (6.1 KiB)  TX bytes:468 (468.0 b)

# Create a floatingip and associate it to the vm(51.0.0.3)

    [root@xianghui-10-9-1-141 ~]# vi /etc/nova/nova.conf
    [DEFAULT]
    default_floating_pool = ext_net

    [root@xianghui-10-9-1-141 ~]# neutron floatingip-create e27e26b1-8b31-4957-8ec0-d9b0b16d6368
    Created a new floatingip:
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | fixed_ip_address    |                                      |
    | floating_ip_address | 192.168.12.11                        |
    | floating_network_id | e27e26b1-8b31-4957-8ec0-d9b0b16d6368 |
    | id                  | f8b48ab7-ea51-4f29-bc84-0ab179808dbb |
    | port_id             |                                      |
    | router_id           |                                      |
    | tenant_id           | adc4e7a4effa44ffa3c6e48dd5a8555a     |
    +---------------------+--------------------------------------+

    [root@xianghui-10-9-1-141 ~]# neutron floatingip-list
    +--------------------------------------+------------------+---------------------+---------+
    | id                                   | fixed_ip_address | floating_ip_address | port_id |
    +--------------------------------------+------------------+---------------------+---------+
    | f8b48ab7-ea51-4f29-bc84-0ab179808dbb |                  | 192.168.12.11       |         |
    +--------------------------------------+------------------+---------------------+---------+

    [root@xianghui-10-9-1-141 ~]# neutron port-list
    +--------------------------------------+------+-------------------+--------------------------------------------------------------------------------------+
    | id                                   | name | mac_address       | fixed_ips                                                                            |
    +--------------------------------------+------+-------------------+--------------------------------------------------------------------------------------+
    | 0d06055b-2f31-4d8e-b8da-e048d76a07cc |      | fa:16:3e:d7:f4:19 | {"subnet_id": "5c62752f-27ba-4d38-9702-2ca17ec2741d", "ip_address": "51.0.0.1"}      |
    | 224838e1-031a-4a7b-86a4-4ad67353fe6b |      | fa:16:3e:de:f8:21 | {"subnet_id": "4705cdf1-d3ac-4b5e-817b-d547d22c641b", "ip_address": "192.168.12.11"} |
    | 3413c6bd-7d3e-45a8-ad7c-1412d356a9ee |      | fa:16:3e:39:0b:da | {"subnet_id": "5c62752f-27ba-4d38-9702-2ca17ec2741d", "ip_address": "51.0.0.7"}      |
    | 5e80c1dd-9659-4642-a6f4-36e36e51d805 |      | fa:16:3e:dd:b6:0f | {"subnet_id": "5c62752f-27ba-4d38-9702-2ca17ec2741d", "ip_address": "51.0.0.5"}      |
    | 79663b18-dc5f-4cab-9343-dfc473fd3134 |      | fa:16:3e:46:66:3a | {"subnet_id": "4705cdf1-d3ac-4b5e-817b-d547d22c641b", "ip_address": "192.168.12.10"} |
    | aded2aae-4e93-4f7f-b79f-c7ee5881c2f5 |      | fa:16:3e:e9:93:36 | {"subnet_id": "5c62752f-27ba-4d38-9702-2ca17ec2741d", "ip_address": "51.0.0.4"}      |
    | b0797fe6-b799-41ea-86d0-9d9bfa0b2eb9 |      | fa:16:3e:f8:93:9d | {"subnet_id": "5c62752f-27ba-4d38-9702-2ca17ec2741d", "ip_address": "51.0.0.3"}      |
    | dac8ab52-b9b2-4595-94cd-0aac192e7269 |      | fa:16:3e:05:51:7d | {"subnet_id": "5c62752f-27ba-4d38-9702-2ca17ec2741d", "ip_address": "51.0.0.2"}      |
    +--------------------------------------+------+-------------------+--------------------------------------------------------------------------------------+

    [root@xianghui-10-9-1-141 ~]# neutron floatingip-associate f8b48ab7-ea51-4f29-bc84-0ab179808dbb b0797fe6-b799-41ea-86d0-9d9bfa0b2eb9
Associated floatingip f8b48ab7-ea51-4f29-bc84-0ab179808dbb

    [root@xianghui-10-9-1-141 ~]# neutron floatingip-list
    +--------------------------------------+------------------+---------------------+--------------------------------------+
    | id                                   | fixed_ip_address | floating_ip_address | port_id                              |
    +--------------------------------------+------------------+---------------------+--------------------------------------+
    | f8b48ab7-ea51-4f29-bc84-0ab179808dbb | 51.0.0.3         | 192.168.12.11       | b0797fe6-b799-41ea-86d0-9d9bfa0b2eb9 |
    +--------------------------------------+------------------+---------------------+--------------------------------------+

Test the vm can be accessed

    [root@xianghui-10-9-1-141 ~]# ping 192.168.12.11
    PING 192.168.12.11 (192.168.12.11) 56(84) bytes of data.
    64 bytes from 192.168.12.11: icmp_seq=1 ttl=64 time=1.83 ms
    64 bytes from 192.168.12.11: icmp_seq=2 ttl=64 time=0.581 ms
    64 bytes from 192.168.12.11: icmp_seq=3 ttl=64 time=0.574 ms

    --- 192.168.12.11 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 2309ms
    rtt min/avg/max/mdev = 0.574/0.995/1.830/0.590 ms

Below iptable rule are being written

    [root@xianghui-10-9-1-141 ~]# iptables -t nat -L
    Chain neutron-l3-agent-PREROUTING (1 references)
    num  target     prot opt source               destination
    1    DNAT       all  --  0.0.0.0/0            192.168.12.11       to:51.0.0.3

    Chain neutron-l3-agent-float-snat (1 references)
    num  target     prot opt source               destination
    1    SNAT       all  --  51.0.0.3             0.0.0.0/0           to:192.168.12.11

The database is updated

    [root@xianghui-10-9-1-141 ~]# nova list
    +--------------------------------------+----------------+--------+------------+-------------+---------------------------------+
    | ID                                   | Name           | Status | Task State | Power State | Networks                        |
    +--------------------------------------+----------------+--------+------------+-------------+---------------------------------+
    | fc47aaf0-ef35-4547-918f-3e2fc9c3e486 | test_local_1   | ERROR  | None       | NOSTATE     |                                 |
    | 1ff6e84b-843a-46f0-851a-66adb5c77290 | test_vcenter_1 | ACTIVE | None       | Running     | flat-51=51.0.0.3, 192.168.12.11 |
    +--------------------------------------+----------------+--------+------------+-------------+---------------------------------+

