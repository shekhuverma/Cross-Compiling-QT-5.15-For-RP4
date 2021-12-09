# Cross-Compiling Qt 5.15.0 for Raspberry Pi 4

I have faced some issues when compiling Qt for Raspberry Pi 4. With every update to the software and hardware, the instructions that were written for an older version seem to become slightly broken with the new version. 
When compiling the source directly on the Pi, it can take a long time to realise an issue even occured, due to the slower CPU.
 
It would mean that I can do all the CPU intensive work directly on my PC and I can also fail faster, speeding up my ability to find and troubleshoot issues.

This guide documents the steps I followed to corss-compile Qt 5.15.0 for the Raspberry Pi 4B and make it work with *PyQT5 using python*. Hope it might be useful to anyone else wanting to achieve something similar.

## Set up as tested:
**Hardware**  
Host: Ryzen 7 5800H + 16 GB RAM + RTX 3050  
Target: Raspberry Pi 4 Model B Rev (4 Gb Variant)

**Software**  
Host: Ubuntu 21.10 64-bit   
Target: Raspberry Pi OS (32-bit)   
Cross Compiler: gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf  

**Other Notes**  
Networking: Your Raspberry Pi REQUIRES internet access to follow these instructions. It will also need to be on the same network as the host PC

## Achnowledgements
This guide is heavily based on this guide which can be found here.
https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4

I also used the following guides for reference:  
- https://forums.raspberrypi.com/viewtopic.php?t=325245
- https://stackoverflow.com/questions/38030140/import-qtquick-controls-2-0-not-working-qqmlapplicationengine-failed-to-load-c
- https://github.com/xuancong84/public/blob/master/fix-stupidity/qeglfsbrcmintegration.cpp

