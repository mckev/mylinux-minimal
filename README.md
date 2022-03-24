# mylinux-minimal
A bootable 12 MB Linux with networking


## Build OS

Debian 10.

    ```
    parallels@debian-gnu-linux-10:~$ uname -a
    Linux debian-gnu-linux-10 4.19.0-17-amd64 #1 SMP Debian 4.19.194-3 (2021-07-18) x86_64 GNU/Linux
    parallels@debian-gnu-linux-10:~$ cat /etc/issue
    Debian GNU/Linux 10 \n \l
    parallels@debian-gnu-linux-10:~$ id
    uid=1000(parallels) gid=1000(parallels) groups=1000(parallels),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),109(netdev),112(bluetooth),116(scanner)
    ```


# Build

 1. Preparations:
       ```
       $ sudo mkdir /data
       $ sudo chown $USER:`id -gn $USER` /data

       $ WORKDIR=/data/mylinux
       $ mkdir $WORKDIR
       $ cd $WORKDIR/
       ```


 2. Compile Linux Kernel:
       - Download:
            ```
            $ mkdir $WORKDIR/kernel
            $ cd $WORKDIR/kernel/
            $ wget https://kernel.org/pub/linux/kernel/v5.x/linux-5.13.12.tar.xz
            $ tar xvf linux-5.13.12.tar.xz
            $ cd linux-5.13.12/
            ```

       - Preparations:
            ```
            $ sudo apt install -y flex bison libssl-dev bc libelf-dev
            - Clean:
                 $ make mrproper
            - Create .config:
                 $ make defconfig
            - Edit .config:
                 $ vi .config
            ```

       - Build kernel:
            ```
            $ make bzImage
            ...
              OBJCOPY arch/x86/boot/vmlinux.bin
              HOSTCC  arch/x86/boot/tools/build
              BUILD   arch/x86/boot/bzImage
            Kernel: arch/x86/boot/bzImage is ready  (#1)

            $ ls -al $WORKDIR/kernel/linux-5.13.12/arch/x86/boot/bzImage
            ```

       - Generate kernel headers:
            ```
            $ rm -rf $WORKDIR/kernel_headers
            $ mkdir $WORKDIR/kernel_headers
            $ sudo apt install -y rsync
            $ make INSTALL_HDR_PATH=$WORKDIR/kernel_headers headers_install
            ```


 3. Compile glibc:
       - Download:
            ```
            $ mkdir $WORKDIR/glibc_src
            $ cd $WORKDIR/glibc_src/
            $ wget https://ftp.gnu.org/gnu/glibc/glibc-2.33.tar.bz2
            $ tar xvf glibc-2.33.tar.bz2
            $ cd glibc-2.33/
            ```

       - Preparations:
            ```
            $ sudo apt install -y gawk
            ```

       - Compile:
            ```
            $ rm -rf $WORKDIR/glibc_obj
            $ mkdir $WORKDIR/glibc_obj
            $ cd $WORKDIR/glibc_obj/
            $ $WORKDIR/glibc_src/glibc-2.33/configure --prefix= --with-headers=$WORKDIR/kernel_headers/include --without-gd --without-selinux
            $ make
            ```

       - Copy:
            ```
            $ rm -rf $WORKDIR/glibc
            $ mkdir $WORKDIR/glibc
            $ make install DESTDIR=$WORKDIR/glibc
            ```


 4. Create sysroot:
       ```
       $ rm -rf $WORKDIR/sysroot
       $ mkdir $WORKDIR/sysroot
       $ cp -r $WORKDIR/glibc/* $WORKDIR/sysroot/
       $ cp -r $WORKDIR/kernel_headers/include $WORKDIR/sysroot/
       $ mkdir $WORKDIR/sysroot/usr
       $ ln -s ../include $WORKDIR/sysroot/usr/include
       $ ln -s ../lib $WORKDIR/sysroot/usr/lib
       ```


 5. Compile busybox:
       - Download:
            ```
            $ mkdir $WORKDIR/busybox_src
            $ cd $WORKDIR/busybox_src/
            $ wget https://busybox.net/downloads/busybox-1.34.0.tar.bz2
            $ tar xvf busybox-1.34.0.tar.bz2
            $ cd busybox-1.34.0/
            ```

       - Build:
            ```
            $ make distclean
            $ make defconfig
            $ vi .config
              Replace:
              CONFIG_SYSROOT=""
              with:
              CONFIG_SYSROOT="/data/mylinux/sysroot"

              Replace:
              CONFIG_EXTRA_CFLAGS=""
              with:
              CONFIG_EXTRA_CFLAGS="-L/data/mylinux/sysroot/lib"

            $ make busybox
              Output:
              ...
                CC      util-linux/volume_id/xfs.o
                AR      util-linux/volume_id/lib.a
                LINK    busybox_unstripped
              Trying libraries: crypt m resolv rt
               Library crypt is not needed, excluding it
               Library m is needed, can't exclude it (yet)
               Library resolv is needed, can't exclude it (yet)
               Library rt is not needed, excluding it
               Library m is needed, can't exclude it (yet)
               Library resolv is needed, can't exclude it (yet)
              Final link with: m resolv
            ```

       - Copy:
            ```
            $ rm -rf $WORKDIR/busybox
            $ mkdir $WORKDIR/busybox
            $ make CONFIG_PREFIX="/data/mylinux/busybox" install
            ```


 6. Create rootfs:
       - Create:
            ```
            $ rm -rf $WORKDIR/rootfs
            $ cd $WORKDIR/
            $ tar xvzf rootfs_overlay.tgz
            $ mv rootfs_overlay rootfs

            $ cp -r $WORKDIR/busybox/* $WORKDIR/rootfs/
            $ rm -f $WORKDIR/rootfs/linuxrc
            $ mkdir $WORKDIR/rootfs/lib64
            $ cp $WORKDIR/sysroot/lib/ld-linux-x86-64.so.2 $WORKDIR/rootfs/lib64/
            $ mkdir $WORKDIR/rootfs/lib
            $ cp $WORKDIR/sysroot/lib/libc.so.6 $WORKDIR/rootfs/lib/
            $ cp $WORKDIR/sysroot/lib/libm.so.6 $WORKDIR/rootfs/lib/
            $ cp $WORKDIR/sysroot/lib/libresolv.so.2 $WORKDIR/rootfs/lib/
            $ cp $WORKDIR/sysroot/lib/libnss_dns.so.2 $WORKDIR/rootfs/lib/
            $ cp $WORKDIR/sysroot/lib/libnss_files.so.2 $WORKDIR/rootfs/lib/
            $ ln -s libnss_dns.so.2 $WORKDIR/rootfs/lib/libnss_dns.so
            $ ln -s libnss_files.so.2 $WORKDIR/rootfs/lib/libnss_files.so

            $ strip --strip-debug $WORKDIR/rootfs/bin/* $WORKDIR/rootfs/sbin/*
            $ strip --strip-debug $WORKDIR/rootfs/lib/* $WORKDIR/rootfs/lib64/*
            ```


       - Pack rootfs:
            ```
            $ rm -f $WORKDIR/rootfs.cpio.xz
            $ cd $WORKDIR/rootfs/
            $ find . | cpio -R root:root -H newc -o | xz -9 --check=crc32 > $WORKDIR/rootfs.cpio.xz
            ```


 7. Create UEFI boot image:
       - Create:
            ```
            $ cd $WORKDIR/
            $ tar xvzf uefi_overlay.tgz
            ```

       - Create uefi.img:
            ```
            $ rm -f $WORKDIR/uefi.img
            $ truncate -s 12582912 $WORKDIR/uefi.img
            $ LOOP_DEVICE_HDD=$(sudo /sbin/losetup -f)
            $ echo $LOOP_DEVICE_HDD
            /dev/loop0
            $ sudo losetup $LOOP_DEVICE_HDD $WORKDIR/uefi.img
            $ sudo mkfs.vfat $LOOP_DEVICE_HDD
            mkfs.fat 4.1 (2017-01-24)
            ```

       - Populate uefi.img:
            ```
            $ mkdir $WORKDIR/uefi
            $ sudo mount $WORKDIR/uefi.img $WORKDIR/uefi
            $ sudo cp -r $WORKDIR/uefi_overlay/* $WORKDIR/uefi/
            $ sudo cp $WORKDIR/kernel/linux-5.13.12/arch/x86/boot/bzImage $WORKDIR/uefi/minimal/x86_64/kernel.xz
            $ sudo cp $WORKDIR/rootfs.cpio.xz $WORKDIR/uefi/minimal/x86_64/rootfs.xz
            $ sudo umount $WORKDIR/uefi
            $ rmdir $WORKDIR/uefi
            ```


 8. Create ISO image:
       - Pre-requisites:
            ```
            $ sudo apt install -y xorriso
            ```
       - Download syslinux:
            ```
            $ mkdir $WORKDIR/syslinux
            $ cd $WORKDIR/syslinux/
            $ wget https://kernel.org/pub/linux/utils/boot/syslinux/syslinux-6.03.tar.xz
            $ tar xvf syslinux-6.03.tar.xz
            ```
       - Create:
            ```
            $ rm -rf $WORKDIR/isoimage
            $ mkdir -p $WORKDIR/isoimage/boot
            $ cp $WORKDIR/uefi.img $WORKDIR/isoimage/boot/
            $ xorriso -as mkisofs -isohybrid-mbr $WORKDIR/syslinux/syslinux-6.03/bios/mbr/isohdpfx.bin -e boot/uefi.img -no-emul-boot -isohybrid-gpt-basdat -o $WORKDIR/minimal_linux_live.iso $WORKDIR/isoimage/
            ```
       - File $WORKDIR/minimal_linux_live.iso is generated.


