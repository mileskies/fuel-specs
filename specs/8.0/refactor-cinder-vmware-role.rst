..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Refactor cinder-vmware role
===========================

https://blueprints.launchpad.net/fuel/+spec/refactor-cinder-vmware-role

Currently, cinder-vmware [0]_ (cinder-volume) installed with default
cinder-volume service. This leads to many problems, since such behavior is not
considered. Moreover, it is wrong from a logical point of view, because we have
an extra unaccounted default cinder-volume service (not for VMware). It is also
necessary to alter start\stop scripts for cinder-vmware.


--------------------
Problem description
--------------------

Deploying cinder-vmware occurs abnormal scenario. At first configure and run
all services cinder (cinder-api, cinder-scheduler,cinder-volume) configuration
for a given environment (LVM or Ceph). Once configured and started
cinder-volume with vmware-backend.

This behavior leads to some problems:

* we have unaccounted and unnecessary services
  (cinder-api, cinder-scheduler,cinder-volume) for cinder-vmware role.
* this leads to a variety of errors in running services (cinder-api,
  cinder-scheduler, cinder-volume) due to the fact that the deployment does not
  imply the existence of these services for the cinder-vmware role. For example
  [1]_.

Also, it is necessary to rework the start/stop script that is copied when we
install cinder-volume (MOS), for support systemd and sysv-rc.


----------------
Proposed changes
----------------

To implement this blueprint, do the following:

* removed from the package cinder-volume (MOS) custom start/stop script and use
  the standard script of the package with the necessary changes.
* change the puppet manifest (openstack-cinder and cinder-vmware).

Web UI
======

None.


Nailgun
=======

None.

Data model
----------

None.


REST API
--------

None.


Orchestration
=============

None.


RPC Protocol
------------

None.


Fuel Client
===========

None.


Plugins
=======

None.


Fuel Library
============

It is planned to make changes to openstack-cinder and cinder-vmware manifests.


------------
Alternatives
------------

There are no alternatives, because if leave as is, we need to prepare for a
large number of bugs and problems.


--------------
Upgrade impact
--------------

None.


---------------
Security impact
---------------

None.


--------------------
Notifications impact
--------------------

None.


---------------
End user impact
---------------

None.


------------------
Performance impact
------------------

None.


-----------------
Deployment impact
-----------------

None.


----------------
Developer impact
----------------

None.


---------------------
Infrastructure impact
---------------------

We remove custom upstart cinder-volume-vmware.conf from cinder-volume package.


--------------------
Documentation impact
--------------------

None.


--------------
Implementation
--------------

Assignee(s)
===========

======================= ==============================================
Primary assignee        - Alexander Arzhanov <aarzhanov@mirantis.com>
Developers              - Alexander Arzhanov <aarzhanov@mirantis.com>

QA engineers            - Ilya Bumarskov <ibumarskov@mirantis.com>
Mandatory design review - Igor Zinovik <izinovik@mirantis.com>
                        - Sergii Golovatiuk <sgolovatiuk@mirantis.com>
======================= ==============================================


Work Items
==========

* make changes to openstack-cinder and cinder-vmware manifests.
* remove custom upstart cinder-volume-vmware.conf from cinder-volume package
  and use the standard script of the package with the necessary changes.


Dependencies
============

None.


------------
Testing, QA
------------

* Most part of our OSTF tests are disabled when we use neutron instead
  nova-network. In this case we should modify the vCenter OSTF tests to make
  them work with DVS plugin.
* Necessary to check all main actions with volumes in vCenter availability
  zone:

  * Create empty volume
  * Create volume from VMDK image
  * Delete volume
* manual testing.


Acceptance criteria
===================

User is able to deploy cluster with vCenter and cinder-vmware role.
After deploy user can use create volume, create volume from image, etc for
vCenter availability zone.


----------
References
----------

.. [0] https://blueprints.launchpad.net/fuel/+spec/cinder-vmdk-role
.. [1] https://bugs.launchpad.net/fuel/+bug/1493441
