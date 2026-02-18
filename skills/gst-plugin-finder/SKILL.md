---
name: gst-plugin-finder
description: Find the right GStreamer elements and plugins for a given task. Use when a user needs to identify which GStreamer element handles a specific codec, protocol, effect, or media operation.
---

# GStreamer Plugin Finder

Identify the right GStreamer elements for any multimedia task. Covers codecs, protocols, effects, sources, sinks, and utility elements.

## How to Search for Elements

```bash
# List all elements
gst-inspect-1.0

# Search by keyword
gst-inspect-1.0 | grep -i h264
gst-inspect-1.0 | grep -i rtmp
gst-inspect-1.0 | grep -i pulse

# Inspect element details
gst-inspect-1.0 x264enc

# List elements in a plugin package
gst-inspect-1.0 --plugin x264

# Check which package provides an element (Debian/Ubuntu)
apt-file search gst-plugin-scanner  # Find plugin dirs
apt list --installed 2>/dev/null | grep gstreamer
```

## Element Catalog by Task

### Video Decoders

| Codec | Software | Hardware (VAAPI) | Hardware (NVIDIA) | Hardware (V4L2) |
|-------|----------|------------------|-------------------|-----------------|
| H.264 | `avdec_h264` | `vaapih264dec` | `nvh264dec` | `v4l2h264dec` |
| H.265 | `avdec_h265` | `vaapih265dec` | `nvh265dec` | `v4l2h265dec` |
| VP8 | `vp8dec` | `vaapivp8dec` | - | `v4l2vp8dec` |
| VP9 | `vp9dec` | `vaapivp9dec` | `nvvp9dec` | `v4l2vp9dec` |
| AV1 | `av1dec`, `dav1ddec` | `vaapiav1dec` | `nvav1dec` | - |
| MPEG-2 | `avdec_mpeg2video` | `vaapimpeg2dec` | - | - |

### Video Encoders

| Codec | Software | Hardware (VAAPI) | Hardware (NVIDIA) | Hardware (V4L2) |
|-------|----------|------------------|-------------------|-----------------|
| H.264 | `x264enc` | `vaapih264enc` | `nvh264enc` | `v4l2h264enc` |
| H.265 | `x265enc` | `vaapih265enc` | `nvh265enc` | `v4l2h265enc` |
| VP8 | `vp8enc` | `vaapivp8enc` | - | - |
| VP9 | `vp9enc` | `vaapivp9enc` | `nvvp9enc` | - |
| AV1 | `av1enc`, `svtav1enc` | `vaav1enc` | `nvav1enc` | - |
| JPEG | `jpegenc` | `vaapijpegenc` | - | `v4l2jpegenc` |

### Audio Codecs

| Codec | Encoder | Decoder | Plugin Package |
|-------|---------|---------|----------------|
| AAC | `voaacenc`, `fdkaacenc`, `avenc_aac` | `avdec_aac`, `faad` | gst-plugins-bad, gst-libav |
| MP3 | `lamemp3enc` | `mpg123audiodec`, `avdec_mp3` | gst-plugins-good/ugly, gst-libav |
| Opus | `opusenc` | `opusdec` | gst-plugins-base |
| Vorbis | `vorbisenc` | `vorbisdec` | gst-plugins-base |
| FLAC | `flacenc` | `flacdec` | gst-plugins-good |
| PCM/WAV | `wavenc` | `wavparse` | gst-plugins-good |

### Muxers / Demuxers

| Container | Muxer | Demuxer |
|-----------|-------|---------|
| MP4/MOV | `mp4mux`, `qtmux` | `qtdemux` |
| MKV | `matroskamux` | `matroskademux` |
| WebM | `webmmux` | `matroskademux` |
| MPEG-TS | `mpegtsmux` | `tsdemux` |
| FLV | `flvmux` | `flvdemux` |
| AVI | `avimux` | `avidemux` |
| Ogg | `oggmux` | `oggdemux` |
| WAV | `wavenc` | `wavparse` |

### Source Elements

| Source | Element | Notes |
|--------|---------|-------|
| File | `filesrc` | Any file type |
| URI (auto) | `uridecodebin`, `urisourcebin` | HTTP, file, RTSP |
| Webcam (Linux) | `v4l2src` | `/dev/video*` |
| Webcam (macOS) | `avfvideosrc` | - |
| Screen (X11) | `ximagesrc` | X11 screen capture |
| PulseAudio | `pulsesrc` | Linux audio capture |
| PipeWire | `pipewiresrc` | Modern Linux audio/video |
| ALSA | `alsasrc` | Direct ALSA capture |
| RTSP | `rtspsrc` | RTSP client |
| RTMP | `rtmpsrc` | RTMP pull |
| SRT | `srtsrc` | SRT receive |
| UDP/RTP | `udpsrc` | Raw UDP packets |
| Test video | `videotestsrc` | Test patterns |
| Test audio | `audiotestsrc` | Sine wave, noise |
| App (programmatic) | `appsrc` | Push buffers from code |

