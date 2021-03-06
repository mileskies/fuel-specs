..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Support for multi-rack in Fuel UI
=================================

https://blueprints.launchpad.net/fuel/+spec/multirack-in-fuel-ui

Fuel UI should allow End User to manage OpenStack deployment for multiple
racks with a scalable underlay network design.


-------------------
Problem description
-------------------

Current Fuel UI does not have a support for multi-rack management. Fuel users
are forced to use CLI or raw API to manage their multi-rack environments.

Also it is impossible to observe all Fuel nodes regardless environment
assignment in Fuel UI. At the same time such a node list with standard
functionality (sorting, filtering, labelling) will make multi-racks management
more efficient.


----------------
Proposed changes
----------------

Web UI
======

Environment page changes:

#. Existing sorting and filtering controls in node management panel should
   include new `Node network group` option (which corresponded to `group_id`
   attribute of `Node` model).
   All the other sorting and filtering behaviour remains the same.

   When `Node network group` filter is applied and specific node network group
   chosen, only nodes from this network group should be shown in the node
   list.
   `Node network group` filter should contain a list of network group names
   (that can be taken from `GET /api/nodegroups?cluster_id=<id>` request
   response). Node network groups should be sorted by their `id` in the
   options list, so that a default node network group, which has a minimum
   id among the environment node network groups, will go first in the list.

   When `Node network group` sorter is applied, node list should be grouped by
   node network group name. Nodes from default node network group should go
   first in the list.

   This change affects environment node list only, as well as unallocated
   nodes do not belong to any node network group (`group_id` node attribute
   has `null` value in this case).

     .. image:: ../../images/8.0/multirack-in-fuel-ui/
        node-net-group-sorter-filter.png
        :scale: 75 %

#. Networks tab changes:

   #. The list of node network groups can be seen at Networks tab

   #. Networks tab is segmented into sections for node network groups:

      * one, called 'Networks' if only one node network group is given;

      * separate sections in case of multi-rack OpenStack environment (several
        node network groups)

   #. Neutron L2 and Neutron L3 settings will form separate groups, as well as
      network verification control. For multi-rack OpenStack environments
      there will be a warning telling that network verification is disabled
      for environments containing more than one node group:

      .. image:: ../../images/8.0/multirack-in-fuel-ui/
         verification_control.png

   In case of Nova network - Nova configuration will also be available in a
   separate section.

   Also it will be possible to see all node network groups at once by clicking
   'Show all networks' checkbox.

   Node network groups will appear as pills, sorted by id, on the left side of
   the screen on Networks tab, so by switching them the user will be able to
   see which networks and which  parameters correspond to which node network
   group and configure them.

     .. image:: ../../images/8.0/multirack-in-fuel-ui/node_network_groups.png

   After changing some network settings for particular node network group it
   will be possible to switch to the other node network group and change
   settings for it without confirmation dialog on leaving unsaved data. In
   case of invalid network settings appropriate section will be marked as
   invalid. All the changes are applied only after clicking Save Settings
   button at the bottom of the page. Saving errors will be displayed at the
   bottom of the tab, right above 'Save' button. Network verification result
   will be shown only on Network Verification section.

   Before deleting a node network group user will see a confirmation dialog to
   prevent accidental node network group deletion. It should be not possible
   to delete Default node network group. To show this there will be a prompt
   on mouseover, describing defaut node network group specialties.

   To create a new node network group user will be prompted to enter its name:

     .. image:: ../../images/8.0/multirack-in-fuel-ui/new_group.png

   Additional notes:

   #. Default node network group should always be the first one on the list.

   #. It should not be possible to create new node network group without
      saving changes. So, it would be neccessary to first save existing
      changes and only after that - create a new node network group.

   #. Gateways must be set for networks in clusters with two or more node
      network groups is present (in case of multi-rack environment).

   #. Arbitrary node network group names should be supported - it should be
      possible to modify node network group names on UI.

New root-level 'Equipment' page:

#. 'Equipment' page should appear in main Fuel menu and display a list of
   nodes in Fuel across all environments, including unallocated nodes.

#. 'Equipment' page should represent a standart node list functionality:

   * display list of grouped nodes
   * support node list sorting (by status (default sorting in the list),
     environment, roles, node network group, name, MAC address, IP address,
     manufacturer, real cores, total cores, HDD size, disks sizes, RAM size,
     interfaces amount)
   * support node list filtering (by node network group, environment, roles,
     status, manufacturer, real cores, total cores, HDD size, disks amount,
     RAM size, interfaces amount)
   * support searching through the node list by node name, MAC or IP address
   * support standard and compact modes of node view
   * support node labels management
   * support of node actions, which do not depend on environment
     (like removing from Fuel for offline nodes)

   .. image:: ../../images/8.0/multirack-in-fuel-ui/equipment-page.png
      :scale: 75 %

Node pop-up should be extended with the following data:

* node network group name (if the node assigned to some group)
* environment name (if the node assigned to some environment)

  .. image:: ../../images/8.0/multirack-in-fuel-ui/node-details-popup.png
     :scale: 75 %


Nailgun
=======

Data model
----------

