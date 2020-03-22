# Linux Video Conferencing using OBS and Gstreamer

There's currently a market shortage in standard USB webcams due to current
events causing many people to be working from home. I didn't manage to buy one
before they all sold out. I have a few IP security apartments in my apartment,
so I figured I'd try to repurpose one for usage with video conferencing as a
proof-of-concept.

I'm running Ubuntu 18.04. Instructions may differ for other distributions.

OBS isn't strictly required, but it can be useful for easy color correction and
other video broadcasting tools. If you want to use multiple webcams as sources,
then OBS is required. I should also mention that I haven't gotten the gstreamer
commands I've been using to work with Google Hangouts on Firefox without OBS.

## Camera Setup

In order for this to work, you need to have an IP camera capable of producing
RTSP streams. I am using the Wyze cam since I have one lying around, but any
camera capable of being an RTSP video source should work. RTSP setup for
cameras varies from manufacturer to manufacturer, so I can't cover them all.

* [Wyze Cam RTSP Setup](https://support.wyzecam.com/hc/en-us/articles/360026245231-Wyze-Cam-RTSP)

Fundamentally, what you'll need is the RTSP URI. On Wyze cams with RTSP
enabled, it looks like:

```
rtsp://username:password@WYZE_CAM_IP/live
```

This will be different for other camera manufacturers.

## Linux Setup

### Prerequisite packages

The following Ubuntu packages are necessary for this tutorial, which can be
installed via `apt`.

* `v4l2loopback-utils`
* `gstreamer1.0-tools`
* `obs-studio`

I forget which gstreamer plugin set that I needed to get to get stuff working,
but I have the good, bad, and ugly plugin packages installed from the Ubuntu
repositories, as well as the `gstreamer1.0-libav` plugin package.

### Setting up Video4Linux

In order to treat an IP camera as a webcam, Linux needs to pipe the video feed
into a virtual camera device. This can be done by loading the `v4l2loopback`
kernel module to create virtual video devices that OBS can read from and write
to. Let's say that you want to create a video device for OBS to output to and
have two camera sources that you want to use as video sources. The following
command will set that up for you.

```
$ sudo modprobe v4l2loopback video_nr=10,11,12 card_label="OBS Video","Camera 1",Camera 2"
```

The stock `v4l2loopback` Ubuntu packages let you create a maximum of eight
devices using `v4l2loopback` and `modprobe`.

If you `ls /dev/video*`, you should see the devices `/dev/video10`,
`/dev/video11`, and `/dev/video12`. You're all set up.

#### Automatically loading v4l2loopback on boot

The above command for setting up `v4l2` devices via `modprobe` will not persist
after reboot. However, it is possible to configure Linux to automatically
configure the virtual cameras at boot time with a couple of configuration
files. It will be assumed that the video setup in this configuration is the
same as what was created from the `modprobe` command listed above. Feel free to
adjust as necessary for the video setup that you need.

Place the following contents in `/etc/modprobe.d/v4l2loopback.conf`:

```
options v4l2loopback video_nr=10,11,12 card_label="OBS Video","Camera 1","Camera 2"
```

Place the following contents in `/etc/modules-load.d/v4l2loopback.conf`:

```
v4l2loopback
```

If you reboot your computer, you should see the devices `/dev/video10`,
`/dev/video11`, and `/dev/video12` automatically configured for you.

## Capturing Video

### Start capturing the IP camera with gstreamer

Now that your video devices are set up, it's time to start piping the RTSP
stream into them. The following command takes one RTSP stream encoded as h264
video (many IP cameras encode with h264 or h265 by default), decodes it, and
puts it into a video device:

```
$ gst-launch-1.0 rtspsrc drop-on-latency=true location=<rtsp uri> ! decodebin ! videoconvert ! v4l2sink device=/dev/video11
```

Awesome! See the
[`rtspsrc`](https://gstreamer.freedesktop.org/documentation/rtsp/rtspsrc.html)
and
[`v4l2sink`](https://gstreamer.freedesktop.org/documentation/video4linux2/v4l2sink.html)
gstreamer documentation for more options for configuring the gstreamer pipeline
for the IP camera device.

Once you have gstreamer launched for all of the cameras you're using, you can
configure them for usage with OBS Studio as V4L2 Video Capture Devices.

### Configuring OBS Studio to output to a V4L2 device.

By default OBS doesn't have the capability of outputting video to a V4L2
device. However, there is a plugin that can be used for that. This is required
to use the output of OBS for video conferencing. Instructions for installing
the OBS V4L2 output plugin can be found at the
[`obs-v4l2sink`](https://github.com/CatxFish/obs-v4l2sink) GitHub page.

Once you have that installed, you can use `/dev/video10` (or whatever you've
configured as the OBS output device via `v4l2loopback` on your system) with the
V4L2 output plugin under the OBS tools menu.
