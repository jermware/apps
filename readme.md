# GPX QuickLook Extension

A macOS QuickLook Preview Extension that renders `.gpx` files directly in Finder with a map, elevation chart, and detailed statistics.

## Features

- **Map tab** — full MapKit track overlay with start/finish markers, waypoints, and standard/satellite/hybrid style toggle
- **Elevation tab** — Swift Charts area graph with min/max/gain/loss summary
- **Stats tab** — distance, time, speed, elevation, metadata, and file info
- Supports tracks, routes, and waypoints
- Parses Garmin TrackPointExtension fields (heart rate, cadence, power, temperature)
- Works with any GPX 1.0 / 1.1 file

## Project Setup

### 1. Create the Xcode project

1. **File → New → Project → macOS → App**
   - Product Name: `GPXViewer` (or anything)
   - Language: Swift, Interface: SwiftUI
   - Bundle ID: `com.yourname.gpxviewer`

2. **File → New → Target → macOS → Quick Look Preview Extension**
   - Product Name: `GPXQuickLook`
   - This creates `GPXQuickLook/PreviewViewController.swift` automatically

### 2. Add files to targets

Copy the files from this repo into your project:

| File | Target |
|------|--------|
| `GPXQuickLook/Models/GPXModels.swift` | GPXQuickLook |
| `GPXQuickLook/Parser/GPXParser.swift` | GPXQuickLook |
| `GPXQuickLook/PreviewViewController.swift` | GPXQuickLook (replace default) |
| `GPXQuickLook/Views/GPXPreviewView.swift` | GPXQuickLook |
| `GPXQuickLook/Views/MapView.swift` | GPXQuickLook |
| `GPXQuickLook/Views/ElevationChartView.swift` | GPXQuickLook |
| `GPXQuickLook/Views/StatsView.swift` | GPXQuickLook |

### 3. Replace Info.plist files

- Replace **`GPXQuickLook/Info.plist`** with the one provided
- Replace (or merge) **host app `Info.plist`** with the UTI declarations provided

### 4. Add frameworks to the extension target

Select **GPXQuickLook** target → **General → Frameworks and Libraries**, add:
- `MapKit.framework`
- `Charts.framework` (ships with Xcode 14+ / macOS 13+)

> **Note:** Do NOT add these only to the host app — they must be in the **extension** target.

### 5. Entitlements

The extension runs in the sandbox automatically. No special entitlements are needed for read-only file access. The URL passed to `preparePreviewOfFile(at:)` has an implicit security-scoped bookmark you don't need to manage manually.

If you want to add **App Groups** (to share preferences with the host app):
- Add `com.apple.security.application-groups` to both targets' entitlements
- Use the same group identifier (e.g. `group.com.yourname.gpxviewer`)

### 6. Deployment target

Set **macOS 13.0** or later on both targets (required for `MapPolyline`, `MapCameraPosition`, and the Charts framework).

### 7. Build & test

```bash
# Install the host app (this registers the extension with the system)
# Run from Xcode, or:
open /path/to/GPXViewer.app

# Test the extension directly
qlmanage -p /path/to/track.gpx

# Force-reload the QuickLook daemon (use after every rebuild during dev)
qlmanage -r
killall Finder    # optional; makes Finder pick up UTI changes
```

### Debugging the extension

Set the extension scheme's **Run → Executable** to `/usr/bin/qlmanage` with arguments `-p $(your.gpx.file)`. This lets you attach the debugger normally.

Alternatively: run the host app, then find the extension process in **Debug → Attach to Process** once you trigger a preview in Finder.

## Architecture

```
GPXViewer.app  (host, minimal UI)
└── GPXQuickLook.appex  (extension)
    ├── PreviewViewController   — QLPreviewingController, async entry point
    ├── GPXParser               — SAX-based XMLParser for performance
    ├── GPXModels               — value types + GPXStatistics computation
    └── Views
        ├── GPXPreviewView      — root shell (header, tab bar)
        ├── MapView             — MapKit polyline + markers
        ├── ElevationChartView  — Swift Charts area graph
        └── StatsView           — scrollable stat grid
```

## Customisation

### Adding heart rate / power charts
`ElevationChartView` can be extended: the parser already populates `GPXTrackPoint.heartRate`, `.cadence`, `.power`, and `.temperature` from Garmin extensions. Add extra `Chart` views and a tab.

### Colour-coding by speed or grade
Replace the solid/gradient `MapPolyline` with multiple shorter segments, each coloured by the computed value. MapKit supports multiple overlapping `MapPolyline` annotations.

### Thumbnail generation
Add a second target: **Quick Look Thumbnail Extension**. Use `MKMapSnapshotter` to render a static map image of the track.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Finder doesn't invoke the extension | Run `qlmanage -r && killall Finder` |
| Extension not listed in `qlmanage -m` | Build & run the host app first; the system registers extensions from installed `.app` bundles |
| Map blank / MapKit crash | Ensure MapKit is linked to the *extension* target, not just the host app |
| Charts import error | Requires macOS 13+ deployment target |
| `preparePreviewOfFile` never called | Check `QLSupportedContentTypes` in extension `Info.plist` matches the file's actual UTI (`mdls yourfile.gpx` to inspect) |
