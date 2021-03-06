:orphan:

===================================================================
 ceph-disk -- Ceph disk utility for OSD
===================================================================

.. program:: ceph-disk

Synopsis
========

| **ceph-disk** **prepare** [--cluster *clustername*] [--cluster-uuid *uuid*]
	[--fs-type *xfs|ext4|btrfs*] [*data-path*] [*journal-path*]
	[--setuser *user*] [--setgroup *group*]

| **ceph-disk** **activate** [*data-path*] [--activate-key *path*]
        [--mark-init *sysvinit|upstart|systemd|auto|none*]
        [--no-start-daemon] [--reactivate]
        [--setuser *user*] [--setgroup *group*]

| **ceph-disk** **activate-all**

| **ceph-disk** **list**

| **ceph-disk** **deactivate** [--cluster *clustername*] [*device-path*]
        [--deactivate-by-id *id*] [--mark-out]

| **ceph-disk** **destroy** [--cluster *clustername*] [*device-path*]
        [--destroy-by-id *id*] [--dmcrypt-key-dir *KEYDIR*] [--zap]

Description
===========

:program:`ceph-disk` is a utility that can prepare and activate a disk, partition or
directory as a Ceph OSD. It is run directly or triggered by :program:`ceph-deploy`
or ``udev``. It can also be triggered by other deployment utilities like ``Chef``,
``Juju``, ``Puppet`` etc.

It actually automates the multiple steps involved in manual creation and start
of an OSD into two steps of preparing and activating the OSD by using the
subcommands ``prepare`` and ``activate``.

:program:`ceph-disk` also automates the multiple steps involved to manually stop
and destroy an OSD into two steps of deactivating and destroying the OSD by using
the subcommands ``deactivate`` and ``destroy``.

Subcommands
============

prepare
--------

Prepare a directory, disk for a Ceph OSD. It creates a GPT partition,
marks the partition with Ceph type ``uuid``, creates a file system, marks the
file system as ready for Ceph consumption, uses entire partition and adds a new
partition to the journal disk. It is run directly or triggered by
:program:`ceph-deploy`.

Usage::

	ceph-disk prepare --cluster [cluster-name] --cluster-uuid [uuid] --fs-type
	[ext4|xfs|btrfs] [data-path] [journal-path]

Other options like :option:`--osd-uuid`, :option:`--journal-uuid`,
:option:`--zap-disk`, :option:`--data-dir`, :option:`--data-dev`,
:option:`--journal-file`, :option:`--journal-dev`, :option:`--dmcrypt`
and :option:`--dmcrypt-key-dir` can also be used with the subcommand.

activate
--------

Activate the Ceph OSD. It mounts the volume in a temporary location, allocates
an OSD id (if needed), remounts in the correct location
``/var/lib/ceph/osd/$cluster-$id`` and starts ceph-osd. It is triggered by
``udev`` when it sees the OSD GPT partition type or on ceph service start with
``ceph disk activate-all``. It is also run directly or triggered by
:program:`ceph-deploy`.

Usage::

	ceph-disk activate [PATH]

Here, [PATH] is path to a block device or a directory.

An additional option :option:`--activate-key` has to be used with this
subcommand when a copy of ``/var/lib/ceph/bootstrap-osd/{cluster}.keyring``
isn't present in the OSD node.

Usage::

	ceph-disk activate [PATH] [--activate-key PATH]

Another option :option:`--mark-init` can also be used with this
subcommand.  ``--mark-init`` provides init system to manage the OSD
directory. It defaults to ``auto`` which detects the init system
suitable for ceph (either ``sysvinit``, ``systemd`` or
``upstart``). The argument can be used to override the init system. It
may be convenient when an operating system supports multiple init
systems, such as Debian GNU/Linux jessie with ``systemd`` and
``sysvinit``. If the argument is ``none``, the OSD is not marked with
any init system and ``ceph-disk activate`` needs to be called
explicitely after each reboot.


Usage::

	ceph-disk activate [PATH] [--mark-init *sysvinit|upstart|systemd|auto|none*]

If the option :option:`--no-start-daemon` is given, the activation
steps are performed but the OSD daemon is not started.

The latest option :option:`--reactivate` can re-activate the OSD which has been
deactivated with the ``deactivate`` subcommand.

Usage::

	ceph-disk activate [PATH] [--reactivate]

activate-journal
----------------

Activate an OSD via it's journal device. ``udev`` triggers
``ceph-disk activate-journal <dev>`` based on the partition type.

Usage::

	ceph-disk activate-journal [DEV]

Here, [DEV] is the path to a journal block device.

Others options like :option:`--activate-key` and :option:`--mark-init` can also
be used with this subcommand.

``--mark-init`` provides init system to manage the OSD directory.

Usage::

	ceph-disk activate-journal [--activate-key PATH] [--mark-init INITSYSTEM] [DEV]

activate-all
------------

Activate all tagged OSD partitions. ``activate-all`` relies on
``/dev/disk/by-parttype-uuid/$typeuuid.$uuid`` to find all partitions. Special
``udev`` rules are installed to create these links. It is triggered on ceph
service start or run directly.

Usage::

	ceph-disk activate-all

Others options like :option:`--activate-key` and :option:`--mark-init` can
also be used with this subcommand.

``--mark-init`` provides init system to manage the OSD directory.

Usage::

	ceph-disk activate-all [--activate-key PATH] [--mark-init INITSYSTEM]

