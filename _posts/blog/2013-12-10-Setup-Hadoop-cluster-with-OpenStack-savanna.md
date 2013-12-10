---
layout: post
title: Set up Hadoop cluster with OpenStack Savanna
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Home</p>

# Deploy OpenStack primary components
*keystone/glance/nova/neutron/horizon*

Refer to [QuickStartLatest - RDO][1], the script can be performed duplicated.

If using neutron instead of nova-network
  
    [root@xianghui workplace]# packstack --allinone --os-neutron-install=y

After preceding successful deployment, following steps are to install savanna, using dashboard other than cli.
Refer [Savanna UI Installation Guide][2]

# Install Savanna
**To install with RDO**
Start by following the Quickstart[1] to install and setup OpenStack.
Install the savanna-api service with,
    
    $ yum install openstack-savanna

Configure the savanna-api service to your liking. The configuration file is located in `/etc/savanna/savanna.conf`.
Start the savanna-api service with,

    $ service openstack-savanna-api start

**To install into a virtual environment(install savanna with tarball)**
First you need to install python-setuptools, python-virtualenv and python headers using yourOS package manager. The python headers package name depends on OS. For Ubuntu it is python-dev,for Red Hat - python-devel.
For Fedora:

    $ sudo yum install gcc python-setuptools python-virtualenv python-devel

Setup virtual environment for Savanna:

    $ virtualenv savanna-venv

This will install python virtual environment into savanna-venv directoryin your current working directory. This command does not require superuser privileges and could be executed in any directory current user haswrite permission.
You can install the latest Savanna release version from pypi:

    $ savanna-venv/bin/pip install savanna

Or you can get Savanna archive from [Index of /savanna][3] and install it using pip:

    $ savanna-venv/bin/pip install 'http://tarballs.openstack.org/savanna/savanna-master.tar.gz'

Note that savanna-master.tar.gz contains the latest changes and might not be stable at the moment.We recommend browsing [Index of /savanna][3] and selecting the latest stable release.

After installation you should create configuration file. Sample config file locationdepends on your OS. For Ubuntu it is /usr/local/share/savanna/savanna.conf.sample,for Red Hat -/usr/share/savanna/savanna.conf.sample. Below is an example for Ubuntu:
$ mkdir savanna-venv/etc$ cp savanna-venv/share/savanna/savanna.conf.sample savanna-venv/etc/savanna.conf
check each option in savanna-venv/etc/savanna.conf, and make necessary changes
To start Savanna call:
$ savanna-venv/bin/python savanna-venv/bin/savanna-api --config-file savanna-venv/etc/savanna.conf
</code></pre>

# Configure Savanna
Edit file `/etc/savanna/savanna.conf`
<pre><code>
[DEFAULT]
port=8386
os_auth_host=127.0.0.1
os_auth_port=35357
os_admin_username=admin
os_admin_password=openstack1
os_admin_tenant_name=service
use_floating_ips=false
use_neutron=true
debug=true
verbose=true
log_file=savanna.log
log_dir=/var/log/savanna/
plugins=vanilla,hdp
[plugin:vanilla]
plugin_class=savanna.plugins.vanilla.plugin:VanillaProvider
[plugin:hdp]
plugin_class=savanna.plugins.hdp.ambariplugin:AmbariPlugin
[database]
#connection=sqlite:////savanna/openstack/common/db/$sqlite_db
</code></pre>

Create directory for savanna
    
    [root@xianghui workplace]#  mkdir /var/log/savanna

Create log file savanna.log

    [root@xianghui workplace]#  vi /var/log/savanna/savanna.log 
    [root@xianghui workplace]#  chown savanna:savanna /var/log/savanna/savanna.log

Restart savanna after finishing configuring 

# Set up savanna UI
Go to the machine where Dashboard resides and install Savanna UI:
For RDO:
    
    $ sudo yum install python-django-savanna

Otherwise:

    $ sudo pip install savanna-dashboard

This will install latest stable release of Savanna UI. If you want to install master branch of Savanna UI:

    $ sudo pip install 'http://tarballs.openstack.org/sa...

