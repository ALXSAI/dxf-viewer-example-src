# dxf-viewer Integration Guide

A practical reference for using the [`dxf-viewer`](https://github.com/vagran/dxf-viewer) npm package in your own project.

---

## Overview

`dxf-viewer` is a high-performance 2D DXF file viewer for the browser. It renders drawings using **WebGL** via [three.js](https://threejs.org/) and offloads heavy parsing work to a **Web Worker**, keeping the UI thread unblocked even with large files.

Key facts:
- Renders lines, arcs, splines, hatches, and text from DXF files
- Text requires TTF fonts to be explicitly provided
- Works in any framework (Vue, React, vanilla JS)
- WebGL canvas auto-sizes to its container

---

## Installation

```bash
npm install dxf-viewer three
```

`three` is a required peer dependency. Match the version to what `dxf-viewer` expects (currently `^0.161.0`).

---

## Quick Start (Vanilla JS)

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    html, body { margin: 0; height: 100%; }
    #viewer { width: 100%; height: 100%; }
  </style>
</head>
<body>
  <div id="viewer"></div>
  <input type="file" id="fileInput" accept=".dxf" style="position:fixed;top:10px;left:10px;z-index:10" />

  <script type="module">
    import { DxfViewer } from "dxf-viewer"
    import * as THREE from "three"
    import DxfViewerWorker from "./DxfViewerWorker.js"  // see Web Workers section

    const container = document.getElementById("viewer")

    const viewer = new DxfViewer(container, {
      clearColor: new THREE.Color("#1a1a2e"),
      autoResize: true,
      colorCorrection: true,
    })

    document.getElementById("fileInput").addEventListener("change", async (e) => {
      const file = e.target.files[0]
      if (!file) return
      const url = URL.createObjectURL(file)
      try {
        await viewer.Load({
          url,
          fonts: ["/fonts/Roboto-LightItalic.ttf"],
          workerFactory: DxfViewerWorker,
          progressCbk: (phase, size, total) => {
            console.log(phase, total ? Math.round(size/total*100)+"%" : "...")
          }
        })
      } finally {
        URL.revokeObjectURL(url)
      }
    })
  </script>
</body>
</html>
```

---

## Sizing the Viewer

The viewer creates a `<canvas>` that fills its container element. **The container must have explicit dimensions** — if it has no height, you'll get a blank screen.

### Full-page viewer

```css
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  overflow: hidden;
}

#viewer {
  width: 100%;
  height: 100%;
}
```

### Fixed size

```css
#viewer {
  width: 800px;
  height: 600px;
}
```

### Flex layout (viewer + sidebar)

```css
.layout {
  display: flex;
  height: 100vh;
}

.viewer-container {
  flex: 1;          /* takes all remaining space */
  position: relative;
  min-width: 100px;
  min-height: 100px;
}

.sidebar {
  width: 260px;
  border-left: 1px solid #ddd;
  overflow-y: auto;
}
```

```html
<div class="layout">
  <div class="viewer-container" id="viewer"></div>
  <div class="sidebar" id="layers"></div>
</div>
```

### `autoResize` option

When `autoResize: true` (recommended), the viewer uses a `ResizeObserver` to watch the container and automatically adjusts the WebGL canvas resolution. You never need to call anything manually on resize.

Set `autoResize: false` only if you want to manage this yourself.

---

## Text Rendering

Text in DXF files is **not rendered at all** unless you provide fonts. This is the most common "why is my text missing?" issue.

### How it works

The `fonts` array in the `Load()` call is an ordered list of TTF font URLs. When rendering a character, the viewer tries each font in sequence and uses the first one that contains a glyph for that character. This lets you stack a primary font with fallbacks for different scripts.

### Providing fonts

Pass an array of TTF font URLs to `Load()`:

```js
await viewer.Load({
  url: dxfUrl,
  fonts: [
    "/fonts/Roboto-LightItalic.ttf",           // latin
    "/fonts/NotoSansDisplay-SemiCondensed.ttf", // extended latin + symbols
    "/fonts/HanaMinA.ttf",                      // CJK (Chinese/Japanese/Korean)
    "/fonts/NanumGothic-Regular.ttf",           // Korean
  ],
  workerFactory: DxfViewerWorker,
})
```

### Webpack / Vue CLI — import fonts as URLs

```js
import mainFont  from "./assets/fonts/Roboto-LightItalic.ttf"
import cjkFont   from "./assets/fonts/HanaMinA.ttf"

