---
layout: post
title: Using ESXI as an OpenStack Compute node
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Home</p>

OS : RHEL6.4

OpenStack version : Havana


Controller node : KVM

Compute node : ESXI 5.5


# Configure controller node succesfully

*Recommend to use devstack/packstack*
service Keystone/Glance/Nova/Neutron works well

# Download ESXI SDK 5.5 

    unzip to directory /var/lib/

# Configure the ESXI5.5 host and create a cluster

create a network port group named "br-int" in ESXI driver with vSphere client on the network configuration items.

1. Click "Configuration" item on ESXI with the vsphereclient

[![ESXI-1](/images/tech/esxi1.jpg)](/images/tech/esxi1.jpg)

2. Click "Add networking", Choose "Virtual machine" , then "Next"

[![ESXI-2](/images/tech/esxi2.jpg)](/images/tech/esxi2.jpg)

3. Create or choose the default switch , then "Next"

[![ESXI-3](/images/tech/esxi3.jpg)](/images/tech/esxi3.jpg)

4. Fill the "Port group properties"

[![ESXI-4](/images/tech/esxi4.jpg)](/images/tech/esxi4.jpg)

# Edit config file on controller node

    compute_driver = vmwareapi.VMwareESXDriver
    vmware_vif_driver="nova.virt.vmwareapi.vif.VMWareVlanBridgeDriver"
    [vmware]
    cluster_name = cluster01
    # ESXI ip
    host_ip = 10.9.0.133
    host_username=root
    host_password=password

    wsdl_location=file:///var/lib/SDK/wsdl/vim25/vimService.wsdl

# Restart service Nova

    for svc in api scheduler compute conductor; do service openstack-nova-$svc restart; done

It's probably correct if appears below logs in file `/var/log/nova/compute.log`

    2013-11-10 21:26:18.952 18723 INFO nova.openstack.common.periodic_task 
    [-] NV-7A0397C Skipping periodic task _periodic_update_dns because its interval is negative
    2013-11-10 21:26:19.115 18723 INFO nova.virt.driver [-] NV-91AF767 
    Loading compute driver 'vmwareapi.VMwareESXDriver'
    2013-11-10 21:26:26.992 18723 INFO nova.openstack.common.rpc.impl_qpid 
    [req-28af9fb3-b7ce-4c41-ac83-4a65f9b296f3 None None] NV-1A047FB Connected 
    to AMQP server on localhost:5672
    2013-11-10 21:26:27.001 18723 INFO nova.openstack.common.rpc.impl_qpid 
    [req-28af9fb3-b7ce-4c41-ac83-4a65f9b296f3 None None] NV-1A047FB Connected
    to AMQP server on localhost:5672

# Download vmdk image

*vmware needs the 'flat' type images and extra options to register to glance, below is an example to boot an vm successfully and has been verified.*

    wget  http://partnerweb.vmware.com/programs/vmdkimage/trend-tinyvm1-flat.vmdk
    $ glance image-create --name trend-thin --is-public=True --container-format=bare --disk-format=vmdk --property vmware_disktype="thin" --property vmware_adaptertype="ide" < trend-tinyvm1-flat.vmdk


**Note**: Errors will be reported if **IDE Controller** attach volume, [bug]:  https://bugs.launchpad.net/cinder/+bug/1226543

Below is the clarification from vmware ：

    Kartik Bommepally (kartikaditya) wrote on 2013-10-24:   #6
    The issue is when the instance uses an IDE controller.

    IDE controller does not support hot adding of a virtual disk http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1025883

    However if an IDE controller is not being used for the primary disk of the virtual machine then the issue must not arise.

    One possible fix would be to use a SCSI controller (one of http://pubs.vmware.com/vsphere-51/index.jsp?topic=%2Fcom.vmware.wssdk.apiref.doc%2Fvim.vm.device.VirtualSCSIController.html) for volume's disk irrespective of primary disk's controller.


    glance add name="ubuntuLTS" disk_format=vmdk container_format=ovf \
    is_public=true vmware_adaptertype="lsiLogic" vmware_disktype="preallocated"\
    vmware_ostype="ubuntu64Guest" < ubuntuLTS-flat.vmdk

referrence : 

<https://wiki.openstack.org/wiki/NovaVMware/DeveloperGuide>
<http://docs.openstack.org/havana/config-reference/content/vmware.html>
