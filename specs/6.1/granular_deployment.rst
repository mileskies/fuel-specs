..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Granular deployment for fuel roles
==================================

Problem description
===================

Our deployment process is very complicated. There are a lot of Puppet modules
used together by our manifests and dependencies between these modules are also
very complex and abundant.
This leads to the following consequences:
- It becomes very difficult to add new features. Even changes that could
look minor from the first glance can easily and randomly break any other
functionality. Trying to guess how any change could affect dependencies
and ordering of deployment process is very hard and error-prone.
- Debugging is also affected. Localizing bugs could be very troublesome due
to manifests being complex and hard to understand. Debugging tools are
also almost non-existent.
- Reproducing bugs and testing takes lots of time because we have no easy
and reliable way to repeat only some part of the deployment. The only thing
we can do is to start the process again and wait for several hours to get any
results. Snapshots are not very helpful because deployment cannot be reliably
stopped and the state saved. These actions most likely break deployment or at
least change its outcome.
- New members of our team or outside developers who want to add some new
functionality to our project are completely out of luck. They will have to
spend many days just to gain minor understanding how our deployment works.
And most likely will make a lot of hard to debug mistakes.
- Using our product is also not as easy as we would like it to be for customers
and other people in the community. People usually cannot easily understand how
the deployment works and have to just follow every step in documentation. It
makes them unable to act reasonably if something goes wrong.


Proposed change
===============

If we want to address any of these issues we should find a way to make our
architecture less complex and more manageable. It’s known that the best way
to understand any large and monolithic structure is to to take it apart and
then learn how does each of the pieces work and then how do they interact with
each other.

So we should try to separate the whole deployment process to many small parts
that could do only one or several closely related tasks. Each of these parts
would be easy to understand for a single developer. Testing and debugging could
also be done separately so localizing and fixing bugs would be much easier than
it is now.

Thinking about the deployment process as a list of atomic tasks will make our
reference architectures and server roles much more dynamic. If you change what
tasks you are going to perform and their order you can create as many custom
sets of roles as you need without any modification of the tasks themselves.

Each task can have some internal dependencies but most likely there would not
be too many of them. It will make manual analyzing of dependency graph possible
within a single task. The task can also have some requirements. System should
be in the specific state before the task can be started.

The introduction of Granular Deployment will be a rather extensive change
to almost all components of the Fuel project and a serious architectural
modification.


Graph-based Task API
---------------------

Several types of tasks will be introduced in addition to basic deployment
types, like puppet, shell, rsync, upload_file. This types are groups and
stages, and they will serve the purpose to build flexible graph of tasks.

Types of tasks::

    - type: group - grouping of the tasks based on nailgun role entities
    - type: stage - skeleton of deployment graph in fuel, right now there
      is 3 stages: pre_deployment, deployment, post_deploment

    deployment tasks:
     - type: puppet - executes puppet manifests with puppet apply
     - type: shell - executes any shell command, like python script.py,
     /script.sh
     - type: upload_file - used for configuration upload to target nodes,
     repo creation
     - type: rsync

* pre_deployment - other actions, including plugin
* deployment - executes main deployment stage
* post_deployment - actions that can be executed only after whole deployment is
  done

Ideally all dependencies between tasks should be described with
requires and required_for attributes, it will allow us to build graph
of tasks in nailgun and then serialize it into orchestrator acceptable format
(workbooks for mistral, or astute-speficic roles with priorities).

type: GROUP
-------------

::

    id: controller
    type: group
    role: [<roles>]
    requires: [<groups>]
    required_for: [<stages>, <roles>]
    tasks: [<tasks>]
    parameters:
        strategy:
            type: parallel
            amount: 8

- each chunk of nodes with this role (8 in this example) will be executed
  in parallel

::
    strategy:
        type: one_by_one

- all nodes with this role will be executed sequentially

::
    strategy:
        type: parallel

- all nodes with this role will be executed in parallel

::
    tasks: [<tasks>]

- this section required for ease of understanding, which tasks belong where

type: STAGE
------------

::

    id: deploy
    type: stage
    requires: [<stages>]

Right now we are using hardcoded set of stages, but it is completely possible
to make it flexible, and define them with API.

type: DEPLOYMENT TASK TYPES
----------------------------

::

    id: deploy_legacy
    type: puppet
    role: [primary-controller, controller,
           cinder, compute, ceph-osd]
    requires: [<tasks>]
    required_for: [<stage>]
    parameters:
        puppet_manifest: /etc/puppet/manifests/site.pp
        puppet_modules: /etc/puppet/modules
        timeout: 360

    id: network
    type: shell
    groups: [primary-controller, controller]
    requires: [deploy_legacy]
    required_for: [deploy]
    parameters:
        cmd: python /opt/setup_network.py
        timeout: 600