// These are resolved to hashed asset URLs by the bundler
const fonts = [mainFont, cjkFont]
```

### Vite — import fonts as URLs

```js
const mainFont = new URL("./assets/fonts/Roboto-LightItalic.ttf", import.meta.url).href
const cjkFont  = new URL("./assets/fonts/HanaMinA.ttf",           import.meta.url).href

const fonts = [mainFont, cjkFont]
```

### Recommended font stack

| Purpose | Font |
|---|---|
| Latin / general | Roboto, Open Sans, or any TTF |
| Extended symbols | Noto Sans Display |
| Chinese / Japanese / Korean | HanaMinA (free, large coverage) |
| Korean | NanumGothic |

All of these are freely available from Google Fonts or Noto project. Only TTF format is supported.

---

## Configuration Options

Pass an options object as the second argument to `new DxfViewer(container, options)`.

```js
const viewer = new DxfViewer(container, {
  clearColor: new THREE.Color("#ffffff"),
  autoResize: true,
  colorCorrection: true,
  sceneOptions: {
    wireframeMesh: true,
  },
})
```

| Option | Type | Default | Description |
|---|---|---|---|
| `clearColor` | `THREE.Color` | — | Background color of the canvas. Use `new THREE.Color("#hex")` or `new THREE.Color(r, g, b)`. |
| `autoResize` | `boolean` | — | Automatically resize the canvas when the container size changes. Recommended to set `true`. |
| `colorCorrection` | `boolean` | — | Improves color rendering output through WebGL color space correction. |
| `sceneOptions.wireframeMesh` | `boolean` | — | Renders filled mesh entities as wireframe. Useful for debugging. |

**Background color examples:**

```js
// White background (good for printing)
clearColor: new THREE.Color("#ffffff")

// Dark background (easier on the eyes)
clearColor: new THREE.Color("#1a1a2e")

// Using RGB values (0–1 range)
clearColor: new THREE.Color(0.1, 0.1, 0.15)
```

---

## Loading a DXF File

### `viewer.Load(params)` — async

```js
await viewer.Load({
  url,          // string  — URL to the DXF file (blob: or http:)
  fonts,        // string[] — ordered array of TTF font URLs (omit to skip text)
  progressCbk,  // function(phase, size, totalSize) — optional
  workerFactory // Web Worker constructor — required for parsing
})
```

Throws on error. Always wrap in try/catch.

### Loading a local file

```js
fileInput.addEventListener("change", async (e) => {
  const file = e.target.files[0]
  const url = URL.createObjectURL(file)
  try {
    await viewer.Load({ url, fonts, workerFactory: DxfViewerWorker })
  } catch (err) {
    console.error("Failed to load DXF:", err)
  } finally {
    URL.revokeObjectURL(url)  // always revoke after load completes
  }
})
```

### Loading a remote URL

```js
await viewer.Load({
  url: "https://example.com/drawing.dxf",
  fonts,
  workerFactory: DxfViewerWorker,
})
```

For cross-origin URLs, either configure CORS on your server or use a proxy. The example app uses `https://api.allorigins.win/raw?url=` as a CORS proxy for public URLs.

### Clearing the viewer

```js
viewer.Clear()  // removes current drawing, fires "cleared" event
```

### Destroying the viewer

```js
viewer.Destroy()  // releases WebGL context and all resources
```

Always call `Destroy()` when the component/page is unmounted to prevent memory leaks.

---

## Progress Tracking

The `progressCbk` callback is called repeatedly during loading with the current phase and byte counts.

```js
function onProgress(phase, size, totalSize) {
  // phase:     "font" | "fetch" | "parse" | "prepare"
  // size:      bytes processed so far
  // totalSize: total bytes, or null if unknown
}
```

### Phase meanings

| Phase | What's happening |
|---|---|
| `"font"` | Downloading TTF font files |
| `"fetch"` | Downloading the DXF file |
| `"parse"` | Parsing DXF entities |
| `"prepare"` | Building WebGL geometry |

### Indeterminate vs determinate progress

```js
function onProgress(phase, size, totalSize) {
  if (totalSize === null) {
    showIndeterminateBar()   // unknown size (e.g. chunked HTTP)
  } else {
    showProgress(size / totalSize)  // 0.0 → 1.0
  }
}
```

---

## Layer Management

After loading, you can get all layers and toggle their visibility.

### Get layers

```js
viewer.Subscribe("loaded", () => {
  const layers = viewer.GetLayers(true)  // true = include all layers
  console.log(layers)
  // [{ name: "0", displayName: "0", color: 16711680 }, ...]
})
```

Each layer object:

