.. _vm_guide_2:

==========================
Managing Virtual Machines
==========================

This guide follows the :ref:`Creating Virtual Machines guide <vm_guide>`. Once a Template is instantiated to a Virtual Machine, there are a number of operations that can be performed using the ``onevm`` command.

.. _vm_life_cycle_and_states:

Virtual Machine Life-cycle
==========================

The life-cycle of a Virtual Machine within OpenNebula includes the following stages:

.. warning:: Note that this is a simplified version. If you are a developer you may want to take a look at the complete diagram referenced in the :ref:`xml-rpc api page <api>`):

+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Short state |     State      |                                                                                                                                                 Meaning                                                                                                                                                  |
+=============+================+==========================================================================================================================================================================================================================================================================================================+
| ``pend``    | ``Pending``    | By default a VM starts in the pending state, waiting for a resource to run on. It will stay in this state until the scheduler decides to deploy it, or the user deploys it using the ``onevm deploy`` command.                                                                                           |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``hold``    | ``Hold``       | The owner has held the VM and it will not be scheduled until it is released. It can be, however, deployed manually.                                                                                                                                                                                      |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``prol``    | ``Prolog``     | The system is transferring the VM files (disk images and the recovery file) to the host in which the virtual machine will be running.                                                                                                                                                                    |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``boot``    | ``Boot``       | OpenNebula is waiting for the hypervisor to create the VM.                                                                                                                                                                                                                                               |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``runn``    | ``Running``    | The VM is running (note that this stage includes the internal virtualized machine booting and shutting down phases). In this state, the virtualization driver will periodically monitor it.                                                                                                              |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``migr``    | ``Migrate``    | The VM is migrating from one resource to another. This can be a life migration or cold migration (the VM is saved and VM files are transferred to the new resource).                                                                                                                                     |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``hotp``    | ``Hotplug``    | A disk attach/detach, nic attach/detach operation is in process.                                                                                                                                                                                                                                         |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``snap``    | ``Snapshot``   | A system snapshot is being taken.                                                                                                                                                                                                                                                                        |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``save``    | ``Save``       | The system is saving the VM files after a migration, stop or suspend operation.                                                                                                                                                                                                                          |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``epil``    | ``Epilog``     | In this phase the system cleans up the Host used to virtualize the VM, and additionally disk images to be saved are copied back to the system datastore.                                                                                                                                                 |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``shut``    | ``Shutdown``   | OpenNebula has sent the VM the shutdown ACPI signal, and is waiting for it to complete the shutdown process. If after a timeout period the VM does not disappear, OpenNebula will assume that the guest OS ignored the ACPI signal and the VM state will be changed to **running**, instead of **done**. |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``stop``    | ``Stopped``    | The VM is stopped. VM state has been saved and it has been transferred back along with the disk images to the system datastore.                                                                                                                                                                          |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``susp``    | ``Suspended``  | Same as stopped, but the files are left in the host to later resume the VM there (i.e. there is no need to re-schedule the VM).                                                                                                                                                                          |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``poff``    | ``PowerOff``   | Same as suspended, but no checkpoint file is generated. Note that the files are left in the host to later boot the VM there.                                                                                                                                                                             |
|             |                |                                                                                                                                                                                                                                                                                                          |
|             |                | When the VM guest is shutdown, OpenNebula will put the VM in this state.                                                                                                                                                                                                                                 |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``unde``    | ``Undeployed`` | The VM is shut down. The VM disks are transfered to the system datastore. The VM can be resumed later.                                                                                                                                                                                                   |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``fail``    | ``Failed``     | The VM failed.                                                                                                                                                                                                                                                                                           |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``unkn``    | ``Unknown``    | The VM couldn't be reached, it is in an unknown state.                                                                                                                                                                                                                                                   |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``done``    | ``Done``       | The VM is done. VMs in this state won't be shown with ``onevm list`` but are kept in the database for accounting purposes. You can still get their information with the ``onevm show`` command.                                                                                                          |
+-------------+----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

