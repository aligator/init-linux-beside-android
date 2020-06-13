# Introduction
It is very easy to run a linux envireonment inside chroot or even without root using proot. There are some apps out there to help you doing this:
* https://github.com/meefik/linuxdeploy
* https://github.com/CypherpunkArmory/UserLAnd

But I wanted to know if it is possible to run a linux environment _beside_ android.

There are some articles already covering that:
* http://whiteboard.ping.se/Android/Debian
* https://jsteward.moe/replace-android-init-with-test-script.html
* https://yhcting.wordpress.com/2011/05/31/android-creating-minimum-set-of-android-kernel-adbd-ueventd-for-android-kernel-test/

However I had to change some steps to be successful on Android 9.

__important:__ Android 10 changes the location of the initramfs. It is moved from the boot.img to the system.img. So this instructions are only working on Android 9 (or maybe lower)

# Step 1 - modify the initramfs
First it is needed to modify the initramfs to be able to change the startup behaviour. 

I tried several unpacking and repacking tools, but even with an unaltered initramfs the images were not usable.
Until I found the magisk tools which are part of the most popular rooting tool currently out there:
https://github.com/topjohnwu/Magisk

The flashable zip of it also includes tools for unpacking the boot.img. So just go to https://github.com/topjohnwu/Magisk/releases and download the newest zip (e.g. Magisk-v20.4.zip)

Unpack the zip and you will find a folder `x86` in it which contains some tools.
We will use one of them to unpack and repack the initramfs.

1. Create a working directory for this project.
1. Copy `magiskboot`into the working directiory.
1. Obtain the boot.img for your current rom. (Either by dumping it or by extracting it from the flashable zip you used to flash your rom, keep in mind that if you installed magisk after flashing, your boot.img will be altered and using the one from the zip will be withoud root.)
1. Run: `./magiskboot unpack boot.img`
1. Create a new folder called `unpacked`: `mkdir unpacked`
1. Switch into it: `cd unpacked`
1. Unpack the `ramdisk.cpio` using `cpio -id < ../ramdisk.cpio` (You may have to install the `cpio` package from the repos of your linux distribution).
1. Now you can modify the initramfs however you like. (more on that later) For the first time I suggest to leave it as it is to verify that unpacking and repacking works.
1. For repacking the initramfs run: `find . | cpio -o --format='newc' > ../ramdisk.cpio`
1. Then `cd ..` into your main working dir and run `./magiskboot repack boot.img`. The boot.img has to be the _original_ one and it will not be modified. Instead the `new-boot.img` is generated.
1. the newly packed `new-boot.img` can be booted using fastboot. (go to the bootloader and run `fastboot boot boot.img`)

If it worked, your phone should just start as normal. (If you did not modify the initramfs.)

# Step 2 - create a custom initramfs
To get some unix tools to work with you can add busybox to your initramfs. 

There are two ways to get a compiled busybox binary.
1. compile yourself
1. use a precompiled one

