# __A cross-compilation guide for Qt5.15.12 for Raspberry Pi 4 running Raspberry Pi OS Bullseye__
This page documents an easy to follow guide to cross-compile the latest version of Qt5 (Qt5.15.12) for a Raspberry Pi 4 running Bullseye.  
I have been using Raspberry Pi OS Buster for quite some times, and it was easy to get Qt5 running on it thanks to the plethora of guides available. After I moved to Bullseye, I quickly found out that most of the guides do not work anymore because some majors changes between the two versions. After days of experimentation, building, erasing, rebuilding ... I finally got everything right. And I decided to put all the steps here, to save time for anyone else interested in building Qt5 for a Raspberry Pi 4 running Bullseye.

## __Setup__
### __Hardware__
Host: Ryzen 5 4000 + 16 GB RAM + RTX 3060
Target: Raspberry Pi 4 Model B.

### __Software__
Host: Ubuntu 20.04 LTS 64 bit.
Target: Raspberry Pi OS Bullseye 32 bit.
Cross compiler: gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf.

<u>__Notes__</u>:  
- A Ubuntu virtual machine can be used if you don't have it installed.
- The process requires at least 10 GB of disk space
- Make sure thatt your Raspberry Pi is on the same network as your host PC and that it has internet access.
- This guide is only for
Raspberry Pi4. It can be used for the Raspberry Pi 3 Model B (change the -device argument to linux-rasp-pi3-vc4-g++ in the configure step) though I did not tested that. Same for Raspberry Zero 2W. If you succeed in building for those, let me know so I can update the guide.  
I personally used the 32 bit desktop version of Bullseye, and I didn't have the time to test it on the 64 bit version yet. If you succeed to build for the other versions, please let me know so that I can update this guide.


## __Acknowledgements__
This guide is highly inspired by the one published by @UvinduW for [cross-compiling Qt5 for Raspberry 4 running Buster](https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4). It has been my reference guide for cross-compiling Qt for all Raspberry Pi variant under Buster.

## __1 Flash an SD Card with the Raspberry Pi OS image__
If you already have a Bullseye image running, you can skip this section.  

The Raspberry Pi Foundation has made the flash process very easy bu providing the [Raspberry Pi Imager](https://www.raspberrypi.com/software/). It allows setting up the user credentials, SSH, and WiFI configuration at the same time.
Follow the link to grab the latest version and install it.  
- Use [this link](https://downloads.raspberrypi.com/raspios_oldstable_armhf/images/raspios_oldstable_armhf-2023-12-06/2023-12-05-raspios-bullseye-armhf.img.xz?_gl=1*18pubvb*_ga*MTI0NzA5MzgzNC4xNjkxMzI5Mjkz*_ga_22FD70LWDS*MTcwNjAwNjQwMi4zNC4xLjE3MDYwMTE1MTMuMC4wLjA.) to download the 32 bit desktop version.
- Run the Raspberry Pi Imager, and click __CHOOSE OS -> Use custom__ to pick the downloaded file.
- Choose your device, and click Next.
- Click __EDIT SETINGS__ customize the installation (WiFI, SSH).
  - Use the **_General_** tab to set a hostname, a username and password, Configure the WiFI, ...
  -  Under the **_Services_** tab, check the **Enable SSH** box.
  - Click __SAVE__ to save the configs.
- Click __YES__ to continue, then confirm the flash operation and wait for the process to complete.

![Raspberry Pi Imager image](./raspi-imager_custom.png)

The Imager can also download automatically the OS itself before flashing it, but I prefer the manual method, as it is faster.

## __2 Configure the Raspberry Pi__
### __2.1 Enable Development Sources__
Power up your Raspberry Pi and use SSH to log into it from your Host. If you have a display and keyboard connected to your Pi, you can also run the following commands directly on it.

The build process requires some packages to be downloaded from development sources. You first need to enable development sources in the source list. Enter the following command into a terminal.  
```
sudo nano /etc/apt/sources.list
```
That will open the file inside a nano editor. Locate the following line and uncomment it by removing the # character (add it if it does not exist):
```
deb-src http://raspbian.raspberrypi.org/raspbian/ bullseye main contrib non-free rpi
```
Now, press `Ctrl+S` to save the file, and `Ctrl+X` to quit.

### __2.2 Update the system__
Run the following commands to update the system and reboot.
``` 
    sudo apt-get update
    sudo apt-get dist-upgrade
    sudo reboot
```

### __2.3 Enable rsync with elevated rights__
Later in this guide, we will be using `rsync` to sync files between the PC and Pi. Some of these files required root rights.
Here, we will configure the Pi to grand this right to the `rsync` command.  
First, use the following command to find the path to rsync:
```
which rsync
```
On my Pi (and most probably on yours), this command outputs:
```
/usr/bin/rsync
```
Now, open the sudoers file with:
```
sudo visudo
```
The file should be opened with the nano editor. At the end of the file, add an entry with the following structure:
```
 <username> ALL=NOPASSWD:<path to rsync>
 ```
 In my case, it was:
 ```
 pi ALL=NOPASSWD:/usr/bin/rsync
 ```

 rsync is now granted root rights.
### __2.4 Install the development packages__
Run the following commands to install the packages required to build Qt.
```
sudo apt-get build-dep qt5-qmake
sudo apt-get build-dep libqt5gui5
sudo apt-get build-dep libqt5webengine-data
sudo apt-get build-dep libqt5webkit5
sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 gdbserver
sudo apt-get install libgles2-mesa-dev libgbm-dev libdrm-dev

```
At this stage, you might be prefer making a backup of your SD card so that you could revert back if something wrong happen later on. In my case I used 'Win32DiskImager' for this purpose (Yeah, I have Windows in dual-boot 😉).

If you want to include Qt Multimedia or Bluetooth in the build, you can run the following commands to install the requires packages (I did, and everything went well).
```
sudo apt-get install gstreamer1.0-plugins*
sudo apt-get install libgstreamer1.0-dev  libgstreamer-plugins-base1.0-dev libopenal-data libsndio7.0 libopenal1 libopenal-dev pulseaudio
sudo apt-get install bluez-tools
sudo apt-get install libbluetooth-dev
```
### __2.5 Create a directory for the Qt install__



