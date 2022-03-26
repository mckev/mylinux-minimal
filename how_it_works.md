How it works
============

 1. UEFI


 2. kernel
       - Kernel initializes hardware.

       - Kernel executes "/init".
         ```
         From kernel/linux-5.13.12/init/main.c
            - start_kernel()  -->  arch_call_rest_init();  -->  rest_init();  -->  kernel_thread(kernel_init, NULL, CLONE_FS);
            - kernel_init()  -->  run_init_process("/init");
         ```


 3. initrd
       - /init
            - echo -e "Welcome to Minimal Linux Live (/init)"

            - /etc/01_prepare.sh
                 ```
                 mount -t devtmpfs none /dev
                 mount -t proc none /proc
                 mount -t tmpfs none /tmp -o mode=1777
                 mount -t sysfs none /sys
                 mkdir -p /dev/pts
                 mount -t devpts none /dev/pts
                 ```

            - exec /etc/02_overlay.sh

       - /etc/02_overlay.sh
            - ```
              echo -e "Copying the root file system to /mnt."
              for dir in */ ; do
                    cp -a $dir /mnt
              done
              ```

            - ```
              mount --move /dev /mnt/dev
              mount --move /sys /mnt/sys
              mount --move /proc /mnt/proc
              mount --move /tmp /mnt/tmp
              echo -e "Mount locations /dev, /sys, /tmp and /proc have been moved to /mnt."
              ```

            - ```
              echo "Switching from initramfs root area to overlayfs root area."
              exec switch_root /mnt /etc/03_init.sh
              ```

       - /etc/03_init.sh
            - ```
              echo -e "Executing /sbin/init as PID 1."
              exec /sbin/init
              ```


 4. rootfs
       - /sbin/init  -->  ../bin/busybox
            - It reads /etc/inittab
              ```
              ::sysinit:/etc/04_bootscript.sh
              ::once:cat /etc/motd
              tty2::once:cat /etc/motd
              tty2::respawn:/bin/sh
              ```

       - /etc/04_bootscript.sh
            - echo -e "Welcome to Minimal Linux Live (/sbin/init)"

            - ```
              # Autorun functionality
              if [ -d /etc/autorun ] ; then
              for AUTOSCRIPT in /etc/autorun/*
                do
                  if [ -f "$AUTOSCRIPT" ] && [ -x "$AUTOSCRIPT" ]; then
                    echo -e "Executing $AUTOSCRIPT in subshell."
                    $AUTOSCRIPT
                  fi
                done
              fi
              ```

       - /etc/autorun/20_network.sh
            - ```
              # DHCP network
              for DEVICE in /sys/class/net/* ; do
                echo "Found network device ${DEVICE##*/}"
                ip link set ${DEVICE##*/} up
                [ ${DEVICE##*/} != lo ] && udhcpc -b -i ${DEVICE##*/} -s /etc/05_rc.dhcp
              done
              ```

            - /sbin/udhcpc  -->  ../bin/busybox

            - Output:
                 ```
                 Found network device eth0
                 udhcpc: started, v1.34.0
                 udhcpc: broadcasting discover
                 udhcpc: broadcasting select for 10.211.55.7, server 10.211.55.1
                 udhcpc: lease of 10.211.55.7 obtained from 10.211.55.1, lease time 1800
                 DHCP configuration for device eth0
                 IP:     10.211.55.7
                 mask:   24
                 router: 10.211.55.1

                 Found network device lo
                 ```

       - Prints /etc/motd  (from /etc/inittab)
            ```
            ###################################
            #                                 #
            #  Welcome to Minimal Linux Live  #
            #                                 #
            ###################################
            ```


       - Executes /bin/sh  (from /etc/inittab)