|Virtual Machine States|

Managing Virtual Machines
=========================

The following sections show the basics of the ``onevm`` command with simple usage examples. A complete reference for these commands can be found :ref:`here <cli>`.

Create and List Existing VMs
----------------------------

.. warning:: Read the :ref:`Creating Virtual Machines guide <vm_guide>` for more information on how to manage and instantiate VM Templates.

.. warning:: Read the complete reference for :ref:`Virtual Machine templates <template>`.

Assuming we have a VM Template registered called *vm-example* with ID 6, then we can instantiate the VM issuing a:

.. code::

    $ onetemplate list
      ID USER     GROUP    NAME                         REGTIME
       6 oneadmin oneadmin vm_example            09/28 06:44:07

    $ onetemplate instantiate vm-example --name my_vm
    VM ID: 0


If the template has :ref:`USER INPUTS <vm_guide_user_inputs>` defined the CLI will prompt the user for these values:

.. code::

    $ onetemplate instantiate vm-example --name my_vm
    There are some parameters that require user input.
      * (BLOG_TITLE) Blog Title: <my_title>
      * (DB_PASSWORD) Database Password:
    VM ID: 0

Afterwards, the VM can be listed with the ``onevm list`` command. You can also use the ``onevm top`` command to list VMs continuously.

.. code::

    $ onevm list
        ID USER     GROUP    NAME         STAT CPU     MEM        HOSTNAME        TIME
         0 oneadmin oneadmin my_vm        pend   0      0K                 00 00:00:03

After a Scheduling cycle, the VM will be automatically deployed. But the deployment can also be forced by oneadmin using ``onevm deploy``:

.. code::

    $ onehost list
      ID NAME               RVM   TCPU   FCPU   ACPU   TMEM   FMEM   AMEM   STAT
       2 testbed              0    800    800    800    16G    16G    16G     on

    $ onevm deploy 0 2

    $ onevm list
        ID USER     GROUP    NAME         STAT CPU     MEM        HOSTNAME        TIME
         0 oneadmin oneadmin my_vm        runn   0      0K         testbed 00 00:02:40

and details about it can be obtained with ``show``:

.. code::

    $ onevm show 0
    VIRTUAL MACHINE 0 INFORMATION
    ID                  : 0
    NAME                : my_vm
    USER                : oneadmin
    GROUP               : oneadmin
    STATE               : ACTIVE
    LCM_STATE           : RUNNING
    START TIME          : 04/14 09:00:24
    END TIME            : -
    DEPLOY ID:          : one-0

    PERMISSIONS
    OWNER          : um-
    GROUP          : ---
    OTHER          : ---

    VIRTUAL MACHINE MONITORING
    NET_TX              : 13.05
    NET_RX              : 0
    USED MEMORY         : 512
    USED CPU            : 0

    VIRTUAL MACHINE TEMPLATE
    ...

    VIRTUAL MACHINE HISTORY
     SEQ        HOSTNAME REASON           START        TIME       PTIME
       0         testbed   none  09/28 06:48:18 00 00:07:23 00 00:00:00

Terminating VM Instances...
---------------------------

You can terminate a running instance with the following operations (either as ``onevm`` commands or through Sunstone):

-  ``shutdown``: Gracefully shuts down a running VM, sending the ACPI signal. Once the VM is shutdown the host is cleaned, and persistent and deferred-snapshot disk will be moved to the associated datastore. If after a given time the VM is still running (e.g. guest ignoring ACPI signals), OpenNebula will returned the VM to the ``RUNNING`` state.

-  ``shutdown --hard``: Same as above but the VM is immediately destroyed. Use this action instead of ``shutdown`` when the VM doesn't have ACPI support.

If you need to terminate an instance in any state use:

-  ``delete``: The VM is immediately destroyed no matter its state. Hosts are cleaned as needed but no images are moved to the repository, leaving then in error. Think of delete as kill -9 for a process, an so it should be only used when the VM is not responding to other actions.

