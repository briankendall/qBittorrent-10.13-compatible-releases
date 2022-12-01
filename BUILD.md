For those of us still running macOS 10.13, here are instructions on how to make 10.13 compatible builds of qBittorrent 4.3.x-4.4.0 (and possibly later versions as well).

While these instructions may require modifications for future qBitorrent releases, they should generally continue to work so long as qBittorrent is still compatible with Qt 5.15.2.

These steps have been tailored for and tested on a clean installation of macOS 10.13.

Note that this does not take any extra steps to build a universal x86_64 / arm64 version of qBittorrent, as arm64 is of course not at all supported in macOS 10.13 anyway.

## Setting up the build environment:

1. Install Command Line Tools (macOS 10.13) for Xcode 10.1, and Xcode 10.1. Both can be obtained from here:    
[https://developer.apple.com/download/more/](https://developer.apple.com/download/more/)    
Note that this page requires signing in with an Apple ID, but not an Apple developer account.

2. Launch Xcode and agree to the licensing agreement, and wait for it to install components

3. Install MacPorts: [https://www.macports.org/install.php](https://www.macports.org/install.php)    
Note: Previous versions of these instructions suggested using Homebrew, and you could if you really prefer it, but a) I find MacPorts to overall be a much better, safer, and more reliable package manager, and b) Homebrew has long since dropped support for earlier macOS versions, while MacPorts remains committed to maintaining support for older macOS releases.

4. Install llvm + clang. As of the time of writing this, the latest version is 15.0.5:    
`sudo port install clang-15`

5. Install cmake:    
    `sudo port install cmake`    

6. Create an Xcode toolchain for LLVM:
    1. `mkdir -p ~/Library/Developer/Toolchains/llvm15.xctoolchain`
    2. `cd ~/Library/Developer/Toolchains/llvm15.xctoolchain`
    3. `echo -e '<?xml version="1.0" encoding="UTF-8"?>\n<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">\n<plist version="1.0">\n<dict>\n\t<key>CFBundleIdentifier</key>\n\t<string>org.llvm.15.0.5</string>\n\t<key>CompatibilityVersion</key>\n\t<integer>2</integer>\n</dict>\n</plist>' > Info.plist`
    4. `ln -s usr /opt/local/libexec/llvm-15`

7. Make llvm's toolchain the default in the terminal:    
    1. `export TOOLCHAINS=llvm`    
    2. `echo "export TOOLCHAINS=llvm" >> ~/.bash_profile`

8. Make llvm's toolchain the default for Xcode
    1. Open Xcode
    2. Open its preferences
    3. Click Components
    4. Click Toolchains
    5. Select org.llvm.15.0.5
    6. Quit Xcode
    
    This may not actually be necessary, but I did it anyway just to be safe.

To double check everything is working, try running the following commands, and make sure their output mentions to correct version of llvm and references the correct paths:

```
$ clang++ --version
clang version 15.0.5
Target: x86_64-apple-darwin17.7.0
Thread model: posix
InstalledDir: /Users/moof/Library/Developer/Toolchains/llvm15.xctoolchain/usr/bin

$ c++ --version
clang version 15.0.5
Target: x86_64-apple-darwin17.7.0
Thread model: posix
InstalledDir: /Users/moof/Library/Developer/Toolchains/llvm15.xctoolchain/usr/bin

$ xcodebuild -find-executable clang++
/Users/moof/Library/Developer/Toolchains/llvm15.xctoolchain/usr/bin/clang++
```

## Modified macOS build instructions 

Now we set up qbittorrent compilation following a modified version of the steps here:
https://github.com/qbittorrent/qBittorrent/wiki/Compilation:-macOS

To keep things simpler and complete, I'm going to copy all of the instructions from that qBittorrent wiki page and make modifications as needed.

### Conventions used in this document

The following directory tree structure will be used through the document:

```text
$HOME/tmp/qbt
 |_ src         <-- directory for all sources
 |_ ext         <-- directory for all compiled dependencies
```

It is assumed that all listed commands should be issued from `$HOME/tmp/qbt/src` unless otherwise specified.

### Build environment setup

Follow the instructions above.

### Required sources

qBittorrent has few required dependencies, and they also have their own. The full list of required sources (including qBittorrent itself) is next:

- OpenSSL 1.1.x
- Qt 5.15.x
- Boost 1.60 or above, 1.70 or above is recommended
- libtorrent (libtorrent-rasterbar) 1.2.x, 1.2.6 or above is recommended
- qBittorrent 4.3.3 or above

