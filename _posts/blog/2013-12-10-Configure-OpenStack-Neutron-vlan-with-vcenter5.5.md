---
layout: post
title: Configure OpenStack Neutron vlan network with vcenter5.5
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Home</p>

OS: RHEL6.5

OpenStack: Icehouse

Controller: KVM

Compute: Vmware vcenter 5.5

Referrence:

[VMware-vSphere-support][1]

[DeveloperGuide][2]

[VMWare_networking][3]

# Install vcenter

[vSphere Web Services SDK 5.5 Release Notes][4]

[VMware vSphere][5]

[vSphere 5.5 Documentation Center][6]

[Downloads | VMware Communities][7]

# Install and configure OpenStack(`packstack` `devstack`)

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

# Download vmware flat type vmdk image

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

# Create ovs bridge `br-eth1` and binding to the `eth1`

    [root@xianghui-10-9-1-141 ~]# ovs-vsctl add-br br-eth1
    [root@xianghui-10-9-1-141 ~]# ovs-vsctl add-port br-eth1 eth1
    [root@xianghui-10-9-1-141 ~]# ifconfig eth1 up
    [root@xianghui-10-9-1-141 ~]# ifconfig eth1
    eth1      Link encap:Ethernet  HWaddr 00:50:56:97:13:9F
          inet6 addr: fe80::250:56ff:fe97:139f/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:38 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2280 (2.2 KiB)  TX bytes:238 (238.0 b)

# Configure Neutron vlan network in the controller node
Edit file `/etc/neutron/neutron.conf`
<pre><code>
[DEFAULT]
lock_path = $state_path/lock
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
connection = mysql://neutron:neutron@$host/ovs_neutron
[service_providers]
service_provider=LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
[AGENT]
root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
report_interval = 15
</code></pre>

Edit file `/etc/neutron/plugins/ml2/ml2_conf.ini`

    [ml2]
    type_drivers = vlan,flat
    tenant_network_types = vlan,flat
    mechanism_drivers = openvswitch
    [ml2_type_vlan]
    network_vlan_ranges = physnet1:10:2999

Edit file `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini`

    [ovs]
    bridge_mappings = physnet1:br-eth1
    [SECURITYGROUP]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

Restart services

    [root@xianghui-10-9-1-141 ~]# service neutron-server restart
    [root@xianghui-10-9-1-141 ~]# service neutron-openvswitch-agent restart

# Create vlan network

    [root@xianghui-10-9-1-141 ~]# neutron net-create vlan-109 --provider:network_type vlan --provider:physical_network physnet1 --provider:segmentation_id 109
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | id                        | 75f4506d-314c-4814-9afe-fa5c935a2b17 |
    | name                      | vlan-109                             |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  | 109                                  |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tenant_id                 | adc4e7a4effa44ffa3c6e48dd5a8555a     |
    +---------------------------+--------------------------------------+

    [root@xianghui-10-9-1-141 ~]# neutron subnet-create vlan-109 90.0.0.0/24
    Created a new subnet:
    +------------------+--------------------------------------------+
    | Field            | Value                                      |
    +------------------+--------------------------------------------+
    | allocation_pools | {"start": "90.0.0.2", "end": "90.0.0.254"} |
    | cidr             | 90.0.0.0/24                                |
    | dns_nameservers  |                                            |
    | enable_dhcp      | True                                       |
    | gateway_ip       | 90.0.0.1                                   |
    | host_routes      |                                            |
    | id               | f5c20675-aa7d-4912-8213-20b04705811a       |
    | ip_version       | 4                                          |
    | name             |                                            |
    | network_id       | 75f4506d-314c-4814-9afe-fa5c935a2b17       |
    | tenant_id        | adc4e7a4effa44ffa3c6e48dd5a8555a           |
    +------------------+--------------------------------------------+

# Ensure vlan network configured correctly
Checking dhcp

    [root@xianghui-10-9-1-141 ~]# service neutron-dhcp-agent restart
    Stopping neutron-dhcp-agent:                               [  OK  ]
    Starting neutron-dhcp-agent:                               [  OK  ]

    [root@xianghui-10-9-1-141 ~]# ps -ef|grep dnsmasq
    nobody   28188     1  0 03:30 ?        00:00:00 /usr/sbin/dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=tapa97dcd80-16 --except-interface=lo --pid-file=/var/lib/neutron/dhcp/75f4506d-314c-4814-9afe-fa5c935a2b17/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/75f4506d-314c-4814-9afe-fa5c935a2b17/host --dhcp-optsfile=/var/lib/neutron/dhcp/75f4506d-314c-4814-9afe-fa5c935a2b17/opts --leasefile-ro --dhcp-range=set:tag0,90.0.0.0,static,86400s --dhcp-lease-max=256 --conf-file= --domain=openstacklocal
    root     28196 14491  0 03:30 pts/6    00:00:00 grep dnsmasq

