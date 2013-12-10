---
layout: post
title: Configure OpenStack Neutron flat with vcenter5.5
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Home</p>

OS: RHEL6.5

OpenStack:  Icehouse

Controller: KVM

Compute: Vcenter 5.5

Referrence:

[VMware-vSphere-support][1]
[DeveloperGuide][2]
[VMWare_networking][3]

# Install vcenter
[vSphere Web Services SDK 5.5 Release Notes][4]
[VMware vSphere][5]
[vSphere 5.5 Documentation Center][6]
[Downloads | VMware Communities][7]

# Install and configure OpenStack(keystone/glance/nova/neutron)
Recommend `devstack`/`packstack`
<pre><code>
[root@xianghui-10-9-1-141 ~]# keystone service-list
+----------------------------------+------------+----------------+------------------------------+
|                id                |    name    |      type      |         description          |
+----------------------------------+------------+----------------+------------------------------+
| 8e9e6a50b26b42e49d8060f4da9611b0 | ceilometer |    metering    | OpenStack Ceilometer service |
| 48de54344f004595a9123a296076288f |   cinder   |     volume     |        Cinder Service        |
| ce49fbd7917d4efda09a0f181af895e0 |   glance   |     image      |     Glance Image Service     |
| 107258af226f4ed9a6a014eea28b7836 |    heat    | orchestration  |           Heat API           |
| 324937dab6064b0eae5032a9f26be32b |  heat-cfn  | cloudformation |   Heat CloudFormation API    |
| 097e2bd5b0b94f85aa40bf70a0420cb1 |  keystone  |    identity    |  Keystone Identity Service   |
| 78040099207c4a0a9b1697017ec643e7 |  neutron   |    network     | OpenStack Networking service |
| dcdb941cb9b84a26b7052fd125de85ab |    nova    |    compute     |     Nova Compute Service     |
| 5f40d29454ab43c68fb95a5f74364d7f |   swift    |  object-store  |    Object Storage Service    |
+----------------------------------+------------+----------------+------------------------------+
</code></pre>

# Download vmware flat type vmdk image
<pre><code>
[root@xianghui-10-9-1-141 ~]# wget  http://partnerweb.vmware.com/programs/vmdkimage/trend-tinyvm1-flat.vmdk
[root@xianghui-10-9-1-141 ~]# glance image-create --name trend-thin --is-public=True --container-format=bare --disk-format=vmdk --property vmware_disktype="thin" --property vmware_adaptertype="ide" < trend-tinyvm1-flat.vmdk
+-------------------------------+--------------------------------------+
| Property                      | Value                                |
+-------------------------------+--------------------------------------+
| Property 'vmware_adaptertype' | ide                                  |
| Property 'vmware_disktype'    | thin                                 |
| checksum                      | 10477e5a7c756f77974d5dfec2a7afa1     |
| container_format              | bare                                 |
| created_at                    | 2013-11-18T03:11:04                  |
| deleted                       | False                                |
| deleted_at                    | None                                 |
| disk_format                   | vmdk                                 |
| id                            | 2c1b230e-c338-4572-8f1b-183ef38231b9 |
| is_public                     | True                                 |
| min_disk                      | 0                                    |
| min_ram                       | 0                                    |
| name                          | trend-thin                           |
| owner                         | adc4e7a4effa44ffa3c6e48dd5a8555a     |
| protected                     | False                                |
| size                          | 268435456                            |
| status                        | active                               |
| updated_at                    | 2013-11-18T03:11:05                  |
+-------------------------------+--------------------------------------+
</code></pre>

# Bring up virtual bridge "br-phy" and bind with eth1 to create neutron flat network
    [root@xianghui-10-9-1-141 ~]# ovs-vsctl add-br br-phy
    [root@xianghui-10-9-1-141 ~]# ovs-vsctl add-port br-phy eth1
    [root@xianghui-10-9-1-141 ~]# ovs-vsctl show
    2b7f8c35-c900-4a96-802a-5a898aad8226
    Bridge br-phy
        Port br-phy
            Interface br-phy
                type: internal
        Port "eth1"
            Interface "eth1"
    Bridge br-int
        Port br-int
            Interface br-int
                type: internal
        Port "tapebf875a8-22"
            tag: 1
            Interface "tapebf875a8-22"
    ovs_version: "1.10.0"


    [root@xianghui-10-9-1-141 ~]# ifconfig eth1 up
    [root@xianghui-10-9-1-141 ~]# ifconfig eth1
    eth1      Link encap:Ethernet  HWaddr 00:50:56:97:13:9F
              inet6 addr: fe80::250:56ff:fe97:139f/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:38 errors:0 dropped:0 overruns:0 frame:0
              TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:2280 (2.2 KiB)  TX bytes:238 (238.0 b)


