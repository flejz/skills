---
name: gst-pipeline-builder
description: Build GStreamer pipelines from high-level descriptions. Use when a user wants to construct a multimedia pipeline for tasks like video playback, transcoding, streaming, recording, mixing, or any media processing workflow using GStreamer.
---

# GStreamer Pipeline Builder

Construct GStreamer pipelines from natural-language descriptions. Produce both `gst-launch-1.0` command lines and equivalent programmatic code (C, Python, or Rust) depending on what the user needs.

## Pipeline Construction Process

1. **Clarify the task**: Identify source, processing, and sink stages
2. **Select elements**: Choose the right GStreamer elements for each stage
3. **Wire caps**: Insert explicit caps filters where negotiation needs guidance
4. **Add queues**: Place `queue` elements at thread boundaries
5. **Validate**: Mentally trace data flow and confirm pad compatibility

## gst-launch-1.0 Syntax Reference

```
# Basic pipe: element ! element
gst-launch-1.0 videotestsrc ! autovideosink

# Named elements and tee
gst-launch-1.0 videotestsrc ! tee name=t \
  t. ! queue ! autovideosink \
  t. ! queue ! x264enc ! filesink location=out.mp4

# Caps filter (inline)
gst-launch-1.0 videotestsrc ! video/x-raw,width=1280,height=720 ! autovideosink

# Muxing audio + video
gst-launch-1.0 videotestsrc ! x264enc ! mux. \
  audiotestsrc ! audioconvert ! voaacenc ! mux. \
  mp4mux name=mux ! filesink location=out.mp4
```

## Common Pipeline Patterns

### Playback
```
uridecodebin uri=URI ! autovideosink
uridecodebin uri=URI ! audioconvert ! autoaudiosink
playbin uri=URI
```

### Transcoding
```
uridecodebin uri=INPUT ! videoconvert ! x264enc ! h264parse ! mp4mux ! filesink location=OUTPUT
```

### Camera Capture
```
v4l2src device=/dev/video0 ! videoconvert ! video/x-raw,width=1280,height=720 ! autovideosink
```

### Screen Capture
```
ximagesrc ! videoconvert ! x264enc tune=zerolatency ! mp4mux ! filesink location=screen.mp4
```

### RTMP Streaming
```
v4l2src ! videoconvert ! x264enc tune=zerolatency bitrate=2500 ! video/x-h264,profile=main ! flvmux streamable=true ! rtmpsink location='rtmp://server/live/key'
```

### Audio Recording
```
pulsesrc ! audioconvert ! vorbisenc ! oggmux ! filesink location=recording.ogg
```

### Picture-in-Picture (compositor)
```
compositor name=comp sink_0::xpos=0 sink_0::ypos=0 sink_1::xpos=10 sink_1::ypos=10 sink_1::width=320 sink_1::height=240 ! autovideosink \
  videotestsrc pattern=snow ! comp.sink_0 \
  videotestsrc ! comp.sink_1
```

### Test Pattern with Overlay
```
videotestsrc ! clockoverlay halignment=center valignment=bottom text="Stream" ! autovideosink
```

## Element Selection Guidelines

| Task | Preferred Elements |
|------|--------------------|
| Decode anything | `uridecodebin`, `decodebin3` |
| Encode H.264 | `x264enc`, `vaapih264enc`, `nvh264enc` |
| Encode H.265 | `x265enc`, `vaapih265enc`, `nvh265enc` |
| Encode VP8/VP9 | `vp8enc`, `vp9enc` |
| Encode AV1 | `av1enc`, `svtav1enc`, `vaav1enc` |
| Encode AAC | `voaacenc`, `fdkaacenc`, `avenc_aac` |
| Mux MP4 | `mp4mux`, `qtmux` |
| Mux MKV | `matroskamux` |
| Mux WebM | `webmmux` |
| Mux MPEG-TS | `mpegtsmux` |
| Mix video | `compositor` |
| Mix audio | `audiomixer` |
| Resize | `videoscale` |
| Convert colorspace | `videoconvert` |
| Convert audio format | `audioconvert` |
| Resample audio | `audioresample` |
| Add text overlay | `textoverlay`, `clockoverlay`, `timeoverlay` |
| RTP payloading | `rtph264pay`, `rtpvp8pay`, `rtpopuspay` |
| WebRTC | `webrtcbin` |

## Programmatic Pipeline (Python Example)

```python
import gi
gi.require_version('Gst', '1.0')
from gi.repository import Gst, GLib

Gst.init(None)

pipeline = Gst.parse_launch(
    'videotestsrc ! videoconvert ! autovideosink'
)
pipeline.set_state(Gst.State.PLAYING)

loop = GLib.MainLoop()
bus = pipeline.get_bus()
bus.add_signal_watch()
bus.connect('message::eos', lambda *_: loop.quit())
bus.connect('message::error', lambda _, msg: print(msg.parse_error()))

try:
    loop.run()
finally:
    pipeline.set_state(Gst.State.NULL)
```

## Guidelines

- Always insert `videoconvert` before sinks and encoders that expect specific formats
- Always insert `audioconvert ! audioresample` before audio encoders and sinks
- Use `queue` to decouple threads and prevent deadlocks in complex pipelines
- Prefer `decodebin3` over `decodebin` for new pipelines
- Prefer `playbin3` over `playbin` when available
- Use `tune=zerolatency` on `x264enc` for live/low-latency scenarios
- Set `async-handling=true` on bins when mixing live and non-live sources
- For file output, always include a proper muxer before `filesink`
- When in doubt about caps, use `gst-inspect-1.0 ELEMENT` to check pad templates
