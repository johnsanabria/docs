.. _cluster_guide:

==================
Managing Clusters
==================

A Cluster is a group of :ref:`Hosts <host_guide>`. Clusters can have associated :ref:`Datastores <sm>` and :ref:`Virtual Networks <vgg>`, this is how the administrator sets which Hosts have the underlying requirements for each Datastore and Virtual Network configured.

Cluster Management
==================

Clusters are managed with the :ref:`''onecluster'' command <cli>`. To create new Clusters, use ``onecluster create <name>``. Existing Clusters can be inspected with the ``onecluster list`` and ``show`` commands.

.. code::

    $ onecluster list
      ID NAME            HOSTS NETS  DATASTORES

    $ onecluster create production
    ID: 100

    $ onecluster list
      ID NAME            HOSTS NETS  DATASTORES
     100 production      0     0     0

    $ onecluster show production
    CLUSTER 100 INFORMATION
    ID             : 100
    NAME           : production

    HOSTS

    VNETS

    DATASTORES

Add Hosts to Clusters
---------------------

Hosts can be created directly in a Cluster, using the ``–cluster`` option of ``onehost create``, or be added at any moment using the command ``onecluster addhost``. Hosts can be in only one Cluster at a time.

To delete a Host from a Cluster, the command ``onecluster delhost`` must be used. When a Host is removed from a Cluster, it is seen as part of the Cluster 'none', more about this below.

In the following example, we will add Host 0 to the Cluster we created before. You will notice that the ``onecluster show`` command will list the Host ID 0 as part of the Cluster.

.. code::

    $ onehost list
      ID NAME         CLUSTER     RVM   TCPU   FCPU   ACPU   TMEM   FMEM   AMEM STAT
       0 host01       -             7    400    290    400   3.7G   2.2G   3.7G   on

    $ onecluster addhost production host01

    $ onehost list
      ID NAME         CLUSTER     RVM   TCPU   FCPU   ACPU   TMEM   FMEM   AMEM STAT
       0 host01       producti      7    400    290    400   3.7G   2.2G   3.7G   on

    $ onecluster show production
    CLUSTER 100 INFORMATION
    ID             : 100
    NAME           : production

    HOSTS
    0

    VNETS

    DATASTORES

Add Resources to Clusters
-------------------------

Datastores and Virtual Networks can be added to one Cluster. This means that any Host in that Cluster is properly configured to run VMs using Images from the Datastores, or is using leases from the Virtual Networks.

For instance, if you have several Hosts configured to use :ref:`Open vSwitch networks <openvswitch>`, you would group them in the same Cluster. The :ref:`Scheduler <schg>` will know that VMs using these resources can be deployed in any of the Hosts of the Cluster.

These operations can be done with the ``onecluster`` ``addvnet/delvnet`` and ``adddatastore/deldatastore``:

.. code::

    $ onecluster addvnet production priv-ovswitch

    $ onecluster adddatastore production iscsi

    $ onecluster list
      ID NAME            HOSTS NETS  DATASTORES
     100 production      1     1     1

    $ onecluster show 100
    CLUSTER 100 INFORMATION
    ID             : 100
    NAME           : production

    CLUSTER TEMPLATE

    HOSTS
    0

    VNETS
    1

    DATASTORES
    100

The System Datastore for a Cluster
----------------------------------

You can associate an specific System DS to a cluster to improve its performance (e.g. balance VM I/O between different servers) or to use different system DS types (e.g. shared and ssh).

To use a specific System DS with your cluster, instead of the default one, just create it (with TYPE=SYSTEM\_DS in its template), and associate it just like any other datastore (onecluster adddatastore). Check the :ref:`System DS guide for more information <system_ds>`.

Cluster Properties
------------------

Each cluster includes a generic template where cluster configuration properties or attributes can be defined. The following list of attributes are recognized by OpenNebula:

