# MediaEdit — Documentation

Browser-based audio and video editor. No backend, no account, no upload to any server. Everything runs locally using FFmpeg compiled to WebAssembly.

---

## Table of contents

1. [Deployment](#deployment)
2. [Google Analytics setup](#google-analytics-setup)
3. [SEO configuration](#seo-configuration)
4. [Interface overview](#interface-overview)
5. [Features](#features)
6. [Export](#export)
7. [Technology](#technology)
8. [Known limitations](#known-limitations)
9. [File structure](#file-structure)

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

Open `index.html` directly in a browser, or serve it from a local server:

```bash
python -m http.server 8000
# then open http://localhost:8000
```

FFmpeg is loaded from the CDN on first use (~25 MB, cached by the browser afterwards).

---

## Google Analytics setup

Google Analytics 4 (GA4) is already wired into the code. You only need to plug in your Measurement ID.

### Step 1 — Create a GA4 property

1. Go to [analytics.google.com](https://analytics.google.com) and sign in with your Google/Gmail account.
2. Click **Start measuring**.
3. **Account name**: anything (e.g. `MediaEdit`).
4. **Property name**: your site name, timezone, and currency.
5. **Choose a platform**: click **Web**.
6. **Website URL**: your GitHub Pages URL, e.g. `https://yourusername.github.io/your-repo`.
7. Click **Create stream**.
8. Copy the **Measurement ID** shown at the top right — it looks like `G-AB12CD34EF`.

### Step 2 — Add your Measurement ID to index.html

Open `index.html` and replace both occurrences of `G-XXXXXXXXXX` (lines 26 and 31) with your real ID:

```html
<script async src="https://www.googletagmanager.com/gtag/js?id=G-AB12CD34EF"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){ dataLayer.push(arguments); }
  gtag('js', new Date());
  gtag('config', 'G-AB12CD34EF');
  ...
</script>
```

### Custom events tracked

Custom events appear in GA4 under **Reports → Engagement → Events** within 24 hours of the first visit.

| Event | When it fires | Parameters |
|---|---|---|
| `file_loaded` | User uploads a file | `file_type` (audio/video), `file_ext` (mp4, mp3 …) |
| `keep_selection` | "Keep Selection" clicked | `media_type` |
| `cut_selection` | "Cut Selection" clicked | `media_type` |
| `export_started` | Export button clicked | `format`, `media_type`, `has_crop`, `rotate`, `muted`, `audio_only` |
| `crop_applied` | Crop overlay confirmed | `width`, `height` (in video pixels) |
| `rotate_set` | Rotation button clicked | `degrees` (90° CW, 90° CCW, 180°) |
| `mute_video` | Video audio muted | — |
| `extract_audio` | Extract to MP3 clicked | — |

---

## SEO configuration

SEO meta tags, Open Graph, and Twitter cards are already in the `<head>`. Replace the placeholder values before going live.

### Replace the domain placeholder

Search for `YOUR-DOMAIN-HERE` in `index.html` and replace all 4 occurrences with your actual URL:

```html
<link rel="canonical" href="https://yourusername.github.io/your-repo/">
<meta property="og:url"    content="https://yourusername.github.io/your-repo/">
<meta property="og:image"  content="https://yourusername.github.io/your-repo/og-image.png">
<meta name="twitter:image" content="https://yourusername.github.io/your-repo/og-image.png">
```

### OG image (recommended)

Take a screenshot of the editor in use, crop it to **1200 × 630 px**, and save it as `og-image.png` in the same folder as `index.html`. This image appears when the link is shared on WhatsApp, Twitter, LinkedIn, and Facebook.

### Tags included

| Tag | Purpose |
|---|---|
| `<title>` | Browser tab and Google search result title |
| `<meta name="description">` | Google search result snippet |
| `<meta name="keywords">` | Secondary keyword signal |
| `<meta name="robots">` | Tells search engines to index the page |
| `<link rel="canonical">` | Prevents duplicate-content issues |
| `og:title / og:description / og:image / og:url` | Social media link previews |
| `twitter:card / twitter:title / twitter:description / twitter:image` | Twitter / X card |

---

## Interface overview

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

### Supported formats

| Type  | Formats accepted on upload |
|-------|---------------------------|
| Audio | MP3, WAV, OGG, FLAC, M4A, OPUS |
| Video | MP4, MOV, AVI, WebM, MKV, FLV, WMV, M4V |

Format is detected from the MIME type first, then from the file extension.

---

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

**Selection-bounded playback**: playback stops automatically at the end of the selection and the playhead returns to the start of the selection.

---

### Trim panel

#### Time inputs

| Field | Behaviour |
|-------|-----------|
| **Start** | Sets the left handle position. Accepts `m:ss.t` (e.g. `1:23.5`), `m:ss`, or plain seconds. |
| **End** | Sets the right handle position. Same format. |
| **Duration** | Sets the end point relative to the start (`end = start + duration`). |

All three fields are bidirectional: dragging a handle updates the fields, and typing in a field moves the handle.

#### Action buttons

| Button | What it does |
|--------|-------------|
| **Keep Selection** | Discards everything outside the selection. For audio this is instant. For video it uses FFmpeg stream copy (fast, no quality loss). |
| **Cut Selection** | Removes the selected region and joins the two remaining parts. |
| **Select All** | Resets the handles to cover the full file. |
| **↩ Undo** | Restores the previous state. Appears only after at least one edit has been applied. |

After applying Keep or Cut the player reloads with the result, the waveform or thumbnail strip updates, and the selection handles reset to the full new duration.

**Video stream copy details**: the trim uses `-ss` as an input seek (fast keyframe alignment), `-t` for duration, `-map 0:v? -map 0:a?` to preserve both streams, `-movflags +faststart` for browser compatibility, and `-avoid_negative_ts make_zero` to normalise timestamps. Output is always MP4 for preview regardless of the original container.

---

### Volume panel

| Control | Range | Notes |
|---------|-------|-------|
| **Volume slider** | 0 % – 300 % | Live playback feedback up to 100 % (browser limit). Values above 100 % are applied via FFmpeg `volume=` filter on export. |
| **Fade in** | 0 s – 15 s | Gradual volume ramp from silence at the start. Applied on export. |
| **Fade out** | 0 s – 15 s | Gradual volume ramp to silence at the end. Applied on export. |

---

### Speed panel

| Control | Range | Notes |
|---------|-------|-------|
| **Speed slider** | 0.25× – 4.0× | Applied on export via FFmpeg `atempo` filter, chained for values outside the 0.5–2.0 native range. |
| Preset buttons | 0.5×, 0.75×, 1×, 1.5× | Tap to jump to a common speed. |

Speed change affects the exported file only. The preview always plays at normal speed.

---

### Video panel

Visible only when a video file is loaded.

#### Crop tool

The crop tool lets you select a rectangular region of the video (by aspect ratio) and export only that region. This is a spatial cut — it does not affect the timeline selection.

| Control | Details |
|---------|---------|
| **Aspect ratio** | Choose 16:9, 9:16, 1:1, 4:3, 3:4, or enter custom W:H values. |
| **Show overlay** | Displays a semi-transparent mask over the video preview. The selected region is highlighted; everything outside is darkened. |
| **Drag the box** | Repositions which part of the frame will be kept. |
| **Drag the corner handle (bottom-right)** | Resizes the crop box while maintaining the chosen aspect ratio. |
| **Pixel readout** | The box shows live pixel dimensions as you drag. |
| **Apply crop** | Saves the crop region. The button turns purple to indicate an active crop. |
| **Clear** | Removes the crop. |

The crop region is stored as pixel coordinates in the original video resolution and applied on export via FFmpeg `crop=w:h:x:y`.

#### Rotate

Clicking a rotation button immediately applies a CSS transform to the video preview so you can see the result before exporting.

| Button | Export filter |
|--------|--------------|
| 90° CW | `transpose=1` |
| 90° CCW | `transpose=2` |
| 180° | `transpose=1,transpose=1` |
| Reset | No filter |

The active rotation button turns purple. Export re-encodes with the correct `transpose` filter.

#### Audio controls

| Button | Effect |
|--------|--------|
| **Mute** | Immediately mutes the video element as a live preview. On export the audio track is removed (`-an`). Click again to unmute. |
| **Extract to MP3** | Switches the export format to MP3 and removes the video track. |

---

## Export

1. Choose the output format from the dropdown in the export bar.
2. Click **⬇ Export**.
3. FFmpeg loads on first use (~25 MB, cached by the browser afterwards).
4. The file is processed in-browser and a download is triggered automatically.
5. The downloaded filename is `<original-name>_edited.<format>`.

### Supported output formats

| Format | Codec |
|--------|-------|
| MP4 | H.264 video + AAC audio |
| WebM | VP9 video + Opus audio |
| MP3 | libmp3lame |
| WAV | PCM |
| OGG | libvorbis |
| FLAC | flac |
| M4A | AAC |
| OPUS | libopus |

### Filter order in the FFmpeg command

When multiple effects are set, filters are applied in this order:

1. Crop (`crop=`)
2. Rotate (`transpose=`)
3. Speed (`setpts=` for video, `atempo=` for audio)
4. Volume (`volume=`)
5. Fade in / fade out (`afade=`)

---

## Technology

| Piece | Version | Purpose |
|-------|---------|---------|
| `@ffmpeg/ffmpeg` | 0.12.6 | FFmpeg JavaScript wrapper |
| `@ffmpeg/core` | 0.12.6 | Single-threaded FFmpeg WebAssembly (no SharedArrayBuffer required — works on GitHub Pages without special headers) |
| `@ffmpeg/util` | 0.12.1 | `toBlobURL` helper for loading WASM from CDN |
| Web Audio API | Browser built-in | Audio decoding, waveform drawing, instant audio preview edits |
| Canvas API | Browser built-in | Waveform rendering, video thumbnail generation |
| HTML5 `<video>` | Browser built-in | Video playback and crop overlay |
| Google Analytics 4 | gtag.js | Usage and feature tracking |

All CDN resources are loaded from `cdn.jsdelivr.net`.

### FFmpeg WASM loading

The library normally creates a module Worker using `new Worker(cdnUrl, { type: 'module' })`, which is blocked by browsers when the page origin differs from the CDN origin. The workaround used here:

1. Fetch `worker.js` source text from the CDN.
2. Rewrite all relative imports (`from './'`, `import('./')`) to absolute CDN URLs so the worker can resolve its own dependencies when running from a blob URL.
3. Create a blob URL from the modified source.
4. Monkey-patch `window.Worker` with a subclass that redirects the CDN worker URL to the blob URL.
5. Call `ffmpeg.load()` with blob URLs for the core JS and WASM files.
6. Restore `window.Worker` after load completes.

This approach requires no COOP/COEP headers and works from `file://`, `localhost`, and GitHub Pages.

---

## Known limitations

- **Amplification above 100 % is not previewed** — browser audio elements cannot exceed `volume = 1.0`. Values above 100 % apply on export via FFmpeg.
- **Speed change is not previewed** — the modified speed applies only on export.
- **Crop is not applied to the preview video** — the overlay shows what will be cropped; the actual crop applies on export.
- **Very large files** may slow down waveform decoding or thumbnail generation (main-thread operations).
- **No multi-track support** — only the first audio channel pair is shown in the waveform.
- **MOV / MKV stream-copy segments** during Cut Selection are converted to MP4 for the preview regardless of the original container.

---

## File structure

```
audio_tools/
├── index.html     # Complete single-file application (HTML + CSS + JS)
├── og-image.png   # Social media preview image — 1200×630 px (create this)
├── .nojekyll      # Prevents GitHub Pages from running Jekyll
└── DOCS.md        # This file
```
