---
layout: post
title: OpenStack unit tests environment debugging
---

{{ page.title }}
================

<p class="meta">9 Dec 2012 - BeiJing Ring Building</p>

Most cases, it needs to provide corresponding unit tests for each project's source code in OpenSTack community,
but due to  the dependency of packages version, unit test may be failed, this post just briefly introduce how to
run `/project/run_tests.sh`.

Use `OpenStack/Neutron` as an example:

##### If using virtual env to deploy OpenStack:

*/opt/stack/neutron/requirements.txt is the dependency requirement of neutron project*

    tools/with_venv.sh pip install --upgrade /opt/stack/neutron/requirements.txt 

*/opt/stack/neutron/test_requirements.txt is the dependency requirement of neutron project test*

    tools/with_venv.sh pip install --upgrade /opt/stack/neutron/test_requirements.txt

##### If not using virtual env to deploy OpenStack:

    pip install --upgrade -r requirements.txt
    pip install --upgrade -r  test-requirements.txt

Results appear:

    Successfully installed amqplib jsonrpclib python-neutronclient alembic six Mako
    Cleaning up...

Run test cases under projects

    root@ubuntu:/opt/stack/heat#./run_tests.sh

Run /heat/tests/test_instance.py

    root@ubuntu:/opt/stack/heat# ./run_tests.sh heat.tests.test_instance

Results including all key_words:`test_instance`

<pre><code>Ran 106 (-1407) tests in 2.115s (-53.826s)
PASSED (id=12)
Slowest Tests
Test id                                                                                                   Runtime (s)
--------------------------------------------------------------------------------------------------------  -----------
heat.tests.test_instance_group_update_policy.InstanceGroupTest.test_instance_group_update                 0.340
heat.tests.test_instance_group_update_policy.InstanceGroupTest.test_instance_group_update_policy_removed  0.318
heat.tests.test_instance.InstancesTest.test_build_nics                                                    0.274
heat.tests.test_instance_group.InstanceGroupTest.test_handle_update_size                                  0.255
heat.tests.test_instance.InstancesTest.test_instance_create_duplicate_image_name_err                      0.236
heat.tests.test_instance_group.InstanceGroupTest.test_update_error                                        0.149
heat.tests.test_instance.InstancesTest.test_instance_update_instance_type_failed                          0.117
heat.tests.test_instance_group.InstanceGroupTest.test_update_fail_badkey                                  0.115
heat.tests.test_instance_group.InstanceGroupTest.test_update_fail_badprop                                 0.111
heat.tests.test_instance_group.InstanceGroupTest.test_create_error                                        0.086</pre></code>

To test /heat/tests/test_instance.py  `Class InstancesTest`:

    root@ubuntu:/opt/stack/heat# ./run_tests.sh heat.tests.test_instance.InstancesTest

<pre><code>
Ran 68 (+15) tests in 1.305s (-0.802s)
PASSED (id=13)
Slowest Tests
Test id                                                                           Runtime (s)
--------------------------------------------------------------------------------  -----------
heat.tests.test_instance.InstancesTest.test_instance_create                       0.266
heat.tests.test_instance.InstancesTest.test_build_nics                            0.256
heat.tests.test_instance.InstancesTest.test_instance_update_instance_type_failed  0.140
heat.tests.test_instance.InstancesTest.test_instance_create_unexpected_status     0.138
heat.tests.test_instance.InstancesTest.test_instance_status_verify_resize         0.110
heat.tests.test_instance.InstancesTest.test_instance_status_password              0.086
heat.tests.test_instance.InstancesTest.test_instance_status_rescue                0.085
heat.tests.test_instance.InstancesTest.test_instance_status_resize                0.081
heat.tests.test_instance.InstancesTest.test_instance_resume_volumes_step          0.075
heat.tests.test_instance.InstancesTest.test_instance_suspend_volumes_step         0.074</pre></code>

To test /heat/tests/test_instance.py `Class InstancesTest function test_build_nics()`:
<pre><code>Ran 2 (-32) tests in 0.191s (-1.106s)
PASSED (id=14)
Slowest Tests
Test id                                                 Runtime (s)
------------------------------------------------------  -----------
heat.tests.test_instance.InstancesTest.test_build_nics  0.188</pre></code>