Master node settings DB model should be extended with a new `ui_settings`
attribute of JSON type to store settings of 'Equipment' page. Default value
of the attribute should be:

  .. code-block:: json

     {
       "view_mode": "standard",
       "filter": {},
       "sort": [{"status": "asc"}],
       "filter_by_labels": {},
       "sort_by_labels": [],
       "search": ""
     }

* `view_mode` property has one of the following values: "standard" (default)
  or "compact"
* `filter` is a hash of applied filters in the following format:

  .. code-block:: json

     {"status": ["discovered", "error", ...], ...}

  (no filters applied by default).

* `sort` is a list of applied sorters in the following format:

  .. code-block:: json

    [{"status": "asc"}, {"roles": "desc"}, ...]

  Default is a `[{"status": "asc"}]` for the property.

* `filter_by_labels` has the same format as `filter` but filter names
  are from Fuel node labels scope (no label filters applied by default).
* `sort_by_labels` has the same format as `sort` but sorter names
  are from Fuel node labels scope (no label sorters applied by default).
* `search` is a string to search nodes by their name, IP or MAC address
  (default value is an empty string that means no active search).

All the properties above are mandatory for `ui_settings` attribute value.

This new `ui_settings` attribute should be validated on backend using JSON
schema.

REST API
--------

No changes to the existing API is required. The following existing API entries
will be reused:

#. To create a new node network group POST request should be sent to
   `/api/v1/nodegroups/`:

   .. code-block:: json

      {
        "cluster_id": 1,
        "name": "group Name"
      }

#. To rename a node network group it is necessary to send a PUT request to
   `/api/v1/nodegroups/<node_network_group_id>/`:

   .. code-block:: json

      {
        "name": "group Name"
      }


#. To remove a node network group it is needed to send DELETE request to
   `/api/v1/nodegroups/<node_network_group_id>`

#. To get or change node network group networks configuration GET or PUT
   request will be send to `/api/clusters/<cluster_id>/network_configuration
   /neutron` with configuration data

#. To get UI settings for 'Nodes' page GET request should be sent to
   `/api/settings`

#. To update UI settings of 'Nodes' page PUT request should be sent to
   `/api/settings` with the data of the following format (the method response
   should have exactly the same format):

   .. code-block:: json

      {
        "node_list_settings": {
          "view_mode": "standard",
          "filter": {"status": ["error", ...], ...},
          "sort": [{"environment": "asc"}, ...],
          "filter_by_labels": {"some_label": ["value1", ...], ...},
          "sort_by_labels": [],
          "search": ""
        }
      }

   Ok code 200, expected error HTTP response code: 400 Bad Request in case of
   malformed request body or missing parameters.


Orchestration
=============

No changes required.


RPC Protocol
------------

No changes required.


Fuel Client
===========

No changes required.


Plugins
=======

No changes required.


Fuel Library
============

No changes required.


------------
Alternatives
------------

None


--------------
Upgrade impact
--------------

Since there is a data model impact, apropriate Alembic migration should be
prepared.


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

Fuel UI feature only.


------------------
Performance impact
------------------

None


-----------------
Deployment impact
-----------------

None


----------------
Developer impact
----------------

New `ui_settings` attribute of master node settings model will be available
for storing other UI settings, which are not related to some particular
environment.


---------------------
Infrastructure impact
---------------------

None


--------------------
Documentation impact
--------------------

User guide should be updated to document the changes described above.


--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  vkramskikh (vkramskikh@mirantis.com)

Mandatory design review:
  vkramskikh (vkramskikh@mirantis.com)
  alekseyk-ru (akasatkin@mirantis.com)

Other contributors:
  jkirnosova (jkirnosova@mirantis.com)
  astepanchuk (astepanchuk@mirantis.com)
  bdudko (bdudko@mirantis.com)

QA engineer:
  apalkina (apalkina@mirantis.com)


Work Items
==========

#. Include node network group option to environment node list sorters
   and filters scope
#. Display node network group name in the node details pop-up
#. Reorganize Networks tab to include common network settings, verification
   block changes and node network groups list
#. Implement node network groups creation and editing support
#. Create new root-level 'Equipment' page in Fuel UI with all Fuel nodes list
   and standart node list management functionality
#. Display environment name in the node details pop-up
#. Support saving and updating of node list settings of 'Equipment' page
   from UI
#. Extend master node settings DB model with the new `ui_settings` attribute
#. Prepare Alembic migrations and update JSON schemas
#. Cover the changes with tests


Dependencies
============

None


------------
Testing, QA
------------

* Manual testing
* Functional UI auto-tests should be updated according the changes


Acceptance criteria
===================

* It is possible to sort environment node list by node network group
  in Fuel UI
* It is possible to filter environment node list by node network group
  in Fuel UI
* Node network group name is shown in node details pop-up (if the node
  assigned to some group)
* Networks tab is segmented with the list of node network groups
* It is possible to create a new node network group
* It is possible to edit and delete existing node network groups
* It is possible to configure networks separately for each node network group
* It should not be possible to delete default node network group
* It should be possible to view and manage all Fuel nodes across all
  environments including unallocated nodes
* Node list custom settings (applied sorters, filters, etc.) of 'Equipment'
  page are stored in DB, so the page has the same configuration after refresh
* Environment name should be shown in node pop-up (if the node assigned
  to some environment)

----------
References
----------

#fuel-ui on freenode
