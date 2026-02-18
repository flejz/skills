---
name: gst-caps-negotiation
description: Understand and resolve GStreamer caps negotiation issues. Use when a user encounters caps-related errors, format mismatches, or needs to control media formats between pipeline elements.
---

# GStreamer Caps Negotiation

Resolve caps (capabilities) negotiation failures and control media format flow between GStreamer elements.

## What Are Caps?

Caps describe the media format flowing between elements. Each pad has a set of caps it supports. Two linked pads must agree on a compatible format during negotiation.

```
# Caps structure
media-type, field1=value1, field2=value2, ...

# Examples
video/x-raw, format=I420, width=1920, height=1080, framerate=30/1
audio/x-raw, format=S16LE, rate=44100, channels=2, layout=interleaved
video/x-h264, stream-format=avc, alignment=au, profile=high, level=(string)4.1
image/jpeg, width=1280, height=720
```

## Common Video Formats

| Format | Description | Typical Use |
|--------|-------------|-------------|
| `video/x-raw` | Uncompressed video | Between processing elements |
| `video/x-h264` | H.264 encoded | Streaming, recording |
| `video/x-h265` | H.265/HEVC encoded | High-efficiency recording |
| `video/x-vp8` | VP8 encoded | WebM, WebRTC |
| `video/x-vp9` | VP9 encoded | WebM, YouTube |
| `video/x-av1` | AV1 encoded | Next-gen streaming |
| `image/jpeg` | JPEG frames | MJPEG cameras |

## Common Audio Formats

| Format | Description | Typical Use |
|--------|-------------|-------------|
| `audio/x-raw` | Uncompressed audio | Between processing elements |
| `audio/mpeg` | AAC/MP3 | Streaming, recording |
| `audio/x-opus` | Opus encoded | WebRTC, VoIP |
| `audio/x-vorbis` | Vorbis encoded | Ogg containers |
| `audio/x-flac` | FLAC lossless | Archival |

## Raw Video Format Field Values

```
format:    I420, NV12, YUY2, UYVY, BGRA, RGBA, BGRx, RGBx, BGR, RGB, GRAY8, ...
width:     integer (e.g., 1920)
height:    integer (e.g., 1080)
framerate: fraction (e.g., 30/1, 25/1, 60/1, 0/1 for variable)
interlace-mode: progressive, interleaved, mixed
pixel-aspect-ratio: fraction (e.g., 1/1)
colorimetry: bt709, bt601, smpte240m
chroma-site: mpeg2, jpeg, none
```

## Negotiation Failure Patterns and Fixes

### Pattern 1: Encoder expects specific raw format
```
# FAILS: x264enc only accepts I420, YV12, NV12
videotestsrc ! x264enc ! ...

# FIX: Insert videoconvert
videotestsrc ! videoconvert ! x264enc ! ...

# BEST: Explicit caps for predictability
videotestsrc ! videoconvert ! video/x-raw,format=I420 ! x264enc ! ...
```

### Pattern 2: Resolution mismatch
```
# FAILS: Sink expects different size than source
src ! sink expecting 1280x720

# FIX: Insert videoscale + caps filter
src ! videoscale ! video/x-raw,width=1280,height=720 ! sink
```

### Pattern 3: Framerate mismatch
```
# FAILS: Muxer expects constant framerate, source is variable
camera ! muxer

# FIX: Insert videorate
camera ! videorate ! video/x-raw,framerate=30/1 ! muxer
```

### Pattern 4: Audio sample rate mismatch
```
# FAILS: Encoder expects 44100 Hz, source produces 48000 Hz
src ! encoder

# FIX: Insert audioresample
src ! audioconvert ! audioresample ! audio/x-raw,rate=44100 ! encoder
```

### Pattern 5: Channel count mismatch
```
# FAILS: Mono source, stereo sink
src ! audio/x-raw,channels=1 ! stereo_sink

# FIX: Insert audioconvert
src ! audioconvert ! audio/x-raw,channels=2 ! stereo_sink
```

## Conversion Element Reference

| Problem | Solution Element | Purpose |
|---------|-----------------|---------|
| Wrong pixel format | `videoconvert` | Converts between video color formats |
| Wrong resolution | `videoscale` | Scales video to target resolution |
| Wrong framerate | `videorate` | Adjusts framerate by duplicating/dropping |
| Wrong audio format | `audioconvert` | Converts between audio sample formats |
| Wrong sample rate | `audioresample` | Resamples audio to target rate |
| Need parsed stream | `h264parse`, `mpegaudioparse` | Parses encoded bitstream |

## Inspecting Caps

```bash
# View element pad templates (what formats it accepts/produces)
gst-inspect-1.0 x264enc | grep -A 20 "Pad Templates"

# See negotiated caps at runtime
GST_DEBUG=GST_CAPS:4 gst-launch-1.0 ...

# Use identity to print caps flowing through
... ! identity silent=false ! ...

# Use dot graph to see caps on every link
GST_DEBUG_DUMP_DOT_DIR=/tmp gst-launch-1.0 ...
```

## Caps Filter Syntax in gst-launch

```bash
# Inline caps (shorthand)
... ! video/x-raw,width=1280,height=720 ! ...

# capsfilter element (explicit)
... ! capsfilter caps="video/x-raw,width=1280,height=720" ! ...

# Multiple caps alternatives (element negotiates best match)
... ! "video/x-raw,format=I420; video/x-raw,format=NV12" ! ...

# Range values
... ! video/x-raw,width=[640,1920],height=[480,1080] ! ...

# List values
... ! video/x-raw,format={I420,NV12,YV12} ! ...
```

## The Safe Conversion Chain

When unsure about format compatibility, this chain handles most conversions:

```
# Video: handles format, size, and rate
... ! videoconvert ! videoscale ! videorate ! capsfilter caps="TARGET_CAPS" ! ...

# Audio: handles format, rate, and channels
... ! audioconvert ! audioresample ! capsfilter caps="TARGET_CAPS" ! ...
```

## Guidelines

- `videoconvert` is nearly zero-cost when input and output formats already match - it is safe to insert liberally
- Always check pad templates with `gst-inspect-1.0` to understand what an element can accept
- Prefer explicit caps filters over relying on auto-negotiation in production pipelines
- When debugging caps, `GST_DEBUG=GST_CAPS:4` shows the negotiation process step by step
- Parsers (`h264parse`, `aacparse`, etc.) are often needed between decoders and muxers to ensure proper stream framing
- `decodebin3` auto-inserts decoders and converters, but for custom pipelines you often need manual conversion elements