## Error handling:

1.AttributeError: `module` object has no attribute `DeprecatedOpt`

<pre><code>
FAIL: unittest.loader.ModuleImportFailure.neutron.tests.unit.test_security_groups_rpc
unittest.loader.ModuleImportFailure.neutron.tests.unit.test_security_groups_rpc
----------------------------------------------------------------------
_StringException: ImportError: Failed to import test module: neutron.tests.unit.test_security_groups_rpc
Traceback (most recent call last):
  File "/usr/lib/python2.7/unittest/loader.py", line 252, in _find_tests
    module = self._get_module_from_name(name)
  File "/usr/lib/python2.7/unittest/loader.py", line 230, in _get_module_from_name
    __import__(name)
  File "/opt/stack/neutron/neutron/tests/unit/test_security_groups_rpc.py", line 29, in <module>
    from neutron import context
  File "/opt/stack/neutron/neutron/context.py", line 24, in <module>
    from neutron.db import api as db_api
  File "/opt/stack/neutron/neutron/db/api.py", line 22, in <module>
    from neutron.db import model_base
  File "/opt/stack/neutron/neutron/db/model_base.py", line 19, in <module>
    from neutron.openstack.common.db.sqlalchemy import models
  File "/opt/stack/neutron/neutron/openstack/common/db/sqlalchemy/models.py", line 29, in <module>
    from neutron.openstack.common.db.sqlalchemy.session import get_session
  File "/opt/stack/neutron/neutron/openstack/common/db/sqlalchemy/session.py", line 283, in <module>
    deprecated_opts=[cfg.DeprecatedOpt('sql_connection',
AttributeError: 'module' object has no attribute 'DeprecatedOpt'</pre></code>

Solution: (From <https://bugs.launchpad.net/tripleo/+bug/1194807>)

    cd /usr/local/
    lib/python2.7/dist-packages/
    rm -rf oslo
    pip install <http://tarballs.openstack.org/oslo.config/oslo.config-1.2.0a2.tar.gz#egg=oslo.config-1.2.0a2>

2. No module named webtest
solution:

    root@ubuntu:/opt/stack/neutron# pip install webtest

<pre><code>Downloading/unpacking webtest
  Downloading WebTest-2.0.7.zip (81kB): 81kB downloaded
  Running setup.py egg_info for package webtest
    
    no previously-included directories found matching 'docs/_build'
    warning: no previously-included files matching '*.pyc' found anywhere in distribution
    warning: no previously-included files matching '__pycache__' found anywhere in distribution
Requirement already satisfied (use --upgrade to upgrade): six in /usr/local/lib/python2.7/dist-packages (from webtest)
Requirement already satisfied (use --upgrade to upgrade): WebOb>=1.2 in /usr/local/lib/python2.7/dist-packages (from webtest)
Downloading/unpacking waitress>=0.8.5 (from webtest)
  Downloading waitress-0.8.7.tar.gz (115kB): 115kB downloaded
  Running setup.py egg_info for package waitress
    
Downloading/unpacking beautifulsoup4 (from webtest)
  Downloading beautifulsoup4-4.3.1.tar.gz (142kB): 142kB downloaded
  Running setup.py egg_info for package beautifulsoup4
    
Requirement already satisfied (use --upgrade to upgrade): setuptools in  
    /usr/local/lib/python2.7/dist-packages (from  waitress>=0.8.5->webtest)
Installing collected packages: webtest, waitress, beautifulsoup4
  Running setup.py install for webtest
    
    no previously-included directories found matching 'docs/_build'
    warning: no previously-included files matching '*.pyc' found anywhere in distribution
    warning: no previously-included files matching '__pycache__' found anywhere in distribution
  Running setup.py install for waitress
    
    Installing waitress-serve script to /usr/local/bin
  Running setup.py install for beautifulsoup4
    
Successfully installed webtest waitress beautifulsoup4
Cleaning up...</pre></code>

