# The .screenstudio file format

A `.screenstudio` file is not a single file — it is a **macOS package directory**. Right-click → "Show Package Contents", or `ls` it in a terminal. Everything inside is plain, unencrypted data. Layout observed in Screen Studio 3.7.x bundles:

```
MyProject.screenstudio/
├── meta.json                    # app version, createdAt
├── project.json                 # ALL edit state (see below)
├── recording-markers.json
└── recording/
    ├── channel-1-system-audio-0.m3u8 (+ ~300 .m4s fragments, + .m4a remux)
    ├── channel-2-display-0.m3u8      # main screen capture, h264
    ├── channel-3-microphone-0.m3u8   (+ .m4a remux)
    ├── channel-4-webcam-0.m3u8       # webcam, h264 (absent if no camera)
    ├── enhanced/                     # noise-reduced voice (m4a), if generated
    ├── cursors/*.png                 # every cursor sprite used
    ├── cursors.json                  # per-sprite hotSpot + standardSize (POINT units)
    ├── mousemoves-0.json             # ~10ms-resolution positions + cursorId
    ├── mouseclicks-0.json            # mouseDown/mouseUp PAIRS (N entries = N/2 clicks)
    ├── keystrokes-0.json
    ├── metadata.json                 # recorders, sessions, bounds, timing anchors
    └── polyrecorder.log
```

## Media

Each channel is **HLS/fMP4**: an init segment (`-0000.mp4`) plus `.m4s` fragments and an `.m3u8` VOD playlist. ffmpeg reads the playlist directly — `ffmpeg -i channel-2-display-0.m3u8 -c copy out.mp4` reassembles losslessly.

**⚠️ The display capture is VFR** (variable frame rate): `r_frame_rate` says 60 but `avg_frame_rate` is typically ~45. Any renderer that counts frames (like ffmpeg's `zoompan`) MUST first normalize with `fps=60,setpts=PTS-STARTPTS`, or video drifts ahead of audio progressively. Stream start PTS is a huge wall-clock-like number; `setpts=PTS-STARTPTS` zero-bases it.

## Coordinate spaces & timing

- Mouse coordinates and `cursors.json` geometry are in **points** (the display's logical `bounds`, e.g. 1710×1107). Video pixels are `bounds × scale` (typically ×2 → 3420×2214).
- Event time anchor: `t_video_seconds = (processTimeMs − processTimeStartMs) / 1000`, where `processTimeStartMs` comes from the input recorder's session in `metadata.json`.
- The OS cursor is **not** baked into the screen capture. Screen Studio records it as data (positions + sprite ids + sprite PNGs with hotspots) and re-renders it at export. That's why raw extractions have no pointer.

## project.json (the edit state)

Everything is under `.json`:

- `scenes[0].slices` — the cuts: `sourceStartMs`/`sourceEndMs` per kept span (+ `timeScale` for speed ramps).
- `scenes[0].zoomRanges` — zooms: `startTime`/`endTime` (source ms), `zoom` factor, `type` (`follow-click-groups` = center on the click cluster; `manual` = `manualTargetPoint`).
- `config` (~70 keys) — `backgroundType/Color/Gradient/SystemName`, `backgroundPaddingRatio`, `windowBorderRadius`, `shadow*`, `camera*` (webcam size/corner/roundness), `cursorSize`, `clickEffect`, `recordingCrop`, `recordingRange`, mute flags, spring parameters (`screenMovementSpring`, `mouseMovementSpring`) for the app's proprietary easing.

Notes: `backgroundSystemName` (e.g. `macOS/sonoma-river.jpg`) references an app resource **not present in the bundle**. `showTranscript` may be true, but no transcript is stored in the bundle — captions are generated at export time by the app.

## What this means

Since the bundle holds raw media + full effect parameters + full input telemetry, a finished export can be reproduced outside the app: normalize the VFR video, overlay a cursor layer rendered from the telemetry, drive `zoompan` from the zoom ranges and click positions, apply the cuts, composite background/webcam/shadow, and mux the chosen audio. That is exactly what [this repo's scripts](../scripts) do.