All the above operations free the resources used by the VM

Pausing VM Instances...
-----------------------

There are two different ways to temporarily stop the execution of a VM: short and long term pauses. A **short term** pause keeps all the VM resources allocated to the hosts so its resume its operation in the same hosts quickly. Use the following ``onevm`` commands or Sunstone actions:

-  ``suspend``: the VM state is saved in the running Host. When a suspended VM is resumed, it is immediately deployed in the same Host by restoring its saved state.

-  ``poweroff``: Gracefully powers off a running VM by sending the ACPI signal. It is similar to suspend but without saving the VM state. When the VM is resumed it will boot immediately in the same Host.

-  ``poweroff --hard``: Same as above but the VM is immediately powered off. Use this action when the VM doesn't have ACPI support.

.. note:: When the guest is shutdown from within the VM, OpenNebula will put the VM in the ``poweroff`` state.

You can also plan a **long term pause**. The Host resources used by the VM are freed and the Host is cleaned. Any needed disk is saved in the system datastore. The following actions are useful if you want to preserve network and storage allocations (e.g. IPs, persistent disk images):

-  ``undeploy``: Gracefully shuts down a running VM, sending the ACPI signal. The Virtual Machine disks are transferred back to the system datastore. When an undeployed VM is resumed, it is be moved to the pending state, and the scheduler will choose where to re-deploy it.

-  ``undeploy --hard``: Same as above but the running VM is immediately destroyed.

-  ``stop``: Same as ``undeploy`` but also the VM state is saved to later resume it.

When the VM is successfully paused you can resume its execution with:

-  ``resume``: Resumes the execution of VMs in the stopped, suspended, undeployed and poweroff states.

Resetting VM Instances...
-------------------------

There are two ways of resetting a VM: in-host and full reset. The first one does not frees any resources and reset a RUNNING VM instance at the hypervisor level:

-  ``reboot``: Gracefully reboots a running VM, sending the ACPI signal.

-  ``reboot --hard``: Performs a 'hard' reboot.

A VM instance can be reset in any state with:

-  ``delete --recreate``: Deletes the VM as described above, but instead of disposing it the VM is moving again to PENDING state. As the delete operation this action should be used when the VM is not responding to other actions. Try undeploy or undeploy --hard first.

Delaying VM Instances...
------------------------

The deployment of a PENDING VM (e.g. after creating or resuming it) can be delayed with:

-  ``hold``: Sets the VM to hold state. The scheduler will not deploy VMs in the ``hold`` state. Please note that VMs can be created directly on hold, using 'onetemplate instantiate --hold' or 'onevm create --hold'.

Then you can resume it with:

-  ``release``: Releases a VM from hold state, setting it to pending. Note that you can automatically release a VM by scheduling the operation as explained below

.. _vm_guide_2_disk_snapshots:

Disk Snapshots
--------------

There are two kinds of operations related to disk snapshots:

- ``disk-snapshot-create``, ``disk-snapshot-revert``, ``disk-snapshot-delete``: Allows the user to take snapshots of the disk states and return to them during the VM life-cycle. It is also possible to delete snapshots.
- ``disk-saveas``: Exports VM disk (or a previusly created snapshot) to an image. This is a live action.

.. _vm_guide_2_disk_snapshots_managing:

Managing disk snapshots
^^^^^^^^^^^^^^^^^^^^^^^

A user can take snapshots of the disk states at any moment in time (if the VM is in ``RUNNING``, ``POWEROFF`` or ``SUSPENDED`` states). These snapshots are organized in a tree-like structure, meaning that every snapshot has a parent, except for the first snapshot whose parent is ``-1``. At any given time a user can revert the disk state to a previously taken snapshot. The active snapshot, the one the user has last reverted to, or taken, will act as the parent of the next snapshot. In addition, it's possible to delete snapshots that are not active and that have no children.