Check network nic
    [root@xianghui-10-9-1-141 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
    DEVICE="eth0"
    BOOTPROTO="static"
    DNS1="10.9.0.14"
    GATEWAY="10.9.0.1"
    IPADDR="10.9.1.141"
    IPV6INIT="no"
    MTU="1500"
    NETMASK="255.255.252.0"
    NM_CONTROLLED="no"
    ONBOOT="yes"
    TYPE="Ethernet"

    [root@xianghui-10-9-1-141 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth1
    DEVICE=eth1
    TYPE=Ethernet
    ONBOOT=no
    NM_CONTROLLED=no
    BOOTPROTO=dhcp

# Configure Neutron 

Edit file `/etc/neutron/neutron.conf`
<pre><code>
[Default]
notification_driver = neutron.openstack.common.notifier.rpc_notifier
auth_strategy = keystone
rpc_backend = neutron.openstack.common.rpc.impl_qpid
qpid_hostname = localhost
verbose = True
allow_overlapping_ips = False
agent_down_time = 20
rpc_thread_pool_size = 128
rpc_conn_pool_size = 60
rpc_response_timeout = 600
service_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin

[quotas]
quota_driver = neutron.db.quota_db.DbQuotaDriver

[keystone_authtoken]
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = neutron
admin_password = neutron
signing_dir = /var/lib/neutron/keystone-signing

[database]
connection = mysql://neutron:neutron@$host_ip/ovs_neutron

[service_providers]
service_provider=LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default

[AGENT]
root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
report_interval = 15
</code></pre>

Edit file `/etc/neutron/plugins/ml2/ml2_conf.ini`
<pre><code>
[ml2]
type_drivers = flat
tenant_network_types = flat
mechanism_drivers = openvswitch

[ml2_type_flat]
flat_networks = physnet1
</code></pre>

Edit file `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini`
<pre><code>
[ovs]
bridge_mappings = physnet1:br-phy
[SECURITYGROUP]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
</code></pre>

Edit file `/etc/neutron/dhcp_agent.ini`
<pre><code>
[DEFAULT]
use_namespaces = False
auth_url = Page on Localhost:5000
admin_username = neutron
admin_password = neutron
admin_tenant_name = service
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
</code></pre>

Restart neutron services:
    [root@xianghui-10-9-1-141 ~]# service neutron-server restart
    Stopping neutron:                                          [  OK  ]
    Starting neutron:                                          [  OK  ]
    [root@xianghui-10-9-1-141 ~]# service neutron-openvswitch-agent restart
    Stopping neutron-openvswitch-agent:                        [  OK  ]
    Starting neutron-openvswitch-agent:                        [  OK  ]
    [root@xianghui-10-9-1-141 ~]# service neutron-dhcp-agent restart
    Stopping neutron-dhcp-agent:                               [  OK  ]
    Starting neutron-dhcp-agent:                               [  OK  ]
    [root@xianghui-10-9-1-141 ~]# service dnsmasq  stop
    Shutting down Lightweight caching nameserver (dnsmasq):    [  OK  ]

# Create flat network, ip range 50.0.0.0/24
<pre><code>
[root@xianghui-10-9-1-141 ~]# neutron net-create flat-50 --provider:network_type flat --provider:physical_network physnet1
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 055200c9-b0ca-4ad8-ae00-4ae4b37b8898 |
| name                      | flat-50                              |
| provider:network_type     | flat                                 |
| provider:physical_network | physnet1                             |
| provider:segmentation_id  |                                      |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | adc4e7a4effa44ffa3c6e48dd5a8555a     |
+---------------------------+--------------------------------------+

[root@xianghui-10-9-1-141 ~]# neutron subnet-create flat-50 50.0.0.0/24
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
| id               | 1bea52fd-31c8-4753-abe0-65e4ed398b5d       |
| ip_version       | 4                                          |
| name             |                                            |
| network_id       | 055200c9-b0ca-4ad8-ae00-4ae4b37b8898       |
| tenant_id        | adc4e7a4effa44ffa3c6e48dd5a8555a           |
+------------------+--------------------------------------------+

</code></pre>
<pre><code>
[root@xianghui-10-9-1-141 ~]# service neutron-dhcp-agent restart
Stopping neutron-dhcp-agent:                               [  OK  ]
Starting neutron-dhcp-agent:                               [  OK  ]
</code></pre>
Check dhcp process
<pre><code>
[root@xianghui-10-9-1-141 ~]# ps -ef|grep dnsmasq
nobody   29961     1  0 21:34 ?        00:00:00 /usr/sbin/dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=tap4eb5f850-5a --except-interface=lo --pid-file=/var/lib/neutron/dhcp/055200c9-b0ca-4ad8-ae00-4ae4b37b8898/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/055200c9-b0ca-4ad8-ae00-4ae4b37b8898/host --dhcp-optsfile=/var/lib/neutron/dhcp/055200c9-b0ca-4ad8-ae00-4ae4b37b8898/opts --leasefile-ro --dhcp-range=set:tag0,50.0.0.0,static,86400s --dhcp-lease-max=256 --conf-file= --domain=openstacklocal
root     29983 10373  0 21:34 pts/1    00:00:00 grep dnsmasq
</code></pre>
Check networking
<pre><code>
[root@xianghui-10-9-1-141 ~]# ifconfig
br-int    Link encap:Ethernet  HWaddr F6:BB:6E:3F:75:40
          inet6 addr: fe80::94c3:13ff:fe4a:200/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:37900 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:8092480 (7.7 MiB)  TX bytes:468 (468.0 b)

br-phy    Link encap:Ethernet  HWaddr 00:50:56:97:13:9F
          inet6 addr: fe80::c839:2eff:fe65:9909/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:6331 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:438956 (428.6 KiB)  TX bytes:468 (468.0 b)

eth0      Link encap:Ethernet  HWaddr 52:54:00:09:01:8D
          inet addr:10.9.1.141  Bcast:10.9.3.255  Mask:255.255.252.0
          inet6 addr: fe80::5054:ff:fe09:18d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:5904146 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3462485 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1646733554 (1.5 GiB)  TX bytes:4601743281 (4.2 GiB)

eth1      Link encap:Ethernet  HWaddr 00:50:56:97:13:9F
          inet6 addr: fe80::250:56ff:fe97:139f/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:6328 errors:0 dropped:0 overruns:0 frame:0
          TX packets:9 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:438726 (428.4 KiB)  TX bytes:698 (698.0 b)

int-br-phy Link encap:Ethernet  HWaddr 6A:47:02:60:7C:84
          inet6 addr: fe80::6847:2ff:fe60:7c84/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2016 errors:0 dropped:0 overruns:0 frame:0
          TX packets:11 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:138832 (135.5 KiB)  TX bytes:846 (846.0 b)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:1946068 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1946068 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:4450317873 (4.1 GiB)  TX bytes:4450317873 (4.1 GiB)

phy-br-phy Link encap:Ethernet  HWaddr EA:0F:66:07:06:18
          inet6 addr: fe80::e80f:66ff:fe07:618/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:11 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2016 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:846 (846.0 b)  TX bytes:138832 (135.5 KiB)

tap4eb5f850-5a Link encap:Ethernet  HWaddr FA:16:3E:08:F4:39
          inet addr:50.0.0.2  Bcast:50.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::649a:e8ff:fe0d:f4af/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:136 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:10056 (9.8 KiB)  TX bytes:468 (468.0 b)
</code></pre>

# Create a port group named [br-int][8]

# Configure Nova file `/etc/nova/nova.conf` to load vcenter driver

<pre><code>
[DEFAULT]
log_dir = /var/log/nova
state_path = /var/lib/nova
lock_path = /var/lib/nova/tmp
dhcpbridge = /usr/bin/nova-dhcpbridge
dhcpbridge_flagfile = /etc/nova/nova.conf
injected_network_template = /usr/share/nova/interfaces.template
libvirt_inject_partition = -1
network_manager = nova.network.manager.FlatDHCPManager
sql_connection = mysql://nova:nova@10.9.1.141/nova?charset=utf8
#compute_driver = libvirt.LibvirtDriver
compute_driver = vmwareapi.VMwareVCDriver
rpc_backend = nova.openstack.common.rpc.impl_qpid
enabled_apis = osapi_compute,metadata
verbose = true
auth_strategy = keystone
auth_uri = Page on 9
api_paste_config = /etc/nova/api-paste.ini
rpc_response_timeout = 960
rpc_conn_pool_size = 60
rpc_thread_pool_size = 2048
firewall_driver = nova.virt.firewall.NoopFirewallDriver
libvirt_type = kvm
image_service = nova.image.glance.GlanceImageService
glance_api_servers = 10.9.1.141:9292
network_api_class = nova.network.neutronv2.api.API
neutron_admin_username = neutron
neutron_admin_password = neutron
neutron_admin_auth_url = Page on Localhost:5000
neutron_auth_strategy = keystone
neutron_admin_tenant_name = service
neutron_url = Page on Localhost:9696
libvirt_vif_driver = nova.virt.libvirt.vif.LibvirtGenericVIFDriver
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
security_group_api = neutron
linuxnet_ovs_integration_bridge = br-int
neutron_ovs_bridge = br-int

[vmware]
host_ip = 10.9.1.43
host_username = administrator@vsphere.local
host_password = Passw0rd$$
cluster_name = cluster01
wsdl_location=file:///var/lib/SDK/wsdl/vim25/vimService.wsdl

[keystone_authtoken]
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = nova
auth_version = v2.0

</code></pre>
<pre><code>
[root@xianghui-10-9-1-141 ~]# service openstack-nova-compute restart
Stopping openstack-nova-compute:                           [  OK  ]
Starting openstack-nova-compute:                           [  OK  ]
</code></pre>

The vcenter driver loaded successfully if has below logs:
<pre><code>
[root@xianghui-10-9-1-141 ~]# vi /var/log/nova/compute.log
2013-11-17 22:05:02.619 5457 INFO nova.virt.driver [-] NV-91AF767 Loading compute driver 'vmwareapi.VMwareVCDriver'
2013-11-17 22:05:15.810 5457 INFO nova.openstack.common.rpc.impl_qpid [req-9283ef4b-8f46-4d97-b7a2-bf216496b5cf None None] NV-1A047FB Connected to AMQP server on localhost:5672
2013-11-17 22:05:15.820 5457 INFO nova.openstack.common.rpc.impl_qpid [req-9283ef4b-8f46-4d97-b7a2-bf216496b5cf None None] NV-1A047FB Connected to AMQP server on localhost:5672
2013-11-17 22:05:15.863 5457 AUDIT nova.service [-] Starting compute node 
2013-11-17 22:05:16.754 5457 AUDIT nova.compute.resource_tracker [-] NV-313322F Auditing locally available compute resources
2013-11-17 22:05:18.056 5457 AUDIT nova.compute.resource_tracker [-] NV-E42374F Free ram (MB): 361830
2013-11-17 22:05:18.057 5457 AUDIT nova.compute.resource_tracker [-] NV-5D132CB Free disk (GB): 199
2013-11-17 22:05:18.057 5457 AUDIT nova.compute.resource_tracker [-] NV-97E8645 Free VCPUS: 48
2013-11-17 22:05:18.121 5457 INFO nova.compute.resource_tracker [-] Compute_service record created for xianghui-10-9-1-141(cluster01)
2013-11-17 22:05:18.157 5457 AUDIT nova.compute.manager [-] Deleting orphan compute node 1
2013-11-17 22:05:18.185 5457 INFO nova.openstack.common.rpc.impl_qpid [-] NV-1A047FB Connected to AMQP server on localhost:5672
</code></pre>

Check nova services

<pre><code>
[root@xianghui-10-9-1-141 ~]# nova-manage service list
Binary           Host                                 Zone             Status     State Updated_At
nova-cells       xianghui-10-9-1-141   internal         enabled    :-)   2013-11-18 05:09:04
nova-conductor   xianghui-10-9-1-141   internal         enabled    :-)   2013-11-18 05:09:04
nova-console     xianghui-10-9-1-141   internal         enabled    :-)   2013-11-18 05:09:04
nova-consoleauth xianghui-10-9-1-141   internal         enabled    :-)   2013-11-18 05:09:05
nova-scheduler   xianghui-10-9-1-141   internal         enabled    :-)   2013-11-18 05:09:06
nova-cert        xianghui-10-9-1-141   internal         enabled    :-)   2013-11-18 05:09:05
nova-compute     xianghui-10-9-1-141   nova             enabled    :-)   2013-11-18 05:08:58
</code></pre>
<pre><code>
[root@xianghui-10-9-1-141 ~]# glance index
ID                                   Name                           Disk Format          Container Format     Size
------------------------------------ ------------------------------ -------------------- -------------------- --------------
2c1b230e-c338-4572-8f1b-183ef38231b9 trend-thin                     vmdk                 bare                      268435456
[root@xianghui-10-9-1-141 ~]# neutron net-list
+--------------------------------------+---------+--------------------------------------------------+
| id                                   | name    | subnets                                          |
+--------------------------------------+---------+--------------------------------------------------+
| 055200c9-b0ca-4ad8-ae00-4ae4b37b8898 | flat-50 | 1bea52fd-31c8-4753-abe0-65e4ed398b5d 50.0.0.0/24 |
+--------------------------------------+---------+--------------------------------------------------+
</code></pre>
Create virtual machine
<pre><code>
[root@xianghui-10-9-1-141 ~]# nova boot --image 2c1b230e-c338-4572-8f1b-183ef38231b9 --flavor 2 --nic net-id=055200c9-b0ca-4ad8-ae00-4ae4b37b8898 test_vcenter_1

