
![alt tag](https://raw.githubusercontent.com/lateralblast/ansible-lvm-luks/master/images/cat_in_a_box.jpg)

Automating KVM and LUKS with Ansible
------------------------------------

Introduction
------------

This is a quick example of how to automate LVM on LUKS with Anisble.

Requirements
------------

Required software:

- Ansible
- Cryptsetup packages

Overview
--------

This example does the following:

- Creates GPT partions on /dev/sd[b,c,d,e]
- Creates and LVM unencrpted partition on /dev/sde1 and mounts it as /vault to store keys
- Creates a directory /vault/secrets to store the keys and changes the permissions so only root has access
- Generates random keys and stores them in /vault/secrets and makes sure only root has access
- Creates encrypted partitions on /dev/sd[b.c.d]1 using keys and opens then so they can be written to
- Create LVM volumes on the encrypted partions (/dev/mapper/crypt-[net,data,backup])
- Creates ext4 filesystems on the LVM volumes
- Create /etc/cryptab entries for the LVM volumes
- Create /etc/fstab entries for filesystems and mount them

The disk/volume/filesystem end up looking like this (via lsblk):

```
sdb                 8:16   0 465.3G  0 disk
└─sdb1              8:17   0 465.3G  0 part
  └─crypt-net     253:12   0 465.2G  0 crypt
    └─vg02-net    253:13   0 465.2G  0 lvm   /net
sdc                 8:32   0   931G  0 disk
└─sdc1              8:33   0   931G  0 part
  └─crypt-data    253:10   0   931G  0 crypt
    └─vg03-data   253:14   0   931G  0 lvm   /data
sdd                 8:48   0   1.1T  0 disk
└─sdd1              8:49   0   1.1T  0 part
  └─crypt-backup  253:11   0   1.1T  0 crypt
    └─vg04-backup 253:15   0   1.1T  0 lvm   /backup
sde                 8:64   0  14.9G  0 disk
└─sde1              8:65   0  14.9G  0 part
  └─vg05-vault    253:9    0  14.9G  0 lvm   /vault
```