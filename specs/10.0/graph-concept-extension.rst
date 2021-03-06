..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Fuel Graph Concept Extension And Usage
======================================

https://blueprints.launchpad.net/fuel/+spec/graph-concept-extension

There is introduced a new opportunity that allows to execute graphs
for different purposes by the Fuel graph concept extension.


-------------------
Problem description
-------------------

Currently, the Fuel graph concept is tied to the deployment process. For
example, we can't use graphs for provisioning, deletion or verification.
Those actions are hardcoded in Nailgun and Astute, and there's no way to
extend extend them easily.

Meantime we want to see every action as a graph in order to make it pluggable
and extendable, since end users usually want to somehow change them.  For
instance, some of them want to use torrent protocol for image delivering
instead of HTTP and there's no way to change it so far.

Another problem is that we can't verify advanced network configuration in
bootstrap mode. The problem lies in our approach where network-checker is
responsible only for basic configuration while we need l23network manifest
to be applied in order to verify network against real configuration.
Having everything in the graphs allows to reuse that puppet manifest, and
hence prepare network for verification.

There're plenty of places where we have hardcoded actions instead of
declarative ones. Moving them into graphs will help to clean and simplify
our code base, as well as provide opportunity to customize them manually
or via plugins.


----------------
Proposed changes
----------------

#. **Transaction Manager**

   Nailgun should have a general transaction manager for running graphs as
   well as a bunch of them within a single transaction.

   The transaction manager must be used by the new RESTful API endpoint
   for executing graphs. See REST API section for details.

#. **Default Graphs for Basic Actions**

   At minimum we want to see the following actions as graphs:

   * Deployment (done)
   * Provisioning
   * Verification
   * Deletion

   Hence, fuel-library should provide tasks for those graphs the same
   way they provide them for deployment. The proposed way is to separate
   them on filesystem (drop into different directories) and sync them
   one be one by passing additional argument to Fuel CLI. Example:

   .. code-block:: console

      fuel rel --sync-deployment-tasks --dir /etc/puppet/ --graph provision

#. **Scenarios**

   Scenarios is the way to run specified graphs one-by-one, each on pre-defined
   set of nodes. A set of nodes could be specified either explicitly or by
   using YAQL expression.

   Scenarios is a good way to provide a high level orchestration flows such
   as "Deploy Changes" in declarative manner.

#. **New Astute tasks**

   In order to support existing scenarios as graphs we need to implement the
   following tasks in task-based format in Astute:

   * ``erase_node`` - run mcollective erase_node action
   * ``master_shell`` - execute a task on the master node with a particular
     node context
   * ``move_to_bootstrap`` - reregister node with a bootstrap profile in
     cobbler

#. **New method of nodes statuses update**

   In order to get rid of hardcoded state machine of node statuses, we
   need to provide a way to set node statuses in a data driven format.
   Hence, it's proposed to add a set of callbacks: ``on_success``, ``on_error``
   and ``on_stop``.

   .. code-block:: yaml

      graph_metadata:
         on_success:
            node_attributes:
               status: ready
         on_error:
            node_attributes:
               status: error
               error_type: deploy
         on_stop: null


Web UI
======

Custom graphs management in Fuel UI was described and implemented within the
[1], although the ability to execute a sequence of graphs is introduced in this
spec as extension.

Working in 'Custom Scenarios' deployment mode, user should be able to specify
a sequence of space-separated graph types, that he wants to execute.

Also, it is necessary to use a new ``/api/v1/graphs/execute/`` handler (that
works with transactions manager) in Fuel UI to run a graph/graphs.


Nailgun
=======

Data model
----------

#. Having everything defined as a graph and mechanism to run few graphs within
   a single transaction, simple means we can't rely on task's name anymore. It
   makes more sense to distinguish runs by two criteria: ``graph_type`` and
   ``dry_run``. So it's proposed to extend ``tasks`` table with those columns
   and mark ``tasks.name`` as deprecated column.

