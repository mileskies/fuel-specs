..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Adopt 4.4 kernel in MOS9 / Ubuntu 14.04
=======================================

https://blueprints.launchpad.net/fuel/+spec/adopt-4-4-kernel


-------------------
Problem description
-------------------

As a Cloud Owner I want to be able to deploy MOS 9.x with Linux kernel v4.4 so
that I could use latest models of servers for my cloud.

MOS 9.x is currently shipped with Linux kernel v3.x - this will likely become
an issue in 2017, when customers want to use newer models of servers for
their cloud capacity. Next to that OVS and KVM may get improvements by
upgrading the Kernel.

Latest Ubuntu 14.04 release, 14.04.5, includes kernel 4.4 oficially
backported from Ubuntu Xenial 16.04 that will be supported until 14.04 EOL:

* `14.04.x Ubuntu Kernel Support`_

* `LTS Kernel Support Schedule`_

Kernel v4.4 is available in official Ubuntu repositories, so no additional
work is needed here - no need to build a custom kernel or enable additional
repositories.

This gives us a good opportunity to replace 3.13 kernel with 4.4, to get
the following advantages:

* updated drivers (we even can remove several drivers since stock versions
  are good), e.g.

  * hpsa-dkms package can be removed

  * updated iscsi driver (resolves issue with some storages)

  * updated pci driver (resolves issue with some Intel NICs)

* support for more hardware out of the box (e.g. it fixes issue with multiple
  SR-IOV NICs installed in one chassis)

* a kernel with a lot of bug/security fixes


----------------
Proposed changes
----------------


Web UI
======

None

Nailgun
=======


Data model
----------

None

REST API
--------

None

Orchestration
=============


RPC Protocol
------------

None

Fuel Client
===========

None

Plugins
=======

There is a little chance that some plugins are not compatible with 4.4 kernel.
However, since upgrading to 4.4 kernel is a manual operation, this should be
checked before an upgrade.


Fuel Library
============

The following CRs needed:

* https://review.fuel-infra.org/#/c/22955/


------------
Alternatives
------------

There is no alternative way if you want functionality of a new kernel.


--------------
Upgrade impact
--------------

None


---------------
Security impact
---------------

None


--------------------
Notifications impact
--------------------

None

---------------
End user impact
---------------

End user (or support staff) should manually upgrade their environement in order
to get this feature following `documentation describing upgrade steps`_.

Upgrading the kernel is a process that may result in workload downtime.
The user should carefully plan the kernel upgrade. Expected workflow for
existing environments is:

1.  Live-migrate existing instances from node which should be upgraded to any
    other node in cluster

2.  Upgrade the kernel that now contains no running instances

3.  Reboot the node

4.  Verify node is working correctly with new kernel

5.  (Optionally) live-migrate instances back to original node to spread load

This workflow should be repeated until all nodes are upgraded to latest kernel.


------------------
Performance impact
------------------

None

-----------------
Deployment impact
-----------------

After Fuel Master is upgraded with MOS 9.2 packages, a number of step to
upgrade existing environments / default settings should be done in order to
switch to kernel 4.4. These steps are described in
`documentation describing upgrade steps`_ among with helper scripts. Without
this already deployed MOS 9.0 and 9.1 environments will stay the same,
3.13-based, unless end-user manually upgrades the kernel to v4.4 on specific
nodes.


----------------
Developer impact
----------------

None

---------------------
Infrastructure impact
---------------------

None

--------------------
Documentation impact
--------------------

Upgrade procedure should be documented and officially published.


--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  `Dmitry Teselkin`_

Other contributors:
  `Ivan Suzdal`_

Mandatory design review:
  `Nastya Urlapova`_


Work Items
==========

* Prepare `documentation describing upgrade steps`_.

* Prepare minimal set of scripts to automate routing tasks required to perform
  upgrade.

* Verify upgrade procedure, verify cluster after an upgrade in following cases:

  * upgrade master node only and deploy new environment

  * upgrade master node and existing environment


Dependencies
============

None

------------
Testing, QA
------------

Upgrade procedure is not fully automated process and should be applied
and verified manually. No new tests needs to be added.



Acceptance criteria
===================

* Instructions for upgrade of existing MOS 9.0/9.1 environments into kernel
  v4.4 are created and meet the following criteria:

  * Customers/L2 are expected to follow instructions and upgrade the kernel to
    v4.4 on computes, one by one, nothing should be automated by Fuel,
    instructions are provided as is.

  * Any node in a deployment environment that is currently using v3.x should
    stay on v3.x unless customer manually upgrades the kernel to newer version.


----------
References
----------

.. _`14.04.x Ubuntu Kernel Support`: https://wiki.ubuntu.com/Kernel/LTSEnablementStack#Kernel.2FSupport.A14.04.x_Ubuntu_Kernel_Support
.. _`LTS Kernel Support Schedule`: https://wiki.ubuntu.com/Kernel/Support?action=AttachFile&do=view&target=LTS+Kernel+Support+Schedule.svg
.. _`Dmitry Teselkin`: https://launchpad.net/~teselkin-d
.. _`Ivan Suzdal`: https://launchpad.net/~isuzdal
.. _`Nastya Urlapova`: https://launchpad.net/~aurlapova
.. _`documentation describing upgrade steps`: https://review.fuel-infra.org/27600
