---
title: Using CSI camera with a Dragonboard-410c
permalink: /documentation/consumer/dragonboard/dragonboard410c/guides/camera-module.md.html
redirect_from:
- /db410c-getting-started/Guides/CameraModule.md/
- /documentation/ConsumerEdition/DragonBoard-410c/Guides/CameraModule.md.html
- /documentation/consumer/dragonboard410c/guides/camera-module.md.html
---
# Using CSI camera

## Introduction

The Linaro Debian and/or OpenEmbedded releases for the DragonBoard 410c include drivers for:

* OV5645 camera sensor
* Qualcomm Camera Subsystem (CAMSS)
* Qualcomm Camera Control Interface (CCI)

## OV5645

The OV5645 is a 5MP MIPI CSI2 camera sensor. The driver supports three sensor modes:

* 2592x1944 15fps (full frame)
* 1920x1080 30fps (cropped from full frame)
* 1280x960 30fps

The driver was accepted upstream and was merged in the Linux Kernel v4.12. It is also backported in the Linaro releases kernel branches.

## CAMSS

The Camera Subsystem hardware found on 8x16 processors consists of:

* 2 CSIPHY modules. They handle the Physical layer of the CSI2 receivers. A separate camera sensor can be connected to each of the CSIPHY module;
* 2 CSID (CSI Decoder) modules. They handle the Protocol and Application layer of the CSI2 receivers. A CSID can decode data stream from any of the CSIPHY. Each CSID also contains a TG (Test Generator) block which can generate artificial input data for test purposes;
* ISPIF (ISP Interface) module. Handles the routing of the data streams from the CSIDs to the inputs of the VFE;
* VFE (Video Front End) module. Contains a pipeline of image processing hardware blocks. The VFE has different input interfaces. The PIX input interface feeds the input data to the image processing pipeline. The image processing pipeline contains also a scale and crop module at the end. Three RDI input interfaces bypass the image processing pipeline. The VFE also contains the AXI bus interface which writes the output data to memory.


The driver implements the V4L2 subdev interface and Media Controller API. With the goal to model the hardware links between the modules and to expose a clean, logical and usable interface, the driver is split into V4L2 sub-devices as follows:

* 2 CSIPHY sub-devices - each CSIPHY is represented by a single sub-device;
* 2 CSID sub-devices - each CSID is represented by a single sub-device;
* 2 ISPIF sub-devices - ISPIF is represented by a number of sub-devices equal to the number of CSID sub-devices;
* 4 VFE sub-devices - VFE is represented by a number of sub-devices equal to the number of the input interfaces (3 RDI and 1 PIX).

The driver supports:

* Input from a camera sensor via CSIPHY
* Generation of test input data by the TG in CSID
* RDI interface of VFE - raw dump of the input data to memory. Supported formats:
  * YUYV/UYVY/YVYU/VYUY (packed YUV 4:2:2)
  * MIPI RAW8 (8bit Bayer RAW)
  * MIPI RAW10 (10bit packed Bayer RAW)
  * MIPI RAW12 (12bit packed Bayer RAW)
* PIX interface of VFE
  * Format conversion of the input data
    * Supported input formats:
       * YUYV/UYVY/YVYU/VYUY (packed YUV 4:2:2).
    * Supported output formats:
       * NV12/NV21 (two plane YUV 4:2:0);
       * NV16/NV61 (two plane YUV 4:2:2).
  * Scaling. Configuration of the VFE Encoder Scale module for downscalling with ratio up to 16x.
  * Cropping. Configuration of the VFE Encoder Crop module.
* Concurrent and independent usage of two data inputs - could be camera sensors and/or TG.

The driver was accepted upstream and is expected in the Linux Kernel v4.14. It is also backported in the Linaro releases kernel branches.

## CCI

The CCI is a I2C controller dedicated for camera control.

The driver is implemented as an I2C adapter driver and will be upstreemed.

## Enable camera

The Linaro release is configured with no camera and users with camera are expected to update the device tree to enable a camera. This can be done by applying a device tree overlay over the base device tree. This adds the required nodes and properties for the camera. Example device tree overlay source and binary files are provided in Linaro kernel in arch/arm64/boot/dts/qcom/overlays/. Should any changes be needed the device tree overlays are rebuilt when the kernel is rebuilt. There are several approaches to update the device tree.

### Enable camera by updating the device tree in a bootimage

