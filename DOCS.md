# MediaEdit — Documentation

Browser-based audio and video editor. No backend, no account, no upload to any server. Everything runs locally using FFmpeg compiled to WebAssembly.

---

## Deployment

The project is a single HTML file (`index.html`) with no build step.

### GitHub Pages

1. Create a repository on GitHub.
2. Push the contents of this folder:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/<your-user>/<your-repo>.git
   git push -u origin main
   ```
3. In the repository settings go to **Pages → Source** and select the `main` branch, root folder (`/`).
4. The site will be live at `https://<your-user>.github.io/<your-repo>/`.

The `.nojekyll` file is required so GitHub Pages does not try to process the file with Jekyll.

### Local development

Open `index.html` directly in a browser. No server needed for the UI. FFmpeg is loaded from the CDN on first export.

---

## Supported formats

| Type  | Formats accepted on upload                     |
|-------|------------------------------------------------|
| Audio | MP3, WAV, OGG, FLAC, M4A, OPUS                |
| Video | MP4, MOV, AVI, WebM, MKV, FLV, WMV, M4V       |

Format is detected from the MIME type first, then from the file extension.

---

## Interface overview

### Upload screen

- Drag and drop a file onto the drop zone, or click anywhere in the zone to open the file picker.
- Files never leave the device.

### Editor screen

After a file is loaded the editor is shown. It has five sections stacked vertically:

```
┌─────────────────────────────┐
│  Header (filename, back)    │
├─────────────────────────────┤
│  Media preview              │
│  (video player or waveform) │
├─────────────────────────────┤
│  Timeline                   │
├─────────────────────────────┤
│  Playback controls          │
├─────────────────────────────┤
│  Edit panels (tabs)         │
├─────────────────────────────┤
│  Export bar                 │
└─────────────────────────────┘
```

---

## Features

### Media preview

- **Audio**: the waveform of the entire file is drawn using the Web Audio API and rendered on a Canvas element. The waveform redraws after any edit is applied.
- **Video**: the native HTML5 `<video>` element is used. A thumbnail strip showing frames from the video is generated and displayed on the timeline.

---

### Timeline

A horizontal bar that represents the full duration of the loaded file.

- **Purple shaded region**: the current selection (between the two handles).
- **Left handle (▐)**: marks the start of the selection. Drag it left or right.
- **Right handle (▌)**: marks the end of the selection. Drag it left or right.
- **Green line**: the playhead. Shows the current playback position. Click anywhere on the timeline to seek.

Handles can be dragged with a mouse or a finger on touch screens.

---

### Playback controls

| Button | Action |
|--------|--------|
| ⏮ | Skip back 5 seconds |
| ▶ / ⏸ | Play / Pause |
| ⏭ | Skip forward 5 seconds |

**Selection-bounded playback**: when you press play, playback stops automatically at the end of the selection and the playhead returns to the start of the selection. This lets you hear the exact region you have selected before committing to an edit.

- If the playhead is already past the selection end when you press play, it jumps to the selection start first.
- Changing a handle position while audio is playing reschedules the stop point immediately.

---

### Trim panel

#### Time inputs

Three editable fields below the timeline work together:

| Field | Behaviour |
|-------|-----------|
| **Start** | Sets the left handle position. Accepts `m:ss.t` (e.g. `1:23.5`), `m:ss`, or plain seconds. |
| **End** | Sets the right handle position. Same format. |
| **Duration** | Sets the end point relative to the start (`end = start + duration`). |

All three fields are bidirectional: dragging a handle updates the fields, and typing in a field moves the handle. Press **Enter** or click away to commit a typed value.

#### Action buttons

| Button | What it does |
|--------|-------------|
| **Keep Selection** | Applies the edit immediately: discards everything outside the selection. The player reloads with the result. For audio this is instant (Web Audio API). For video it uses FFmpeg stream copy (~1–2 s). |
| **Cut Selection** | Applies the edit immediately: removes the selected region and joins the two remaining parts. |
| **Select All** | Resets the handles to cover the full file. |
| **↩ Undo** | Restores the previous state. Appears only after at least one edit has been applied. Multiple undos are supported. |

