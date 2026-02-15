---
title: "Creating shared storage for Proxmox VMs"
description: "A quick guide for setting up shared storage between VMs when using ZFS"
categories:
  - Homelab
  - Storage
tags:
  - ZFS
  - Proxmox
# image:
#   path: /assets/images/misc/code.png
#   alt: "Staring at lines of code - Generated with Fooocus"
pin: false
---

## The Reasoning
It is quite often useful to have a shared folder that is attached in multiple VMs in a Proxmox environment, 
for example, to share a configuration file or to not have to re-download files across VMs.

This way of setting up file sharing in between VMs is meant to be used in an environment where users who have
read/write access to the shared folder are trusted and ideally there is little to no concurrent reading/writing
on the same file. 

## Setting up a ZFS dataset

Suppose we have already setup a ZFS pool named `zfspool`, we can set up our shared dataset inside that pool
using the following command:

```bash
zfs create zfspool/shared
```

Then it is recommended to set the following ZFS parameters:

```bash
zfs set compression=zstd zfspool/shared
zfs set atime=off zfspool/shared
```
We can also set a quota for out dataset:
```bash
zfs set quota=100.0G zfspool/shared
```

## Mounting dataset inside the VM
Now that our dataset is set up, we can proceed with mounting our shared dataset inside the VM. This can be done
through the proxmox interface by setting up a VirtioFS mount.

Before mounting the dataset inside the VM we must first set up a *Directory Mapping* to our shared ZFS folder. This
can be done by navigating to **Datacenter &#8594; Directory Mappings** and clicking `Add` and specifying a name for 
our directory map (`sharedzfs`), the mount path (`/zfspool/shared/`) and specifying the node our mount lives on. 

> The name we set up here will be our mount tag for the following steps, so choose carefully.
{: .prompt-warning }

Then we can navigate to the VM configuration and under the `Hardware` tab, we can `Add` &#8594; `Virtiofs`. Then we
can select the Directory ID we set up earlier (`sharedzfs`).

At this point we are ready to mount the directory inside our VM. For this example we choose a mount point of `/mnt/shared`.
As such we create the directory:
```bash
sudo mkdir -p /mnt/shared
```
Then we can mount our VirtioFS share:
```bash
sudo mount -t virtiofs sharedzfs /mnt/shared
```
We can verify that our mount is working by checking the contents of the directory (if present) and by checking the output of `df -h`.
If everything has happened correctly, it should return something along the lines of:
```
df -h
...
sharedzfs                   100G  509M  100G   1% /mnt/shared
...
```
Once we have verified that our mount is sucessful, we can make it **persistent**. This can be done by editing `/etc/fstab` and adding the
following line:
```
sharedzfs  /mnt/shared  virtiofs  defaults  0  0
```
The following line can be interpreted as follows:
- `sharedzfs`: Our mount tag, set in the first step
- `/mnt/shared`: The mount path we chose
- `virtiofs`: The filesystem type, which is in this case is a host-backed filesystem passthrough 
- `defaults`: Standard mount behavior
- `0`: `Dump` &#8594; Don't back this up using `Dump`
- `0`: `fsck` &#8594; Don't run filesystem checks at boot. This prevents Linux from trying to *repair* this mount, as VirtioFS is not a real
block device.

We can verify that our entry is correct by running:
```bash
sudo mount -a
```
If this returns no output, then our entry is valid and it is safe to reboot the VM.

## Setting up permissions for the shared dataset

Even though our filesystem is mounted, we can quickly notice that if we try to write anything to it, we will be hit by a permission denied error:
```
touch test.txt
touch: cannot touch 'test.txt': Permission denied
```
This happens because VirtioFS doesn't override the permissions set by the host filesystem. ZFS enforces the permissions of the host. Because of the way
we set up our dataset, it is owned by `root`, which means that while other user can read/execute from it, they cannot write. We can fix this by introducing
a new usergroup that owns our shared dataset and that we can assign to any user we want to be able to write to it. Usergroups in Linux are typically set up
such that:

> To be continued once I have more time :)
{: .prompt-warning }

