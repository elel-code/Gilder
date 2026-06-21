# M8 Video Optimization Plan

This is an archived note. It records the removed GTK, native-wgpu and native
`playbin/waylandsink` video optimization work as historical baseline evidence.
Those paths are no longer buildable or executable validation targets.

Current video work lives in:

- `docs/vulkan-migration.md`: native Wayland/Vulkan architecture, importer
  direction, Vulkan Video and GStreamer appsink/DMA handoff plan.
- `docs/video-validation.md`: current codec, native Vulkan Wayland and process
  sampling commands.
- `docs/todo.md`: remaining implementation and validation checklist.

## Retired Baselines

The old measurements remain useful only as comparison points:

- GTK direct sink showed a practical H.264 4K/240 baseline but retained too much
  process/GPU memory and depended on GTK/GDK/GSK behavior outside our control.
- Native-wgpu proved a GStreamer GPU-memory handoff could reach roughly
  240fps with lower private dirty memory, but it is no longer maintained as a
  separate backend.
- Native `playbin/waylandsink` reduced some memory categories but was unstable
  and was not the desired direct-DMA/Vulkan direction.

## Current Decision

The project now uses one visible native path:

- native Wayland hosts layer-shell surface/output/scale/viewport/dmabuf feedback;
- native Vulkan owns import/decode/render/present;
- GStreamer provides demux/parser/appsink/audio/clock only;
- display sinks such as GTK paintable sinks and `waylandsink` do not own the
  visible wallpaper surface.

The old scripts and helper binaries were intentionally removed so future video
validation cannot accidentally fall back to the retired paths.
