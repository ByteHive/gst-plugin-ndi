GStreamer NDI Plugin for Linux
====================

*Compiled and tested with NDI SDK 4.0, 4.1, 5.0, and 6.x*

**Features:**
- Full multi-channel audio support (supports 1 to unlimited audio channels)
- Compatible with NDI SDK 6.x (latest version)
- Uses NDI v3 API for optimal performance and compatibility
- Advanced timestamping and synchronization capabilities
- Support for NDI timecode and timestamp metadata
- Audio/Video clock synchronization for precise frame timing
- Seamless NDI to WebRTC conversion for browser-based streaming
- Hardware-accelerated encoding support (VA-API, NVENC)

This is a plugin for the [GStreamer](https://gstreamer.freedesktop.org/) multimedia framework that allows GStreamer to receive a stream from a [NDI](https://www.newtek.com/ndi/) source. This plugin has been developed by [Teltek](http://teltek.es/) and was funded by the [University of the Arts London](https://www.arts.ac.uk/) and [The University of Manchester](https://www.manchester.ac.uk/).

Currently the plugin has a source element for receiving from NDI sources, a sink element to provide an NDI source and a device provider for discovering NDI sources on the network.

Some examples of how to use these elements from the command line:

```console
# Information about the elements
$ gst-inspect-1.0 ndi
$ gst-inspect-1.0 ndisrc
$ gst-inspect-1.0 ndisink

# Discover all NDI sources on the network
$ gst-device-monitor-1.0 -f Source/Network:application/x-ndi

# Audio/Video source pipeline
$ gst-launch-1.0 ndisrc ndi-name="GC-DEV2 (OBS)" ! ndisrcdemux name=demux   demux.video ! queue ! videoconvert ! autovideosink  demux.audio ! queue ! audioconvert ! autoaudiosink

# Audio/Video sink pipeline
$ gst-launch-1.0 videotestsrc is-live=true ! video/x-raw,format=UYVY ! ndisinkcombiner name=combiner ! ndisink ndi-name="My NDI source"  audiotestsrc is-live=true ! combiner.audio

# Multi-channel audio sink pipeline (8 channels example)
$ gst-launch-1.0 videotestsrc is-live=true ! video/x-raw,format=UYVY ! ndisinkcombiner name=combiner ! ndisink ndi-name="My NDI source"  audiotestsrc is-live=true ! audio/x-raw,channels=8 ! combiner.audio

# Audio-only source pipeline with multi-channel support
$ gst-launch-1.0 ndisrc ndi-name="Audio Source" ! ndisrcdemux name=demux demux.audio ! queue ! audioconvert ! autoaudiosink
```

Synchronization and Timestamping
-------

The NDI plugin provides comprehensive synchronization and timestamping capabilities for professional broadcast workflows.

### NDI Source (ndisrc) - Timestamp Modes

The `ndisrc` element supports multiple timestamp modes via the `timestamp-mode` property:

- **`receive-time-vs-timecode`** (default): Synchronizes receive time with NDI timecode for optimal clock recovery
- **`receive-time-vs-timestamp`**: Synchronizes receive time with NDI timestamp
- **`timecode`**: Uses NDI timecode directly as PTS
- **`timestamp`**: Uses NDI timestamp directly as PTS
- **`receive-time`**: Uses local receive time as PTS

**Example with timestamp mode:**
```console
$ gst-launch-1.0 ndisrc ndi-name="Camera 1" timestamp-mode=timecode ! ...
```

### NDI Reference Timestamps

When compiled with `reference-timestamps` feature (enabled by default), the plugin adds NDI timecode and timestamp as GStreamer reference timestamp metadata. This allows downstream elements to access the original NDI timing information.

### NDI Sink (ndisink) - Clock Synchronization

The `ndisink` element supports NDI's built-in clock synchronization:

- **`clock-video`**: Enable video frame rate limiting/clocking by NDI
- **`clock-audio`**: Enable audio frame rate limiting/clocking by NDI

When enabled, NDI SDK will automatically rate-limit frames to maintain proper timing and synchronization across the network.

**Example with clock synchronization:**
```console
# Synchronized video and audio output
$ gst-launch-1.0 videotestsrc is-live=true ! video/x-raw,format=UYVY ! \
    ndisinkcombiner name=combiner ! \
    ndisink ndi-name="Synced Source" clock-video=true clock-audio=true \
    audiotestsrc is-live=true ! combiner.audio

# Video-only with precise timing
$ gst-launch-1.0 videotestsrc is-live=true ! video/x-raw,format=UYVY ! \
    ndisink ndi-name="Precise Video" clock-video=true
```

### Multi-Source Synchronization

For applications requiring frame-accurate synchronization across multiple NDI sources:

1. Use the same `timestamp-mode` on all `ndisrc` elements
2. Enable `clock-video` and `clock-audio` on `ndisink` elements
3. Consider using GStreamer's `netsync` or PTP clock for system-wide synchronization

**Example - Synchronized multi-camera setup:**
```console
# Camera 1
$ gst-launch-1.0 ndisrc ndi-name="Camera 1" timestamp-mode=timecode ! \
    ndisrcdemux name=d1 d1.video ! queue ! ...

# Camera 2
$ gst-launch-1.0 ndisrc ndi-name="Camera 2" timestamp-mode=timecode ! \
    ndisrcdemux name=d2 d2.video ! queue ! ...
```

NDI to WebRTC
-------

Convert NDI streams to WebRTC for low-latency browser-based viewing and streaming applications.

### Prerequisites

Install GStreamer WebRTC plugins:
```console
$ apt-get install gstreamer1.0-plugins-good gstreamer1.0-plugins-bad \
    gstreamer1.0-nice gstreamer1.0-libav
```

### Basic NDI to WebRTC Pipeline

**Simple video-only WebRTC streaming:**
```console
# VP8 encoding (widely supported)
$ gst-launch-1.0 ndisrc ndi-name="Camera 1" ! ndisrcdemux name=demux \
    demux.video ! queue ! videoconvert ! vp8enc deadline=1 target-bitrate=2000000 ! \
    rtpvp8pay ! webrtcbin name=sendrecv
```

**H.264 encoding (hardware acceleration):**
```console
$ gst-launch-1.0 ndisrc ndi-name="Camera 1" ! ndisrcdemux name=demux \
    demux.video ! queue ! videoconvert ! x264enc tune=zerolatency bitrate=2000 speed-preset=ultrafast ! \
    rtph264pay ! webrtcbin name=sendrecv
```

### Full Audio/Video WebRTC Pipeline

**Complete NDI to WebRTC with audio (VP8 + Opus):**
```console
$ gst-launch-1.0 ndisrc ndi-name="Studio Camera" timestamp-mode=timecode ! \
    ndisrcdemux name=demux \
    demux.video ! queue max-size-buffers=1 leaky=downstream ! videoconvert ! \
        vp8enc deadline=1 target-bitrate=3000000 cpu-used=4 ! rtpvp8pay ! \
        queue ! application/x-rtp,media=video,encoding-name=VP8,payload=96 ! \
        webrtcbin name=sendrecv \
    demux.audio ! queue max-size-buffers=1 leaky=downstream ! audioconvert ! \
        audioresample ! opusenc bitrate=128000 ! rtpopuspay ! \
        queue ! application/x-rtp,media=audio,encoding-name=OPUS,payload=97 ! \
        sendrecv.
```

**With H.264 and AAC (better compatibility):**
```console
$ gst-launch-1.0 ndisrc ndi-name="Studio Camera" ! ndisrcdemux name=demux \
    demux.video ! queue ! videoconvert ! x264enc tune=zerolatency bitrate=3000 ! \
        rtph264pay config-interval=1 ! queue ! \
        application/x-rtp,media=video,encoding-name=H264,payload=96 ! \
        webrtcbin name=sendrecv \
    demux.audio ! queue ! audioconvert ! voaacenc bitrate=128000 ! \
        rtpmp4apay ! queue ! \
        application/x-rtp,media=audio,encoding-name=MPEG4-GENERIC,payload=97 ! \
        sendrecv.
```

### Low-Latency WebRTC Streaming

**Optimized for minimal latency:**
```console
$ gst-launch-1.0 ndisrc ndi-name="Camera" connect-timeout=1000 timeout=1000 ! \
    ndisrcdemux name=demux \
    demux.video ! queue max-size-time=0 max-size-buffers=1 leaky=downstream ! \
        videoconvert ! videoscale ! video/x-raw,width=1280,height=720 ! \
        vp8enc deadline=1 target-bitrate=2000000 cpu-used=8 lag-in-frames=0 ! \
        rtpvp8pay mtu=1200 ! queue max-size-time=0 ! \
        application/x-rtp,media=video,encoding-name=VP8,payload=96 ! \
        webrtcbin latency=0 name=sendrecv \
    demux.audio ! queue max-size-time=0 max-size-buffers=1 leaky=downstream ! \
        audioconvert ! audioresample ! audio/x-raw,rate=48000 ! \
        opusenc bitrate=96000 frame-size=10 ! rtpopuspay ! \
        queue max-size-time=0 ! \
        application/x-rtp,media=audio,encoding-name=OPUS,payload=97 ! \
        sendrecv.
```

### Multi-Channel Audio WebRTC

**NDI multi-channel audio to WebRTC stereo:**
```console
# Downmix 8-channel NDI audio to stereo for WebRTC
$ gst-launch-1.0 ndisrc ndi-name="Multi-Channel Source" ! ndisrcdemux name=demux \
    demux.video ! queue ! videoconvert ! vp8enc deadline=1 ! rtpvp8pay ! \
        queue ! application/x-rtp,media=video,encoding-name=VP8,payload=96 ! \
        webrtcbin name=sendrecv \
    demux.audio ! queue ! audioconvert ! audioresample ! \
        audio/x-raw,channels=2,rate=48000 ! opusenc ! rtpopuspay ! \
        queue ! application/x-rtp,media=audio,encoding-name=OPUS,payload=97 ! \
        sendrecv.
```

### Hardware-Accelerated Encoding

**Using VA-API (Intel/AMD GPUs):**
```console
$ gst-launch-1.0 ndisrc ndi-name="Camera" ! ndisrcdemux name=demux \
    demux.video ! queue ! vaapipostproc ! vaapih264enc rate-control=cbr bitrate=3000 ! \
        rtph264pay ! queue ! \
        application/x-rtp,media=video,encoding-name=H264,payload=96 ! \
        webrtcbin name=sendrecv \
    demux.audio ! queue ! audioconvert ! opusenc ! rtpopuspay ! \
        queue ! application/x-rtp,media=audio,encoding-name=OPUS,payload=97 ! \
        sendrecv.
```

**Using NVENC (NVIDIA GPUs):**
```console
$ gst-launch-1.0 ndisrc ndi-name="Camera" ! ndisrcdemux name=demux \
    demux.video ! queue ! videoconvert ! nvh264enc rc-mode=cbr bitrate=3000 ! \
        rtph264pay ! queue ! \
        application/x-rtp,media=video,encoding-name=H264,payload=96 ! \
        webrtcbin name=sendrecv \
    demux.audio ! queue ! audioconvert ! opusenc ! rtpopuspay ! \
        queue ! application/x-rtp,media=audio,encoding-name=OPUS,payload=97 ! \
        sendrecv.
```

### WebRTC Signaling

Note: The examples above show the media pipeline only. For complete WebRTC functionality, you need:

1. **Signaling server**: To exchange SDP offers/answers and ICE candidates
2. **STUN/TURN servers**: For NAT traversal
3. **WebRTC signaling implementation**: Using GStreamer's webrtcbin signals

**Python example with signaling:**
```python
import gi
gi.require_version('Gst', '1.0')
gi.require_version('GstWebRTC', '1.0')
from gi.repository import Gst, GstWebRTC

# Initialize
Gst.init(None)

# Create pipeline
pipeline = Gst.parse_launch('''
    ndisrc ndi-name="Camera 1" ! ndisrcdemux name=demux
    demux.video ! queue ! videoconvert ! vp8enc deadline=1 ! rtpvp8pay !
        application/x-rtp,media=video,encoding-name=VP8,payload=96 ! webrtcbin name=sendrecv
    demux.audio ! queue ! audioconvert ! opusenc ! rtpopuspay !
        application/x-rtp,media=audio,encoding-name=OPUS,payload=97 ! sendrecv.
''')

webrtc = pipeline.get_by_name('sendrecv')

# Connect signaling callbacks
webrtc.connect('on-negotiation-needed', on_negotiation_needed)
webrtc.connect('on-ice-candidate', on_ice_candidate)

# Set STUN server
webrtc.set_property('stun-server', 'stun://stun.l.google.com:19302')

pipeline.set_state(Gst.State.PLAYING)
```

For complete working examples with signaling, see:
- [GStreamer WebRTC demos](https://gitlab.freedesktop.org/gstreamer/gst-examples/-/tree/master/webrtc)
- [gst-plugins-rs webrtc examples](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/-/tree/main/net/webrtc)

Feel free to contribute to this project. Some ways you can contribute are:
* Testing with more hardware and software and reporting bugs
* Doing pull requests.

Compilation of the NDI element
-------
To compile the NDI element it's necessary to install Rust, the NDI SDK and the following packages for gstreamer:

```console
$ apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
      gstreamer1.0-plugins-base

```
To install the required NDI library there are two options:
1. Download NDI SDK from NDI website and move the library to the correct location.
2. Use a [deb package](https://github.com/Palakis/obs-ndi/releases/download/4.5.2/libndi3_3.5.1-1_amd64.deb) made by the community. Thanks to [NDI plugin for OBS](https://github.com/Palakis/obs-ndi).

To install Rust, you can follow their documentation: https://www.rust-lang.org/en-US/install.html

Once all requirements are met, you can build the plugin by executing the following command from the project root folder:

```
cargo build
export GST_PLUGIN_PATH=`pwd`/target/debug
gst-inspect-1.0 ndi
```

By default GStreamer 1.18 is required, to use an older version. You can build with `$ cargo build --no-default-features --features whatever_you_want_to_enable_of_the_above_features`
      

If all went ok, you should see info related to the NDI element. To make the plugin available without using `GST_PLUGIN_PATH` it's necessary to copy the plugin to the gstreamer plugins folder.

```console
$ cargo build --release
$ sudo install -o root -g root -m 644 target/release/libgstndi.so /usr/lib/x86_64-linux-gnu/gstreamer-1.0/
$ sudo ldconfig
$ gst-inspect-1.0 ndi
```

More info about GStreamer plugins written in Rust:
----------------------------------
https://gitlab.freedesktop.org/gstreamer/gstreamer-rs
https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs


License
-------
This plugin is licensed under the LGPL - see the [LICENSE](LICENSE) file for details


Acknowledgments
-------
* University of the Arts London and The University of Manchester.
* Sebastian Dröge (@sdroege).