+--------------------------------------+--------------------------------------+
| Property                             | Value                                |
+--------------------------------------+--------------------------------------+
| OS-EXT-STS:task_state                | scheduling                           |
| image                                | trend-thin                           |
| OS-EXT-STS:vm_state                  | building                             |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000001                    |
| OS-SRV-USG:launched_at               | None                                 |
| flavor                               | m1.small                             |
| id                                   | 470f3962-b95f-42d2-993b-89b35261c0d9 |
| security_groups                      | [{u'name': u'default'}]              |
| user_id                              | cd781463be9d4a4ebbcf239560df056c     |
| OS-DCF:diskConfig                    | MANUAL                               |
| accessIPv4                           |                                      |
| accessIPv6                           |                                      |
| progress                             | 0                                    |
| OS-EXT-STS:power_state               | 0                                    |
| OS-EXT-AZ:availability_zone          | nova                                 |
| config_drive                         |                                      |
| status                               | BUILD                                |
| updated                              | 2013-11-18T05:10:24Z                 |
| hostId                               |                                      |
| OS-EXT-SRV-ATTR:host                 | None                                 |
| OS-SRV-USG:terminated_at             | None                                 |
| key_name                             | None                                 |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | None                                 |
| name                                 | test_vcenter_1                       |
| adminPass                            | F3mmiQp93nMT                         |
| tenant_id                            | adc4e7a4effa44ffa3c6e48dd5a8555a     |
| created                              | 2013-11-18T05:10:24Z                 |
| os-extended-volumes:volumes_attached | []                                   |
| metadata                             | {}                                   |
+--------------------------------------+--------------------------------------+
</code></pre>