- ``disk-snapshot-create <vmid> <diskid> <name>``: Creates a new snapshot of the specified disk.
- ``disk-snapshot-revert <vmid> <diskid> <snapshot_id>``: Reverts to the specified snapshot. The snapshots are immutable, therefore the user can revert to the same snapshot as many times as he wants, the disk will return always to the state of the snapshot at the time it was taken.
- ``disk-snapshot-delete <vmid> <diskid> <snapshot_id>``: Deletes a snapshot if it has no children and is not active.

.. warning::

  ``disk-snapshot-create`` and ``disk-snapshot-revert`` actions are not in sync with the hypervisor. If the VM is in ``RUNNING`` state make sure the disk is unmounted (preferred), synced or quiesced in some way before taking the snapshot.

``disk-snapshot-create`` and ``disk-snapshot-revert`` can take place when the VM is in ``RUNNING`` state. The way OpenNebula handles this operation varies depending on the configuration and on the backend used. When configuring the ``VM_MAD`` in ``/etc/one/oned.conf``, depending on the arguments passed, the administrator can decide what strategy to use when creating and reverting snapshots in ``RUNNING`` state:

- ``-d suspend`` (default): The VM is suspended (the memory state is written to the system datastore), the snapshot operation takes place (create or revert). This is the safest strategy but implies some downtime (the time it takes for the memory state to be written and to be re-read again).
- ``-d detach``: the disk is detached while the VM is kept active and running. The snapshot operation takes place, and the disk is re-attached. This is a dangerous operation as if the OS has active file descriptors opening the disk, the OS will not be able to release the target (e.g. ``sbd``) and when it is re-attached the OS will place it in a new target instead (e.g. ``sdc``). This is problematic as there will be a discrepancy between the target defined by OpenNebula and the real target inside the guest VM, which could make future disk-attach operations fail. In order to avoid this, the disk must be fully unmounted with no active file descriptors in use. On the other hand, this technique is the fastest as it requires no down-time.

Additionally, one can activate the live snapshots option (``-i``), which is only supported for some drivers. If this option is enabled **and** if the driver that will create the snapshot supports it, it will use hypervisor operations to create the snapshot while running. This strategy is as robust as ``suspend`` but has the benefit of not implying any downtime. However it is only supported for:

- Hypervisor ``VM_MAD=kvm``, System Datastore ``TM_MAD=shared``, Image datastore ``DS_MAD=fs`` and ``TM_MAD=qcow2``. In this case OpenNebula will request that the hypervisor executes ``virsh snapshot-create``.

Note that the live disk snapshot calls a diferent TM action than the regular one, as documented by the :ref:`Storage Driver <sd_tm>` guide.

Persistent image snapshots
^^^^^^^^^^^^^^^^^^^^^^^^^^

These actions are available for both persistent and non-persistent images. In the case of persistent images the snapshots **will** be preserved upon VM termination and will be able to be used by other VMs using that image. See the :ref:`snapshots <img_guide_snapshots>` section in the Images guide for more information.

Backend implementations
^^^^^^^^^^^^^^^^^^^^^^^

The snapshot operations are implemented differently depending on the storage backend:

+----------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------+---------------------------------------------------------------------------+------------------------------+
| **Operation/TM_MAD** |                                           Ceph                                          |                  Shared  and SSH                  |                                   Qcow2                                   | Dev,  FS_LVM,  LVM and  vmfs |
+======================+=========================================================================================+===================================================+===========================================================================+==============================+
| Snap Create          | Creates a protected snapshot                                                            | Copies the file.                                  | Creates a new qcow2 image with the previous disk as the backing file.     | *Not Supported*              |
+----------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------+---------------------------------------------------------------------------+------------------------------+
| Snap Create (live)   | *Not Supported*                                                                         | *Not Supported*                                   | (For KVM only) Launches ``virsh snapshot-create``.                        | *Not Supported*              |
+----------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------+---------------------------------------------------------------------------+------------------------------+
| Snap Revert          | Overwrites the active disk by creating a new snapshot of an existing protected snapshot | Overwrites the file with a previously copied one. | Creates a new qcow2 image with the selected snapshot as the backing file. | *Not Supported*              |
+----------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------+---------------------------------------------------------------------------+------------------------------+
| Snap Delete          | Deletes a protected snapshot                                                            | Deletes the file.                                 | Delestes the selected qcow2 snapshot.                                     | *Not Supported*              |
+----------------------+-----------------------------------------------------------------------------------------+---------------------------------------------------+---------------------------------------------------------------------------+------------------------------+