And many thanks to [Ankit Sharma](https://github.com/maglash64) for his guidance and support!

### If you already have Raspberry Pi setup with Raspbian and SSH Running , you can go to step 

## Step 1: Download the Raspberry Pi OS image
I downloaded the image and prepared the SD card on my Windows machine. There are a few steps to do here:
	I used Raspberry pi Imager to burn the OS image in the SD card.
	- Version I used
	````
	cat /etc/os-release

	PRETTY_NAME="Raspbian GNU/Linux 11 (bullseye)"
	NAME="Raspbian GNU/Linux"
	VERSION_ID="11"
	VERSION="11 (bullseye)"
	VERSION_CODENAME=bullseye
	ID=raspbian
	ID_LIKE=debian
	```

- Latest version: https://downloads.raspberrypi.org/raspios_full_armhf_latest
- Note the above links are for the desktop version bundled with all the recommended software. You can also use the standard desktop version, or even the minimal version if you prefer
- Flash the image onto an SD card (I used Balena Etcher)
- I wanted to set up my Raspberry Pi to be accessible remotely so I set up WiFi and SSH as well directly on the SD card after flashing as described below

### 1.1 Set up WiFi
With the SD card connected to your computer, create a new file called `wpa_supplicant.conf`  
on the /boot/ partition

The file should have the following contents:

	country=GB
	ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
	update_config=1

	network={
		ssid="NETWORK-NAME"
		psk="NETWORK-PASSWORD"
		priority=2
	}
	
	network={
		ssid="SECOND-NETWORK-NAME"
		psk="SECOND-NETWORK-PASSWORD"
		priority=1
	}	
	
Modify the contents of the network block to match that of your WiFi connection. You can add as many network blocks as you would like, if you want the device to be able to automatically connect to one of many networks. I left the template to allow for two networks.  
Use the priority value to select which network the RPi should prioritise if multiple are visible. Higher number = higher priority

### 1.2 Set up SSH
I wanted the ability to SSH into the Pi straight away. To enable ssh, simply create an empty file called `ssh` in the /boot/ parition

On my desktop, I used `kitty` (a popular fork of  `putty`) to SSH into the RPi. If you're doing it within the virtual machine, you can also use Ubuntu's built in SSH capability directly from any terminal.

Now the SD card can be inserted into the RPi.  

If you aren't planning on connecting a monitor and keyboard, you can use your network router's interface to figure out what the IP address of the RPi is.


## Step 2: Configure the Raspberry Pi
For our build we need to have SSH enabled. This can be done through the `raspi-config` utility.  
If you followed the instructions above, SSH should already enabled. If not I've detailed how to do it below:

On the Raspberry Pi terminal, type:

	sudo raspi-config
	
A menu should pop up on your terminal

## 2 Enable SSH
To enable SSH, go to:

	Interfacing Options -> P2 SSH -> Yes
	
Press Enter/Return to enable it

### 2.1 Enable Development Sources
You need to edit your sources list to enable development sources. To do this, enter the following into a terminal

	sudo nano /etc/apt/sources.list
	
In the nano text editor, uncomment the following line by removing the `#` character (the line should exist already, if not then add it):

	deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
	
Now press `Ctrl+X` to quit. You will be asked if you want to save the changes. Press `y` for yes, and then press `Enter` to keep the same filename.

### 2.3 Update the system

Run the following commands in terminal to update the system and reboot

	sudo apt-get update
	sudo apt-get dist-upgrade
	sudo reboot

### 2.4 Enable rsync with elevated rights
Later in this guide, we will be using the `rsync` command to sync files between the PC and the RPi. For some of these files, root rights (i.e. sudo) are required.  
In this step, we will change a setting to allow this.

First, find the path to rsync with the following command:

	which rsync

On my RPi it was here:

	/usr/bin/rsync

Now we need to edit the sudoers file. You can edit it by typing the following into terminal:

	sudo visudo
	
Now the sudoers file should be opened with nano. You need to add an entry to the end with the following structure:

	<username> ALL=NOPASSWD:<path to rsync>
	
In my case (and for most others else as well), it was:

	pi ALL=NOPASSWD:/usr/bin/rsync

That's it. Now rsync should be setup to run with sudo if needed.

### 2.5 Install the required development packages

Run the following commands in terminal to install the required packages

	sudo apt-get build-dep qt5-qmake
	sudo apt-get build-dep libqt5gui5
	sudo apt-get build-dep libqt5webengine-data
	sudo apt-get build-dep libqt5webkit5
	sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 gdbserver
	
At this stage I made a backup of my SD card image using `Win32DiskImager` so that I can easily revert to this state if something later on broke the installation.

### 2.6 Create a directory for the Qt install
This is where the built Qt sources will be deployed to on the Rasberry Pi. Run the following to create the directory:

	sudo mkdir /usr/local/qt5.15
	sudo chown -R pi:pi /usr/local/qt5.15
	

## Step 3: Configure PC
This guide assumes that you have Ubuntu already installed on your machine, either natively or running within a virtual machine.

### 3.1 Update the PC
Run the following to update your system and install some needed dependancies:

	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get install gcc git bison python gperf pkg-config gdb-multiarch
	sudo apt install build-essential
	
### 3.2 Set up SSH keys to speed up connecting with the Raspberry Pi
Normally, everytime you connect from your PC to the RPi, you will need to provide the login credentials. We can use SSH keys to avoid this and speed up the process.

Detailed instructions on how to do this are provided in the Raspberry Pi documentation here:
https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md

For simplicity, I opted not to use a passphrase when generating the key.

### 3.3 Set up the directory structure
I chose to create a directory called "rpi" to use as my workspace for the cross-compiler on the PC. Use these commands to create the directory structure:

	sudo mkdir ~/rpi
	sudo mkdir ~/rpi/build
	sudo mkdir ~/rpi/tools
	sudo mkdir ~/rpi/sysroot
	sudo mkdir ~/rpi/sysroot/usr
	sudo mkdir ~/rpi/sysroot/opt
	sudo chown -R 1000:1000 ~/rpi
	cd ~/rpi
	
The second to last line makes the first user of the computer (hopefully you) the owner of that folder. You can replace the `1000` with your user name if you want to be sure.

The last command should have changed your current directory to ~/rpi. If not, run the last line to make sure you are inside `~/rpi` as the next steps assume you're running your commands from that directory.

## Step 4: Build Preparation & Build

### 4.1 Download Qt sources
Now we can download the source files for Qt. As mentioned before, this guide is for Qt 5.15.0, which is the latest version available at the time of running. It is also the latest LTS version.

Run the following line to download the source files:

	sudo wget http://download.qt.io/archive/qt/5.15/5.15.0/single/qt-everywhere-src-5.15.0.tar.xz
	
Extract the downloaded tar file with the following command:

	sudo tar xfv qt-everywhere-src-5.15.0.tar.xz 
	
We need to slightly modify the a mkspec file within the source files to allow us to use our cross compiler. We will copy an existing directory within the source files, and modify the name of the directory and the 
contents of the qmake.conf file within that directory to follow the name of our compiler.  

To do this, run the following two command:

	cp -R qt-everywhere-src-5.15.0/qtbase/mkspecs/linux-arm-gnueabi-g++ qt-everywhere-src-5.15.0/qtbase/mkspecs/linux-arm-gnueabihf-g++
	
	sed -i -e 's/arm-linux-gnueabi-/arm-linux-gnueabihf-/g' qt-everywhere-src-5.15.0/qtbase/mkspecs/linux-arm-gnueabihf-g++/qmake.conf


### 4.2 Download the cross-compiler

Most cross-compilation guides for the Raspberry Pi will use the tools provided by Raspberry Foundation themselves, but this will not work here as the gcc version present in those files is too old. We will download a newer linaro compiler. I used version 7.4.1 of the compiler.
It is possible that newer versions will also work.
We will download this into the tools folder. Let's first change into that directory with the following command:
	
	cd ~/rpi/tools
	
Run the following to download the compiler:

	sudo wget https://releases.linaro.org/components/toolchain/binaries/7.4-2019.02/arm-linux-gnueabihf/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
	
Once it is downloaded, we can extract it using the following command:

	tar xfv gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
	
Let's move back into the rpi folder as needed for the next sections:

	cd ~/rpi
	
### 4.3 Sync our sysroot
We now need to sync up our sysroot folder with the system files from the Raspberry Pi. The sysroot folder will then have all the necessary files to run a system, and therefore also compile for that system.
rsync will let us sync any changes we make on the Raspberry Pi with our PC and vice versa as well. It is convenient because if you make changes to your RPi later and want to update the sysroot folder on your host machine, rsync will only copy over the changes, potentially saving a lot of time.

For now, we want to sync (i.e. copy) over all the relevant files from the RPi. To do this, enter the following commands [change *192.168.1.7* with the IP address for your RPi]:

	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/lib sysroot
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/usr/include sysroot/usr
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/usr/lib sysroot/usr
	rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/opt/vc sysroot/opt

*Replace "pi" with your username and 192.168.1.7 with your SSH Address*

Note: Double check after each of the above commands that all the files have been copied. There will be an information message if there were any issues. 
You can run any of those lines again as much as you want if you want to check that all files have been copied. rsync only copies files if any changes have been made.

The `--rsync-path="sudo rsync"` option allows us to access files on the target system (RPi) that may require elevated rights. The rsync configuration changes we made on the RPi itself is what allowed us to use this flag.  

The `--delete` option will delete any files from our host system if they have also been deleted on the RPi. You can probably omit this but I used it as I was troubleshooting library install issues on the RPi.

### 4.4 Fix symbolic links
The files we copied in the previous step still have symbolic links pointing to the file system on the Raspberry Pi. We need to alter this so that they become relative links from the new sysroot directory on the host machine.
We can do this with a downloadable python script. To download it, enter the following:

	wget https://raw.githubusercontent.com/riscv/riscv-poky/master/scripts/sysroot-relativelinks.py
	
Once it is downloaded, you just need to make it executable and run it, using the following commands:

	sudo chmod +x sysroot-relativelinks.py
	./sysroot-relativelinks.py sysroot
	
### 4.5 Install Correct Gcc/G++ Version
By default, ubuntu comes with Gcc version 11 which will not work for this guide as some types are not natively defined in Gcc 11
	
Check for Gcc & g++ version

	gcc -v
	g++ -v

If they are version 11 or above then run the below commands

	sudo apt-get install gcc-10
	sudo apt-get install g++-10

Creating softlinks so that complier can use gcc-10

	cd ~
	cd /bin
	ln -s gcc gcc-10
	ln -s g++ g++-10

### 4.6 Making some changes in the QT source files

 In qt-everywhere-src-5.15.0/qtbase/src/gui/configure.json

	--- a/qtbase/src/gui/configure.json	2020-10-27 03:02:11.000000000 -0500
	+++ b/qtbase/src/gui/configure.json	2021-12-07 11:15:01.768446081 -0600
	@@ -862,7 +862,10 @@
	             "type": "compile",
	             "test": {
	                 "include": [ "EGL/egl.h", "bcm_host.h" ],
	-                "main": "vc_dispmanx_display_open(0);"
	+                "main": [
	+                    "vc_dispmanx_display_open(0);",
	+                    "EGL_DISPMANX_WINDOW_T *eglWindow = new EGL_DISPMANX_WINDOW_T;"
	+                ]
	             },
	             "use": "egl bcm_host"
	         },