<pre><code>
[root@xianghui-10-9-1-141 ~]# nova list
+--------------------------------------+----------------+--------+------------+-------------+------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks         |
+--------------------------------------+----------------+--------+------------+-------------+------------------+
| 470f3962-b95f-42d2-993b-89b35261c0d9 | test_vcenter_1 | BUILD  | spawning   | NOSTATE     | flat-50=50.0.0.3 |
+--------------------------------------+----------------+--------+------------+-------------+------------------+

[root@xianghui-10-9-1-141 ~]# nova list
+--------------------------------------+----------------+--------+------------+-------------+------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks         |
+--------------------------------------+----------------+--------+------------+-------------+------------------+
| 470f3962-b95f-42d2-993b-89b35261c0d9 | test_vcenter_1 | ACTIVE | None       | Running     | flat-50=50.0.0.3 |
+--------------------------------------+----------------+--------+------------+-------------+------------------+
</code></pre>

[1]:  https://wiki.openstack.org/wiki/VMware-vSphere-support
[2]:  https://wiki.openstack.org/wiki/NovaVMware/DeveloperGuide
[3]:  http://docs.openstack.org/trunk/config-reference/content/vmware.html#VMWare_networking
[4]:  http://www.vmware.com/support/developer/vc-sdk/wssdk_5_5_releasenotes.html
[5]:  https://my.vmware.com/cn/web/vmware/details?downloadGroup=WEBSDK550&productId=353
[6]:  http://pubs.vmware.com/vsphere-55/index.jsp?topic=%2Fcom.vmware.wssdk.pg.doc%2FPG_Preface.html
[7]:  https://communities.vmware.com/community/vmtn/developer/downloads
[8]:  http://panghua.github.io/2013/12/10/Using-ESXI-as-OpenStack-Compute_node.html