#. Transient node statuses shouldn't be persisted in database. That means
   ``nodes::status`` attribute should contain either ``discover`` or
   ``provisioned`` or ``deployed``. Statuses ``provisioning``, ``deploying``
   and ``error`` should be calculated based on node attributes.

   * ``provisioning`` = ``discovery`` + ``progress >= 0``
   * ``deploying`` = ``provisioned`` + ``progress >= 0``
   * ``error`` = ``error_type`` is not ``null``

   When any action is committed the ``progress`` should be resetted to
   ``100``.

   ``error_type`` should not be limited to pre-defined set of types.

#. In order to implement scenarios, we need to design a database schema for
   new entity. Here's a proposed solution:

   .. code-block:: text

      .
                                    SCENARIOS_ACTS
        SCENARIOS               +--------------------+
      +-----------+             | + id (pk)          |
      | + id (pk) |<------------| + scenario_id (fk) |
      | + name    |             | + order            |
      +-----------+             | + graph_type       |
                                | + nodes            |
                                +--------------------+

   where:

   * ``scenarios::name`` is a unique identifier to be used by clients for
     running scenarios;
   * ``scenarios_acts::scenario_id`` is a foreign key to ``scenarios``;
   * ``scenarios_acts::order`` is an execution order in scenario;
   * ``scenarios_acts::graph_type`` is a graph type to run;
   * ``scenarious_acts::nodes`` is a JSON column that may contain either
     hardcoded JSON array with nodes IDs or JSON object with ``yaql_exp`` key
     for getting nodes IDs on fly;

   Executing scenarios mean: run its graphs on corresponding set of nodes
   within a single transaction.


REST API
--------

#. **Graphs Execution**

   .. http:post:: /graphs/execute

      Execute passed graphs.

      **Request:**

      .. code-block:: http

         POST /graphs/execute HTTP/1.1

         {
            "cluster": <cluster-id>,
            "graphs": [
               {
                  "type": "graph-type-1",
                  "nodes": [1, 2, 3, 4],
                  "tasks": ["task-a", "task-b"]
               },
               {
                  "type": "graph-type-2",
                  "nodes": [3, 4],
                  "tasks": ["task-c", "task-d"]
               },
            ],
            "dry_run": false,
            "force": false
         }

      where:

      * ``cluster`` -- cluster id;
      * ``graphs`` -- list of graphs to be executed, with optional ``nodes``
        and ``tasks`` params;
      * ``dry_run`` (optional, default: false) -- run graphs in dry run mode;
      * ``force`` (optional, default: false) -- execute tasks anyway; don't
        take into account previous runs.

      **Response:**

      .. code-block:: http

         HTTP/1.1 202 Accepted

         {
            "task_uuid": "transaction-uuid",
            ...
         }

      where:

      * ``task_uuid`` -- unique ID of accepted transaction

   As the graph term was extended, some requests should be modified to avoid
   misunderstanding. In the following requests the deployment/deploy word
   should be removed:

   * ``GET /releases/<release_id>/deployment_graphs/``
   * ``GET/POST/PUT/PATCH/DELETE /releases/<release_id>/deployment_graphs/<graph_type>/``
   * ``GET /releases/<release_id>/deployment_tasks/``
   * ``GET /clusters/<cluster_id>/deployment_graphs/``
   * ``GET /clusters/<cluster_id>/deployment_tasks/``
   * ``GET/POST/PUT/PATCH/DELETE /clusters/<cluster_id>/deployment_graphs/<graph_type>/``
   * ``GET /plugins/<plugin_id>/deployment_graphs/``
   * ``GET/POST/PUT/PATCH/DELETE /plugins/<plugin_id>/deployment_graphs/<graph_type>/``
   * ``GET /clusters/<cluster_id>/deploy_tasks/graph.gv``

#. **Scenarios**

   .. http:post:: /scenarios

      Create a new workflow.

      **Request:**

      .. code-block:: http

         POST /scenarios HTTP/1.1

         {
            "name": "deploy-changes",
            "scenario": [
               {
                  "graph_type": "provision",
                  "nodes": {
                     "yaql_exp": "select nodes for provisioning"
                  }
               },
               {
                  "graph_type": "deployment"
                  "nodes": ...,
               }
               ...
            ]
         }

   .. http:get:: /scenarios

      List available scenarios.

      **Response:**

      .. code-block:: http

         HTTP/1.1 200 Ok

         [
            {
               "id": 1,
               "name": "deploy-changes",
               "scenario": [
                  ... scenario's acts ...
               ]
            },
            {
               "id": 2,
               ...
            }
         ]

   .. http:post:: /scenarios/:name/execute

      Run a scenarios with a given ``name``. If successful a transaction ID
      is returned.

      **Response:**

      .. code-block:: http

         HTTP/1.1 202 Accepted

         {
            "task_uuid": "transaction uuid"
         }