After applying Keep or Cut:

- The player is reloaded with the result. Press play to hear or watch the edit.
- The waveform (audio) or thumbnail strip (video) updates to reflect the new content.
- The selection handles reset to cover the full new duration.
- The status line confirms what was done and reminds you to export when satisfied.

---

### Volume panel

| Control | Range | Notes |
|---------|-------|-------|
| **Volume slider** | 0 % – 300 % | Live playback feedback up to 100 % (browser limit). Values above 100 % amplify via FFmpeg on export. |
| **Fade in** | 0 s – 15 s | Gradual volume ramp from silence at the start of the selection. Applied on export. |
| **Fade out** | 0 s – 15 s | Gradual volume ramp to silence at the end of the selection. Applied on export. |

---

### Speed panel

| Control | Range | Notes |
|---------|-------|-------|
| **Speed slider** | 0.25× – 4.0× | Applied on export via FFmpeg `atempo` filter, chained for values outside the 0.5–2.0 native range. |
| Preset buttons | 0.5×, 0.75×, 1×, 1.5× | Tap to jump directly to a common speed. |

Speed change affects the exported file only. The browser preview always plays at normal speed.

---

### Video panel

Visible only when a video file is loaded.

| Control | Options | Notes |
|---------|---------|-------|
| **Resize** | Original, 1080p, 720p, 480p, 360p, Custom | Uses `scale` filter with Lanczos resampling on export. |
| **Rotate** | 90° CW, 90° CCW, 180°, Reset | Uses FFmpeg `transpose` filter on export. |
| **Remove audio** | Button | Sets volume to 0 and strips the audio track on export. |
| **Extract to MP3** | Button | Switches the export format to MP3 and removes the video track. |

---

### Export bar

> ⚠️ **Status: pending — export does not currently work reliably.**
>
> The export pipeline (FFmpeg re-encode with filters) is implemented but has known issues. The in-browser preview edits (Keep / Cut via stream copy) do work. Use those to prepare your file. The export button and format selector are present in the UI but the output is not guaranteed to be correct.

When export is fixed, the workflow will be:

1. Select output format from the dropdown (e.g. MP3, WAV, MP4, WebM).
2. Click **⬇ Export**.
3. FFmpeg loads on first use (~25 MB, cached by the browser afterwards).
4. The file is processed in-browser and a download is triggered automatically.
5. The downloaded filename is `<original-name>_edited.<format>`.

---

## Technology

| Piece | Version | Purpose |
|-------|---------|---------|
| `@ffmpeg/ffmpeg` | 0.12.10 | FFmpeg JavaScript wrapper |
| `@ffmpeg/core` | 0.12.6 | Single-threaded FFmpeg WebAssembly core (no SharedArrayBuffer required, works on GitHub Pages) |
| `@ffmpeg/util` | 0.12.1 | `toBlobURL` helper for loading wasm from CDN |
| Web Audio API | Browser built-in | Audio decoding, waveform drawing, instant audio preview edits |
| Canvas API | Browser built-in | Waveform rendering, video thumbnail generation |
| HTML5 `<video>` / `<audio>` | Browser built-in | Playback |

All CDN resources are loaded from `unpkg.com`.

---

## Known limitations

- **Export is broken** — see Export bar section above.
- **Amplification above 100 % is not heard in preview** — browser audio elements cannot go above `volume = 1.0`. Values above 100 % are stored and will be applied on export (via FFmpeg `volume=` filter) once export is fixed.
- **Speed change is not previewed** — the browser plays at original speed; the modified speed only applies on export.
- **MOV / MKV / AVI stream-copy preview** may have container compatibility issues on some browsers because these containers are not always natively supported by the HTML5 video element.
- **Very large files** may cause the page to slow down during waveform decoding or thumbnail generation because these operations run on the main thread.
- **No multi-track support** — only the first audio channel pair is shown in the waveform.

---

## File structure

```
audio_tools/
├── index.html   # Complete single-file application (HTML + CSS + JS)
├── .nojekyll    # Prevents GitHub Pages from running Jekyll
└── DOCS.md      # This file
```