Conditional tasks
----------------------

Major part of tasks will require conditional expressions.
There is several ways to solve it:

1. Implement python framework for pluging a task. Each task will have
clear interface for defining condition for a task, and if this condition passes
- task will be serialized.
This the most scalable and solid solution, but developing such
framework will require a lot of effor, and we wont be able to land it in 6.1

2. Define conditions in custom expression parser that is also used on UI.
There is couple of downsides with this approach:
- Not all conditions can be expressed. For example,
if zabbix-role present in cluster - deploy zabbix-agent for each role
- It is new expression language, which we need to support ourselves
- It depends on context data, which is quite easy to change

3. Define certain groups for tasks, and each mutually exclusive
task will be able to specify its group.
- This wont work with conditions that are not mutually exclusive.

4. Using strict API for conditions that can be used in expressions parsing.
Pros:
- it is not a new language
- it has very strict api, so atleast we can try to guarantee its stability
- complex abstract logic can be hidden in simple python methods

Stements will be expressed in the form of:

api.cluster_status == 'operational'
api.role_in_deployment('ceph-osd')
api.role_in_cluster('zabbix-server')
api.cluster_status == 'new' and api.nodes_count() > 10

::

    class ExpressionApi(object):

        def __init__(self, cluster, nodes):
            self.cluster = cluster
            self.nodes = nodes

        def role_in_deployment(self, role):
            for node in self.nodes:
                if role in node.roles:
                    return True
            return False

        def role_in_cluster(self, role):
            for node in self.cluser.nodes:
                if role in node.roles:
                    return True
            return False

        def nodes_count(self):
            return len(self.nodes)

        @property
        def cluster_status(self):
            return self.cluster.status

    env = jinja2.Environment()
    expr = env.compile_expression("api.cluster_status == 'operational'
                                   and api.nodes_count() < 9")
    print expr(api=API)

In 6.1 we will either stick to existing expression language that is used
for cluster settings validation.

Operators are available in [2].

Usage of graph in nailgun
------------------------------------
Based on provided tasks and dependencies between tasks we will build
graph object with help of networkx library [1].
Format of serialized information will depend on orchestrator that we will use
in any particular release.

Let me provide an example:

Consider that we have several types of roles:

::

    - id: deploy
      type: stage
    - id: primary-controller
      type: group
      role: [primar-controller]
      required_for: [deploy]
      parameters:
        strategy:
          type: one_by_one
    - id: controller
      type: group
      role: [controller]
      requires: [primary-controller]
      required_for: [deploy]
      parameters:
        strategy:
          type: parallel
          amount: 2
    - id: cinder
      type: group
      role: [cinder]
      requires: [controller]
      required_for: [deploy]
      parameters:
        strategy:
          type: parallel
    - id: compute
      type: group
      role: [compute]
      requires: [controller]
      required_for: [deploy]
      parameters:
        strategy:
            type: parallel
    - id: network
      type: group
      role: [network]
      requires: [controller]
      required_for: [compute, deploy]
      parameters:
        strategy:
            type: parallel

And there is defined tasks for each role:

::

    - id: setup_services
      type: puppet
      requires: [setup_network]
      groups: [controller, primary-controller, compute, network, cinder]
      required_for: [deploy]
      parameters:
        puppet_manifests: /etc/puppet/manifests/controller.pp
        puppet_modules: /etc/puppet/modules
        timeout: 360
    - id: setup_network
      type: shell
      groups: [controller, primary-controller, compute, network, cinder]
      required_for: [deploy]
      parameters:
        cmd: run_setup_network.sh
        timeout: 120

For each role we can define different subsets of tasks, but for simplicity
lets make this tasks applicable for each role.

Based on this configuration nailgun will send to orchestrator config
in expected by orchestator format.

For example we have several nodes for deployment:

::
    primary-controller: [node-1]
    controller: [node-4, node-2, node-3, node-5]
    cinder: [node-6]
    network: [node-7]
    compute: [node-8]

This nodes will be executed in following order:
Deploy primary-controller node-1
Deploy controller node-4, node-2 - you can see that parallel amount is 2
Deploy controller node-3, node-5
Deploy network role node-7 and cinder node-6 - they depend on controller
Deploy compute node-8 - compute depends both on network and controller

During deployment for each node 2 tasks will be executed sequentially:

Run shell script setup_network
Run puppet setup_services

Pre/post_deployment task examples
---------------------------------

::

    - id: update_hosts
      type: puppet
      role: '*'
      stage: post_deployment
      requires: [upload_nodes_info]
      parameters:
        puppet_manifest: /etc/pupppet/modules/update_hosts_file.pp
        puppet_modules: /etc/puppet/modules 16
        timeout: 3600
        cwd: /

    - id: rsync_puppet
      type: rsync
      role: '*'
      stage: pre_deployment
      parameters:
        src: /etc/pupppet/{VERSION}
        dst: /etc/puppet/modules
        timeout: 3600


Alternatives
------------

Execute deployment based not on roles, but on tasks.
To consider this as alternative we need to modularize atleast each deployment
role as separate manifest. So in current deployment model, there will be
next set of manifests:

    - controller.pp
    - mongo.pp
    - ceph_osd.pp
    - cinder.pp
    - zabbix.pp
    - compute.pp

After this is done it is quite easy to transfrom this in simple set of tasks:

::

    - id: primary-controller
      type: puppet
      required_for: [deploy]
      role: [primary-controller]
      strategy:
          type: one_by_one
      parameters:
        puppet_manifest: /etc/puppet/controller.pp
    - id: controller
      type: puppet
      requires: [primary-controller]
      required_for: [deploy]
      strategy:
          type: parallel
          amount: 2
      parameters:
        puppet_manifest: /etc/puppet/controller.pp
    - id: compute
      type: puppet
      requires: [controller]
      strategy:
        type: parallel
      parameters:
        puppet_manifest: /etc/puppet/compute.pp
    - id: cinder
      type: puppet
      requires: [controller]
      strategy:
        type: parallel
      parameters:
        puppet_manifest: /etc/puppet/cinder.pp
    - id: ceph-osd
      type: puppet
      requires: [controller]
      strategy:
        type: parallel
      parameters:
        puppet_manifest: /etc/puppet/ceph.pp

As you see there is no separation between tasks and roles.
For example there is next set of roles to nodes:

::

    primary-controller: [node-1]
    controller: [node-4, node-2, node-3, node-5]
    cinder: [node-6]
    ceph-osd: [node-7]
    compute: [node-8]

Deploy /etc/puppet/controller.pp on uids [1]
Deploy /etc/puppet/controller.pp on uids [2,3] in parallel
Deploy /etc/puppet/controller.pp on uids [4,5] in parallel
Deploy /etc/puppet/compute.pp on uids [8] and
Deploy /etc/puppet/cinder.pp on uids [6] and
Deploy /etc/puppet/cinder.pp on uids [7] in parallel

Current model will allow us to make multiple cross-reference tasks, like:

::

    - id: put_compute_into_maintenance_mode
      type: puppet
      role: [primary-controller]
    - id: migrate_vms_from_compute
      type: puppet
      role: [primary-controller]
      requires: [put_vm_into_maintenance_mode]
    - id: reinstall_ovs
      type: puppet
      role: [compute]
      requires: [put_vm_into_maintenance_mode, migrate_vms_from_compute]
    - id: make_compute_available
      role: [primary-controller]
      requires: [reinstall_vs]

It is not full format, but in general it will do next things:

1. Put vm into maintanance mode
2. Migrate all virtual machines from this vm
3. Reinstall ovs (or any risky/disruptibe action)
4. Put this vm back into available mode

In nailgun rpc receiver we will need to track status of each node deployment
ourselvers, by validations process of tasks performed. So task executor
(astute) will send which task is completed after each puppet execution.

In case if role was not present at the time of writing deployment_graph,
it will specify all tasks it wants to execute in metadata for this role.

Data model impact
-----------------

Astute facts:
Nailgun will generate additional section for astute facts.
This section will contain list of tasks with its priorities for specific role.
Astute fact will be extended with tasks exactly in same format it is stored
in database, so if we are generating fact for compute role,
astute will have section like:
::

    tasks:
        -
          priority: 100
          type: puppet
          uids: [1] - this is done for compatibility reasons
          parameters:
            puppet_manifest: /etc/network.pp
            puppet_modules: /etc/puppet
            timeout: 360
            cwd: /
        -
          priority: 100
          type: puppet
          uids: [2]
          parameters:
            puppet_manifest: /etc/controller.pp
            puppet_modules: /etc/puppet
            timeout: 360
            cwd: /


Each astute.yaml will have part of deployment graph executed for
that particular role.

REST API impact
---------------

Several API requests will be added:

GET/PUT clusters/<cluster_id>/deployment_tasks
Reads, updates deployment configuration for concrete cluster.
It will be usefull if someone wants to execute deployment in unique order.


GET/PUT releases/<release_id>/deployment_tasks
Reads, updates deployment configuration for release

GET will support filters by start_task and end_task parameters:

GET releases/<release_id>/deployment_tasks/?end_task=netconfig&start_task=hiera

Will return all tasks that should start from start_task and finish
at end_task


CLI Api impact
--------------

Several commands will be added to operate on tasks and to manipulate
deployment API

Download/Upload deployment tasks from nailgun API will be available both
for clusters and releases, by default dir parameter is current directory.

fuel rel --rel 2 --deployment-tasks --download --dir ./
fuel rel --rel 2 --deployment-tasks --upload --dir ./

fuel env --env 2 --deployment-tasks --download --dir ./
fuel env --env 2 --deployment-tasks --upload --dir ./

Sync deployment tasks for releases:

fuel rel --sync-deployment-tasks --dir /etc/puppet

All tasks.yaml that will be found recursively in directory "dir" will be merged
and sended for correct release version, there is 2 approaches that can be taken
to match releases to tasks:
1. Match them by path
2. Match by config file that will on root level of tasks directory structure

::
  fuel rel --sync-deployment-tasks will be performed during master bootstrap.

Next set of commands is about deployment API, in general we will have
ability to construct custom graph for concrete nodes.

::
  fuel node --node 2 --env 2 --tasks netconfig hiera

Only this tasks will be executed on specified nodes.

::
  fuel node --node 2,3 --env 2 --skip netconfig

Tasks specified in netconfig will be dropped from deployment.

::
  fuel node --node 2,3,4 --env 2 --end pre_deployment

Tasks required for pre_deployment to be ready will be executed,
in this API we will traverse graph up to pre_deployment and execute those tasks

::
  fuel node --node 2,3,4 --env 2 --start netconfig --end galera

Start at netconfig and end execution at task that is used for galera
installation.


Upgrade impact
--------------

After 6.1 release that task API that will be done as part of this feature
will be considered as stable task API and we are going to support tasks
described in that order.

Versioning will be done based on MOS version, so all tasks in any
given version should conform to certain API version or not.

Deployment configuration will be stored in

Cluster.deployment_tasks
Release.deployemtn_tasks

Initially graph configuraton will be filled on bootstrap_master_node stage,
by api call to /release/<id>/deployment_tasks

If there will be any kind of incopatibilities with new deployment code and
previous stored data - it will be possible to solve by migration or
modification from upgrade script (by API calls).

Security impact
---------------

Notifications impact
--------------------

Other end user impact
---------------------

Performance Impact
------------------

Wont significantly affect deployment time.
Maybe for some cases puppet run will be shorter.

Other deployer impact
---------------------

We will need to put tasks from fuel-library for each release in nailgun,
at the stage of bootstrap admin node.

Developer impact
----------------

Implementation
==============

Assignee(s)
-----------

Feature lead:
- Dmitry Shulyak dshulyak@mirantis.com

Devs:
- Vladimir Sharshov vsharhov@mirantis.com
- Sebastian Kalinowski skalinowski@mirantis.com
- Kamil Sambor ksambor@mirantis.com

Library:
- Dmitry Ilyin dilyin@mirantis.com
- Alex Didenko adidenko@mirantis.com

QA:
- Tatyana Leontovich tleontovich@mirantis.com
- Denis Dmitriev ddmitriev@mirantis.com
- Anastasia Palkin apalkina@mirantis.com


Work Items
----------

1. Graph based API for nailgun (config-defined tasks and roles)
2. Add hooks support for deployment stage in astute
3. Remove pre/post tasks from astute, orchestration to nailgun,
   functionality to library (reuse plugins mechanism)
4. Modularizing puppet

Dependencies
============

python networkx library [1]

Testing
=======

Every new piece of code will be covered by unit tests.
This is internal functionality, therefore it will be covered by
system tests without any modifications.
Additional tests that will verify that we dont have regression in time of
deployment.
Tests that will create new task and add it into deployment graph,
and then verify that node is in expected state.
Acceptance critirea for each task granule will be added in another spec,
eithre library modularizarion or modular tests.

Documentation Impact
====================

Requires update to developer and plugin documentation.

References
==========

1. https://networkx.github.io/ - Python utilities for working with graph's
2. http://docs.mirantis.com/
   fuel-dev/develop/nailgun/customization/settings.html#expression-syntax