In EGL/eglplatform.h Replace all code with [this](https://github.com/xuancong84/public/blob/master/fix-stupidity/qeglfsbrcmintegration.cpp)

### 4.7 Configure Qt Build
Now most of the work we need to set things up has been completed. We can now configure our Qt build.

Let's move into the build directory that we created earlier inside the rpi folder:

	cd ~/rpi/build
	
Now we need to run the configure script to configure our build. This configure script is actually located inside the qt sources directory. We don't want to build within that source directory as it can get messy, so we will access it from
within this build directory. This is the command you need to run to configure the build, including all the necessary options:

	../qt-everywhere-src-5.15.0/configure -release -opengl es2  -eglfs -device linux-rasp-pi4-v3d-g++ -device-option CROSS_COMPILE=~/rpi/tools/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf- -sysroot ~/rpi/sysroot -prefix /usr/local/qt5.15 -extprefix ~/rpi/qt5.15 -opensource -confirm-license -skip qtscript -skip qtwayland -skip qtwebengine -nomake tests -make libs -pkg-config -no-use-gold-linker -v -recheck
	
The configure script may take a few minutes to complete. Once it is completed you should get a summary of what has been configured. Make sure the following options appear:

<pre><code>
QPA backends:
  DirectFB ............................... no
  <b>EGLFS .................................. yes	[SHOULD BE YES]</b>
  EGLFS details:
    EGLFS OpenWFD ........................ no
    EGLFS i.Mx6 .......................... no
    EGLFS i.Mx6 Wayland .................. no
    EGLFS RCAR ........................... no
    <b>EGLFS EGLDevice ...................... yes	[SHOULD BE YES]</b>
    EGLFS GBM ............................ yes
    EGLFS VSP2 ........................... no
    EGLFS Mali ........................... no
    <b>EGLFS Raspberry Pi ................... no	[SHOULD BE NO]</b>
    EGLFS X11 ............................ yes
  LinuxFB ................................ yes
  VNC .................................... yes
</code></pre>  

If the your configuration summary doesn't have the EGLFS features set to what's shown above, something has probably gone wrong. You can look at the config.log file in the build directory to try and diagnose what the issue might be.
If you see an error at the bottom of your config summary along the lines of "EGLFS was enabled but...." that could also be an indication something went wrong. Look through the config.log files to try and diagnose the error.  
You may get a warning about QDoc not being compiled. This can be safely ignored unless you specifically need this.

If you have any issues, before running configure again, delete the current contents with the following command (save a copy of config.log first if you need to):

	rm -rf *

The configuration command I provided above skips QtWebEngine as this seems to have some additional dependacies. If you need it, you can try compiling that module seperately later. I've also skipped Wayland support, but if you need it, you can remove that option from the configure command provided.
I've skipped QtScripts as this library is being depracated and it is probably best to avoid using it.

If all looks good and all libraries you need have been installed we can continue to the next section

### 4.7 Build Qt
Our build has been configured now, and it is time to actually build the source files. Ensure you are still in the build directory, and run the following command:

	make -j4
	
The -j4 option indicates that the job should be spread into 4 threads and run in parallel. I allocated 4 CPU cores to my Ubuntu virtual machine, so I would think the system will make use of that and distribute the workload among the 4 cores.

This process will take some time :coffee:

Once it is completed, we can install the built package using the following command:

	make install
	
This should install the files in the correct directories

### 4.8 Deploy Qt to our Raspberry Pi
We can now deploy Qt to our RPi. We will again make use of the rsync command. First move back into the rpi folder using the following command:

	cd ~/rpi
	
You should now see a new folder named "qt5.15" here. Copy this to the raspberry pi using the following command [replace 192.168.1.7 with your RPi's IP address]:

	rsync -avz --rsync-path="sudo rsync" qt5.15 pi@192.168.1.7:/usr/local


## Step 5: Update linker on Raspberry Pi
(I'm not entirely sure if this step is needed)

Enter the following command to update the device letting the linker to find the new Qt library files:

	echo /usr/local/qt5.15/lib | sudo tee /etc/ld.so.conf.d/qt5.15.conf
	sudo ldconfig

The Qt wiki for installing on a Raspberry Pi 2 suggests the following:

	If you're facing issues with running the example, 
	try to use 00-qt5pi.conf instead of qt5pi.conf, to introduce proper order.

Something to try if you're having issues running your projects.

That should be it! You have now (hopefully) succesfully installed Qt 5.15 on the Raspberry Pi 4B.

In the next step, we will build an example application, just to check everything works.

## Step 6: Build an example application
On the PC, run the following commands to build one of the example OpenGL projects that comes bundled with Qt and deploy it to the Raspberry Pi:

To make a copy of the example project, run the following command:

	cp -r ~/rpi/qt-everywhere-src-5.15.0/qtbase/examples/opengl/qopenglwidget ~/rpi/

Move into that new folder and build the source files with these commands:

	cd ~/rpi/qopenglwidget
	../qt5.15/bin/qmake
	make

Copy the built binary file to the Rasberry Pi with this command [change IP address to your RPi's]:

	scp qopenglwidget pi@192.168.1.7:/home/pi
	
Now **switch to the Raspberry Pi** and navigate to the home directory:

	cd ~
	
Now run the compiled executable that we copied over from the host machine:

	./qopenglwidget
	
The demo should start running on the display connected to the Raspberry Pi.

## Step 6: Windowed vs Full Screen Mode
In previous builds of Qt I've used on a Raspberry Pi, the apps I developed ran on the frame buffer directly, bypassing the X Window Manager. This also meant that the apps always ran full screen. 
I have found that this is not the case with this build of Qt.  

If I boot into the desktop mode and launch the app, it will run in windowed mode. This does have the benefit of being able to VNC into the Raspberry Pi and view what the result looks like.

However, if you want it to bypass the X Window Manager, the only solution I have found so far is to boot the RPi directly into CLI. This way the X Window Manager is not running, and the app runs full screen when launched.

[Pablojr has pointed out](https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4/issues/2) that you can access the app through VNC even when you're in command line mode by adding the `-platform vnc` flag when launching it, like this:

	./qopenglwidget -platform vnc

For this to work, you first need to disable the Raspberry Pi's built in VNC server through `raspi-config`. I did run into issues with applications which use OpenGL components (such as the example above), where the OpenGL graphics fail to render.

## Step 7: Installing Python Bindings(Yet to be verified)
	sudo apt install qml-module-qtquick-controls2
	sudo apt-get install qtbase5-dev qtchooser
	sudo apt-get install qt5-qmake qtbase5-dev-tools