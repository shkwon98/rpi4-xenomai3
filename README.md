# Raspberry Pi 4 Xenomai 3 Patch
These steps will help you through the installation of xenomai on the **Raspberry Pi 4**. The kernel that is used is the 4.19.86 linux kernel.<br>
Patches from Tantham-h are used to help the installation and make the USB work through the PCIE-bus that is added in the Raspberry Pi 4.<br>
<br>



## Requirements:
* Raspberry Pi 4
* 16gb Micro-SD card + reader
* Computer with Ubuntu
* Wifi/LAN connection

> Create a new raspbian image on the micro-SDcard with the Pi imager. Use the file on the link below.<br>
> [2020-02-13-raspbian-buster.zip](http://downloads.raspberrypi.org/raspbian/images/raspbian-2020-02-14/2020-02-13-raspbian-buster.zip)<br>
<br>



## A. Directory initialization

1. Start by pulling the linux repository on the host computer.
```console
host@ubuntu:~$ git clone -b rpi-4.19.y https://github.com/raspberrypi/linux.git linux-rpi-4.19.86-xeno3
```

2. Create a linked folder for easy access.
```console
host@ubuntu:~$ ln -s linux-rpi-4.19.86-xeno3 linux
```

3. Download the xenomai tar.bz2 and extract it. Also create a linked folder for the xenomai installation.
```console
host@ubuntu:~$ wget https://xenomai.org/downloads/xenomai/stable/xenomai-3.1.tar.bz2
host@ubuntu:~$ tar -xjvf xenomai-3.1.tar.bz2
host@ubuntu:~$ ln -s xenomai-3.1 xenomai
```
> Modify 'xenomai/scripts/prepare-kernel.sh' file<br>
> Replace 'ln -sf' by 'cp'  so that it will copy all neccessary xenomai files to linux source

4. Download the patches into xeno3-patches
```console
host@ubuntu:~$ mkdir xeno3-patches && cd xeno3-patches
host@ubuntu:~/xeno3-patches$ wget https://github.com/shkwon98/rpi4-xenomai3/blob/main/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch
host@ubuntu:~/xeno3-patches$ wget https://github.com/shkwon98/rpi4-xenomai3/blob/main/pre-rpi4-4.19.86-xenomai3-simplerobot.patch
host@ubuntu:~/xeno3-patches$ cd ..
```

<br>

## B. Patching linux with xenomai

1. Patch linux with the pre-patch from Tantham-h (If the patch returns with error, just ignore)
```console
host@ubuntu:~$ cd linux
host@ubuntu:~/linux$ patch -p1 <../xeno3-patches/pre-rpi4-4.19.86-xenomai3-simplerobot.patch
```

2. Patch linux with xenomai
```console
host@ubuntu:~/linux$ ../xenomai/scripts/prepare-kernel.sh --linux=./ --arch=arm --ipipe=../xeno3-patches/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch
```

<br>

## C. Building linux

1. Preparation on host PC
```console
host@ubuntu:~$ sudo apt-get install gcc-arm-linux-gnueabihf
host@ubuntu:~$ sudo apt-get install --no-install-recommends ncurses-dev bc
```

2. Get the default config of the BCM2711 into the linux directory
```console
host@ubuntu:~/linux$ make -j4 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
```

3. Set the menuconfig to the right settings
```console
host@ubuntu:~/linux$ make -j4 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```

4. Edit the following variables:
```
Kernel features —> Timer frequency 1000Hz
General setup —> (-v7l-xeno3) Local version - append to kernel release
CPU powermanagement –> CPU Frequency scaling –> [] CPU Frequency scaling
MemoryManagament options —> [] Allow for memory compaction
Kernel hacking —> [] KDGB: kernel debugger
Xenomai/cobalt —> Drivers —> Real-time GPIO drivers —> [*] GPIO controller
Xenomai/cobalt —> Drivers —> Real-time IPC drivers –> [*] RTIPC protocol familty
```

5. Build the linux kernel (This can take some time, so get coffee or tea...)
```console
host@ubuntu:~/linux$ make -j4 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bzImage modules dtbs
```

<br>

## D. Installing the linux kernel on the Micro-SD card

1. Create a shortcut to the identifier for the PI4 ARM version of the kernel
```console
host@ubuntu:~/linux$ KERNEL=kernel7l
```

2. In terminal, locate the fresh installed SD card and see which mounts points are created
```console
host@ubuntu:~/linux$ lsblk
```
> Now we assume that **sdb1** being the FAT (boot) partition, and **sdb2** being the ext4 filesystem (root) partition.

3. Create mount points for both these partitions and mount the sd card to them.
```console
host@ubuntu:~/linux$ mkdir mnt
host@ubuntu:~/linux$ mkdir mnt/fat32
host@ubuntu:~/linux$ mkdir mnt/ext4
host@ubuntu:~/linux$ sudo mount /dev/sdb1 mnt/fat32
host@ubuntu:~/linux$ sudo mount /dev/sdb2 mnt/ext4
```

4. Install the modules of the build linux system and install them on the root partition.
```console
host@ubuntu:~/linux$ sudo env PATH=$PATH make O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=mnt/ext4 modules_install
```

5. Install the kernel image, bts packets and the overlays into the boot partition.<br>
Also the original kernel image is saved, in the case that the kernel does not boot.
```console
host@ubuntu:~/linux$ sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img
host@ubuntu:~/linux$ sudo cp build/arch/arm/boot/zImage mnt/fat32/$KERNEL.img
host@ubuntu:~/linux$ sudo cp build/arch/arm/boot/dts/*.dtb mnt/fat32/
host@ubuntu:~/linux$ sudo cp build/arch/arm/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
```

6. Edit the neccessary boot configs and cmdline such that the pi can boot, also add the ssh access file
```console
host@ubuntu:~/linux$ touch mnt/fat32/ssh
host@ubuntu:~/linux$ sudo nano mnt/fat32/cmdline.txt
```
> Add at the end of the first line:
```
dwc_otg.fiq_enable=0 dwc_otg.fiq_fsm_enable=0 dwc_otg.nak_holdoff=0 isolcpus=0,1 xenomai.supported_cpus=0x3
```
```console
host@ubuntu:~/linux$ sudo nano mnt/fat32/config.txt
```
> Add in the beginning:
```
total_mem=3072
```

7. Unmount the SD card and insert it into the Raspberry pi and power it up with the ethernet cable attached to the host computer.
```console
host@ubuntu:~/linux$sudo umount mnt/fat32
host@ubuntu:~/linux$sudo umount mnt/ext4
```

8. Boot the raspberry pi and ssh into it, check the linux kernel.
```console
pi@raspberrypi:~$ uname -r
```

<br>

## E. Installing the xenomai libraries on the Raspberry pi

1. Build the xenomai libraries on the host PC.
```console
host@ubuntu:~$ cd xenomai
host@ubuntu:~/xenomai$ ./scripts/bootstrap
host@ubuntu:~/xenomai$ ./configure --host=arm-linux-gnueabihf --enable-smp --with-core=cobalt
host@ubuntu:~/xenomai$ make
host@ubuntu:~/xenomai$ sudo make install
host@ubuntu:~/xenomai$ tar -cjvf rpi4-xeno3-deploy.tar.bz2 /usr/xenomai
```

2. Copy the constructed tar.bz2 file to your raspberry pi.
```console
host@ubuntu:~/xenomai$ scp rpi4-xeno3-deploy.tar.bz2 pi@<raspberry pi IP address>
```

3. Switch to the raspberry pi (SSH into the raspberry pi)
```console
host@ubuntu:~/xenomai$ ssh pi@<raspberry pi IP address>:/home/pi
```
4. Inside the raspberry pi, unpack the tar.bz2 and copy it to /usr
```console
pi@raspberrypi:~$ sudo tar -xjvf rpi4-xeno3-deploy.tar.bz2 -C /
```

5. Create a config file and add the xenomai libraries to it.
```console
pi@raspberrypi:~$ sudo nano /etc/ld.so.conf.d/xenomai.conf
```
> Add to this file:
```
#xenomai lib path
/usr/local/lib
/usr/xenomai/lib
```

6. Load the config and reboot.
```console
pi@raspberrypi:~$ sudo ldconfig
pi@raspberrypi:~$ sudo reboot
```

7. Run some tests and you are done!
```console
pi@raspberrypi:~$ sudo /usr/xenomai/bin/xeno-test
pi@raspberrypi:~$ sudo /usr/xenomai/bin/latency
```
> If latency tool get start and show some result, you are now have realtime kernel for your rpi