.. warning::

  Depending on the ``CACHE`` the live snapshot may or may not work correctly. For more security use ``CACHE=writethrough`` although this delivers the slowest performance.

Exporting disk images with ``disk-saveas``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Any VM disk can be exported to a new image (if the VM is in ``RUNNING``, ``POWEROFF`` or ``SUSPENDED`` states). This is a live operation that happens immediately. This operation accepts ``--snapshot <snapshot_id>`` as an optional argument, which specifies a disk snapshot to use as the source of the clone, instead of the current disk state (value by default).

.. note::

  This action is called ``onevm disk-snapshot --live`` in OpenNebula <= 4.14 but has been renamed to ``onevm disk-saveas``

.. warning::

  This action is not in sync with the hypervisor. If the VM is in ``RUNNING`` state make sure the disk is unmounted (preferred), synced or quiesced in some way or another before taking the snapshot.

Disk Hotpluging
---------------

New disks can be hot-plugged to running VMs with the ``onevm`` ``disk-attach`` and ``disk-detach`` commands. For example, to attach to a running VM the Image named **storage**:

.. code::

    $ onevm disk-attach one-5 --image storage

To detach a disk from a running VM, find the disk ID of the Image you want to detach using the ``onevm show`` command, and then simply execute ``onevm detach vm_id disk_id``:

.. code::

    $ onevm show one-5
    ...
    DISK=[
      DISK_ID="1",
    ...
      ]
    ...

    $ onevm disk-detach one-5 1

|image2|

.. _vm_guide2_nic_hotplugging:

NIC Hotpluging
--------------

You can hotplug network interfaces to VMs in the ``RUNNING``, ``POWEROFF`` or ``SUSPENDED`` states. Simply specify the network where the new interface should be attach to, for example:

.. code::

    $ onevm show 2

    VIRTUAL MACHINE 2 INFORMATION
    ID                  : 2
    NAME                : centos-server
    USER                : ruben
    GROUP               : oneadmin
    STATE               : ACTIVE
    LCM_STATE           : RUNNING
    RESCHED             : No
    HOST                : cloud01

    ...

    VM NICS
    ID NETWORK      VLAN BRIDGE   IP              MAC
     0 net_172        no vbr0     172.16.0.201    02:00:ac:10:0

    VIRTUAL MACHINE HISTORY
     SEQ HOST            REASON           START            TIME     PROLOG_TIME
       0 cloud01         none    03/07 11:37:40    0d 00h02m14s    0d 00h00m00s
    ...

    $ onevm nic-attach 2 --network net_172

After the operation you should see two NICs, 0 and 1:

.. code::

    $ onevm show 2
    VIRTUAL MACHINE 2 INFORMATION
    ID                  : 2
    NAME                : centos-server
    USER                : ruben
    GROUP               : oneadmin

    ...


    VM NICS
    ID NETWORK      VLAN BRIDGE   IP              MAC
     0 net_172        no vbr0     172.16.0.201    02:00:ac:10:00:c9
                                  fe80::400:acff:fe10:c9
     1 net_172        no vbr0     172.16.0.202    02:00:ac:10:00:ca
                                  fe80::400:acff:fe10:ca
    ...

Also, you can detach a NIC by its ID. If you want to detach interface 1 (MAC=02:00:ac:10:00:ca), just execute:

