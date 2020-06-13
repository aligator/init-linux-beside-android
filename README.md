# Introduction
It is very easy to run a linux envireonment inside chroot or even without root using proot. There are some apps out there to help you doing this:
https://github.com/meefik/linuxdeploy
https://github.com/CypherpunkArmory/UserLAnd

But I wanted to know if it is possible to run a linux environment _beside_ android.

There are some articles already covering that:
http://whiteboard.ping.se/Android/Debian

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
1. Run `TODO`
1. Create a new folder called `unpacked`: `mkdir unpacked`
1. Switch into it: `cd unpacked`
1. Unpack the `initramfs.cpio` using `TODO` (You may have to install the `cpio` package from the repos of your linux distribution).
1. Now you can modify the initramfs however you like. (more on that later) For the first time I suggest to leave it as it is to verify that unpacking and repacking works.
1. For repacking the initramfs run: `TODO`
1. Then `cd ..` into your main working dir and run `TODO` to regenerate the boot.img. For this step you need the original boot.img you initially unpacked as the magisk-tools need it.
1. the newly packed `new-boot.img` can be booted using fastboot. (go to the bootloader and run `fastboot boot boot.img`)

If it worked, your phone just starts as normal. (If you did not modify the initramfs.)

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

The rescue is: kernel panic

