# minimal initramfs

This initramfs is the one described in the readme of the repo.
__Important:__ You have to provide your own `original-ramdisk.tar.gz` containing the whole initramfs of your rom (optionally already patched by magisk)

The file provided in this repo is only an empty dummy file.
Also you have to use a `/sbin/busybox` binary for your architecture.

So to use this initramfs:
1. add your own `original-ramdisk.tar.gz`
1. add a matching `/sbin/busybox` binary
1. modify the paths and partitions in the init script as you need