+--------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Attribute                | Description                                                                                                                                                                                                                                |
+==========================+============================================================================================================================================================================================================================================+
| ``DATASTORE_LOCATION``   | \*Default\* path for datastores in the cluster hosts. It **is** the same for all the hosts in the cluster. Note that DATASTORE\_LOCATION is only for the cluster hosts and not for the front-end. It defaults to /var/lib/one/datastores   |
+--------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

You can easily update this values with the ``onecluster`` command:

.. code::

    $ onecluster update production

    -----8<----- editor session ------8<------

    DATASTORE_LOCATION="/mnt/nas/datastores"
    ~
    ~
    ~
    ----->8----- editor session ------>8------

    $oneluster show production
    CLUSTER 100 INFORMATION
    ID               : 100
    NAME             : production
    SYSTEM DATASTORE : 100

    CLUSTER TEMPLATE
    DATASTORE_LOCATION="/mnt/nas/datastores"

    HOSTS
    0

    VNETS
    1

    DATASTORES
    100

You can add as many variables as you want, following the standard template syntax. These variables will be used for now only for informational purposes.

The Default Cluster 'None'
==========================

Hosts, Datastores and Virtual Networks can be grouped into clusters, but this is optional. By default, these resources are created outside of any Cluster, what can be seen as a special Cluster named 'none' in Sunstone. In the CLI, this Cluster name is shown as '-'.

Virtual Machines using resources from Datastores or Virtual Networks in the Cluster 'none' can be deployed in any Host, which must be properly configured.

Hosts in the Cluster 'none' will only run VMs using resources without a Cluster.

Scheduling and Clusters
=======================

Automatic Requirements
----------------------

When a Virtual Machine uses resources (Images or Virtual Networks) from a Cluster, OpenNebula adds the following :ref:`requirement <template_placement_section>` to the template:

.. code::

    $ onevm show 0
    [...]
    AUTOMATIC_REQUIREMENTS="CLUSTER_ID = 100"

Because of this, if you try to use resources from more than one Cluster, the Virtual Machine creation will fail with a message similar to this one:

.. code::

    $ onetemplate instantiate 0
    [TemplateInstantiate] Error allocating a new virtual machine. Incompatible cluster IDs.
    DISK [0]: IMAGE [0] from DATASTORE [1] requires CLUSTER [101]
    NIC [0]: NETWORK [1] requires CLUSTER [100]

Manual Requirements and Rank
----------------------------

The placement attributes :ref:`SCHED\_REQUIREMENTS and SCHED\_RANK <template_placement_section>` can use attributes from the Cluster template. Let’s say you have the following scenario:

.. code::

    $ onehost list
      ID NAME            CLUSTER   RVM      ALLOCATED_CPU      ALLOCATED_MEM STAT
       1 host01          cluster_a   0       0 / 200 (0%)     0K / 3.6G (0%) on
       2 host02          cluster_a   0       0 / 200 (0%)     0K / 3.6G (0%) on
       3 host03          cluster_b   0       0 / 200 (0%)     0K / 3.6G (0%) on

    $ onecluster show cluster_a
    CLUSTER TEMPLATE
    QOS="GOLD"

    $ onecluster show cluster_b
    CLUSTER TEMPLATE
    QOS="SILVER"

You can use these expressions:

.. code::

    SCHED_REQUIREMENTS = "QOS = GOLD"
     
    SCHED_REQUIREMENTS = "QOS != GOLD & HYPERVISOR = kvm"

System Storage
==============

The system datastore holds files for running VMs. Each cluster can use a different system datastore, read more in :ref:`the system datastore guide <system_ds>`.

Managing Clusters in Sunstone
=============================

The :ref:`Sunstone UI interface <sunstone>` offers an easy way to manage clusters and the resources whithin them. You will find the cluster submenu under the infraestructure menu. From there, you will be able to:

-  Create new clusters selecting the resources you want to include in this cluster:

|image0|

-  See the list of current clusters, from which you can update the template of existing ones, or delete them.

|image1|

.. |image0| image:: /images/sunstone_cluster_create.png
.. |image1| image:: /images/sunstone_cluster_list2.png
