---
layout: post
title:  "kvm/lvm manipulation tricks"
tags: kvm 
---

I've benefited over the years from blogs about weird kvm provisioning
setups so I feel like I owe it to anonymous strangers to publish some 
tricks I've found and used for home lab or even more sinister purposes.

For background, I host a variety of VMs for the purposes of home services
or just general experimentation on a Rube Goldberg collection of scraped
together hardware. Compute nodes that mount FreeNAS volumes via iSCSI,
nfs, local SSD, slower local storage, clusters of 64 bit ARM RockPI
boxes, PoE powered distributed compute, you name it. The examples below 
are VMs that run on an 8 core Intel Atom box with a lot of ram on a 
APC1500 UPS with volumes from a redundant LVM pool on SSDs. 

I took a `cloudimg` from the official ubuntu nightly build, 
copied it into a LVM volume on a host, and built that into a VM
after some freaky and poorly decided local edits.  

* Download the latest focal-server-cloudimg-amd64-disk-kvm.img
* lvcreate -L 30g -n machine0 -T my-vg
* qemu-img convert focal.img -O raw /dev/my-vg/machine0

Now you've got the latest cloud image sitting in the first few
hundred megs of a logical volume on a host.

While I recently learned there's a neat trick to package as an iso the cloud-init
utilities I'm used to passing via user-data, my searching for similar led me to
the guestmount/guestedit tools which let you mount images (as loopback mounts) or 
in my case, VM volumes as filesystems.

```
mkdir /mnt/machine0 
guestmount -a /dev/my-vg/machine0 -m /dev/sda1 --rw /mnt/machine0
chroot /mnt/mb0 /bin/bash
```
after that you're running your original host's kernel, but in a chrooted
environment that has a as-yet-unrun VM's filesystem as root. I was able
to quickly set the machine's hostname, make an initial user, setup
static networking if required.  cloud-init via user-data would 
have been the more standard approach, but I was just delighted to
find a simple way to chroot a guest VM's root filesystem before I
ever booted the VM. 

Exit the chroot, unmount the `guestmount` and use `virt-install` to 
create a new VM without install media that uses the already populated `/dev/my-vg/machine0`
LVM

```
virt-install \
--connect qemu:///system \
--import \
--name machine0 \
--ram 2048 \
--disk /dev/my-vg/machine0 \
--vcpus 2 \
--network bridge=br1,model=virtio \
--network bridge=br0,model=virtio \
--graphics none \
--console pty,target_type=serial
```
In this case the VM needed two network interfaces as I was building a mail relay
 but that's unusual. 
 