### Sink Elements

| Sink | Element | Notes |
|------|---------|-------|
| File | `filesink` | Write to file |
| Auto video | `autovideosink` | Best available display |
| Auto audio | `autoaudiosink` | Best available output |
| X11 | `ximagesink`, `xvimagesink` | X11 display |
| Wayland | `waylandsink` | Wayland display |
| PulseAudio | `pulsesink` | PulseAudio output |
| PipeWire | `pipewiresink` | PipeWire output |
| ALSA | `alsasink` | Direct ALSA output |
| RTMP | `rtmpsink`, `rtmp2sink` | RTMP push |
| SRT | `srtsink` | SRT send |
| UDP | `udpsink` | Raw UDP output |
| TCP | `tcpserversink`, `tcpclientsink` | TCP streaming |
| Fake (discard) | `fakesink` | Drop data (testing) |
| App (programmatic) | `appsink` | Receive buffers in code |
| HLS | `hlssink`, `hlssink2` | HLS output |

### Video Processing

| Task | Element |
|------|---------|
| Color format conversion | `videoconvert` |
| Scaling | `videoscale` |
| Framerate adjustment | `videorate` |
| Cropping | `videocrop` |
| Flipping / rotating | `videoflip` |
| Compositing / mixing | `compositor` |
| Text overlay | `textoverlay`, `clockoverlay`, `timeoverlay` |
| Image overlay | `gdkpixbufoverlay` |
| Color balance | `videobalance` |
| Box drawing | `rsvgoverlay` |
| Deinterlacing | `deinterlace` |

### Audio Processing

| Task | Element |
|------|---------|
| Format conversion | `audioconvert` |
| Resampling | `audioresample` |
| Mixing | `audiomixer` |
| Volume control | `volume` |
| Equalization | `equalizer-3bands`, `equalizer-10bands` |
| Echo / effects | `audioecho` |
| Level metering | `level` |
| Noise suppression | `webrtcdsp` |

### Network / Streaming

| Protocol | Elements |
|----------|----------|
| RTP | `rtpbin`, `rtph264pay/depay`, `rtpvp8pay/depay`, `rtpopuspay/depay` |
| RTSP | `rtspsrc`, `rtspclientsink` |
| RTMP | `rtmpsrc`, `rtmpsink`, `rtmp2src`, `rtmp2sink` |
| SRT | `srtsrc`, `srtsink` |
| WebRTC | `webrtcbin` |
| HLS | `hlssink`, `hlssink2`, `hlsdemux` |
| DASH | `dashdemux`, `dashsink` |

### Utility Elements

| Purpose | Element |
|---------|---------|
| Thread decoupling | `queue`, `queue2`, `multiqueue` |
| Branching | `tee` |
| Selecting streams | `input-selector`, `output-selector` |
| Caps filtering | `capsfilter` |
| Identity / debugging | `identity` |
| Valve (block flow) | `valve` |
| Type finding | `typefind` |
| Auto decoder | `decodebin`, `decodebin3` |
| Auto playback | `playbin`, `playbin3` |
| Parse H.264 | `h264parse` |
| Parse H.265 | `h265parse` |
| Parse AAC | `aacparse` |

## Plugin Packages (Debian/Ubuntu)

```
gstreamer1.0-plugins-base    # Core elements: playback, conversion, encoding basics
gstreamer1.0-plugins-good    # High-quality plugins: matroska, jpeg, png, pulse, v4l2
gstreamer1.0-plugins-bad     # Varied quality: x265, opus, srt, webrtc, hls
gstreamer1.0-plugins-ugly    # Patent-encumbered: x264, mp3, mpeg2
gstreamer1.0-libav           # FFmpeg/libav decoders and encoders
gstreamer1.0-vaapi           # VAAPI hardware acceleration
gstreamer1.0-tools           # gst-launch-1.0, gst-inspect-1.0
libgstreamer1.0-dev          # Development headers
```

## Guidelines

- Use `gst-inspect-1.0 ELEMENT` to verify an element exists before using it
- Hardware-accelerated elements (`vaapi*`, `nv*`, `v4l2*`) depend on driver support - fall back to software if unavailable
- When multiple elements do the same thing, prefer: plugins-base > plugins-good > plugins-bad > plugins-ugly > libav
- `decodebin3` and `playbin3` auto-select the best decoder/sink, useful for quick prototyping
- For production, pin specific elements rather than relying on auto-selection
