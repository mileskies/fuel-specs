..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Enabling Fernet tokens by default during Fuel installation
==========================================================

https://blueprints.launchpad.net/fuel/+spec/fernet-tokens-support


Implement Fernet tokens as default authentication method, including initial
generation and propagation of keys to all controllers.


-------------------
Problem description
-------------------

The default Keystone configuration for Fernet tokens does not support keystone
service running on multiple nodes. As a result, the current state requires
Operators that are interested in using Fernet tokens to manually generate and
sync the Fernet token key across controllers. Keystone itself does not have any
knowledge of multiple key locations and does not provide any syncing
capabilities natively. However, it is critical to have the keys synced between
all Keystone nodes in order to provide token validation.

The goal of this spec is to provide a mechanism by which Fernet token keys can
be easily generated, propagated during initial cluster environment setup to
all keystone nodes and enabled by default.

----------------
Proposed changes
----------------

Keystone with Fernet tokens does not natively support multiple controllers,
we propose to implement this functionality using astute and puppet manifests
in fuel-library. This will enable the user to generate and install Fernet keys
to all Keystone nodes seamlessly during the deployment process.

To generate keys, a new 'GenerateFernetKeys’ class will be added to
'pre-deployment actions' of astute. It will use the openssl tool for key
generation. The generated keys will be stored in the following directory on the
Fuel-Master node:

'/var/lib/fuel/keys/"${deployment_id}"/fernet-keys'

To copy the generated Fernet keys to all controllers, a new 'UploadFernetKeys'
class will be added to 'pre-deployment actions' of astute. It will copy Fernet
keys to the following destination:

'/var/lib/astute/fernet-keys'

After copying Fernet keys to all controller nodes, they will be installed to
the appropriate directory using puppets facilities:

'/etc/keystone/fernet-keys'

In order to use Fernet tokens as the default authentication mechanism, the
'token_provider' Keystone parameter will be set to
'keystone.token.providers.fernet.Provider' in the associated puppet manifest.

Web UI
======

None

Nailgun
=======

None

Data model
----------

None

REST API
--------

No FUEL REST API changes.

Orchestration
=============

None

RPC Protocol
------------

None

Fuel Client
===========

None

Plugins
=======

None

Fuel Library
============

The following option should be passed to ::keystone class in order to
enable Fernet tokens:
* token_provider = keystone.token.providers.fernet.Provider

------------
Alternatives
------------

* generate fernet keys using default keystone tool on Fuel Master node
  (keystone-manage fernet_setup) and copy them to all keystone nodes:

  This proposition was rejected as this case could lead to upgrade
  of Keystone on Fuel Master node. And it would require a large overhead cost
  to perform this update.

* generate fernet keys using default keystone tool on Primary node
  (keystone-manage fernet_setup) where keystone service is running keystone
  service is running and copy them to another keystone nodes:

  This case was not implemented as there were some complexities to derive keys
  from deployed Primary node and copy the keys to another keystone node using
  Astute. This case needed an additional researching.

--------------
Upgrade impact
--------------

There should be no impact for Fuel Master upgrades. For existing cloud
environment upgrades, there may be some service interruption until all
controllers have been upgraded to support Fernet tokens and all tokens have
been updated and all clients have new Fernet based tokens.

---------------
Security impact
---------------

It is expected to have the same security level as for ssh keys of mysql,
nova and etc.

--------------------
Notifications impact
--------------------

None

---------------
End user impact
---------------

None

------------------
Performance impact
------------------

Rally run showed the following results:

+---------------------------+-------------------+-------------------+---------+
|  Scenario                 | Load              | Full              | Itera   |
|                           | durations(s)      | duration(s)       | tions   |
+---------------------------+---------+---------+---------+---------+---------+
|                           | uuid    | fernet  | uuid    | fernet  |         |
+===========================+=========+=========+=========+=========+=========+
|keystone                   | 5000.27 | 5000.27 | 5154.19 | 5062.73 | 150000  |
+---------------------------+---------+---------+---------+---------+---------+
|create_and_list_tenants    | 2.761   | 3.189   | 23.574  | 25.295  | 30      |
+---------------------------+---------+---------+---------+---------+---------+
|create_and_list_users      | 4.004   | 4.401   | 17.392  | 22.203  | 90      |
+---------------------------+---------+---------+---------+---------+---------+
|create_delete_user         | 9.945   | 18.189  | 31.679  | 40.501  | 90      |
+---------------------------+---------+---------+---------+---------+---------+
|create_tenant_with_users   | 37.672  | 72.488  | 260.214 | 417.182 | 30      |
+---------------------------+---------+---------+---------+---------+---------+
|assign_and_removeuser_role | 75.359  | 101.323 | 159.812 | 163.355 | 150     |
+---------------------------+---------+---------+---------+---------+---------+
|create_and_delete_role     | 16.571  | 20.585  | 23.143  | 29.165  | 150     |
+---------------------------+---------+---------+---------+---------+---------+
|create_and_delete_service  | 9.567   | 13.987  | 35.691  | 41.265  | 150     |
+---------------------------+---------+---------+---------+---------+---------+
|create_and_list_user_roles | 11.924  | 17.279  | 16.250  | 22.469  | 150     |
+---------------------------+---------+---------+---------+---------+---------+
|get_entities               | 2.431   | 4.724   | 20.309  | 22.459  | 15      |
+---------------------------+---------+---------+---------+---------+---------+
|get_token                  | 1.556   | 2.890   | 6.392   | 17.149  | 15      |
+---------------------------+---------+---------+---------+---------+---------+
|update_and_delete_tenant   | 12.583  | 17.237  | 18.141  | 25.379  | 150     |
+---------------------------+---------+---------+---------+---------+---------+
|update_user_password       | 18.320  | 16.987  | 42.551  | 41.364  | 150     |
+---------------------------+---------+---------+---------+---------+---------+
|boot_and_delete_server     | 269.515 | 311.886 | 297.314 | 347.193 | 300     |
+---------------------------+---------+---------+---------+---------+---------+

-----------------
Deployment impact
-----------------

None

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

Switching to Fernet tokens and manual Fernet keys rotation procedure should be
documented in Fuel Deployment Guide [1].

--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  Maksym Yatsenko <myatsenko>

QA engineers:
  Oleksandr Petrov <apetrov>

Mandatory design review:
  Sergii Golovatiuk <sgolovatiuk>
  Vladimir Kuklin <vkuklin>

Work Items
==========

* Implement enabling Fernet tokens.
* Perform fernet keys generation.
* Copy Fernet keys to all keystone
  nodes during deployment process.

Dependencies
============

None

------------
Testing, QA
------------

Manual Acceptance Tests
=======================

* Deploy HA-mode configuration
* All keystone nodes should contain identical fernet keys

HA/Destructive Tests
====================

* Token verification after controller failure
  * issue a token
  * stop a controller this token was issued
  * make sure token works

Scale
=====

Environment with enabled Fernet tokens should pass all tests currently run on
Scale Lab with no significant performance degradation.

Acceptance criteria
===================

After successfull deployment all keystone nodes contain identical fernet keys,
Keystone functions properly.

----------
References
----------

.. [1] `Fuel documentation <https://github.com/openstack/fuel-docs>`_
.. [2] `Blueprint <https://blueprints.launchpad.net/fuel/+spec/fernet-tokens-support>`_