I just installed busybox on my phone using this app: (It's from the same developer as LinuxDeploy)
https://play.google.com/store/apps/details?id=ru.meefik.busybox&hl=de

Then I copied the binary from `/system/xbin/busybox`to `{initramfs}/sbin/`.
You can use `adb pull` to copy the file to your pc if you have adb debugging enabled.

Now you can start creating a init replacement script
```
#!/sbin/busybox sh
#...
```
But wait... how the hell can I get debug logs?
It took me some time to get a working init because nothing worked and debug logs are not available.
There is no way to write to a console, nor adb nor to a log file in the filesystem.

The rescue is: __kernel panic__

I got the hint here: https://jsteward.moe/replace-android-init-with-test-script.html
It is relatively simple: 
1. Ask the kernel to mount a simple dev-filesystem. The way it was described in the article did not work for me. I use
    `/sbin/busybox mdev -s`
    Mdev is a very simple replacement of e.g. udev and populates the dev directory.
    https://www.google.com/search?channel=trow2&client=firefox-b-d&q=mdev
1. write to /dev/kmsg: `echo MY_OWN_INIT > /dev/kmsg`
1. crash the kernel (by just exiting the init binary, so nothing special to add to the script)  
    So the final initramfs may look like this:
    ```
    #!/sbin/busybox sh

    earlyLog() {
        echo MY_OWN_INIT: $@ > /dev/kmsg
    }

    # set up psuedo-filesystems
    /sbin/busybox mount -t proc proc /proc
    /sbin/busybox mount -t sysfs sysfs /sys
    /sbin/busybox sleep 1
    /sbin/busybox mdev -s
    
    earlyLog yay it works
    ```
1. Repack as described above, go into the bootloader and run `fastboot boot new-boot.img`
1. The phone will reboot, but as we started the phone by using `fastboot boot new-boot.img` and did not flash the new image, it will restart using the original, working initramfs. __important:__ so be sure to have a working boot image flashed.
1. Read the kernel messages from the previous run using `cat /sys/fs/pstore/console-ramoops` or for me it was `cat /sys/fs/pstore/console-ramoops-0`. You can run these commands using adb (`adb shell cat /sys/fs/pstore/console-ramoops-0`) or using a console-app.
1. If it contains your echo (e.g. find it using `cat /sys/fs/pstore/console-ramoops-0 | grep MY_OWN_INIT`), then YAY.

Keep in mind that this only works if the kernel crashed and if it rebooted itself directly. If you just press the power button until it turns off the ramoops file will not exist. So the most easy way is to preserve the normal, working boot image and boot your own image without flashing it.

# Step 3 - install a linux rootfs
I had installed an Arch filesystem using LinuxDeploy to a partition of my sdcard already. So I could just use that.
But you can also install any other linux distribution but keep in mind that systemd won't work. 
In [this](http://whiteboard.ping.se/Android/Debian) article debian was used and it is described how to install it.

# Step 4 - mount the linux partition
I assume here that you have installed linux to a ext partition on your sd card.

First I had to notice that mdev has no idea about what partitions exist. That's why it didn't populate much of the dev filesystem.
But you can add your own nodes using `/sbin/busybox mknod`
For this you have to obtain the _MAJOR MINOR_ values of your partition.
1. Run `mount` on your android console and try to find the partition where the linux is stored. (for me it is `/dev/block/mmcblk1p1` because it is the first partition on my sd card)
1. Then run `ls -l /dev/block/mmcblk1p1`
    You will get an output like this:
    ```
    brw------- 1 root root 179, 65 Feb 16  1970 /dev/block/mmcblk1p1
    ```
    In my case the `179` and `65` are the numbers needed. So we can now mount the sdcard:
1.
    ```
    #!/sbin/busybox sh

    earlyLog() {
        echo MY_OWN_INIT: $@ > /dev/kmsg
    }

    # set up psuedo-filesystems
    /sbin/busybox mount -t proc proc /proc
    /sbin/busybox mount -t sysfs sysfs /sys
    /sbin/busybox sleep 1
    /sbin/busybox mdev -s
    
    # Folders have to be created before using mknod
    /sbin/busybox mkdir /dev/block

    earlyLog create sdcard devices
    /sbin/busybox mknod /dev/block/mmcblk1 b 179 64 &> /dev/kmsg
    /sbin/busybox mknod /dev/block/mmcblk1p1 b 179 65 &> /dev/kmsg
    earlyLog pseudo filesystem setup finished

    earlyLog mount sd card
    /sbin/busybox mkdir /mnt/root
    /sbin/busybox mount -t ext4 -o noatime,nodiratime,errors=panic /dev/block/mmcblk1p1 /mnt/root &> /dev/kmsg

    /sbin/busybox sleep 1


    echo start log > /mnt/root/init.log
    
    # Flush sdcard so that the log is actually written immediately.
    /sbin/busybox sync
    ```
    
Now after running the normal procedure to pack and boot the image, the file `init.log` should have been created on your sd card and contain the word `start log`.

# Step 5 - switch to linux
1. To switch to the linux init just create a file called `/etc/init` in your linux rootfs (add execute permissions using `chmod +x init`)
2. Now just add the switch_root command (and cleanup the environment) at the end of your init script:

    ```
    #!/sbin/busybox sh

    earlyLog() {
        echo MY_OWN_INIT: $@ > /dev/kmsg
    }

    # set up psuedo-filesystems
    /sbin/busybox mount -t proc proc /proc
    /sbin/busybox mount -t sysfs sysfs /sys
    /sbin/busybox sleep 1
    /sbin/busybox mdev -s
    
    # Folders have to be created before using mknod
    /sbin/busybox mkdir /dev/block

    earlyLog create sdcard devices
    /sbin/busybox mknod /dev/block/mmcblk1 b 179 64 &> /dev/kmsg
    /sbin/busybox mknod /dev/block/mmcblk1p1 b 179 65 &> /dev/kmsg
    earlyLog pseudo filesystem setup finished

    earlyLog mount sd card
    /sbin/busybox mkdir /mnt/root
    /sbin/busybox mount -t ext4 -o noatime,nodiratime,errors=panic /dev/block/mmcblk1p1 /mnt/root &> /dev/kmsg

    /sbin/busybox sleep 1
    echo start log > /mnt/root/init.log
    
    
    log cleanup and transfere root
    /sbin/busybox umount /proc
    /sbin/busybox umount /sys
    
    # Flush sdcard so that the log is actually written immediately.
    /sbin/busybox sync

    earlyLog switch root
    exec /sbin/busybox switch_root /mnt/root /etc/init
    ```
3. Your init script can now do whatever it wants.

# Step 6 - start android and let it populate /dev
Now you could populate the whole dev-system on your own, but I have no idea how to do this.
So I use android to populate everything, and wait until it is finished. 

The bad thing about this is that I could not get my android to run just using chroot as described in the other articles.
Also I didn't want to put the original initramfs into a extra partition on the sd card as in the other articles.

Instead I unpacked the original ramfs as described above and packed it back as .tar.gz (leaving it unmodified).
This can be done using `tar -czvf filename.tar.gz .` instead of the `find . | cpio -o --format='newc' > ../ramdisk.cpio` 
Then I copied the tar.gz into my own ramfs and modified the init script to unpack it into a tmpfs partition.  

My final script looks like this:
```
#!/sbin/busybox sh

earlyLog() {
    echo new_era: $@ > /dev/kmsg
}

log() {
    echo $@ >> /mnt/root/init.log
}

# set up psuedo-filesystems
/sbin/busybox mount -t proc proc /proc
/sbin/busybox mount -t sysfs sysfs /sys

/sbin/busybox sleep 1

/sbin/busybox mdev -s

/sbin/busybox mkdir /dev/block

earlyLog create sdcard devices
/sbin/busybox mknod /dev/block/mmcblk1 b 179 64 &> /dev/kmsg
/sbin/busybox mknod /dev/block/mmcblk1p1 b 179 65 &> /dev/kmsg
earlyLog pseudo filesystem setup finished

earlyLog mount sd card
/sbin/busybox mkdir /mnt/root
/sbin/busybox mount -t ext4 -o noatime,nodiratime,errors=panic /dev/block/mmcblk1p1 /mnt/root &> /dev/kmsg

/sbin/busybox sleep 1


echo start log > /mnt/root/init.log
log _________

earlyLog copy initramfs
log copy initramfs
/sbin/busybox mount -t tmpfs -o size=10m tmpfs /mnt/root/android &> /dev/kmsg
/sbin/busybox cp original-ramdisk.tar.gz /mnt/root/android &> /dev/kmsg
cd /mnt/root/android/
/sbin/busybox tar -xzf original-ramdisk.tar.gz &> /dev/kmsg
cd /
earlyLog copied

log cleanup and transfere root
/sbin/busybox umount /proc
/sbin/busybox umount /sys

/sbin/busybox sync

earlyLog switch root
exec /sbin/busybox switch_root /mnt/root /etc/init
```
and my init-script in the linux rootfs looks like this:
```
#!/bin/bash

# Leave all the initialization process to the Android init to handle
echo running init > /stage2.log

sync
cp /usr/bin/busybox /android/sbin/
cd /android
mkdir arch-root

echo patch init to remount the whole android into the arch-chroot >> /stage2.log
sed -i '/^on boot$/a \ \ \ \ mount none / /arch-root/android bind rec' init.rc

pivot_root . arch-root &> /dev/kmsg
mkdir /arch-root/android

/sbin/busybox chroot arch-root /etc/init.stage2 &

exec /sbin/busybox chroot . /init &> /dev/kmsg
```
As you can see I have the problem that I must use pivot_root to basically switch the `/` with the virtual original ramfs tmpfs partition. And therefore loose my linux partition.
To still be able to access the basic tools I copy a busybox installed in my arch-partition to the original ramfs.

Then I start a chroot back to my arch-filesystem in the backround calling stage2.
At the end I chroot into the original android init.
 
The problem is that I could not get the android init running without the `/` being the original initramfs. Just using the chroot without the pivot_root did not work. 

Also I added a patch of the original init.rc which just mounts the whole android after booting android, so that we can access the android filesystem (with all mounts android uses) inside of our chroot.

# Step 7 - wait for android and start background services
This is the `init.stage2` file:
```
#!/bin/bash
export PATH=/sbin:/usr/sbin:/bin:/usr/bin

echo "`date` init stage 2 started - waiting for android" >> /stage2.log

while [ ! -e /android/dev/.coldboot_done ]; do
	/sbin/busybox sleep 1
done

while [ -e /android/dev/.booting ]; do
	/sbin/busybox sleep 1
done

echo setup pseudo file systems >> /stage2.log
mount -o rbind /android/dev /dev
mount -t proc none /proc &>> /stage2.log
mount -t sysfs none /sys &>> /stage2.log

rm -R /tmp/*
mount -t tmpfs -o size=2G tmpfs /tmp &>> /stage2.log

sudo chmod 666 /dev/null

echo start services >> /stage2.log
run-parts /etc/init.d --arg=start &> /services.log

exit 0
```

Basically we are waiting for android to be started and then (bind) mount the pseudo file systems into our chroot.
After that I do some basic setups and finally use run-parts to start my services (e.g. ssh, vnc).

# Conclusion
It was not very easy to find information about this. The articles I linked seem to not work fully with Android 9.
That's why I decided to provide my own foundings about this matter.

Summary:
* I can ssh into the arch linux.
* It runs always, no matter which app is open. __But__ if the screen is turned off android slows down the kernel. So to keep it running at full speed I use the coffeine function my rom includes, which just keeps the screen on.
* I can access the android filesystem as it's mounted to `/android`.
* I even can chroot into it, but its very limited. The PATH is somehow srewed up and so on. But something like 'chroot /android /system/bin/reboot' to reboot works just fine.
* I can access the linux root-fs from android and modify files if ssh refused to start.

I am not sure if this has really a benefit over just starting a normal chroot, but it makes fun hacking with the initrams.

At the end it would be better if I would not need the pivot_root to get android running, because then I would start my linux without chroot. So if anyone has a hint about it...

# Tested devices
Moto G4
