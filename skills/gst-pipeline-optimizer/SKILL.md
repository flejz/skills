---
name: gst-pipeline-optimizer
description: Optimize GStreamer pipeline performance. Use when a user needs to reduce latency, increase throughput, fix dropped frames, tune buffer sizes, leverage hardware acceleration, or profile pipeline bottlenecks.
---

# GStreamer Pipeline Optimizer

Optimize GStreamer pipelines for throughput, latency, CPU usage, and memory. Covers queue tuning, hardware acceleration, threading, and profiling.

## Performance Diagnosis Checklist

1. **Identify the bottleneck**: Is it CPU, GPU, memory, I/O, or network?
2. **Measure first**: Use tracers and profiling before optimizing
3. **Target the slowest element**: One slow element throttles the entire pipeline
4. **Check hardware acceleration**: Software encoding/decoding is the most common bottleneck

## Queue Placement and Tuning

### When to Use Queues
- Between every encoder/decoder and the rest of the pipeline
- Before and after any element that blocks (network sinks, file I/O)
- To create thread boundaries for parallel processing
- In `tee` branches to prevent one slow branch from stalling others

### queue vs queue2

| Feature | `queue` | `queue2` |
|---------|---------|----------|
| Buffering | Memory only | Memory + disk |
| Use case | Thread decoupling | Network stream buffering |
| Overhead | Lower | Higher |
| Temp file support | No | Yes |

### Queue Tuning Properties

```bash
# queue: control thread boundary buffering
queue max-size-buffers=200 max-size-bytes=10485760 max-size-time=1000000000
#       200 buffers          10 MB                  1 second (nanoseconds)

# Disable unneeded limits (set to 0)
queue max-size-buffers=0 max-size-bytes=0 max-size-time=2000000000  # Only time-based limit

# leaky queue: drop old/new buffers when full (live pipelines)
queue leaky=downstream max-size-buffers=3   # Drop oldest when full
queue leaky=upstream max-size-buffers=3     # Drop newest when full

# queue2: network buffering
queue2 use-buffering=true max-size-bytes=20971520  # 20 MB buffer
```

### multiqueue for Parallel Streams
```bash
# Use multiqueue when handling multiple streams (audio + video)
decodebin ! multiqueue name=mq ! videoconvert ! encoder
                       mq. ! audioconvert ! audio_encoder
```

## Hardware Acceleration

### Detection
```bash
# Check for VAAPI support
gst-inspect-1.0 | grep vaapi
vainfo  # Show VAAPI capabilities

# Check for NVIDIA NVDEC/NVENC
gst-inspect-1.0 | grep -i nv
nvidia-smi  # Verify GPU is available

# Check for V4L2 hardware codecs
gst-inspect-1.0 | grep v4l2.*dec
gst-inspect-1.0 | grep v4l2.*enc
v4l2-ctl --list-devices
```

### Hardware-Accelerated Pipeline Examples

```bash
# VAAPI H.264 encoding (Intel/AMD)
v4l2src ! videoconvert ! vaapih264enc rate-control=cbr bitrate=4000 ! h264parse ! mp4mux ! filesink location=out.mp4

# NVIDIA H.264 encoding
v4l2src ! videoconvert ! nvh264enc bitrate=4000 preset=low-latency-hq ! h264parse ! mp4mux ! filesink location=out.mp4

# VAAPI decoding + display (zero-copy)
filesrc location=video.mp4 ! qtdemux ! h264parse ! vaapih264dec ! vaapisink

# NVIDIA decode + encode (transcode on GPU)
filesrc location=input.mp4 ! qtdemux ! h264parse ! nvh264dec ! nvh264enc bitrate=2000 ! h264parse ! mp4mux ! filesink location=output.mp4
```

### Hardware Acceleration Priority
1. **VAAPI** - Best Linux support (Intel, AMD)
2. **NVIDIA NVENC/NVDEC** - Best for NVIDIA GPUs
3. **V4L2** - Embedded systems (Raspberry Pi, Jetson)
4. **Software** - Fallback, always available

## Latency Optimization

### Low-Latency Encoding
```bash
# x264enc low-latency settings
x264enc tune=zerolatency speed-preset=ultrafast bitrate=2500 key-int-max=30

# Key properties:
#   tune=zerolatency     - Disables B-frames, reduces lookahead
#   speed-preset=ultrafast - Fastest encoding, largest file
#   key-int-max=N        - Keyframe interval (lower = more seekable, larger file)
#   threads=N            - Encoding threads (0 = auto)

# VAAPI low-latency
vaapih264enc rate-control=cbr bitrate=2500 keyframe-period=30
```

