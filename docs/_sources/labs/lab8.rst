===========================
Lab 8: Non-volatile Storage
===========================

############
Introduction
############

In previous lab, files were stored in memory, which is volatile. 
In this lab, you'll implement FAT32 that stores data in SD card.

#################
Goals of this lab
#################

* Understand how to interact with SD card. 
* Implement FAT32 file system.
* Understand memory cache for slow storage mediums.

.. warning::

  It's better to do this lab on qemu first, and backup your SD card as disk image when you run on real SD card, 
  so you can ``dd`` to restore if you broke your FAT32 partition

##########
Background
##########

FAT File System
===============

FAT is a simple and widely used file system.
There is at least one file allocation table, which stores the allocation status.
The entry size can be either 12 bit, 16 bit, or 32 bit.

.. note::

  FAT is a portable file system, hence, your FAT32 driver should be compatible with others. 

Short File Names (SFN) and Long File Names (LFN)
================================================

Originally, FAT uses 8 bytes to store file name and 3 bytes to store extension name, 
for example: ``"a.out"``, ``"a"`` stored in the file name, ``"out"`` stored in extension name and ``"."`` is ignored.

However, it limits the file name's length and encoding, hence, LFN was invented to fix this problem.

The ``nctuos.img`` we provide in lab0 is using LFN, 
so we provide a new `sfn_nctuos.img <https://github.com/GrassLab/osdi/raw/master/supplement/sfn_nctuos.img>`_ ,
which is using SFN, and the ``kernel8.img`` inside prints out the first block 
and the first partition block of the SD card.

.. note::
  SFN is easier to implement than LFN, you can choose either one to implement.   
  We won't test against LFN, but you might need to change your configuration to use SFN.   
  Ex. initramfs.cpio needs to be renamed, and modify config.txt so bootloader can load it.   

SD Card
=======

SD Card Driver
--------------

We provide an `SD card driver <https://github.com/GrassLab/osdi/raw/master/supplement/sdhost.c>`_ for you.

You should first call the ``sd_init`` to initialize the SD card.
Then you can use ``readblock`` / ``writeblock`` to R/W 512 bytes from/to SD card.

.. note::
  You should modify the code to meet your requirements.

SD Card on QEMU
---------------

You can add the argument ``-drive if=sd,file=<img>,format=raw`` to attach an disk image to QEMU.

SD Card on Linux
----------------

In Linux, you can set up a loop device for a disk image.

``losetup -fP <img>``: set up a loop device for disk image, 
you can use ``lsblk`` to lookup which loop device it's attached to

``losetup -d <device>``: detach the loop device from the disk image.

Then you can mount loop device

``mount -t msdos <device> <dir>``: SFN.

``mount -t vfat <device> <dir>``: LFN.

Sector
-------

A sector is the smallest unit for a hard drive to manage its data.

Block
------

A block is the smallest unit for a block device driver to read/write a device.

Cluster
--------

A cluster is composed of contiguous blocks.
A file system uses it as a basic unit to store a regular file or a directory.

.. note::
  
  If you use the provided image and driver, sector, block, and cluster are all 512 bytes.

###############
Basic Exercises
###############

In this part, you should mount SD Card's FAT32 partition on ``/boot``.

Get Partition
=============

You should locate the partition before mount.
SD card was formatted as MBR (normally).
You can parse it to get each partition's type, size, and block index.

.. note::

  If you use the provided image, the FAT32 partition's block index is 2048.

Mount FAT32
===========

FAT32 stores its metadata in the first sector of the partition.
You need to do the following things during mounting.

1. Parse the metadata on the SD card.
2. Create a kernel object to store the metadata in memory.
3. Get the root directory cluster number and create it's root directory object.

Basic Exercise 1 - Open and Read - 40%
========================================

In this part, you need to lookup, open, and read existing files in FAT32.

Lookup and Open
----------------

To look up files in FAT32 directory.

1. Get the cluster number of the directory and calculate its block index.
2. Read the first block of the cluster.
3. Traverse the directory entries to find the file.

You can get the first cluster number of the file in the directory entry.

Read
-----

After you get the first cluster number of the file, you can use readblock to read the file.

Basic Exercise 2 - Create and Write - 30%
=========================================

In this part, you need to create, and write files in FAT32.

Create
-------

To create a new file in FAT32

1. Find an empty entry in the FAT table.
2. Find an empty directory entry in the target directory.
3. Set them to proper values.

Write
------

Similar to read, you can use writeblock to write file, you also need to maintain the metadata of file.

##################
Advanced Exercises
##################

In this part, you create an cache layer for your SD card.

Advanced Exercise 1 - Memory Cached SD Card - 40%
==================================================

Accessing an SD card is much slower than accessing memory. 
Before a CPU shutdown or an SD card ejected, it is not necessary to synchronize the data between memory and SD card. 
Hence, it's more efficient to preserve the data in memory and use memory as a cache for external storage.

We can categorize the file data on the storage into three types: file content, directory entry, and file metadata.

File Metadata
-------------

Besides the content of a file, additional information such as file size is stored in external storage, too. 
The additional information is the file's metadata. There is also metadata for a file system such as FAT tables in FAT.

Those metadata are cached by a file system's kernel objects.


Directory Entry
---------------

VFS can reduce the time spend on reading directory block and parsing directory entry by a component name cache mechanism. 
A component name cache mechanism can be implemented as:

1. Look up the component name cache of the directory first.
2. If successfully finds the vnode, return to the vnode. Otherwise, call the lookup method of the underlying file system.
3. The underlying file system looks up from external storage.
4. If it successfully finds the file, it creates a vnode for the file and put it into the component name cache.


File Content
------------

A VFS can cache file content in memory by page frames. A page cache mechanism can be implemented as:

1. Check the existence of the file's page frames when read or write a file.
2. If the page frames don't exist, allocate page frames for the file.
3. The underlying file system populates the page frames with the file's content in external storage if necessary.

Read or write the page frames of the file.


Sync
-----

VFS should synchronize the file's memory cache with the external storage when a user wants to eject it. 
Hence, a VFS should provide an API for users to synchronize the data, and the file system should implement 
the synchronize method for writing data back to the external storage.

.. code:: c

  // syscall number 20
  void sync();
  // this should call VFS api to sync
  // usually it's calling super_operation sync_fs on all file system
  // but we don't care how you design it, as long as it syncs.

.. note::

  You shouldn't write to SD card until sync is called.
  Ex. open() -> write() -> close() -> poweroff   
  shouldn't create a file on SD card.

.. hint::

  Although normally, filesystem are cached differently for different components (so it's possible to partial sync).
  But we only test against full synchronization, so it is possible to cache at block level.
  which means, all block read and write goes through cache layer, and dirty blocks are written back to SD card when synced.

Test
====

Put :download:`vfs2.img <vfs2.img>` in initramfs.cpio 

Put :download:`FAT_R.TXT <rsrc/FAT_R.TXT>` in your sd card

.. note::

  normally for case insensitive file system, the file system driver does case insensitive lookup.
  But for your convenience, all filenames we use in FAT32 are upper case.

* Run ``fat_r`` for Basic Exercise 1
* Run ``fat_w`` for Basic Exercise 2
* Run ``fat_ws`` for Advanced Exercise 1

.. important::

  you should check your sd card with your computer. 
  For ``fat_w`` there should be ``FAT_W.TXT``.
  For ``fat_ws`` there should be ``FAT_WS.TXT``.

.. note::

  If you've implement advance exercise 1, run ``fat_ws`` before ``fat_w`` and then power off.
  There should be ``FAT_WS.TXT`` but not ``FAT_W.TXT``