list
----

List disk partitions and Ceph OSDs. It is run directly or triggered by
:program:`ceph-deploy`.

Usage::

	ceph-disk list

suppress-activate
-----------------

Suppress activate on a device (prefix). Mark devices that you don't want to
activate with a file like ``/var/lib/ceph/tmp/suppress-activate.sdb`` where the
last bit is the sanitized device name (/dev/X without the /dev/ prefix). A
function ``is_suppressed()`` checks for and  matches a prefix (/dev/). It means
suppressing sdb will stop activate on sdb1, sdb2, etc.

Usage::

	ceph-disk suppress-activate [PATH]

Here, [PATH] is path to a block device or a directory.

unsuppress-activate
-------------------

Stop suppressing activate on a device (prefix). It is used to activate a device
that was earlier kept deactivated using ``suppress-activate``.

Usage::

	ceph-disk unsuppress-activate [PATH]

Here, [PATH] is path to a block device or a directory.

deactivate
----------
Deactivate the Ceph OSD. It stops OSD daemon and optionally marks it out. The
content of the OSD is left untouched but the *ready*, *active*, *INIT-specific*
files are removed (so that it is not automatically re-activated by the ``udev``
rules) and the file deactive is created to remember the OSD is deactivated.
If the OSD is dmcrypt, remove the data dmcrypt map. When deactivate finishes,
the OSD is ``down``. A deactivated OSD can later be re-activated using the
:option:`--reactivate` option of the ``activate`` subcommand.

Usage::

	ceph-disk deactivate [PATH]

Here, [PATH] is a path to a block device or a directory.

Another option :option:`--mark-out` can also be used with this subcommand.
``--mark-out`` marks the OSD out. The objects it contains will be remapped.
If you are not sure you will destroy OSD, do not use this option.

You can also use ``osd-id`` to deactivate an OSD with the option :option:`--deactivate-by-id`.

Usage::

	ceph-disk deactivate --deactivate-by-id [OSD-ID]

destroy
-------
Destroy the Ceph OSD. It removes the OSD from the cluster, the crushmap and
deallocates OSD ID. It can only destroy an OSD which is *down*.

Usage::

	ceph-disk destroy [PATH]

Here, [PATH] is a path to a block device or a directory.

Another option :option:`--zap` can also be used with this subcommand.
``--zap`` will destroy the partition table and content of the disk.

Usage::

	ceph-disk destroy [PATH] [--zap]

You can also use the id of an OSD instead of the path with the option
:option:`--destroy-by-id`.

Usage::

	ceph-disk destroy --destroy-by-id [OSD-ID]

zap
---

Zap/erase/destroy a device's partition table and contents. It actually uses
``sgdisk`` and it's option ``--zap-all`` to destroy both GPT and MBR data
structures so that the disk becomes suitable for repartitioning. ``sgdisk``
then uses ``--mbrtogpt`` to convert the MBR or BSD disklabel disk to a GPT
disk. The ``prepare`` subcommand can now be executed which will create a new
GPT partition. It is also run directly or triggered by :program:`ceph-deploy`.

Usage::

	ceph-disk zap [DEV]

Here, [DEV] is path to a block device.

Options
=======

.. option:: --prepend-to-path PATH

   Prepend PATH to $PATH for backward compatibility (default ``/usr/bin``).

.. option:: --statedir PATH

   Directory in which ceph configuration is preserved (default ``/usr/lib/ceph``).

.. option:: --sysconfdir PATH

   Directory in which ceph configuration files are found (default ``/etc/ceph``).

.. option:: --setuser USER

   Specify a user for ceph-disk to use when dropping privileges (default ``ceph``, or ``root`` if ``ceph`` does not exist)

.. option:: --setgroup GROUP

   Specify a group for ceph-disk to use when dropping privileges (default ``ceph``, or ``root`` if ``ceph`` does not exist)

.. option:: --cluster

   Provide name of the ceph cluster in which the OSD is being prepared.

.. option:: --cluster-uuid

   Provide uuid of the ceph cluster in which the OSD is being prepared.

.. option:: --fs-type

   Provide the filesytem type for the OSD. e.g. ``xfs/ext4/btrfs``.

.. option:: --osd-uuid

	Unique OSD uuid to assign to the disk.

.. option:: --journal-uuid

	Unique uuid to assign to the journal.

.. option:: --zap-disk

	Destroy the partition table and content of a disk.

.. option:: --data-dir

	Verify that ``[data-path]`` is of a directory.

.. option:: --data-dev

	Verify that ``[data-path]`` is of a block device.

.. option:: --journal-file

	Verify that journal is a file.

.. option:: --journal-dev

	Verify that journal is a block device.

.. option:: --dmcrypt

	Encrypt ``[data-path]`` and/or journal devices with ``dm-crypt``.

.. option:: --dmcrypt-key-dir

	Directory where ``dm-crypt`` keys are stored.

.. option:: --setuser

    Define the user for ceph-disk's child processes (default ``ceph``, or ``root``)

.. option:: --setgroup

    Define the group for ceph-disk's child processes (default ``ceph``, or ``root``)

Availability
============

:program:`ceph-disk` is part of Ceph, a massively scalable, open-source, distributed storage system. Please refer to
the Ceph documentation at http://ceph.com/docs for more information.

See also
========

:doc:`ceph-osd <ceph-osd>`\(8),
:doc:`ceph-deploy <ceph-deploy>`\(8)
