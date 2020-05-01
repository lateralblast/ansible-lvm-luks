
![alt tag](https://raw.githubusercontent.com/lateralblast/ansible-lvm-luks/master/images/cat_in_a_box.jpg)

Automating LVM and LUKS with Ansible
====================================

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

- Create GPT partions on /dev/sd[b,c,d,e]
- Create an unencrypted LVM volume group and volume on /dev/sde1 and mounts it as /vault to store keys
- Create a directory /vault/secrets to store the keys and changes the permissions so only root has access
- Generate random keys and stores them in /vault/secrets and makes sure only root has access
- Creates encrypted partitions on /dev/sd[b.c.d]1 using keys
- Create /etc/cryptab entries for the LVM volumes and opens LUKS devices so they can be written to
- Create LVM volumes on the encrypted partitions (/dev/mapper/crypt-[net,data,backup])
- Create ext4 filesystems on the LVM volumes
- Create /etc/fstab entries for filesystems and mount them

The disks/volumes/filesystems end up looking like this (via lsblk):

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

Steps
-----

Creating GPT partions on disks:

```
 name: Create Primary Partitions
  parted:
    device: "{{ item.device }}"
    label:  gpt
    number: 1
    state:  present
  loop:
    - { device: "/dev/sdb" }
    - { device: "/dev/sdc" }
    - { device: "/dev/sdd" }
    - { device: "/dev/sde" }
```

Create an unencrypted LVM volume group and volume on /dev/sde1 and mounts it as /vault to store keys

```
- name: Check Vault Volume Group
  lvg:
    vg:   vg05
    pvs:  /dev/sde1

- name: Check Vault Volume
  lvol:
    vg:     vg05
    lv:     vault
    size:   100%FREE
    shrink: false

- name: Check Vault Filesystem
  filesystem:
    fstype: ext4
    dev:    /dev/mapper/vg05-vault

- name: Check Vault Mount
  mount:
    name:   /vault
    src:    "/dev/mapper/vg05-vault"
    fstype: ext4
    state:  mounted
```

Create a directory /vault/secrets to store the keys and changes the permissions so only root has access

```
- name: Check Secret Directory 
  file:
    state: directory
    path:  /vault/secrets
    owner: root
    group: root
    mode:  "0700"
```

Generate random keys and stores them in /vault/secrets and makes sure only root has access

```
- name: Check Secret Files
  stat:
    path:   "/vault/secrets/crypt-{{ item.vol }}"
  register: vols
  loop:
    - { vol: "net" }
    - { vol: "data" }
    - { vol: "backup" }

- name: Check Secret File Creation
  command: sh -c "dd if=/dev/urandom of=/vault/secrets/crypt-{{ item.item.vol }} bs=1024 count=4"
  args:    
    chdir:   /vault/secrets
    creates: "/vault/secrets/crypt-{{ item.item.vol }}"
  when: 
    - item.stat.exists|bool == false
  with_items:
    - "{{ vols.results }}"

- name: Check Secret File Permissions
  file:
    state: file
    path:  "/vault/secrets/crypt-{{ item.vol }}"
    owner: root
    group: root
    mode:  "0400"
  loop:
    - { vol: "net" }
    - { vol: "data" }
    - { vol: "backup" }
```

Create encrypted partitions on /dev/sd[b.c.d]1 using keys

```
- name: Check LUKS Container is inactive
  command:  sh -c "cryptsetup status crypt-{{ item.vol }} |grep active"
  register: crypts
  loop:
    - { device: "/dev/sdb1", vol: "net" }
    - { device: "/dev/sdc1", vol: "data" }
    - { device: "/dev/sdd1", vol: "backup" }

- name: Create LUKS Container if inactive
  luks_device:
    device:       "{{ item.item.device }}"
    name:         "crypt-{{ item.item.vol }}"
    state:        present
    keyfile:      "/vault/secrets/crypt-{{ item.item.vol }}"
  with_items:
    - "{{ crypts.results }}"
  when: 
    - item.stdout.find('inactive') != -1
```

Create /etc/crypttab entries for the LVM volumes and opens LUKS devices so they can be written to

```
- name: Create crypttab entry
  crypttab:
    backing_device: "{{ item.device }}"
    name:           "crypt-{{ item.vol }}" 
    state:          present
    password:       "/vault/secrets/crypt-{{ item.vol }}"
  loop:
    - { device: "/dev/sdb1", vol: "net" }
    - { device: "/dev/sdc1", vol: "data" }
    - { device: "/dev/sdd1", vol: "backup" }

- name: Open LUKS Containers
  luks_device:
    device:       "{{ item.device }}"
    name:         "crypt-{{ item.vol }}"
    state:        opened
    keyfile:      "/vault/secrets/crypt-{{ item.vol }}"
  loop:
    - { device: "/dev/sdb1", vol: "net" }
    - { device: "/dev/sdc1", vol: "data" }
    - { device: "/dev/sdd1", vol: "backup" }
```

Create LVM volumes on the encrypted partitions (/dev/mapper/crypt-[net,data,backup])

```
- name: Check Volume Groups
  lvg:
    vg:   "{{ item.vg }}"
    pvs:  "/dev/mapper/crypt-{{ item.vol }}"
  loop:
    - { vg: "vg02", vol: "net" }
    - { vg: "vg03", vol: "data" }
    - { vg: "vg04", vol: "backup" }

- name: Check Volumes
  lvol:
    vg:     "{{ item.vg }}"
    lv:     "{{ item.vol }}"
    size:   100%FREE
    shrink: false
  loop:
    - { vg: "vg02", vol: "net" }
    - { vg: "vg03", vol: "data" }
    - { vg: "vg04", vol: "backup" 
```

Create ext4 filesystems on the LVM volumes

```
 - name: Check Filesystems
  filesystem:
    fstype: ext4
    dev:    "/dev/mapper/{{ item.vg }}-{{ item.vol }}"
  loop:
    - { vg: "vg02", vol: "net" }
    - { vg: "vg03", vol: "data" }
    - { vg: "vg04", vol: "backup" }
```

Create /etc/fstab entries for filesystems and mount them

```
- name: Check Mounts
  mount:
    name:   "{{ item.mount }}"
    src:    "/dev/mapper/{{ item.vg }}-{{ item.vol }}"
    fstype: ext4
    state:  mounted
  loop:
    - { mount: "/net",     vg: "vg02", vol: "net" }
    - { mount: "/data",    vg: "vg03", vol: "data" }
    - { mount: "/backup",  vg: "vg04", vol: "backup" }
```
