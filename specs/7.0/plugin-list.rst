..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Plugins page in Fuel Web UI
===========================

https://blueprints.launchpad.net/fuel/+spec/ui-plugins-list

Customer who has installed one or more plugins in Fuel, should be able
to view them on the Fuel Web UI.

Problem description
===================

Currently we haven't possibility to view information about installed
plugins on the Fuel Web UI.

Proposed change
===============

Create a separate page in UI with appropriate information about installed
plugins.

This page should be on the same level like Environments, Releases and Support,
between Releases and Support.

Page shows the list of installed plugins, every plugin in the list should
contain next fields: name, version, description, compatibility information
with MOS (releases), provider (authors), category (group).

Installed plugins list information comes from GET request to /api/plugins url:

.. code-block:: json

  [
    {
      "name": "plugin_name",
      "version": "1.0",
      "description": '',
      "authors": '',
      "releases": '',
      "group": ''
    },
    ...
  ]

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None

Upgrade impact
--------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Plugin impact
-------------

Described above.

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Kate Pimenova (kpimenova@mirantis.com)

Developers:

* Vitaly Kramskikh (vkramskikh@mirantis.com) - JS code
* Kate Pimenova (kpimenova@mirantis.com) - JS code
* Bogdan Dudko (bdudko@mirantis.com) - visual design

Work Items
----------

* Implement appropriate UI page.
* Create new link into navigation.

Dependencies
============

None

Testing
=======

* Install a plugin, check that it represented on the Plugins page.
* Plugin lists in UI should be covered by functional tests.

Aceptance Criteria
------------------

* Fuel WEB UI contains new link in the main navigation menu to new
  Plugins page. There is a list of installed plugins, each plugin
  should contain next fields: name, version, description,
  compatibility information with MOS (releases), provider (authors),
  category (group).

Documentation Impact
====================

The documentation should cover how the end user experience has been changed.

References
==========

None
