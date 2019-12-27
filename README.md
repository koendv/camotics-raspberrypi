# Camotics Build Notes 
Build notes for camotics on a raspberry pi 4, 4gb ram, running raspbian "Buster" . To run, download the [AppImage](https://github.com/koendv/camotics-raspberrypi/releases) and install using 
~~~
cd ~/Downloads
chmod +x CAMotics-armhf.AppImage
./CAMotics-armhf.AppImage
~~~

## Intro
![screenshot](https://github.com/koendv/camotics-raspberrypi/raw/master/doc/screenshot.png  "Camotics on Raspberry Pi")
CAMotics simulates g-code. CAMotics can be used to see how a 3d printer would print a workpiece. 

## Prerequisites

Build on a clean  2019-09-26-raspbian-buster-lite. This avoids having multiple libQt versions during compilation.

Install the prerequisites
~~~
sudo apt-get update
~~~

Install Qt5.12 LTS with OpenGL:
~~~
wget https://github.com/koendv/qt5-opengl-raspberrypi/releases/download/v5.12.5-1/qt5-opengl-dev_5.12.5_armhf.deb
sudo apt install ./qt5-opengl-dev_5.12.5_armhf.deb
~~~

For cbang:
~~~
apt-get install scons build-essential libssl-dev binutils-dev libiberty-dev libmariadb-dev-compat libleveldb-dev libsnappy-dev git
apt-get install libbz2-dev libyaml-dev libevent-dev libre2-dev 
~~~
Install libnode-dev to pull in libv8-dev and libplatform.h:
~~~
apt-get install libnode-dev 
~~~
For camotics:
~~~
apt-get install python-six libdxflib-dev libglu1-mesa-dev
~~~
## Build cbang
~~~
git clone https://github.com/CauldronDevelopmentLLC/cbang
export TARGET_ARCH=arm-linux-gnueabihf
scons -C cbang
~~~
Check scons will compile cbang with v8, not ChakraCore:
~~~
Checking for C++ header file ChakraCore.h... no
Need C++ header ChakraCore.h(cached) error: no result
Checking for C++ header file v8.h... yes
Checking for C++ header file libplatform/libplatform.h... yes
Checking for C library v8... yes
~~~

## Build camotics
Tell camotics where cbang is and compile:
~~~
export CBANG_HOME=$PWD/cbang
export QT5DIR=/usr/lib/qt5.12/
export PATH=$QT5DIR/bin:$PATH
export PKG_CONFIG_PATH=/usr/lib/qt5.12/lib/pkgconfig/
git clone https://github.com/CauldronDevelopmentLLC/CAMotics
cd CAMotics
scons
~~~

## Create AppImage
Type:
~~~
rm -rf build/camotics-deb/
scons package
~~~
This creates the debian package `camotics_1.2.1_armhf.deb`.
To continue creating the AppImage, first remove the Debian package info:
~~~
cd build/camotics-deb/
export APPIMAGE_DIR=$PWD
rm -rf DEBIAN/
~~~
Add desktop icon information:
~~~
cp usr/share/applications/CAMotics.desktop .
cp usr/share/pixmaps/camotics.png $APPIMAGE_DIR
cat <<EOD  >> CAMotics.desktop
Version=1.0
X-AppImage-Version=1.2.1
EOD
~~~

Copy Qt translations:
~~~
(cd /; tar cvhf - usr/lib/qt5.12/translations/) | (cd $APPIMAGE_DIR; tar xvpf -)
~~~

Copy library dependencies to AppImage. First make a list of all shared libraries used, then copy these libraries to the AppImage directory.
```
cd $APPIMAGE_DIR/..
wget https://raw.githubusercontent.com/koendv/camotics-raspberrypi/master/appimagelibs-buster
chmod +x ./appimagelibs-buster 
./appimagelibs-buster $APPIMAGE_DIR
```
Copy AppImage files. From `https://github.com/AppImage/AppImageKit/releases/` download `AppRun-armhf` and `appimagetool-armhf.AppImage`.
```
cp ~/Downloads/AppRun-armhf $APPIMAGE_DIR/AppRun
chmod a+x $APPIMAGE_DIR/AppRun
```
Copy AppStream metadata.
```
mkdir -p $APPIMAGE_DIR/usr/share/metainfo/
cp ../CAMotics.appdata.xml $APPIMAGE_DIR/usr/share/metainfo/CAMotics.appdata.xml
```
Create AppImage:
```
cd $APPIMAGE_DIR/..
~/Downloads/appimagetool-armhf.AppImage $APPIMAGE_DIR
```
This produces the file ```CAMotics-armhf.AppImage```.
Test AppImage:
```
./CAMotics-armhf.AppImage
```
Run the AppImage on a clean install of the operating system to check all dependencies have been caught.

not truncated.
