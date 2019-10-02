# Camotics Build Notes 
Build notes for camotics on a raspberry pi 4, 4gb ram, running raspbian "Buster" . To install, download the [Debian binary package](https://github.com/koendv/camotics-raspberrypi/raw/master/camotics_1.2.0_armhf.deb) and install using 
~~~
sudo gdebi camotics_1.2.0_armhf.deb
~~~

## Intro
![screenshot](https://github.com/koendv/camotics-raspberrypi/raw/master/doc/screenshot.png  "Camotics on Raspberry Pi")
CAMotics simulates g-code. CAMotics can be used to see how a 3d printer would print a workpiece. 

## Prerequisites
Install the prerequisites
~~~
sudo apt-get update
~~~
For cbang:
~~~
sudo apt-get -y install scons build-essential libssl-dev libv8-dev git libglu1-mesa-dev libyaml-dev libevent-dev libre2-dev libnode-dev libdxflib-dev
~~~
For Qt5 with mesa:
~~~
sudo apt-get install libgl1-mesa-dev libglu1-mesa-dev mesa-common-dev
~~~
For Qt5:
~~~
apt-get install build-essential libfontconfig1-dev libdbus-1-dev libfreetype6-dev libicu-dev libinput-dev libxkbcommon-dev libsqlite3-dev libssl-dev libpng-dev libjpeg-dev libglib2.0-dev libraspberrypi-dev
~~~
Not needed: qt5-default libqt5websockets5-dev libqt5opengl5-dev 
(We'll build these from source)

## Build Qt5 LTS

Build, following the instructions on [Building Qt 5.12 LTS for Raspberry Pi on Raspbian](https://www.tal.org/tutorials/building-qt-512-raspberry-pi) but with the following differences:

- patch qt sources, else compilation fails:
~~~
diff -rBNu qt-everywhere-src-5.12.5/qtbase/src/plugins/platforms/xcb/gl_integrations/xcb_egl/qxcbeglwindow.cpp qt-everywhere-src-5.12.5.OLD/qtbase/src/plugins/platforms/xcb/gl_integrations/xcb_egl/qxcbeglwindow.cpp
--- qt-everywhere-src-5.12.5/qtbase/src/plugins/platforms/xcb/gl_integrations/xcb_egl/qxcbeglwindow.cpp	2019-09-03 20:52:35.000000000 +0200
+++ qt-everywhere-src-5.12.5.OLD/qtbase/src/plugins/platforms/xcb/gl_integrations/xcb_egl/qxcbeglwindow.cpp	2019-09-30 16:03:17.831619246 +0200
@@ -93,7 +93,7 @@
 {
     QXcbWindow::create();
 
-    m_surface = eglCreateWindowSurface(m_glIntegration->eglDisplay(), m_config, m_window, 0);
+    m_surface = eglCreateWindowSurface(m_glIntegration->eglDisplay(), m_config, (void*)m_window, 0);
 }
 
 QT_END_NAMESPACE
diff -rBNu qt-everywhere-src-5.12.5/qtscript/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h qt-everywhere-src-5.12.5.OLD/qtscript/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h
--- qt-everywhere-src-5.12.5/qtscript/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h	2019-08-23 12:28:19.000000000 +0200
+++ qt-everywhere-src-5.12.5.OLD/qtscript/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h	2019-10-01 15:52:01.966516814 +0200
@@ -367,7 +368,8 @@
 #    define WTF_CPU_ARM_TRADITIONAL 1
 #    define WTF_CPU_ARM_THUMB2 0
 #  else
-#    error "Not supported ARM architecture"
+#    define WTF_CPU_ARM_TRADITIONAL 1
+#    define WTF_CPU_ARM_THUMB2 0
 #  endif
 #elif CPU(ARM_TRADITIONAL) && CPU(ARM_THUMB2) /* Sanity Check */
 #  error "Cannot use both of WTF_CPU_ARM_TRADITIONAL and WTF_CPU_ARM_THUMB2 platforms"
~~~
- In "Configure the Qt build" choose -opengl "desktop"
~~~sh
PKG_CONFIG_LIBDIR=/usr/lib/arm-linux-gnueabihf/pkgconfig:/usr/share/pkgconfig \
../qt-everywhere-src-5.12.5/configure -platform linux-rpi3-g++ \
-v \
-opengl desktop -eglfs \
-no-gtk \
-opensource -confirm-license -release \
-reduce-exports \
-force-pkg-config \
-nomake examples -no-compile-examples \
-skip qtwayland \
-skip qtwebengine \
-no-feature-geoservices_mapboxgl \
-qt-pcre \
-no-pch \
-ssl \
-evdev \
-system-freetype \
-fontconfig \
-glib \
-prefix /usr/local/qt5.12lts \
-qpa eglfs
~~~
Check output for:
~~~
Configure summary:
Build type: linux-rpi3-g++ (arm, CPU features: neon)
Qt Gui:
  OpenGL:
    Desktop OpenGL ....................... yes
~~~
Build and install qt5 lts:
~~~
make 
sudo make install
cd ..
~~~
This installs qt5 in `/usr/local/qt5.12lts`
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
If you already have  a qt5 installation, temporarily move it out of the way:
~~~
cd /usr/include/arm-linux-gnueabihf
mv qt5 qt5.OLD
ln -s /usr/local/qt5.12lts/include/ qt5
cd /usr/lib/arm-linux-gnueabihf/
mv qt5 qt5.OLD
ln -s /usr/local/qt5.12lts/lib/ qt5
~~~

Tell camotics where cbang is and compile:
~~~
export CBANG_HOME=$PWD/cbang
export QT5DIR=/usr/local/qt5.12lts/
export PATH=$QT5DIR/bin:$PATH
git clone https://github.com/CauldronDevelopmentLLC/CAMotics
cd CAMotics
scons
~~~
If scons terminates with the error message
~~~
GL.cpp:(.text+0x240): undefined reference to `QOpenGLFunctions_2_0::versionProfile()'
~~~
take the following action:
In `/usr/local/qt5.12lts/lib/libQt5OpenGL.la` check the dependency libraries for libQt5OpenGL are:
~~~
dependency_libs='-L=/usr/local/qt5.12lts/lib -lQt5Widgets -lQt5Gui -lQt5Core -lpthread'
~~~
It's easiest to just copy-paste  the dependency libs to the command used to link camotics and camsim:
~~~
g++ -o camotics -Wl,--as-needed -Wl,-S -Wl,-x -no-pie -pthread -Wl,-rpath=/usr/local/qt5.12lts//lib build/camotics.o build/qrc_camotics.o -L/home/koen/src/cbang/lib build/libCAMoticsGUI.a build/libclipper.a build/libDXF.a build/libSTL.a build/libGCode.a -lstdc++ -lcairo -lGL -lcbang -lcbang-boost -lv8 -lssl -lcrypto -ldl -lexpat -lbz2 -lz -lpthread -lQt5OpenGL -lQt5Widgets -lQt5Gui -lQt5WebSockets -lQt5Network -lQt5Core build/dxflib/libdxflib.a -L=/usr/local/qt5.12lts/lib -lQt5Widgets -lQt5Gui -lQt5Core -lpthread
g++ -o camsim -Wl,--as-needed -Wl,-S -Wl,-x -no-pie -pthread -Wl,-rpath=/usr/local/qt5.12lts//lib build/camsim.o build/qrc_camotics.o -L/home/koen/src/cbang/lib build/libCAMoticsGUI.a build/libclipper.a build/libDXF.a build/libSTL.a build/libGCode.a -lstdc++ -lcairo -lGL -lcbang -lcbang-boost -lv8 -lssl -lcrypto -ldl -lexpat -lbz2 -lz -lpthread -lQt5OpenGL -lQt5Widgets -lQt5Gui -lQt5WebSockets -lQt5Network -lQt5Core build/dxflib/libdxflib.a -L=/usr/local/qt5.12lts/lib -lQt5Widgets -lQt5Gui -lQt5Core -lpthread
~~~
Everything else ought to build cleanly.
## Create debian package.
Before creating the debian package, patch a small typo in SConstruct:
~~~
--- SConstruct	2019-10-02 08:25:36.962179211 +0200
+++ SConstruct.ORIG	2019-10-02 08:25:25.312341100 +0200
@@ -369,7 +369,7 @@
         deb_section = 'miscellaneous',
         deb_depends =
         'debconf | debconf-2.0, libc6, libglu1, libv8-3.14.5 | libv8-dev, ' +
-        'libglu1-mesa, libssl1.1' + qt_pkgs,
+        'libglu1-mesa libssl1.1' + qt_pkgs,
         deb_priority = 'optional',
         deb_replaces = 'openscam',
~~~
and type:
~~~
scons package
~~~
To complete the debian package, we'll add the Qt libraries we've just built:
~~~
cd build/camotics-deb/
( tar cvhf - /usr/local/qt5.12lts/lib/libQt5Widgets.so.5 /usr/local/qt5.12lts/lib/libQt5Gui.so.5 /usr/local/qt5.12lts/lib/libQt5WebSockets.so.5 /usr/local/qt5.12lts/lib/libQt5Network.so.5 /usr/local/qt5.12lts/lib/libQt5Core.so.5 /usr/local/qt5.12lts/plugins/ /usr/local/qt5.12lts/plugins/ | tar xvf - )
~~~
Note the 'h' option in the tar. 
Copy the build notes (this document) to build/camotics-deb/usr/share/doc/camotics/BUILD_NOTES.md
Re-build the .deb package, this time with the Qt5 libraries added. Go back to the top level of the camotics source:
~~~
cd ../..
ls build/camotics-deb
fakeroot dpkg-deb -b build/camotics-deb .
~~~
This creates the debian package `camotics_1.2.0_armhf.deb`. Don't forget to put the qt5 includes and libs back:
~~~
cd /usr/include/arm-linux-gnueabihf
ls -l qt5
rm qt5
mv qt5.OLD qt5
cd /usr/lib/arm-linux-gnueabihf/
ls -l qt5
rm qt5
mv qt5.OLD qt5
~~~
This completes building the camotics package for Raspbian.
## Install debian package and its dependencies
To install:
~~~
sudo gdebi camotics_1.2.0_armhf.deb
~~~
To remove:
~~~
sudo dpkg -r camotics
~~~

This file installed as `/usr/share/doc/camotics/BUILD_NOTES.md`

not truncated.