Orchestration
=============

None

RPC Protocol
------------

None


Fuel Client
===========

For listing/uploading/downloading will be used the common custom graph commands
[0].

The graph execution command should stay practically the same, however it is
necessary to be able to define several graph types to run them one by one. Also
it should be possible to enforce execution of tasks without skipping and to run
only specific tasks ignoring dependancies.

.. code-block:: console

    fuel2 graph execute --env 1 [--nodes 1 2 3]
                                [--graph-types gtype1 gtype2]
                                [--task-names task1 task2]
                                [--force]
                                [--dry-run]

where

* ``--nodes`` executes only on passed nodes;
* ``--graph-types`` executes passed graphs within one transaction;
* ``--task-names`` executes only passed tasks skipping others;
* ``--force`` executes tasks anyway;
* ``--dry-run`` executes in dry-run mode (doesn't affect nodes)


Plugins
=======

None


Fuel Library
============

* Compose the default provisioning and deletion graphs.

* Compose the default verification graph. This graph should contain tasks
  for the network configuring and checking.

* All default graphs should be loaded during the Fuel installation with
  the corresponding graph types.


------------
Alternatives
------------

None for the whole approach.

For the verification tool:

* Use the standard network verification mechanism, although in this
  case we have a deal with non-realistic network configuration.
* Use connectivity checker plugin [2] to verify network during
  the deployment, but it will take more time to rework.


--------------
Upgrade impact
--------------

Some API endpoints are renamed so it breaks backward compatibility.

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

Ability to:

* execute different graphs for different purposes.

* check the realistic network configuration design before the deployment
  process.


------------------
Performance impact
------------------

None


-----------------
Deployment impact
-----------------

The whole mechanism is more flexible. The provisioning part is configurable
and easier to debug. Thanks to the verification graph mechanism, errors
detection before the deployment stage may save a lot of time in case of
reconfiguration necessity.


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

* API, CLI and UI documentations should be extended according to the
  appropriate changes.


--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  bgaifullin

Other contributors:
  vsharshov (astute)
  sbogatkin (library: deletion, provisioning)
  lefremova (library: verification)
  ikutukov  (client)

Mandatory design review:
  ashtokolov
  vkuklin


Work Items
==========

* Implement transaction manager that runs a bunch of graphs one by one,
  each with own context generated on top of changes committed by previous
  graph.

* Implement new Astute tasks for moving nodes to bootstrap, running shell
  tasks on master node with context of other roles and removing nodes.

* Implement new graphs to run provisioning, deployment, deletion and
  verification.

* Implement CLI interface to run graphs in one transaction.

* Implement Fuel UI to run graphs in one transaction as well as scenarios.


Dependencies
============

Custom graph management on UI [1].


-----------
Testing, QA
-----------

* New logic in nailgun should be covered by unit and integration tests.

* Functional tests that executes verification and provisioning graphs on
  bootstrap nodes should be introduced.


Acceptance criteria
===================

* The Fuel graph concept is extended so we can use a graph mechanism
  for different purposes.

* Network checking tool in Fuel is introduced for realistic configurations
  via execution an appropriate verification graph on bootstrap nodes.
  So as a cloud operator I have the possibility to investigate the production
  specific network defects before the deployment.

* Provisioning and deletion mechanisms also work via the corresponding graphs
  execution.

* While the default graphs for the base actions are loaded during the Fuel
  insallation, user may specify and execute custom graphs.


----------
References
----------

[0] Allow user to run custom graph on cluster
    https://blueprints.launchpad.net/fuel/+spec/custom-graph-execution
[1] Custom graph management on UI
    https://blueprints.launchpad.net/fuel/+spec/ui-custom-graph
[2] Connectivity checker plugin
    https://github.com/xenolog/fuel-plugin-connectivity-checker