.. code::

    $ onevm nic-detach 2 1

|image3|

.. _vm_guide2_snapshotting:

Snapshotting
------------

You can create, delete and restore snapshots for running VMs. A snapshot will contain the current disks and memory state.

.. warning:: The snapshots will only be available during the ``RUNNING`` state. If the state changes (stop, migrate, etc...) the snapshots **will** be lost.

.. code::

    $ onevm snapshot-create 4 "just in case"

    $ onevm show 4
    ...
    SNAPSHOTS
      ID         TIME NAME                                           HYPERVISOR_ID
       0  02/21 16:05 just in case                                   onesnap-0

    $ onevm snapshot-revert 4 0 --verbose
    VM 4: snapshot reverted

Please take into consideration the following limitations:

-  **The snapshots are lost if any life-cycle operation is performed, e.g. a suspend, migrate, delete request.**
-  KVM: Snapshots are only available if all the VM disks use the :ref:`qcow2 driver <img_template>`.
-  VMware: the snapshots will persist in the hypervisor after any life-cycle operation is performed, but they will not be available to be used with OpenNebula.
-  Xen: does not support snapshotting

|image4|

.. _vm_guide2_resizing_a_vm:

Resizing a VM Capacity
----------------------

You may re-size the capacity assigned to a Virtual Machine in terms of the virtual CPUs, memory and CPU allocated. VM re-sizing can be done when the VM is not ACTIVE, an so in any of the following states: PENDING, HOLD, FAILED and specially in POWEROFF.

If you have created a Virtual Machine and you need more resources, the following procedure is recommended:

-  Perform any operation needed to prepare your Virtual Machine for shutting down, e.g. you may want to manually stop some services...
-  Poweroff the Virtual Machine
-  Re-size the VM
-  Resume the Virtual Machine using the new capacity

Note that using this procedure the VM will preserve any resource assigned by OpenNebula (e.g. IP leases)

The following is an example of the previous procedure from the command line (the Sunstone equivalent is straight forward):

.. code::

    > onevm poweroff web_vm
    > onevm resize web_vm --memory 2G --vcpu 2
    > onevm resume web_vm

From Sunstone:

|image5|

.. _vm_guide2_resize_disk:

Resizing a VM Disks
-------------------

If the disks assigned to a Virtual Machine need more size, this can achieved at instantiation time of the VM. The SIZE parameter of the disk can be adjusted and, if it is bigger than the original size of the image, OpenNebula will:

- Increase the size of the disk container prior to launching the VM
- Using the :ref:`contextualization packages <bcont>`, at boot time the VM will grow the filesystem to adjust to the new size.

This can be

.. code::

   DISK=[IMAGE_ID=4,
         SIZE=2000]   # If Image 4 is 1 GB, OpenNebula will resize it to 2 GB

Alternatively, the resize can be created directly using the CLI as follows:

.. code::

  onetemplate instantiate <template> --disk image0:size=20000

This can also be achieved from Sunstone, both in Cloud and Admin View, at the time of instantiating a VM Template:

|image9|


.. _vm_guide2_clone_vm:

Cloning a VM
--------------------------------------------------------------------------------

A VM instance can be saved back to a new VM Template. To do that, ``poweroff`` the VM and then use the ``onevm save`` command:

.. code::

    $ onevm save web_vm copy_of_web_vm
    Template ID: 26

The clone takes into account the customization available to end users through Sunstone. This action clones the VM source Template, replacing the disks with snapshots of the current disks (see the disk-snapshot action). If the VM instance was resized, the current capacity is also used. NIC interfaces are also overwritten with the ones from the VM instance, to preserve any attach/detach action.

Please bear in mind the following limitations:

- The VM's source Template will be used. If this Template was updated since the VM was instantiated, the new contents will be used.
- Volatile disks cannot be saved, and the current contents will be lost. The cloned VM Template will contain the definition for an empty volatile disk.
- Disks and NICs will only contain the target Image/Network ID. If your Template requires extra configuration (such as DISK/DEV_PREFIX), you will need to update the new Template.

