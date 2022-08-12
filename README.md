# RPi4_Xeno3
To build realtime kernel 4.19.86 with xenomai 3 for raspberry pi 4

References
------------
Xenomai 3: https://xenomai.org/installing-xenomai-3-x/
Raspberry pi linux: https://github.com/raspberrypi/linux

1. Then we enter the directory and reset the git repository to

git clone git://github.com/raspberrypi/linux.git linux-rpi-4.19.86-xeno3
cd linux-rpi-4.19.86-xeno3
git reset --hard c078c64fecb325ee86da705b91ed286c90aae3f6
cd ..


2. Then we create a linked folder for easy access.

ln -s linux-rpi-4.19.86-xeno3 linux


3. Download the xenomai tar.bz2 and extract it. Also create a linked folder for the xenomai installation.

wget https://xenomai.org/downloads/xenomai/stable/xenomai-3.1.tar.bz2
tar -xjvf xenomai-3.1.tar.bz2
ln -s xenomai-3.1 xenomai


4. Download the patches into xeno3-patches

mkdir xeno3-patches && cd xeno3-patches
wget https://github.com/thanhtam-h/rpi4-xeno3/blob/master/scripts/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch
wget https://github.com/thanhtam-h/rpi4-xeno3/blob/master/scripts/pre-rpi4-4.19.86-xenomai3-simplerobot.patch
cd ..


1. Patch linux with the pre-patch from Tantham-h

cd linux
patch -p1 <../xeno3-patches/pre-rpi4-4.19.86-xenomai3-simplerobot.patch


2. Patch linux with xenomai

../xenomai/scripts/prepare-kernel.sh --linux=./ --arch=arm --ipipe=../xeno3-patches/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch


1. Get the default config of the BCM2711 into the linux directory

make -j8 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig


2. Set the menuconfig to the right settings

make -j8 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig


make -j4 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bzImage modules dtbs

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

      