This approach is useful when avoiding building the kernel. The kernel source (or at least the device tree overlays from arch/arm64/boot/dts/qcom/overlays/) is still needed.

Get needed tools (dbootimg, dtbtool):

    git clone https://github.com/96boards/dt-update.git

Build needed tools:

    cd dt-update/
    make

Install needed tools:

    sudo make install

Update the Dragonboard bootimage
- extract the device tree binary from the bootimage;
- merge with the camera device tree overlay binary;
- update the device tree binary in the bootimage:

````
    dbootimg boot-db410c.img -x dtb | dtbtool --merge db410c-stm32-rear-ov5645.dtb | dbootimg boot-db410c.img -u dtb
````

Flash or boot the new bootimage.

### Enable camera by updating the device tree before creating a bootimage

This approach can be useful when the kernel is being rebuilt. The device tree binary is updated and then used to create the new bootimage.

Get needed tools. dtbtool or fdtoverlay can be used. For dtbtool check the previous point. For fdtoverlay:

Clone:

    git clone https://git.kernel.org/pub/scm/utils/dtc/dtc.git

Build:

    cd dtc/
    make

Add fdtoverlay to your $PATH (or use full path in next command).

After building the kernel and before creating the new bootimage, update the newly built device tree binary.

For dtbtool:

    dtbtool arch/arm64/boot/dts/qcom/apq8016-sbc.dtb --merge arch/arm64/boot/dts/qcom/overlays/db410c-stm32-rear-ov5645.dtb -o arch/arm64/boot/dts/qcom/apq8016-sbc.dtb

For fdtoverlay:

    fdtoverlay -i arch/arm64/boot/dts/qcom/apq8016-sbc.dtb arch/arm64/boot/dts/qcom/overlays/db410c-stm32-rear-ov5645.dtb -o arch/arm64/boot/dts/qcom/apq8016-sbc.dtb

Append the device tree binary to the kernel binary and build the bootimage the usual way.

## Basic usage

Make sure that the following package is installed:

    sudo apt-get install v4l-utils

To ensure that the sensor is properly connected and detected, you can inspect the output of the following command:

    sudo media-ctl -d /dev/media0 -p

If everything is ok, you should see something like this:
````
- entity 87: ov5645 4-003b (1 pad, 1 link)
             type V4L2 subdev subtype Sensor flags 0
             device node name /dev/v4l-subdev10
        pad0: Source
                [fmt:UYVY8_2X8/1920x1080 field:none
                 crop:(0,0)/1920x1080]
                -> "msm_csiphy0":0 [ENABLED,IMMUTABLE]

````

### Direct dump to memory

You need to configure the Media Controller pipeline: link CSIPHY to CSID, CSID to ISPIF, ISPIF to VFE. Then configure formats on all entities in the pipeline. For direct dump to memory (RDI channels) this looks like this:

    sudo media-ctl -d /dev/media0 -l '"msm_csiphy0":1->"msm_csid0":0[1],"msm_csid0":1->"msm_ispif0":0[1],"msm_ispif0":1->"msm_vfe0_rdi0":0[1]'
    sudo media-ctl -d /dev/media0 -V '"ov5645 4-003b":0[fmt:UYVY8_2X8/1920x1080 field:none],"msm_csiphy0":0[fmt:UYVY8_2X8/1920x1080 field:none],"msm_csid0":0[fmt:UYVY8_2X8/1920x1080 field:none],"msm_ispif0":0[fmt:UYVY8_2X8/1920x1080 field:none],"msm_vfe0_rdi0":0[fmt:UYVY8_2X8/1920x1080 field:none]'

At this point the pipeline should be configured and ready to be used by any application that supports V4L2. For example, you can use GStreamer to take a JPEG picture:

    gst-launch-1.0 v4l2src device=/dev/video0 num-buffers=1 ! 'video/x-raw,format=UYVY,width=1920,height=1080' ! jpegenc ! filesink location=image01.jpg

Or you can use GStreamer to show a live preview from the camera:

    gst-launch-1.0 v4l2src ! 'video/x-raw,format=UYVY,width=1920,height=1080' ! glimagesink

If you have a second camera sensor and intend to use it concurrently then link and configure another pipeline which includes the second camera and the unused entities. Use a v4l2 application the same way only pointing the correct video device node used in this pipeline.

### Format conversion