.. todo:: command in Sunstone

.. _vm_guide2_scheduling_actions:

Scheduling Actions
------------------

Most of the onevm commands accept the '--schedule' option, allowing users to delay the actions until the given date and time.

Here is an usage example:

.. code::

    $ onevm suspend 0 --schedule "09/20"
    VM 0: suspend scheduled at 2013-09-20 00:00:00 +0200

    $ onevm resume 0 --schedule "09/23 14:15"
    VM 0: resume scheduled at 2013-09-23 14:15:00 +0200

    $ onevm show 0
    VIRTUAL MACHINE 0 INFORMATION
    ID                  : 0
    NAME                : one-0

    [...]

    SCHEDULED ACTIONS
    ID ACTION        SCHEDULED         DONE MESSAGE
     0 suspend     09/20 00:00            -
     1 resume      09/23 14:15            -

These actions can be deleted or edited using the 'onevm update' command. The time attributes use Unix time internally.

.. code::

    $ onevm update 0

    SCHED_ACTION=[
      ACTION="suspend",
      ID="0",
      TIME="1379628000" ]
    SCHED_ACTION=[
      ACTION="resume",
      ID="1",
      TIME="1379938500" ]

These are the commands that can be scheduled:

-  ``shutdown``
-  ``shutdown --hard``
-  ``undeploy``
-  ``undeploy --hard``
-  ``hold``
-  ``release``
-  ``stop``
-  ``suspend``
-  ``resume``
-  ``delete``
-  ``delete-recreate``
-  ``reboot``
-  ``reboot --hard``
-  ``poweroff``
-  ``poweroff --hard``
-  ``snapshot-create``

.. _vm_guide2_user_defined_data:

User Defined Data
-----------------

Custom tags can be associated to a VM to store metadata related to this specific VM instance. To add custom attributes simply use the ``onevm update`` command.

.. code::

    $ onevm show 0
    ...

    VIRTUAL MACHINE TEMPLATE
    ...
    VMID="0"

    $ onevm update 0
    ROOT_GENERATED_PASSWORD="1234"
    ~
    ~

    $onevm show 0
    ...

    VIRTUAL MACHINE TEMPLATE
    ...
    VMID="0"

    USER TEMPLATE
    ROOT_GENERATED_PASSWORD="1234"

Manage VM Permissions
---------------------

OpenNebula comes with an advanced :ref:`ACL rules permission mechanism <manage_acl>` intended for administrators, but each VM object has also :ref:`implicit permissions <chmod>` that can be managed by the VM owner. To share a VM instance with other users, to allow them to list and show its information, use the ``onevm chmod`` command:

.. code::

    $ onevm show 0
    ...
    PERMISSIONS
    OWNER          : um-
    GROUP          : ---
    OTHER          : ---

    $ onevm chmod 0 640

    $ onevm show 0
    ...
    PERMISSIONS
    OWNER          : um-
    GROUP          : u--
    OTHER          : ---

Administrators can also change the VM's group and owner with the ``chgrp`` and ``chown`` commands.

.. _life_cycle_ops_for_admins:

Life-Cycle Operations for Administrators
----------------------------------------

There are some ``onevm`` commands operations meant for the cloud administrators:

**Scheduling:**

-  ``resched``: Sets the reschedule flag for the VM. The Scheduler will migrate (or migrate --live, depending on the :ref:`Scheduler configuration <schg_configuration>`) the VM in the next monitorization cycle to a Host that better matches the requirements and rank restrictions. Read more in the :ref:`Scheduler documentation <schg_re-scheduling_virtual_machines>`.
-  ``unresched``: Clears the reschedule flag for the VM, canceling the rescheduling operation.

**Deployment:**

