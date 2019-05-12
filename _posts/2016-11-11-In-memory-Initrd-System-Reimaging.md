---
layout: post
title: Reimage OS from initrd
category: linux
---
- A few years back, we had the problem of upgrading the OS of a fleet of systems which did not have a spare root partition. 
- These were autonomous machines geographically scattered around the world on very slow networks (sort of like ATMs).
- The upgrade process was required to retain and restore certain amount of state from the old install to the new install (think machine identity, private keys, certificates etc.)
- We ended up creating a system for im-memory reimaging of the entire hard disk. 
- Let's understand this process in greater detail.

### Setup the init script 
- Init is the first process called after kernel boots up.
- We add modules for hard disk access, filesystem and a few other
dependencies.
- We start two terminals on tty2 and tty3 to allow diagnosis in case
of errors.
- Thereafter reimaging follows three stages.
{% highlight bash %}
#!/bin/sh

echo Mounting proc filesystem
mount -t proc /proc /proc
echo Mounting sysfs filesystem
mount -t sysfs /sys /sys
echo Creating /dev
mount -o mode=0755 -t tmpfs /dev /dev
echo 3 > /proc/sys/kernel/printk
depmod -A

modprobe ahci
modprobe libata 
modprobe ata_generic
modprobe ata_piix
modprobe crc_t10dif
modprobe sd_mod
modprobe sr_mod
modprobe jbd2
modprobe mbcache
modprobe ext4
modprobe usb_storage

sleep 2
mdev -s
getty -n -l /bin/bash 115200 /dev/tty3  &
getty -n -l /bin/bash 115200 /dev/tty2  &

bash reimage_stage1.sh 2>&1 | tee /log1.txt
bash reimage_stage2.sh
bash reimage_stage3.sh 2>&1 | tee /log2.txt

echo "Error during installation. Dropping to shell."
getty -n -l /bin/bash 115200 /dev/tty1
{% endhighlight %}
### Stage 1: Copy new image and files to be retained to RAM.
{% highlight bash %}
#!/bin/sh
set -e

echo "Starting upgrade..."
mount /dev/sda5 /sysroot
echo "Mounted disk. Copying image to memory."
cp /sysroot/rhel7.img.bz2 /

rm -rf /retentionlist || true

mkdir -p /sysroot/tmp/retentionlist
cp /retentionlist.txt /sysroot/tmp/retentionlist/

cat chroot_commands1.sh | chroot /sysroot

cp -r /sysroot/tmp/retentionlist/ /
rm -rf /sysroot/tmp/retentionlist/
cat /retentionlist/metadata/savedperms

echo "copied rhel7 to /sysroot"
umount /sysroot
echo "Unmounted disk. Disk will be reimaged."
{% endhighlight %}
#### chroot_command1.sh - Retain files and their UID/GID
{% highlight bash %}
mkdir /tmp/retentionlist/data
mkdir /tmp/retentionlist/metadata

for f in `cat /tmp/retentionlist/retentionlist.txt`;
do
    if [ -d $f ]; then
        cp -r -v $f /tmp/retentionlist/data/
    fi
    if [ -f $f ]; then
        cp -v $f /tmp/retentionlist/data/
    fi
    if [ -e $f ]; then
        find `dirname "$f"` -path "$f" -prune -printf '%m\t%u\t%g\t%p\n' >> /tmp/retentionlist/metadata/savedperms
    fi
done
{% endhighlight %}
###  Stage 2: Reimage the disk
{% highlight bash %}
set -e
bzcat rhel7.img.bz2 | pv | dd of=/dev/sda bs=1M
{% endhighlight %}
### Stage 3: Restore files from RAM and copy log files to new install.
{% highlight bash %}
set -e
echo "Starting sync."

sync
echo "Reimaging complete. Refreshing partition table."

hdparm -z /dev/sda
mdev -s

echo "Current partition table is..."
blkid

echo "Mounting RHEL7 root partition."
mount /dev/sda2 /sysroot

echo "Mounting RHEL7 kiosk partition."
mount /dev/sda6 /sysroot/kiosk

echo "Moving retentionlist into chroot"

mkdir /sysroot/tmp/retentionlist
cp -r /retentionlist/ /sysroot/tmp/

cat /chroot_commands.sh | chroot /sysroot

rm -rf /sysroot/tmp/retentionlist
cp /log1.txt /sysroot/kiosk/log/
cp /log2.txt /sysroot/kiosk/log/

echo "Unmounting RHEL7 kiosk partition."
umount /sysroot/kiosk

echo "Unmounting RHEL7 root partition."
umount /sysroot

sync
echo "Restore successful. Installation complete. Rebooting in 5 seconds."
sleep 5
echo s > /proc/sysrq-trigger
echo u > /proc/sysrq-trigger
echo b > /proc/sysrq-trigger
{% endhighlight %}
#### chroot_commands.sh - Restore files and it's owners
{% highlight bash %}
echo "Restoring retention list."
for f in `cut -f4 /tmp/retentionlist/metadata/savedperms`;
do
    if [ -d /tmp/retentionlist/data/`basename $f` ]; then
        cp -r -v /tmp/retentionlist/data/`basename $f` /$f
    fi
    if [ -f /tmp/retentionlist/data/`basename $f` ]; then
        cp -v /tmp/retentionlist/data/`basename $f` /$f
    fi
done

echo "Restoring file owners"
awk '{system("chown -v "$2":"$3" "$4)}' /tmp/retentionlist/metadata/savedperms
sleep 5
echo "Restoring file permissions"
awk '{system("chmod -v "$1" " $4)}' /tmp/retentionlist/metadata/savedperms
sleep 5
{% endhighlight %}

