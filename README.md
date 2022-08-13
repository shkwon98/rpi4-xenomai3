# RPi4_Xeno3
To build realtime kernel 4.19.86 with xenomai 3 for **Raspberry Pi 4**

References
------------
* Xenomai 3: https://xenomai.org/installing-xenomai-3-x/
* Raspberry pi linux: https://github.com/raspberrypi/linux

<br>

## Introduction
These steps will help you through the installation of xenomai on the raspberry pi 4. The kernel that is used is the 4.19.86 linux kernel.<br>
Patches from Tantham-h are used to help the installation and make the USB work through the PCIE-bus that is added in the Raspberry Pi 4.

### Requirements:
* Raspberry pi 4
* 16gb Micro-SD card + reader
* Computer with Ubuntu
* Wifi/LAN connection


<br>


## A. Directory initialization

1. Start by pulling the linux repository on the host computer.
```
git clone -b rpi-4.19.y https://github.com/raspberrypi/linux.git linux-rpi-4.19.86-xeno3
```

2. Create a linked folder for easy access.
```
ln -s linux-rpi-4.19.86-xeno3 linux
```

3. Download the xenomai tar.bz2 and extract it. Also create a linked folder for the xenomai installation.
```
wget https://xenomai.org/downloads/xenomai/stable/xenomai-3.1.tar.bz2
tar -xjvf xenomai-3.1.tar.bz2
ln -s xenomai-3.1 xenomai
```

4. Download the patches into xeno3-patches
```
mkdir xeno3-patches && cd xeno3-patches
wget https://github.com/shkwon98/RPi4_Xeno3/blob/main/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch
wget https://github.com/shkwon98/RPi4_Xeno3/blob/main/pre-rpi4-4.19.86-xenomai3-simplerobot.patch
cd ..
```

<br>

## B. Patching linux with xenomai

1. Patch linux with the pre-patch from Tantham-h (If the patch returns with error, just ignore)
```
cd linux
patch -p1 <../xeno3-patches/pre-rpi4-4.19.86-xenomai3-simplerobot.patch
```

2. Patch linux with xenomai
```
../xenomai/scripts/prepare-kernel.sh --linux=./ --arch=arm --ipipe=../xeno3-patches/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch
```

<br>

## C. Building linux

1. Get the default config of the BCM2711 into the linux directory
```
make -j8 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
```

2. Set the menuconfig to the right settings
```
make -j8 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```

3. Edit the following variables:
```
Kernel features —> Timer frequency 1000Hz
General setup —> (-v7l-xeno3) Local version - append to kernel release
CPU powermanagement –> CPU Frequency scaling –> [] CPU Frequency scaling
MemoryManagament options —> [] Allow for memory compaction
Kernel hacking —> [] KDGB: kernel debugger
Xenomai/cobalt —> Drivers —> Real-time GPIO drivers —> [*] GPIO controller
Xenomai/cobalt —> Drivers —> Real-time IPC drivers –> [*] RTIPC protocol familty
```

4. Build the linux kernel (This can take some time, so get coffee or tea...)
```
make -j4 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bzImage modules dtbs
```

<br>

## D. Installing the linux kernel on the Micro-SD card







Preparation on host PC
------------
      sudo apt-get install gcc-arm-linux-gnueabihf
      sudo apt-get install --no-install-recommends ncurses-dev bc

* Download xenomai-3:

      wget https://xenomai.org/downloads/xenomai/stable/xenomai-3.1.tar.bz2
      tar -xjvf xenomai-3.1.tar.bz2
      ln -s xenomai-3.1 xenomai
MODIFY 'xenomai/scripts/prepare-kernel.sh' file
Replace 'ln -sf' by 'cp'  so that it will copy all neccessary xenomai files to linux source

* Download rpi-linux-4.19.86:

	  git clone -b rpi-4.19.y https://github.com/raspberrypi/linux.git linux-rpi-4.19.86-xeno3
	  cd linux-rpi-4.19.86-xeno3
	  git reset --hard c078c64fecb325ee86da705b91ed286c90aae3f6
	  ln -s linux-rpi-4.19.86-xeno3 linux
    
* Download patches set from:

	  mkdir xeno3-patches
Download all files in this directory and save them to *xeno3-patches* directory

	  https://github.com/thanhtam-h/rpi4-xeno3/tree/master/scripts	  
	
Patching
------------
	 cd linux
    
1. Pre-patch:
		patch -p1 <../pre-rpi4-4.19.86-xenomai3-simplerobot.patch
2. Xenomai patching:	
	  	../xenomai/scripts/prepare-kernel.sh --linux=./  --arch=arm  --ipipe=../xeno3-patches/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch
      

Building kernel
------------
	  
	Refer to rpi3 case: https://github.com/thanhtam-h/rpi23-4.9.80-xeno3/blob/master/scripts/README.md
           
      
Prosprocessing
------------  
	**Disable DWC features which may cause ipipe issue: adding to the end of single-line in */boot/cmdline.txt* file**
	dwc_otg.fiq_enable=0 dwc_otg.fiq_fsm_enable=0 dwc_otg.nak_holdoff=0
	
	**CPU affinity**
	isolcpus=0,1 xenomai.supported_cpus=0x3
	
Test xenomai on rpi
------------      
In order to test whether your kernel is really patched with xenomai, run the latency test from xenomai tool:

      sudo /usr/xenomai/bin/latency
If latency tool get start and show some result, you are now have realtime kernel for your rpi

      