-  ``deploy``: Starts an existing VM in a specific Host.
-  ``migrate --live``: The Virtual Machine is transferred between Hosts with no noticeable downtime. This action requires a :ref:`shared file system storage <sm>`.
-  ``migrate``: The VM gets stopped and resumed in the target host. In an infrastructure with :ref:`multiple system datastores <system_ds_multiple_system_datastore_setups>`, the VM storage can be also migrated (the datastore id can be specified).

Note: By default, the above operations do not check the target host capacity. You can use the -e (-enforce) option to be sure that the host capacity is not overcommitted.

**Troubleshooting:**

-  ``recover``: If the VM is stuck in any other state (or the boot operation does not work), you can recover the VM by simulating the failure or success of the missing action, or you can launch it with the ``--retry`` flag (and optionally the ``--interactive`` if its a Transfer Manager problem) to replay the driver actions. Read the :ref:`Virtual Machine Failures guide <ftguide_virtual_machine_failures>` for more information.
-  ``migrate`` or ``resched``: A VM in the UNKNOWN state can be booted in a different host manually (``migrate``) or automatically by the scheduler (``resched``). This action must be performed only if the storage is shared, or manually transfered by the administrator. OpenNebula will not perform any action on the storage for this migration.

Sunstone
========

You can manage your virtual machines using the :ref:`onevm command <cli>` or :ref:`Sunstone <sunstone>`.

In Sunstone, you can easily instantiate currently defined :ref:`templates <vm_guide>` by clicking ``New`` on the Virtual Machines tab and manage the life cycle of the new instances

|image6|

Using the noVNC Console
-----------------------

In order to use this feature, make sure that:

-  The VM template has a ``GRAPHICS`` section defined, that the ``TYPE`` attribute in it is set to ``VNC``.

-  The specified VNC port on the host on which the VM is deployed is accessible from the Sunstone server host.

-  The VM is in ``running`` state.

If the VM supports VNC and is ``running``, then the VNC icon on the Virtual Machines view should be visible and clickable:

|image7|

When clicking the VNC icon, the process of starting a session begins:

-  A request is made and if a VNC session is possible, Sunstone server will add the VM Host to the list of allowed vnc session targets and create a random token associated to it.

-  The server responds with the session token, then a ``noVNC`` dialog pops up.

-  The VNC console embedded in this dialog will try to connect to the proxy either using websockets (default) or emulating them using ``Flash``. Only connections providing the right token will be successful. Websockets are supported from Firefox 4.0 (manual activation required in this version) and Chrome. The token expires and cannot be reused.

|image8|

In order to close the VNC session just close the console dialog.

.. note:: From Sunstone 3.8, a single instance of the VNC proxy is launched when Sunstone server starts. This instance will listen on a single port and proxy all connections from there.

Information for Developers and Integrators
==========================================

-  Although the default way to create a VM instance is to register a Template and then instantiate it, VMs can be created directly from a template file using the ``onevm create`` command.
-  When a VM reaches the ``done`` state, it disappears from the ``onevm list`` output, but the VM is still in the database and can be retrieved with the ``onevm show`` command.
-  OpenNebula comes with an :ref:`accounting tool <accounting>` that reports resource usage data.
-  The monitoring information, shown with nice graphs in :ref:`Sunstone <sunstone>`, can be retrieved using the XML-RPC methods :ref:`one.vm.monitoring and one.vmpool.monitoring <api>`.

.. |Virtual Machine States| image:: /images/states-simple.png
    :width: 100 %
.. |image2| image:: /images/sunstone_vm_attach.png
.. |image3| image:: /images/sunstone_vm_attachnic.png
.. |image4| image:: /images/sunstone_vm_snapshot.png
.. |image5| image:: /images/sunstone_vm_resize.png
.. |image6| image:: /images/sunstone_vm_list.png
.. |image7| image:: /images/sunstone_vnc.png
.. |image8| image:: /images/sunstonevnc4.png
.. |image9| image:: /images/sunstone_vm_resize.png