Actually, some additional libraries (like zlib) are required, but they are available on each system and have stable binary interfaces, so there is no reason to build them from sources. Moreover, such libraries are part of the SDK.

### Downloading sources

Download and build instructions was split due to some macOS specific stuff, see "[Building sources](#building-sources)" section for details.

### Downloading OpenSSL

OpenSSL is available for download from [the official website](https://www.openssl.org/source/), just download archive for 1.1.x series. At the moment of writing it was 1.1.1s.

Place downloaded archive to `$HOME/tmp/qbt/src` and extract it here:

```sh
tar xf openssl-1.1.1s.tar.gz
```

If you prefer using git, you can also clone the openssl git repo and checkout the latest release by tag.

### Downloading Qt

While git can be used to download Qt, I personally recommend just getting the complete .tar.xz archive of the latest Qt 5.15.x from here: [https://download.qt.io/official_releases/qt/5.15/](https://download.qt.io/official_releases/qt/5.15/)    
...and extract it in `~/tmp/qbt/src`. As of the time of this writing, the latest version is 5.15.7, so I'll use that version number throughout these instructions. But if a newer version is available, use that one, and replace any instance of 5.15.7 with the version number you downloaded.

You made need a separate tool to extract the xz file.

You should now have a folder named `qt-everywhere-src-5.15.7` (replacing the x with whichever version number you actually downloaded).

However, Qt 5.15.7 will not compile properly on macOS 10.13. To fix this, copy the following text exactly:

```
diff --git a/src/corelib/kernel/qcore_mac.mm b/src/corelib/kernel/qcore_mac.mm
index 33c64bc474..a0fe49b071 100644
--- a/src/corelib/kernel/qcore_mac.mm
+++ b/src/corelib/kernel/qcore_mac.mm
@@ -262,6 +262,7 @@ QMacAutoReleasePool::QMacAutoReleasePool()
 
 #ifdef QT_DEBUG
     void *poolFrame = nullptr;
+#if QT_DARWIN_PLATFORM_SDK_EQUAL_OR_ABOVE(__MAC_10_14, __IPHONE_12_0, __TVOS_12_0, __WATCHOS_5_0)
     if (__builtin_available(macOS 10.14, iOS 12.0, tvOS 12.0, watchOS 5.0, *)) {
         void *frame;
         if (backtrace_from_fp(__builtin_frame_address(0), &frame, 1))
@@ -272,6 +273,12 @@ QMacAutoReleasePool::QMacAutoReleasePool()
         if (backtrace(callstack, maxFrames) == maxFrames)
             poolFrame = callstack[maxFrames - 1];
     }
+#else
+    static const int maxFrames = 3;
+    void *callstack[maxFrames];
+    if (backtrace(callstack, maxFrames) == maxFrames)
+        poolFrame = callstack[maxFrames - 1];
+#endif
 
     if (poolFrame) {
         Dl_info info;
```
...to `$HOME/tmp/qbt/src/qtbase_10.13.patch`

Then apply the patch:

```sh
cd ~/tmp/qbt/src/qt-everywhere-src-5.15.7/qtbase
git apply $HOME/tmp/qbt/src/qtbase_10.13.patch
```


### Downloading Boost

Boost is available for download from [official website][boost-site]. Latest available release is a good choice in most cases, so pick it firstly. If libtorrent build fails, download previous version. Preferable archive format is `.tar.bz2` or `.tar.gz`. At the moment of writing latest version was 1.80.0.

Place downloaded archive to `$HOME/tmp/qbt/src` (see [conventions](#conventions-used-in-this-document)) and extract it here:

```sh
tar xf boost_1_80_0.tar.bz2
```

### Downloading libtorrent

libtorrent RC_1_2 branch is used in this guide, but any 1.2.x release is also suitable. 1.2.6 and above is recommended.

First clone the [official repo][libtorrent-repo]. RC_2_0 branch was default at the time of writing, so check out RC_1_2 after cloning:

```sh
git clone --recurse-submodules https://github.com/arvidn/libtorrent.git
git checkout RC_1_2
```

It is also possible to download the release archive or git snapshot instead of cloning the repo.

### Downloading qBittorrent

While the original instructions say to use the qBittorrent master branch, here we will check out a specific release. As of this writing, version 4.3.3 is the latest and we'll use that. In theory later releases will work to so long as they're still using Qt 5.15 and not Qt 6 (which will not support macOS 10.13).

```sh
git clone https://github.com/qbittorrent/qBittorrent.git
cd qBittorrent
git checkout release-4.3.3
```

There is one modification necessary, which is to set qBittorrent's deployment target to macOS 10.13. To fix this, edit the file `macxconf.pri` and change the `QMAKE_MACOSX_DEPLOYMENT_TARGET` variable to `10.13`. The full line should read:    
`QMAKE_MACOSX_DEPLOYMENT_TARGET = 10.13`


### Building OpenSSL

OpenSSL in Qt' dependency, so build it first. In this guide it is built as shared library.

At the time of writing OpenSSL 1.1.1m was the latest, so source directory is `openssl-1.1.1m`. Go to it

```sh
cd openssl-1.1.1m
```

and issue configuration command:

```sh
./config no-comp no-deprecated no-dynamic-engine no-tests no-zlib --openssldir=/etc/ssl --prefix="$HOME/tmp/qbt/ext" -mmacosx-version-min=10.13
```

Please note `-mmacosx-version-min` option at the end of line, this is NOT some OpenSSL configure script option, this option is passed directly to compiler, it is important to place it last. Its value must be the value that was found in Qt sources.

To start build process just run `make`:

```sh
make -j$(sysctl -n hw.ncpu)
```

Build process takes few minutes (3-5), when it finishes, install everything compiled:

```sh
make install_sw
```

Please note '_sw' suffix, it used just to install a subset of available stuff, this is sufficient for successful build.

### Building Qt

The preferred way to build Qt is 'out of tree build'. So create separate build directory at the same level as source directory and go into it:

```sh
mkdir build-qt-5.15.7 && cd build-qt-5.15.7
```

Issue Qt configuration command:

```sh
../qt-everywhere-src-5.15.7/configure -prefix "$HOME/tmp/qbt/ext" -opensource -confirm-license -release -appstore-compliant -c++std c++17 -no-pch -I "$HOME/tmp/qbt/ext/include" -L "$HOME/tmp/qbt/ext/lib" -make libs -no-compile-examples -no-dbus -no-icu -qt-pcre -system-zlib -ssl -openssl-linked -no-cups -qt-libpng -qt-libjpeg -no-feature-testlib -skip qtwebengine -skip qtlocation
```

This configures Qt as shared library (.framework in case of macOS) with no any debug info included.

Note `-I` and `-L` options in that line, they are required to allow Qt' build system to find OpenSSL (it is an optional dependency, but it is required in case of qBittorrent). If you change paths used in this guide to your own, please make sure that these options have correct values, also please set `-prefix` value to corresponding path too.

Also, the `-c++std c++17` argument has been changed from the original build instructions.

The rest of options mostly to minimize the scope of building stuff and decrease build time.

Configuration process takes a few minutes, some required tools are build during it.

When configuration process is finished, build can be started:

```sh
make -j$(sysctl -n hw.ncpu)
```

This step is most time consuming of all that guide, build process takes about 30 minutes.

When it finishes, just install Qt as any other library:

```sh
make install
```

### Building Boost

Actually no Boost binaries are required, libtorrent requires Boost.System and it became header-only since Boost 1.70 (or even 1.69), boost_system library is just a stub left for compatibility, but a lot of tools/scripts rely on it when detecting the presence of Boost. Moreover, there is no option to build header-only version of Boost. Of course, 'stage' version can be used without any building, but it is not suitable for usage with cmake - it doesn't have files allowing cmake to detect it. Such files are generated only during installation process. So build only boost_system library regardless this is just a stub and will not be used:

```sh
cd boost_1_75_0
./bootstrap.sh
./b2 --prefix="$HOME/tmp/qbt/ext" --with-system variant=release link=static cxxflags="-std=c++17 -mmacosx-version-min=10.13" install
```

This produces static version of boost_system library with no debug information. Custom `cxxflags` are left for historical reasons, when Boost.System was not header-only library and apps using it must link boost_system library.

For 10.13 builds, I've changed the `-std=c++14` to `-std=c++17` just to be safe, though this may not be necessary since it's header-only.

### Building libtorrent

libtorrent provides few build systems to choose from to build it. Unfortunately, convenient and easy to use GNU autotools option is not available on macOS, so let's use cmake.

cmake supports build in separate directory, usually subdirectory is used for it, so create such directory and run cmake in it:

```sh
cd libtorrent
mkdir build && cd build
```

Configure libtorrent as static library with all other options set to default:

```sh
cmake -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_CXX_VISIBILITY_PRESET=hidden -DCMAKE_C_VISIBILITY_PRESET=hidden -DCMAKE_PREFIX_PATH="$HOME/tmp/qbt/ext" -DCMAKE_CXX_STANDARD=17 -DCMAKE_CXX_EXTENSIONS=OFF -DCMAKE_OSX_DEPLOYMENT_TARGET=10.13 -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX="$HOME/tmp/qbt/ext" ..
```

Few important things to note in this line are C++ standard version and minimum supported macOS version (`CMAKE_CXX_STANDARD` and `CMAKE_OSX_DEPLOYMENT_TARGET` options), this is required for successful linkage. When compiling qBittorrent 4.3.3 or later for macOS 10.13, they must be `-DCMAKE_CXX_STANDARD=17` and `CMAKE_OSX_DEPLOYMENT_TARGET=10.13` respectively. Also we need to specify precisely which C and C++ compiler we're going to use, forcing cmake to use the version of LLVM we installed with MacPorts rather than the antiquated version included with Xcode. The visibility settings help prevent a compiler warning later on when compiling qBittorrent.

Value of `CMAKE_PREFIX_PATH` is also important, it tells cmake where to find any required dependency. So adjust it in case of using custom paths.

cmake just performs configuration steps and generates platform-specific makefile, so build and install it as usual (remember, you are still in 'build' directory):

```sh
make -j$(sysctl -n hw.ncpu)
make install
```

Build process takes about 5-7 minutes.

### Building qBittorrent

Like libtorrent, qBittorrent also provides few build systems to choose from to build it. GNU autotools is not available on macOS, qmake requires some additional work to setup project, so let's use cmake again.

(Note: the original build instructions mentioned it being necessary to make some adjustments to the source code in order to successfully build with cmake, but as of qBittorrent 4.3.3 that is no longer necessary.)

As with libttorrent, let's create a separate build directory:

```sh
mkdir build && cd build
```

Now everything is ready to issue cmake (from build directory).

```sh
cmake -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_CXX_VISIBILITY_PRESET=hidden -DCMAKE_C_VISIBILITY_PRESET=hidden -DCMAKE_CXX_FLAGS="-fvisibility-inlines-hidden" -DCMAKE_EXE_LINKER_FLAGS="-nostdlib /opt/local/libexec/llvm-15/lib/libc++.a /opt/local/libexec/llvm-15/lib/libc++abi.a /usr/lib/libSystem.dylib" -DCMAKE_PREFIX_PATH="$HOME/tmp/qbt/ext" -DCMAKE_CXX_STANDARD=17 -DCMAKE_CXX_EXTENSIONS=OFF -DCMAKE_OSX_DEPLOYMENT_TARGET=10.13 -DCMAKE_BUILD_TYPE=Release ..
```

Once again we must tell it to use our custom LLVM installation and not Xcode's. And we must explicitly link to LLVM's static runtime libraries as otherwise it won't link due to using C++ features not available in the system runtime. We need to make sure visibility is hidden, including for inlines, otherwise we'll get a "direct access in function XXX to global weak symbol YYY means the weak symbol cannot be overridden at runtime" warning message. Other than that, the important option to note - `CMAKE_PREFIX_PATH`, tells cmake where to find dependencies.

Now run build as usual:

```sh
make -j$(sysctl -n hw.ncpu)
```

Build process takes about 10 minutes, and as result you will get `qbittorrent.app` in the build directory (where cmake and make were run). But this app bundle is incomplete - Qt runtime must be integrated into it. This can be achieved with special Qt tool called `macdeployqt`, it even can produce dmg image ready for redistribute.

```sh
$HOME/tmp/qbt/ext/bin/macdeployqt qbittorrent.app -dmg
```

Now `qbittorrent.app` bundle is ready to use. Also `qbittorrent.dmg` is created (with the same app inside). Drop `-dmg` option if don't want it.

### Final notes

On one of the 10.13 systems I used to build qBittorrent, the compiled app didn't run because it was missing llvm's C++ runtime dynamic libraries: `libc++.1.dylib` and `libc++abi.1.dylib`. If qBittorrent fails to run because either of these are missing, then copy them into `qbittorrent.app/Contents/Frameworks/`.

However, when I tested these instructions on a clean 10.13 system, I found that this was not necessary. I'm not sure why it was required on one 10.13 system and not another!