### Pipeline-Level Latency Settings
```bash
# Set latency on the pipeline (nanoseconds)
# Programmatic:
pipeline.set_latency(100 * Gst.MSECOND)  # 100ms

# For live sources, reduce latency-offset
gst-launch-1.0 v4l2src ! queue max-size-buffers=1 leaky=downstream ! videoconvert ! autovideosink sync=false

# Disable sync on sinks for lowest latency (may cause tearing)
autovideosink sync=false
```

### Network Streaming Latency
```bash
# RTP: minimize jitter buffer
rtpjitterbuffer latency=50  # 50ms (default is 200ms)

# SRT low-latency
srtsrc uri="srt://0.0.0.0:1234" latency=125  # 125ms (default is 125)
srtsink uri="srt://dest:1234" latency=125

# RTMP
rtmpsink location="rtmp://server/live/key live=1"
```

## Throughput Optimization

### Parallel Processing with Threads
```bash
# Each queue creates a new thread - use them to parallelize
src ! queue ! decoder ! queue ! filter ! queue ! encoder ! queue ! sink
#     ^thread1          ^thread2         ^thread3          ^thread4
```

### Batch Processing
```bash
# Process multiple files: use pipeline restart or dynamic pipelines
# For transcoding farms, run multiple gst-launch instances
parallel gst-launch-1.0 filesrc location={} ! decodebin ! x264enc ! mp4mux ! filesink location={.}.mp4 ::: *.avi
```

### Memory Optimization
```bash
# Use buffer pools (automatic for most elements, configure on appsrc)
appsrc format=time block=true max-bytes=1048576  # 1 MB buffer limit

# Reduce queue memory
queue max-size-bytes=1048576 max-size-buffers=5  # Limit memory per queue

# Zero-copy where possible (hardware acceleration, same-GPU processing)
```

## Profiling Tools

### Built-in Tracers
```bash
# Latency tracer: measure end-to-end latency
GST_TRACERS=latency GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...

# Stats tracer: per-element statistics
GST_TRACERS=stats GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...

# Leaks tracer: detect buffer/event leaks
GST_TRACERS=leaks GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...

# Framerate tracer: measure FPS at each point
GST_TRACERS=framerate GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...

# Combined
GST_TRACERS='latency;stats;framerate' GST_DEBUG=GST_TRACER:7 gst-launch-1.0 ...
```

### CPU Profiling
```bash
# Use perf to profile CPU usage per element
perf record gst-launch-1.0 ...
perf report

# Or use sysprof for graphical profiling
sysprof-cli --command "gst-launch-1.0 ..."
```

### Monitoring at Runtime
```bash
# identity element: print buffer timestamps and sizes
... ! identity silent=false ! ...

# fpsdisplaysink: show FPS on screen
... ! fpsdisplaysink video-sink=autovideosink text-overlay=true

# Dot graph at different states
GST_DEBUG_DUMP_DOT_DIR=/tmp gst-launch-1.0 ...
```

## Common Performance Issues and Fixes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Dropped frames | Encoder too slow | Use HW acceleration or faster preset |
| High CPU on encode | Software encoding | Switch to VAAPI/NVENC |
| Audio/video desync | Missing queues | Add queue before encoder and muxer |
| Pipeline stalls | Blocking sink | Add leaky queue before sink |
| Memory grows | Buffer leak | Use leaks tracer, check appsink usage |
| High latency | Large queue/jitter buffer | Reduce queue sizes, lower jitterbuffer latency |
| Choppy playback | No thread separation | Add queues between stages |

## Guidelines

- Measure before optimizing - use tracers to identify the actual bottleneck
- Hardware encoding typically gives 10x+ performance improvement over software
- `tune=zerolatency` on x264enc is the single biggest low-latency win for software H.264
- In live pipelines, use `leaky=downstream` queues to drop old frames rather than building up latency
- `sync=false` on sinks removes display-clock synchronization overhead, useful for benchmarking or processing pipelines
- Zero-copy paths (VAAPI decode -> VAAPI encode, or VAAPI decode -> vaapisink) avoid expensive GPU-to-CPU memory transfers
- Over-queuing wastes memory and increases latency; under-queuing causes stalls. Start with `max-size-time=1000000000` (1s) and adjust
- For network streaming, the bottleneck is usually the encoder - optimize encoding first