Checking ovs flow and interfaces

    [root@xianghui-10-9-1-141 ~]# ovs-vsctl show
    2b7f8c35-c900-4a96-802a-5a898aad8226
    Bridge "br-eth1"
        Port "phy-br-eth1"
            Interface "phy-br-eth1"
        Port "eth1"
            Interface "eth1"
        Port "br-eth1"
            Interface "br-eth1"
                type: internal
    Bridge br-int
        Port int-br-phy
            Interface int-br-phy
        Port "int-br-eth1"
            Interface "int-br-eth1"
        Port br-int
            Interface br-int
                type: internal
        Port "tapa97dcd80-16"
            tag: 1
            Interface "tapa97dcd80-16"
                type: internal
    ovs_version: "1.10.0"

    [root@xianghui-10-9-1-141 ~]# ovs-ofctl dump-ports-desc br-int
    OFPST_PORT_DESC reply (xid=0x2):
    24(tapa97dcd80-16): addr:00:50:56:97:13:9f
        config:     0
        state:      0
        speed: 0 Mbps now, 0 Mbps max
    26(int-br-phy): addr:a6:14:ba:99:9c:c7
        config:     0
        state:      0
        current:    10GB-FD COPPER
        speed: 10000 Mbps now, 0 Mbps max
    28(int-br-eth1): addr:72:c1:4d:22:1b:e0
        config:     0
        state:      0
        current:    10GB-FD COPPER
        speed: 10000 Mbps now, 0 Mbps max
    LOCAL(br-int): addr:00:50:56:97:13:9f
        config:     0
        state:      0
        speed: 0 Mbps now, 0 Mbps max

    [root@xianghui-10-9-1-141 ~]# ovs-ofctl dump-ports-desc br-eth1
    OFPST_PORT_DESC reply (xid=0x2):
     1(eth1): addr:00:50:56:97:13:9f
        config:     0
        state:      0
        current:    10GB-FD COPPER
        advertised: COPPER
        supported:  1GB-FD 10GB-FD COPPER
        speed: 10000 Mbps now, 10000 Mbps max
    3(phy-br-eth1): addr:1a:bc:4d:e7:70:9b
        config:     0
        state:      0
        current:    10GB-FD COPPER
        speed: 10000 Mbps now, 0 Mbps max
    LOCAL(br-eth1): addr:00:50:56:97:13:9f
        config:     0
        state:      0
        speed: 0 Mbps now, 0 Mbps max

    [root@xianghui-10-9-1-141 ~]# ovs-ofctl dump-flows br-int
    NXST_FLOW reply (xid=0x4):
     cookie=0x0, duration=151.935s, table=0, n_packets=0, n_bytes=0, idle_age=151, priority=3,in_port=28,dl_vlan=107 actions=mod_vlan_vid:1,NORMAL
     cookie=0x0, duration=84.919s, table=0, n_packets=0, n_bytes=0, idle_age=84, priority=3,in_port=28,dl_vlan=109 actions=mod_vlan_vid:1,NORMAL
     cookie=0x0, duration=1863.574s, table=0, n_packets=8508, n_bytes=658280, idle_age=0, priority=2,in_port=28 actions=drop
     cookie=0x0, duration=1864.574s, table=0, n_packets=113, n_bytes=4792, idle_age=1641, priority=1 actions=NORMAL
    [root@xianghui-10-9-1-141 ~]# ovs-ofctl dump-flows br-eth1
    NXST_FLOW reply (xid=0x4):
     cookie=0x0, duration=152.422s, table=0, n_packets=0, n_bytes=0, idle_age=152, priority=4,in_port=3,dl_vlan=1 actions=mod_vlan_vid:109,NORMAL
     cookie=0x0, duration=1879.616s, table=0, n_packets=117, n_bytes=5130, idle_age=1657, priority=2,in_port=3 actions=drop
     cookie=0x0, duration=1880.539s, table=0, n_packets=8586, n_bytes=664620, idle_age=0, priority=1 actions=NORMAL

# Capture eth1 packets

[![vlan1](/images/tech/vlan1.jpg)](/images/tech/vlan1.jpg)

# Configure vcenter bridge(vid 109)

[![vlan2](/images/tech/vlan2.jpg)](/images/tech/vlan2.jpg)

# Configure nova to support vcenter driver