## Test

 9. Test using Parallels Desktop:
       - Launch "Parallels Desktop".
       - Right click "Parallels Desktop" from the dock, and then click "Control Center".
       - On Parallels "Control Center", click "File  >  New...".
            - It shows "Installation Assistant".
            - Click "Install Windows or another OS from a DVD or image file".
              Click "Continue".
            - Click "Choose Manually".
            - Click "Image File".
              Click "select a file...".
              Alternatively, you can install from DVD, Image File, or USB Drive.
            - Browse to: minimal_linux_live.iso
              Click "Open".
            - It says "Unable to detect operating system. Click Continue to proceed anyway or try another source.".
              Click "Continue".
            - Please select your operating system: Select "Other  >  Other".
              Click "OK".
            - Check "[X] Customize settings before installation". Click "Create".
            - On ""Other" Configuration":
                 - Click "Hardware" tab.
                 - Click "Boot Order".
                 - Click "Advanced...".
                 - BIOS: EFI 64-bit										--> default: Legacy BIOS
                   Click "OK".
                 - Close ""Other" Configuration".
            - Click "Continue".
       - The VM boots:
            - It shows "Minimal Linux Live" in the boot menu.


10. Test using actual Intel desktop:
       - Download Rufus:
            - Browse to: https://rufus.ie/en/
            - Click "Rufus 3.18 Portable".
       - Burn the .iso file into a USB drive:
            - Double click "rufus-3.18p.exe":
            - There's a warning "Do you want to allow Rufus to check for application updates online?". Click "No".
            - On "Device", it automatically detects our USB drive "Samsung USB (E:) [256 GB]".
            - On "Boot selection", click "SELECT".
            - Browse to: minimal_linux_live.iso
              Click "Open".
            - There's a window with title "ISOHybrid image detected" and body "The image you have selected is an ISOHybrid, but its creators have not made it compatible with ISO/File copy mode. As a result, DD image writing mode will be enforced.".
              Click "OK".
            - Make sure "Partition scheme" is "GPT".
            - Click "START".
            - There's a warning "WARNING: ALL DATA ON DEVICE 'Samsung USB (E:) [256 GB]' WILL BE DESTROYED. To continue with this operation, click OK. To quit click CANCEL.".
              Click "OK".
            - Click "CLOSE".
       - Reboot the desktop.
            - Press "F12" to enter "BOOT MENU". Notice this is very system dependent.
            - Click "UEFI: Samsung Type-C 1100, Partition 1".
            - It shows "Minimal Linux Live" in the boot menu. Then Linux boots up.