Pipeline configuration for the format conversion looks like this:

    sudo media-ctl -d /dev/media0 -l '"msm_csiphy0":1->"msm_csid0":0[1],"msm_csid0":1->"msm_ispif0":0[1],"msm_ispif0":1->"msm_vfe0_pix":0[1]'

Format configuration for NV16/NV61 output:

    sudo media-ctl -d /dev/media0 -V '"ov5645 4-003b":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csiphy0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csid0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_ispif0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":1[fmt:UYVY8_2X8/1280x960 field:none]'

Format configuration for NV12/NV21 output (the format on msm_vfe0_pix source pad - for NV12/NV21 must be UYVY8_1_5_X8 - must be set explicitly):

    sudo media-ctl -d /dev/media0 -V '"ov5645 4-003b":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csiphy0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csid0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_ispif0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":1[fmt:UYVY8_1_5X8/1280x960 field:none]'

And similar Gstreamer pipeline for a JPEG picture:

    gst-launch-1.0 v4l2src device=/dev/video3 num-buffers=1 ! 'video/x-raw,format=NV12,width=1280,height=960,framerate=30/1' ! jpegenc ! filesink location=image02.jpg

### Scale

Format configuration for NV12/NV21 with downscale ratio 2x. The compose element on msm_vfe0_pix sink pad defines the output size from the scaler. Syntax is (left,top)/widthxheight and only width and height are valid as this is scaling only. Downscaling with up to 16x ratio is supported:

    media-ctl -d /dev/media0 -V '"ov5645 4-003b":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csiphy0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csid0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_ispif0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":0[fmt:UYVY8_2X8/1280x960 field:none compose:(0,0)/640x480],"msm_vfe0_pix":1[fmt:UYVY8_1_5X8/640x480 field:none]'

### Crop

Format configuration for NV12/NV21 with crop of the bottom right corner. The crop element on msm_vfe0_pix source pad defines the cropped area. Syntax is (left,top)/widthxheight and all fields are valid:

    media-ctl -d /dev/media0 -V '"ov5645 4-003b":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csiphy0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csid0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_ispif0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":1[fmt:UYVY8_1_5X8/320x240 field:none crop:(960,720)/320x240]'

### Scale + crop

Format configuration for NV12/NV21 with downscale ratio 2x and crop of the center area. Scaling is done first and then cropping (scaler module is in front of the crop module in the hardware pipeline of the VFE):

    media-ctl -d /dev/media0 -V '"ov5645 4-003b":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csiphy0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_csid0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_ispif0":0[fmt:UYVY8_2X8/1280x960 field:none],"msm_vfe0_pix":0[fmt:UYVY8_2X8/1280x960 field:none compose:(0,0)/640x480],"msm_vfe0_pix":1[fmt:UYVY8_1_5X8/320x240 field:none crop:(160,120)/320x240]'

## Video record pipeline

Starting with 17.06 release a video recording GStreamer pipeline is supported involving the camera and video encoder. Currently this has the following limitations:

* A GStreamer video encoder plugin is needed which is not a part of the release. To enable this the user must patch and rebuild `gstreamer1.0-plugins-good` package - this is recommended for advanced users only. Instructions are available in the release notes.

Example GStreamer pipeline for video recording:

    gst-launch-1.0 -e v4l2src device=/dev/video3 ! video/x-raw,format=NV12,width=1280,height=960,framerate=30/1 ! v4l2h264enc extra-controls="controls,h264_profile=4,video_bitrate=2000000;" ! h264parse ! mp4mux ! filesink location=video.mp4


## Using CSID Test Generator

If you do not have any camera sensor, it is possible to use the internal CSID Test Generator (TG). To enable it download and compile the `yavta` tool from [here](https://github.com/fastr/yavta):

    git clone git://git.ideasonboard.org/yavta.git
    cd yavta
    make

Then, enable the test generator:

    ./yavta --no-query -w '0x009f0903 1' /dev/v4l-subdev2

Then, configure the pipeline:

    sudo media-ctl -d /dev/media0 -l '"msm_csid0":1->"msm_ispif0":0[1],"msm_ispif0":1->"msm_vfe0_rdi0":0[1]'
    sudo media-ctl -d /dev/media0 -V '"msm_csid0":1[fmt:UYVY8_2X8/1920x1080 field:none],"msm_ispif0":0[fmt:UYVY8_2X8/1920x1080 field:none],"msm_vfe0_rdi0":0[fmt:UYVY8_2X8/1920x1080 field:none]'

Finally, you can use any v4l2 application, such as GStreamer.