Edit file `/etc/nova/nova.conf`
<pre><code>
[DEFAULT]
debug = False
log_dir = /var/log/nova
state_path = /var/lib/nova
lock_path = /var/lib/nova/tmp
dhcpbridge = /usr/bin/nova-dhcpbridge
dhcpbridge_flagfile = /etc/nova/nova.conf
injected_network_template = /usr/share/nova/interfaces.template
libvirt_inject_partition = -1
#network_manager = nova.network.manager.FlatDHCPManager
sql_connection = mysql://nova:nova@10.9.1.141/nova?charset=utf8
#compute_driver = libvirt.LibvirtDriver
compute_driver = vmwareapi.VMwareVCDriver
rpc_backend = nova.openstack.common.rpc.impl_qpid
enabled_apis = osapi_compute,metadata
verbose = true
auth_strategy = keystone
auth_uri = http://10.9.1.141:5000
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
neutron_admin_auth_url = http://localhost:5000/v2.0/
neutron_auth_strategy = keystone
neutron_admin_tenant_name = service
neutron_url = http://localhost:9696/
libvirt_vif_driver = nova.virt.libvirt.vif.LibvirtGenericVIFDriver
#linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
security_group_api = neutron
#linuxnet_ovs_integration_bridge = br-int
neutron_ovs_bridge = br-int
#vmware_vif_driver="nova.virt.vmwareapi.vif.VMWareVlanBridgeDriver"
default_floating_pool = ext_net
integration_bridge = br-vlan
[vmware]
host_ip = 10.9.1.43
host_username = administrator@vsphere.local
host_password = passw0rd
cluster_name = cluster01
#vlan_interface="vmnic0"
wsdl_location=file:///var/lib/SDK/wsdl/vim25/vimService.wsdl
#integration_bridge = br-vlan
[keystone_authtoken]
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = nova
auth_version = v2.0
</code></pre>

# Restart nova-compute

    [root@xianghui-10-9-1-141 ~]# service openstack-nova-compute restart 

# Create vm

    [root@xianghui-10-9-1-141 ~]# glance index
    0260e6e4-96df-4e90-8fed-0ed6dac06d14 F17                            qcow2                bare                      476704768
    2c1b230e-c338-4572-8f1b-183ef38231b9 trend-thin                     vmdk                 bare                      268435456

    [root@xianghui-10-9-1-141 ~]# neutron net-list
    +--------------------------------------+----------+------------------------------------------------------+
    | id                                   | name     | subnets                                              |
    +--------------------------------------+----------+------------------------------------------------------+
    | 75f4506d-314c-4814-9afe-fa5c935a2b17 | vlan-109 | f5c20675-aa7d-4912-8213-20b04705811a 90.0.0.0/24     |
    | e27e26b1-8b31-4957-8ec0-d9b0b16d6368 | ext_net  | 4705cdf1-d3ac-4b5e-817b-d547d22c641b 192.168.12.0/24 |
    +--------------------------------------+----------+------------------------------------------------------+

    [root@xianghui-10-9-1-141 ~]# nova boot --image 2c1b230e-c338-4572-8f1b-183ef38231b9 --flavor 2 --nic net-id=75f4506d-314c-4814-9afe-fa5c935a2b17 test_vcenter_6
    +--------------------------------------+--------------------------------------+
    | Property                             | Value                                |
    +--------------------------------------+--------------------------------------+
    | OS-EXT-STS:task_state                | scheduling                           |
    | image                                | trend-thin                           |
    | OS-EXT-STS:vm_state                  | building                             |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000017                    |
    | OS-SRV-USG:launched_at               | None                                 |
    | flavor                               | m1.small                             |
    | id                                   | 26bf4e15-ac44-41e2-8345-93f5446d41cd |
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
    | updated                              | 2013-12-02T03:30:35Z                 |
    | hostId                               |                                      |
    | OS-EXT-SRV-ATTR:host                 | None                                 |
    | OS-SRV-USG:terminated_at             | None                                 |
    | key_name                             | None                                 |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | None                                 |
    | name                                 | test_vcenter_6                       |
    | adminPass                            | oSHPDunhB3dh                         |
    | tenant_id                            | adc4e7a4effa44ffa3c6e48dd5a8555a     |
    | created                              | 2013-12-02T03:30:35Z                 |
    | os-extended-volumes:volumes_attached | []                                   |
    | metadata                             | {}                                   |
    +--------------------------------------+--------------------------------------+

Additional

    ovs-ofctl add-flow br-eth1 hard_timeout=0,idle_timeout=0,priority=4,in_port=3,dl_vlan=1,actions=mod_vlan_vid:109,normal
    phy-br-eth1
    ovs-ofctl add-flow br-int hard_timeout=0,idle_timeout=0,priority=3,in_port=28,dl_vlan=107,actions=mod_vlan_vid:1,normal
    int-br-eth1

[ovs][8]

[1]:  https://wiki.openstack.org/wiki/VMware-vSphere-support
[2]:  https://wiki.openstack.org/wiki/NovaVMware/DeveloperGuide
[3]:  http://docs.openstack.org/trunk/config-reference/content/vmware.html#VMWare_networking
[4]:  http://www.vmware.com/support/developer/vc-sdk/wssdk_5_5_releasenotes.html
[5]:  https://my.vmware.com/cn/web/vmware/details?downloadGroup=WEBSDK550&productId=353
[6]:  http://pubs.vmware.com/vsphere-55/index.jsp?topic=%2Fcom.vmware.wssdk.pg.doc%2FPG_Preface.html
[7]:  https://communities.vmware.com/community/vmtn/developer/downloads
[8]:  http://blog.csdn.net/yahohi/article/details/6631934