```ts
{
  name: string        // internal DXF layer name
  displayName: string // human-readable name
  color: number       // RGB as integer (e.g. 16711680 = 0xFF0000 = red)
}
```

### Show / hide a layer

```js
viewer.ShowLayer("WALLS", true)   // show
viewer.ShowLayer("WALLS", false)  // hide
```

### Convert layer color to CSS

```js
function layerColorToCss(colorInt) {
  return "#" + colorInt.toString(16).padStart(6, "0")
}
// layerColorToCss(16711680) → "#ff0000"
```

### Building a layers panel

```js
const layers = viewer.GetLayers(true)

layers.forEach(layer => {
  const row = document.createElement("div")
  
  const swatch = document.createElement("span")
  swatch.style.background = layerColorToCss(layer.color)
  swatch.style.display = "inline-block"
  swatch.style.width = "12px"
  swatch.style.height = "12px"
  swatch.style.marginRight = "6px"

  const checkbox = document.createElement("input")
  checkbox.type = "checkbox"
  checkbox.checked = true
  checkbox.addEventListener("change", () => {
    viewer.ShowLayer(layer.name, checkbox.checked)
  })

  row.appendChild(swatch)
  row.appendChild(checkbox)
  row.appendChild(document.createTextNode(layer.displayName))
  layersPanel.appendChild(row)
})
```

---

## Events

Subscribe to viewer events with `viewer.Subscribe(eventName, handler)`.

```js
viewer.Subscribe("loaded", (e) => {
  console.log("DXF loaded!")
})

viewer.Subscribe("message", (e) => {
  const { level, message } = e.detail
  if (level === DxfViewer.MessageLevel.WARN) console.warn(message)
  if (level === DxfViewer.MessageLevel.ERROR) console.error(message)
})
```

### Full event list

| Event | When it fires |
|---|---|
| `"loaded"` | DXF successfully parsed and rendered |
| `"cleared"` | `viewer.Clear()` was called |
| `"destroyed"` | `viewer.Destroy()` was called |
| `"resized"` | Canvas was resized (container changed size) |
| `"pointerdown"` | Mouse/touch pressed on the canvas |
| `"pointerup"` | Mouse/touch released on the canvas |
| `"viewChanged"` | User panned or zoomed |
| `"message"` | Diagnostic message (warning or error) from the parser |

### Message levels

```js
import { DxfViewer } from "dxf-viewer"

viewer.Subscribe("message", (e) => {
  switch (e.detail.level) {
    case DxfViewer.MessageLevel.WARN:
      // non-fatal issue, drawing may still be partially shown
      break
    case DxfViewer.MessageLevel.ERROR:
      // serious issue
      break
  }
})
```

---

## Web Workers

The viewer uses a Web Worker to parse DXF files off the main thread. You must provide a `workerFactory` in `Load()` or parsing will fail.

### Worker file

Create `DxfViewerWorker.js`:

```js
import { DxfViewer } from "dxf-viewer"
DxfViewer.SetupWorker()
```

### Webpack / Vue CLI

Install `worker-loader`:

```bash
npm install --save-dev worker-loader
```

Import the worker:

```js
import DxfViewerWorker from "worker-loader!./DxfViewerWorker.js"

await viewer.Load({ url, fonts, workerFactory: DxfViewerWorker })
```

Also add to `vue.config.js` (Vue CLI):

```js
transpileDependencies: [
  /[\\\/]node_modules[\\\/]dxf-viewer[\\\/]/
]
```

### Vite

```js
// workerFactory must be a constructor function
const workerFactory = () => new Worker(
  new URL("./DxfViewerWorker.js", import.meta.url),
  { type: "module" }
)

await viewer.Load({ url, fonts, workerFactory })
```

---

## Vue 2 Integration

### DxfViewer.vue

```vue
<template>
  <div class="viewer-container" ref="container">
    <div v-if="loading" class="loading">Loading...</div>
    <div v-if="error" class="error">{{ error }}</div>
  </div>
</template>

<script>
import { DxfViewer } from "dxf-viewer"
import * as THREE from "three"
import DxfViewerWorker from "worker-loader!./DxfViewerWorker"
import mainFont from "./fonts/Roboto-LightItalic.ttf"

export default {
  props: {
    dxfUrl: { type: String, default: null },
  },

  data() {
    return { loading: false, error: null }
  },

  watch: {
    async dxfUrl(url) {
      if (url) await this.load(url)
      else this.viewer.Clear()
    }
  },

  methods: {
    async load(url) {
      this.loading = true
      this.error = null
      try {
        await this.viewer.Load({
          url,
          fonts: [mainFont],
          workerFactory: DxfViewerWorker,
        })
      } catch (e) {
        this.error = e.message
      } finally {
        this.loading = false
      }
    },

    getViewer() { return this.viewer }
  },

  mounted() {
    this.viewer = new DxfViewer(this.$refs.container, {
      clearColor: new THREE.Color("#fff"),
      autoResize: true,
      colorCorrection: true,
    })

    this.viewer.Subscribe("loaded",  e => this.$emit("loaded",  e))
    this.viewer.Subscribe("cleared", e => this.$emit("cleared", e))
    this.viewer.Subscribe("message", e => this.$emit("message", e))

    if (this.dxfUrl) this.load(this.dxfUrl)
  },

  destroyed() {
    this.viewer.Destroy()
  }
}
</script>

<style scoped>
.viewer-container {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 100px;
}
.loading {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
</style>
```