# Configure OpenStack Dashboard.
<pre><code>
In settings.py add savanna to HORIZON_CONFIG = {    'dashboards': ('nova', 'syspanel', 'settings', ..., 'savanna'), and also add savannadashboard to INSTALLED_APPS = (    'savannadashboard',    ....
Note: settings.py file is located in /usr/share/openstack-dashboard/openstack_dashboard/ by default.
Also you have to specify SAVANNA_URL in local_settings.py. For example:
SAVANNA_URL = 'http://localhost:8386/v1.1 If you are using Neutron instead of Nova Network: SAVANNA_USE_NEUTRON = True
</code></pre>

**Note**: For RDO, the local_settings.py file is located in/etc/openstack-dashboard/, otherwise it is in/usr/share/openstack-dashboard/openstack_dashboard/local/.

    $ sudo service httpd reload

You can check that service has been started successfully. Go to Horizon URL and if installation is correct you’ll be able to see the Savanna tab.

# Download prebuild image 
You can download pre-built images with vanilla Apache Hadoop or build this images yourself:
Download and install pre-built image with Fedora 19

    $ wget http://savanna-files.mirantis.co... glance image-create --name=savanna-0.3-vanilla-1.2.1-fedora-19 \  --disk-format=qcow2 --container-format=bare < ./savanna-0.3-vanilla-1.2.1-fedora-19.qcow2

# Login dashboard and find out a page named "savanna", the plugins item including two plugins type: vanilla/hdp

[![plugins1](/images/tech/savanna1.png)](/images/tech/savanna1.png)

# Register prebuild image in dashboard:
note tags:='["vanilla", "1.2.1", "fedora"]'

[![plugins2](/images/tech/savanna2.png)](/images/tech/savanna2.png)

Create `node groups`: `data node`/`name node` templates

[![plugins3](/images/tech/savanna3.png)](/images/tech/savanna3.png)

Create `cluster templates`

[![plugins4](/images/tech/savanna4.png)](/images/tech/savanna4.png)
[![plugins5](/images/tech/savanna5.jpg)](/images/tech/savanna5.jpg)
[![plugins6](/images/tech/savanna6.jpg)](/images/tech/savanna6.jpg)
[![plugins7](/images/tech/savanna7.jpg)](/images/tech/savanna7.jpg)

Create `cluster`

[![plugins8](/images/tech/savanna8.jpg)](/images/tech/savanna8.jpg)

[1]:  http://openstack.redhat.com/QuickStartLatest
[2]:  https://savanna.readthedocs.org/en/latest/horizon/installation.guide.html
[3]:  http://tarballs.openstack.org/savanna/

# Run job
Due to the dashboard only support Swift currently, so use HDFS as the file system, below list including three vms, when each vm is booting, the cloud-init script will automatically running hadoop task of vanilla plugin.
<pre><code>
[root@xianghui ~]# nova list
+--------------------------------------+---------------------------------------+--------+------------+-------------+---------------------------------+
| ID                                   | Name                                  | Status | Task State | Power State | Networks                        |
+--------------------------------------+---------------------------------------+--------+------------+-------------+---------------------------------+
| 0019f2e8-9450-45e7-9455-44f51d4029b8 | test-1-DataNodeGroup-001              | ACTIVE | None       | Running     | flat-80=80.0.0.6                |
| 1cd12dc5-86f5-4e06-bf1f-fd7635dca032 | test-1-DataNodeGroup-002              | ACTIVE | None       | Running     | flat-80=80.0.0.7                |
| e746f0ba-b297-4e07-b430-85840b71fa53 | test-1-NameNodeGroup-001              | ACTIVE | None       | Running     | flat-80=80.0.0.3                |
</code></pre>
Configure ssh-key to ignore the passwd

    [root@xianghui ~]# ssh fedora@80.0.0.3
    Last login: Wed Oct 30 09:48:07 2013 from 80.0.0.2
    [fedora@test-1-namenodegroup-001 ~]$ sudo -i
    [root@test-1-namenodegroup-001 ~]#whereis hadoop
    hadoop: /usr/bin/hadoop /etc/hadoop /usr/etc/hadoop /usr/include/hadoop /usr/share/hadoop

Edit `input file` as the input file of jobs
    
    [root@test-1-namenodegroup-001 ~]# cat input
    dfs
    dfjalkfd
    dkaljf

Running wordcount job of examples, result shows 3, correct!
<pre><code>
[root@test-1-namenodegroup-001 ~]# hadoop jar /usr/share/hadoop/hadoop-examples-1.2.1.jar wordcount input test_output
13/11/14 06:15:34 INFO mapred.JobClient: Running job: job_201310210945_0015
13/11/14 06:15:35 INFO mapred.JobClient:  map 0% reduce 0%
13/11/14 06:15:50 INFO mapred.JobClient:  map 100% reduce 0%
13/11/14 06:16:01 INFO mapred.JobClient:  map 100% reduce 33%
13/11/14 06:16:03 INFO mapred.JobClient:  map 100% reduce 100%
13/11/14 06:16:05 INFO mapred.JobClient: Job complete: job_201310210945_0015
13/11/14 06:16:05 INFO mapred.JobClient: Counters: 29
13/11/14 06:16:05 INFO mapred.JobClient:   Job Counters
13/11/14 06:16:05 INFO mapred.JobClient:     Launched reduce tasks=1
13/11/14 06:16:05 INFO mapred.JobClient:     SLOTS_MILLIS_MAPS=15287
13/11/14 06:16:05 INFO mapred.JobClient:     Total time spent by all reduces waiting after reserving slots (ms)=0
13/11/14 06:16:05 INFO mapred.JobClient:     Total time spent by all maps waiting after reserving slots (ms)=0
13/11/14 06:16:05 INFO mapred.JobClient:     Launched map tasks=1
13/11/14 06:16:05 INFO mapred.JobClient:     Data-local map tasks=1
13/11/14 06:16:05 INFO mapred.JobClient:     SLOTS_MILLIS_REDUCES=12207
13/11/14 06:16:05 INFO mapred.JobClient:   File Output Format Counters
13/11/14 06:16:05 INFO mapred.JobClient:     Bytes Written=26
13/11/14 06:16:05 INFO mapred.JobClient:   FileSystemCounters
13/11/14 06:16:05 INFO mapred.JobClient:     FILE_BYTES_READ=44
13/11/14 06:16:05 INFO mapred.JobClient:     HDFS_BYTES_READ=138
13/11/14 06:16:05 INFO mapred.JobClient:     FILE_BYTES_WRITTEN=110578
13/11/14 06:16:05 INFO mapred.JobClient:     HDFS_BYTES_WRITTEN=26
13/11/14 06:16:05 INFO mapred.JobClient:   File Input Format Counters
13/11/14 06:16:05 INFO mapred.JobClient:     Bytes Read=21
13/11/14 06:16:05 INFO mapred.JobClient:   Map-Reduce Framework
13/11/14 06:16:05 INFO mapred.JobClient:     Map output materialized bytes=44
13/11/14 06:16:05 INFO mapred.JobClient:     Map input records=4
13/11/14 06:16:05 INFO mapred.JobClient:     Reduce shuffle bytes=44
13/11/14 06:16:05 INFO mapred.JobClient:     Spilled Records=6
13/11/14 06:16:05 INFO mapred.JobClient:     Map output bytes=32
13/11/14 06:16:05 INFO mapred.JobClient:     Total committed heap usage (bytes)=163254272
13/11/14 06:16:05 INFO mapred.JobClient:     CPU time spent (ms)=2580
13/11/14 06:16:05 INFO mapred.JobClient:     Combine input records=3
13/11/14 06:16:05 INFO mapred.JobClient:     SPLIT_RAW_BYTES=117
13/11/14 06:16:05 INFO mapred.JobClient:     Reduce input records=3
13/11/14 06:16:05 INFO mapred.JobClient:     Reduce input groups=3
13/11/14 06:16:05 INFO mapred.JobClient:     Combine output records=3
13/11/14 06:16:05 INFO mapred.JobClient:     Physical memory (bytes) snapshot=238972928
13/11/14 06:16:05 INFO mapred.JobClient:     Reduce output records=3
13/11/14 06:16:05 INFO mapred.JobClient:     Virtual memory (bytes) snapshot=1652383744
13/11/14 06:16:05 INFO mapred.JobClient:     Map output records=3
</code></pre>
