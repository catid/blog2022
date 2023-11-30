+++
title = "Intel oneAPI Video Processing Library on Linux"
date = "2023-11-30"
description = "How to use Intel VPU hardware"
+++

## Why do we want oneVPL?

If your goal is to encode or decode video on a modern Intel Linux computer, and you have no AMD/Nvidia graphics card plugged in, then you'll be looking at oneVPL.  This is Intel's newest API for video processing, deprecating the libmfx and Media SDK APIs for Intel Tiger Lake (2020) and newer processors.

So yeah it's a little niche.  But sometimes it's the best way to do some heavy lifting to transcode video on a tiny PC.  Maybe if you have a cluster of Intel servers it's the best way to do transcode in the cloud, rather than depending on expensive GPU instances.

## Pre-requisites

I'm going to explore getting this running on Ubuntu Server 23.10.  I have no monitor plugged in and X is not running, so my output may be slightly different from yours.

Check if you have an Intel GPU, which should show up as an Integrated Graphics Controller and a `/dev/dri/card0` device:

```bash
catid@devnuc:~$ lspci | grep -i VGA
00:02.0 VGA compatible controller: Intel Corporation Alder Lake-P Integrated Graphics Controller (rev 0c)
catid@devnuc:~$ ls /dev/dri
by-path  card0  renderD128
```

Follow the installation instructions found at https://github.com/intel/compute-runtime/releases
This involves downloading a list of `.deb` files and installing them with `dpkg -i *.deb`

Install the `intel_gpu_top`, `vainfo`, `clinfo` application Ubuntu packages:

```bash
sudo apt install vainfo intel-gpu-tools clinfo
```

Add current user to the `video` and `render` groups:

```bash
sudo usermod -a -G render $USER
sudo usermod -a -G video $USER

# Reboot the server
sudo reboot now
```

Now you can run `clinfo` to verify that the drivers are working.  At this point you should be able to use OpenVINO and other OpenCL acceleration.

```bash
catid@devnuc:~$ clinfo
Number of platforms                               1
  Platform Name                                   Intel(R) OpenCL Graphics
  Platform Vendor                                 Intel(R) Corporation
  Platform Version                                OpenCL 3.0
...
```


## Build latest Intel Media Driver software

Next let's get `VAAPI` working, which is the driver used by VPL:

Install the `gmmlib` dependency from source:

```bash
cd ~
git clone https://github.com/intel/gmmlib.git
cd gmmlib
git checkout intel-gmmlib-22.3.12
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j16
sudo make install
```

Install the `libva` dependency from source:

```bash
sudo apt install meson

cd ~
git clone https://github.com/intel/libva/
cd libva
git checkout 2.20.0
mkdir build
cd build
meson .. -Dprefix=/usr -Dlibdir=/usr/lib/x86_64-linux-gnu
ninja
sudo ninja install
```

```bash
cd ~
git clone https://github.com/intel/media-driver.git
cd media-driver
git checkout intel-media-23.3.5
cd ~
mkdir build_media
cd build_media
cmake ../media-driver
make -j16
sudo make install
```

Now you should be able to run `vainfo` and see some results:

```bash
catid@devnuc:~$ vainfo
error: can't connect to X server!
libva info: VA-API version 1.20.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_20
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.20 (libva 2.12.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 23.3.5 (0f3697942)
vainfo: Supported profile and entrypoints
      VAProfileNone                   : VAEntrypointVideoProc
      VAProfileNone                   : VAEntrypointStats
...
```

## Build latest Intel Media SDK software

```bash
cd ~
git clone https://github.com/Intel-Media-SDK/MediaSDK/
cd MediaSDK
git checkout intel-mediasdk-22.5.4
mkdir build
cd build
cmake ..
```

I had to edit one file to get it to build:

```bash
catid@devnuc:~/sources/MediaSDK$ git diff
diff --git a/api/mfx_dispatch/linux/mfxparser.cpp b/api/mfx_dispatch/linux/mfxparser.cpp
index 9d3823ec..467c773a 100644
--- a/api/mfx_dispatch/linux/mfxparser.cpp
+++ b/api/mfx_dispatch/linux/mfxparser.cpp
@@ -22,6 +22,7 @@
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
+#include <stdint.h>
 
 #include <list>
```

```bash
make -j16
sudo make install
```


## Build latest VPL software

Build and install the oneVPL backend:

```bash
sudo apt install git-lfs
git lfs install

cd ~
git clone https://github.com/oneapi-src/oneVPL-intel-gpu onevpl-gpu
cd onevpl-gpu
git checkout intel-onevpl-23.3.4
mkdir build
cd build
cmake ..
make -j16
sudo make install
```

Build and install libvpl and examples:

```bash
cd ~
git clone https://github.com/intel/libvpl.git
cd libvpl
sudo script/bootstrap
```

I modified one script to fix the build to disable warnings-as-errors:

```
catid@devnuc:~/sources/libvpl/examples/api2x/hello-decode/build$ git diff
diff --git a/script/build b/script/build
index a09d344..01fe882 100755
--- a/script/build
+++ b/script/build
@@ -27,7 +27,7 @@ cmake -B "$build_dir" -S "$source_dir" \
       -DENABLE_VA=ON \
       -DENABLE_WAYLAND=ON \
       -DENABLE_X11=ON \
-      -DENABLE_WARNING_AS_ERROR=ON
+      -DENABLE_WARNING_AS_ERROR=OFF

 cmake --build "$build_dir" --verbose      
```

```bash
script/build
sudo script/install
```

Then you can run `vpl-inspect` to verify that VPL API is working:

```bash
catid@devnuc:~$ vpl-inspect

Implementation #0: mfxhw64
  Library path: /opt/intel/mediasdk/lib/libmfxhw64.so.1.35
  AccelerationMode: MFX_ACCEL_MODE_VIA_VAAPI
  ApiVersion: 1.35
  Impl: MFX_IMPL_TYPE_HARDWARE
  VendorImplID: 0x0000
  ImplName: mfxhw64
  License: 
  Version: 1.2
  Keywords: MSDK,x64
  VendorID: 0x8086
  mfxAccelerationModeDescription:
    Version: 1.0
    Mode: MFX_ACCEL_MODE_VIA_VAAPI
  mfxPoolPolicyDescription:
    Version: 1.0
  mfxDeviceDescription:
    MediaAdapterType: MFX_MEDIA_UNKNOWN
    DeviceID: 46a6/0
    Version: 1.1
  mfxDecoderDescription:
    Version: 0.0
  mfxEncoderDescription:
    Version: 0.0
  mfxVPPDescription:
    Version: 0.0
  NumExtParam: 0

Total number of implementations found = 1
```

You can also use `system_analyzer` to check more details:

```bash
catid@devnuc:~$ system_analyzer
------------------------------------
Looking for GPU interfaces available to OS...
FOUND: /dev/dri/renderD128
GPU interfaces found: 1
------------------------------------


------------------------------------
Available implementation details:

Implementation #0: mfxhw64
  Library path: /opt/intel/mediasdk/lib/libmfxhw64.so.1.35
  AccelerationMode: MFX_ACCEL_MODE_VIA_VAAPI
  ApiVersion: 1.35
  Impl: MFX_IMPL_TYPE_HARDWARE
  ImplName: mfxhw64
  MediaAdapterType: MFX_MEDIA_UNKNOWN
  VendorID: 0x8086
  DeviceID: 0x46A6 
  GPU name: unknown (arch=na codename=na)
  PCI BDF: FFFFFFFF:FFFFFFFF:FFFFFFFF.FFFFFFFF
  PCI RevisionID: 0xFFFF
  DRMRenderNodeNum: 128
DeviceName: mfxhw64
------------------------------------
```


## Test a hardware-accelerated decode using VPL

I downloaded a sample video:

```
wget https://filesamples.com/samples/video/hevc/sample_1280x720_surfing_with_audio.hevc
```

And this is where I'm stuck right now.  The API 2.0 does not appear to work:

```bash
catid@devnuc:~/sources/libvpl/examples/api2x/hello-decode/build$ ./hello-decode -i sample_1280x720_surfing_with_audio.hevc 
Cannot create session -- no implementations meet selection criteria
Decoded 0 frames
```

However legacy API appears to work:

```bash
catid@devnuc:~/sources/libvpl/examples/api1x_core/legacy-decode/build$ ./legacy-decode -i sample_1280x720_surfing_with_audio.hevc 
Implementation details:
  ApiVersion:           1.35  
  Implementation type: HW
  AccelerationMode via: VAAPI
  DeviceID:             46a6/0 
  Path: /opt/intel/mediasdk/lib/libmfxhw64.so.1.35

libva info: VA-API version 1.20.0
libva info: User environment variable requested driver 'iHD'
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri//iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_20
libva info: va_openDriver() returns 0
Decoding sample_1280x720_surfing_with_audio.hevc -> out.raw

Decoded 4389 frames
```
