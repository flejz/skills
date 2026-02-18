---
name: gst-pipeline-debugger
description: Debug and fix broken GStreamer pipelines. Use when a user has a pipeline that fails, produces errors, hangs, or behaves unexpectedly. Covers error message interpretation, GST_DEBUG, dot graph generation, and common failure patterns.
---

# GStreamer Pipeline Debugger

Diagnose and fix GStreamer pipeline failures. Interpret error messages, trace negotiation issues, identify missing plugins, and resolve state change problems.

## Debugging Decision Tree

```
Pipeline fails →
├── Error message on bus?
│   ├── RESOURCE error → Missing file, device, or network issue
│   ├── STREAM error → Format or decoding problem
│   ├── CORE error → Missing plugin or element
│   └── LIBRARY error → Dependency issue
├── Pipeline hangs (no error)?
│   ├── Missing queue between threads
│   ├── Preroll stuck → Check live source settings
│   └── Deadlock → Generate dot graph, check queue sizes
├── No audio/video output?
│   ├── Caps negotiation failed → Check with GST_DEBUG=3
│   ├── Wrong sink → Verify autovideosink/autoaudiosink
│   └── Pipeline not in PLAYING state
└── Glitches / artifacts?
    ├── Timestamps → Check PTS/DTS with identity silent=false
    ├── Buffer underrun → Increase queue sizes
    └── Framerate mismatch → Add videorate element
```

## GST_DEBUG Environment Variable

Control debug output verbosity per category:

```bash
# Global debug level (0=none, 1=error, 2=warning, 3=fixme, 4=info, 5=debug, 6=log, 7=trace, 9=memdump)
GST_DEBUG=3 gst-launch-1.0 ...

# Per-category levels
GST_DEBUG=2,videodecoder:5,basesink:4 gst-launch-1.0 ...

# Useful debugging presets
GST_DEBUG=3                                # General issues
GST_DEBUG=GST_CAPS:4                       # Caps negotiation
GST_DEBUG=GST_STATES:4                     # State changes
GST_DEBUG=GST_SCHEDULING:5                 # Push/pull scheduling
GST_DEBUG=GST_PADS:4,GST_CAPS:4           # Linking and negotiation
GST_DEBUG=basesrc:5,basesink:5             # Source/sink issues

# Log to file instead of stderr
GST_DEBUG_FILE=/tmp/gst.log GST_DEBUG=4 gst-launch-1.0 ...

# Colorize output
GST_DEBUG_COLOR_MODE=on GST_DEBUG=3 gst-launch-1.0 ...
```

## Dot Graph Generation

Visualize pipeline topology and state:

```bash
# Enable dot file generation
GST_DEBUG_DUMP_DOT_DIR=/tmp gst-launch-1.0 ...

# Convert dot to PNG
dot -Tpng /tmp/pipeline.dot -o /tmp/pipeline.png

# Programmatic (Python)
Gst.debug_bin_to_dot_file(pipeline, Gst.DebugGraphDetails.ALL, "pipeline")
```

Dot graphs show elements, pads, caps, and current state - essential for diagnosing complex pipelines.

## Common Error Messages and Fixes

### "no element 'X'"
```
# Missing plugin - install it
gst-inspect-1.0 X                    # Check if element exists
apt list --installed | grep gstreamer # Check installed plugins (Debian/Ubuntu)
# Common packages: gstreamer1.0-plugins-{base,good,bad,ugly}, gstreamer1.0-libav
```

### "not-negotiated" / "Internal data stream error"
Caps mismatch between elements. Fix:
```bash
# Add conversion elements
... ! videoconvert ! videoscale ! 'video/x-raw,format=I420,width=1280,height=720' ! encoder ...
# Or for audio
... ! audioconvert ! audioresample ! 'audio/x-raw,rate=44100,channels=2' ! encoder ...
```

### "Could not set property X"
Wrong property name or value type. Check with:
```bash
gst-inspect-1.0 element_name    # Lists all properties with types and ranges
```

### "Failed to connect: No such file or directory"
Device or file not found:
```bash
ls /dev/video*     # Check camera devices
arecord -l         # List audio capture devices
pactl list sources # PulseAudio sources
```

### State change failure (READY -> PAUSED)
Usually a source element problem:
- File source: file doesn't exist
- Camera source: device busy or permissions (`sudo usermod -aG video $USER`)
- Network source: URL unreachable

### Pipeline hangs at PAUSED
Common with live sources:
```bash
# Set pipeline or source to live mode
gst-launch-1.0 v4l2src ! ... # v4l2src is inherently live
# For non-live sources that should act live:
... ! identity sync=true ! ...
```

## Diagnostic Tools

```bash
# List all available elements
gst-inspect-1.0

# Inspect a specific element (pads, properties, signals)
gst-inspect-1.0 x264enc

# Check plugin details
gst-inspect-1.0 --plugin coreelements

# Discover media file info
gst-discoverer-1.0 file.mp4

# Check installed plugin packages (Debian/Ubuntu)
dpkg -l | grep gstreamer

# Check GStreamer version
gst-inspect-1.0 --version
```

## Tracing and Profiling

```bash
# Enable built-in tracers
GST_TRACERS=stats GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...
GST_TRACERS=latency GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...
GST_TRACERS=leaks GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...

# Combine tracers
GST_TRACERS='stats;latency;leaks' GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...
```

## Guidelines

- Start with `GST_DEBUG=3` to see warnings and errors without excessive noise
- Always generate a dot graph for complex pipelines before diving into logs
- Use `gst-inspect-1.0` to verify element availability before assuming it exists
- When a pipeline hangs, check queue fullness: `GST_DEBUG=queue:5`
- State change issues often stem from the first element (source) - debug it in isolation first
- "not-negotiated" almost always means a missing `videoconvert`, `audioconvert`, or explicit caps filter
- Use `fakesink` or `fakesrc` to isolate which part of a pipeline is failing
- Check `dmesg` for kernel-level issues with hardware devices (v4l2, ALSA)