---

## React Integration

```jsx
import { useEffect, useRef } from "react"
import { DxfViewer } from "dxf-viewer"
import * as THREE from "three"
import mainFont from "./fonts/Roboto-LightItalic.ttf"

// Worker factory (Vite)
const workerFactory = () => new Worker(
  new URL("./DxfViewerWorker.js", import.meta.url),
  { type: "module" }
)

export function DxfViewerComponent({ dxfUrl }) {
  const containerRef = useRef(null)
  const viewerRef = useRef(null)

  useEffect(() => {
    const viewer = new DxfViewer(containerRef.current, {
      clearColor: new THREE.Color("#ffffff"),
      autoResize: true,
      colorCorrection: true,
    })
    viewerRef.current = viewer

    return () => {
      viewer.Destroy()
      viewerRef.current = null
    }
  }, [])

  useEffect(() => {
    const viewer = viewerRef.current
    if (!viewer) return

    if (!dxfUrl) {
      viewer.Clear()
      return
    }

    viewer.Load({
      url: dxfUrl,
      fonts: [mainFont],
      workerFactory,
    }).catch(console.error)
  }, [dxfUrl])

  return (
    <div
      ref={containerRef}
      style={{ width: "100%", height: "100%", position: "relative" }}
    />
  )
}
```

Usage:

```jsx
// Local file
const [url, setUrl] = useState(null)

function handleFile(e) {
  const file = e.target.files[0]
  const blobUrl = URL.createObjectURL(file)
  setUrl(prev => { if (prev) URL.revokeObjectURL(prev); return blobUrl })
}

<input type="file" accept=".dxf" onChange={handleFile} />
<div style={{ height: "calc(100vh - 40px)" }}>
  <DxfViewerComponent dxfUrl={url} />
</div>
```

---

## Common Patterns

### Revoke blob URLs

Always revoke blob URLs after loading to free memory:

```js
const url = URL.createObjectURL(file)
try {
  await viewer.Load({ url, fonts, workerFactory })
} finally {
  URL.revokeObjectURL(url)
}
```

### Load from URL query parameter

```js
const params = new URL(location.href).searchParams
const dxfUrl = params.get("dxfUrl")
if (dxfUrl) {
  await viewer.Load({ url: dxfUrl, fonts, workerFactory })
}
```

### Log all viewer warnings and errors

```js
viewer.Subscribe("message", (e) => {
  const prefix = `[dxf-viewer] ${e.detail.level === DxfViewer.MessageLevel.ERROR ? "ERROR" : "WARN"}`
  console.warn(prefix, e.detail.message)
})
```

---

## Troubleshooting

**Text is not showing**
You forgot `fonts` in `Load()`. Text is completely skipped without fonts. Provide at least one TTF font URL.

**Blank canvas**
The container element has no height. Set an explicit `height` (px, %, vh) via CSS. `height: 100%` requires the parent to also have a height.

**`workerFactory` error / parsing hangs**
The Web Worker is not set up. Make sure you have a `DxfViewerWorker.js` that calls `DxfViewer.SetupWorker()` and pass it correctly to `Load()`.

**CORS error when loading remote DXF**
The DXF server doesn't allow cross-origin requests. Either host the file yourself, configure CORS headers on the server, or use a CORS proxy (`https://api.allorigins.win/raw?url=<encoded-url>`).

**Webpack build errors with dxf-viewer**
The package uses modern JS syntax — add it to your transpile list:
```js
// vue.config.js
transpileDependencies: [/node_modules[\\/]dxf-viewer[\\/]/]
```
Or in raw Webpack, include it in your `babel-loader` rule.

**WebGL context lost**
Usually caused by too many WebGL contexts on the page. Call `viewer.Destroy()` when you no longer need a viewer instance.
