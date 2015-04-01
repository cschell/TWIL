# Migrating a virtual machine from Xen to KVM

My company is migrating to a new infrastructure for its virtual machines and switching from XenServer to KVM. This week we had to tackle the migration of existing Xen-VMs to the new hypervisor – turned out that this has been trickier than I thought it would be!

This is a description of how we migrate VMs from a XenServer to an Ubuntu 14.10 with KVM. The VMs run with Debian.

## Get the image to the destination server

You have to get an image from the Xen VM. Our machines are snapshotted every night (using NAUBackup), so for my test vm "TestVM01" I already had the appropriate .xva image.

The next step is to transfer that image to the destination server running KVM. Before you transfer the image you might want to zip it first:

```bash
tar czvf TestVM01.xva.tar.gz /path/to/TestVM01.xva
```

Then copy it to the KVM-server. I've used `scp` for that task:

```bash
scp xenserver.example.com:/path/to/TestVM01.xva.tar.gz ~/xen-migration/  # from the destination server
```

Switch to the download directory and untarzip the whole thing:

```bash
tar xzvf TestVM01.xva.tar.gz
```

## Convert the image

For KVM machines we use raw or qcow2 images. I had to learn that xva images are in fact pretty weird tarballs with many single files that have to somehow be glued together.

Let's untar the image first:

```bash
tar xvf TestVM01.xva
```

You now have a folder with a name like "Ref:1912". In it are said files we have to somehow convert into a KVM image. After some Googleing I found a [python script](https://github.com/hswayne77/xenserver_to_xen) that could do that for me:

```bash
wget https://raw.githubusercontent.com/hswayne77/xenserver_to_xen/master/xenmigrate_new.py
python ./xenmigrate_new.py -c Ref\:1912/ TestVM01.img
```

To be honest, I have no idea how the script works. There are a some similar scripts out there that claim to do the same – of those I tried, only this worked for me.

## Prepare the image

### Access the image
Xen doesn't need Grub as bootloader, so we have to install it. To do so, we have to be able to access the file system, so we have to mount the vm's system partition first and wire up basic system folders so we can chroot into it:

```bash
losetup /dev/loop0 TestVM01.img
kpartx -v -a /dev/loop0 # maybe you have to install that one (apt-get install kpartx)
losetup /dev/loop1 /dev/mapper/loop0p1
mkdir -p /mnt/vm
mount /dev/loop1 /mnt/vm
mount --bind /dev /mnt/vm/dev
mount --bind /sys /mnt/vm/sys
mount --bind /proc /mnt/vm/proc
```

And `chroot`:

```bash
chroot /mnt/vm
```

### Install Grub

**Optional:** In order to download and install grub, I first had to edit the settings to fit the new infrastructure:

  - check `/etc/apt/sources.list`
  - check `/etc/resolve.conf` – if it contains a DNS-server which is not available anymore, you can use 8.8.8.8 as a quick fix


Dealing with Grub is a bit tricky and you might need to play (and google) around with these settings. At first you need to install Grub2 and install it to the disk

```bash
aptitude update
aptitude install grub2
```

Than set the `/boot/grub/device.map`:

```bash
cat > /boot/grub/device.map <<EOF
(hd0) /dev/loop0
(hd0,1) /dev/loop1
EOF
```

Finally I had to remove a setting, that attaches the vm to some kind of Xen terminal – since this environment won't be there anymore, this setting would prevent the VM from booting:

  - Remove every occurrence of `console=hvc0` in `/etc/default/grub`
  - Remove the line(s) containing `hvc0` in `/etc/inittab`

I've also read that you have to check `/etc/fstab` to point at the correct device. But in my case it referenced the partitions' UUIDs (and not a path like `/dev/xvda1`), so I hadn't to do anything.


Install and update Grub:

```bash
grub-install /dev/loop0
update-grub
```

### Configure Networking

Check /etc/network/interfaces to fit the new infrastructure's requirements. We switched from fixed ip addresses to DHCP, so I had to change it like this:

```bash
# vi /etc/network/interfaces
allow-hotplug eth0
iface eth0 inet dhcp
```

### Wrap up
Leave the chroot:

```bash
exit
```

Unwire all the mounts:

```bash
umount /mnt/vm/sys
umount /mnt/vm/proc
umount /mnt/vm/dev
umount /mnt/vm
losetup -d /dev/loop1
kpartx -v -d /dev/loop0
losetup -d /dev/loop0
```

## Install the new VM
Before you setup the new VM, move the image to the path you want it to store permanently. Then install it using:

```bash
virt-install --name TestVM01 --ram 512 -f /path/to/TestVM01.img --import --vnc --connect qemu:///system
```

You now need to connect to the vm **as quickly as you can** via VNC (get the port using `virsh vncdisplay TestVM01`). In my cases I had to press `e` in the bootloader and remove the lines containing `loop1` in order to get Debian booting. Did the VM boot successfully you need to reinstall grub again:

```bash
grub-install /dev/vda
update-grub
```

That did the trick for me! If you have any thoughts or tipps on this, please let me know!

## Sources
https://wiki.debian.org/HowToMigrateBackAndForthBetweenXenAndKvm#Install_grub_and_kernel_in_each_VM_guest
http://michael.stapelberg.de/Artikel/xen_to_kvm/
