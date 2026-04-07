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

The recommended pattern splits the viewer into two components:

1. `DxfViewer.vue` — wraps the raw `DxfViewer` class, manages loading state, re-emits events
2. `ViewerPage.vue` — orchestrates the viewer component with a layers sidebar

### DxfViewer.vue (low-level wrapper)

This component matches the pattern used in the official example app. It accepts a `dxfUrl` prop, handles loading/error state, and proxies all DXF events with a `dxf-` prefix.

```vue
<template>
  <div class="canvasContainer" ref="canvasContainer">
    <!-- Loading spinner -->
    <div v-if="isLoading" class="loading-overlay">Loading...</div>

    <!-- Progress bar -->
    <div v-if="progress !== null" class="progress">
      <div
        class="progress-bar"
        :class="{ indeterminate: progress < 0 }"
        :style="progress >= 0 ? { width: (progress * 100) + '%' } : {}"
      ></div>
      <div v-if="progressText" class="progress-text">{{ progressText }}</div>
    </div>

    <!-- Error display -->
    <div v-if="error !== null" class="error">
      Error: {{ error }}
    </div>
  </div>
</template>

<script>
import { DxfViewer } from "dxf-viewer"
import * as THREE from "three"
import DxfViewerWorker from "worker-loader!./DxfViewerWorker"

/**
 * Emits all DxfViewer events prefixed with "dxf-":
 *   dxf-loaded, dxf-cleared, dxf-destroyed, dxf-resized,
 *   dxf-pointerdown, dxf-pointerup, dxf-viewChanged, dxf-message
 */
export default {
  name: "DxfViewer",

  props: {
    /** URL of the DXF file to load. Set to null to clear. */
    dxfUrl: {
      default: null,
    },
    /**
     * Array of TTF font file URLs.
     * Fonts are tried in order until a glyph is found.
     * Text is not rendered if this is null or empty.
     */
    fonts: {
      default: null,
    },
    /** DxfViewer constructor options. */
    options: {
      default() {
        return {
          clearColor: new THREE.Color("#ffffff"),
          autoResize: true,
          colorCorrection: true,
          sceneOptions: {
            wireframeMesh: true,
          },
        }
      },
    },
  },

  data() {
    return {
      isLoading: false,
      progress: null,
      progressText: null,
      curProgressPhase: null,
      error: null,
    }
  },

  watch: {
    async dxfUrl(dxfUrl) {
      if (dxfUrl !== null) {
        await this._Load(dxfUrl)
      } else {
        this.dxfViewer.Clear()
        this.error = null
        this.isLoading = false
        this.progress = null
      }
    },
  },

  methods: {
    async _Load(url) {
      this.isLoading = true
      this.error = null
      try {
        await this.dxfViewer.Load({
          url,
          fonts: this.fonts,
          progressCbk: this._OnProgress.bind(this),
          workerFactory: DxfViewerWorker,
        })
      } catch (error) {
        console.warn(error)
        this.error = error.toString()
      } finally {
        this.isLoading = false
        this.progressText = null
        this.progress = null
        this.curProgressPhase = null
      }
    },

    /** Expose the underlying DxfViewer instance for parent components. */
    GetViewer() {
      return this.dxfViewer
    },

    _OnProgress(phase, size, totalSize) {
      if (phase !== this.curProgressPhase) {
        switch (phase) {
          case "font":    this.progressText = "Fetching fonts...";            break
          case "fetch":   this.progressText = "Fetching file...";             break
          case "parse":   this.progressText = "Parsing file...";              break
          case "prepare": this.progressText = "Preparing rendering data...";  break
        }
        this.curProgressPhase = phase
      }
      this.progress = totalSize === null ? -1 : size / totalSize
    },
  },

  mounted() {
    this.dxfViewer = new DxfViewer(this.$refs.canvasContainer, this.options)

    // Proxy all DXF viewer events as Vue events with "dxf-" prefix
    const eventNames = [
      "loaded", "cleared", "destroyed", "resized",
      "pointerdown", "pointerup", "viewChanged", "message",
    ]
    for (const eventName of eventNames) {
      this.dxfViewer.Subscribe(eventName, (e) => this.$emit("dxf-" + eventName, e))
    }

    // Load immediately if url was already set
    if (this.dxfUrl) {
      this._Load(this.dxfUrl)
    }
  },

  destroyed() {
    this.dxfViewer.Destroy()
    this.dxfViewer = null
  },
}
</script>

<style scoped>
.canvasContainer {
  position: relative;
  width: 100%;
  height: 100%;
  min-width: 100px;
  min-height: 100px;
}

.loading-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
  background: rgba(255, 255, 255, 0.6);
}

.progress {
  position: absolute;
  z-index: 20;
  width: 90%;
  margin: 20px 5%;
}

.progress-bar {
  height: 4px;
  background: #1976d2;
  transition: width 0.2s;
}

.progress-bar.indeterminate {
  width: 40%;
  animation: indeterminate 1.4s infinite linear;
}

@keyframes indeterminate {
  0%   { margin-left: -40%; }
  100% { margin-left: 100%; }
}

.progress-text {
  margin-top: 6px;
  font-size: 13px;
  color: #333;
  text-align: center;
}

.error {
  position: absolute;
  inset: 0;
  z-index: 20;
  padding: 24px;
  color: #c00;
  font-weight: 500;
}
</style>
```

### ViewerPage.vue (viewer + layers panel)

This page-level component wires together the `DxfViewer` wrapper with a layers toggle UI and message notifications — exactly the pattern used in the example app.

```vue
<template>
  <div class="viewer-page">
    <!-- DXF canvas -->
    <div class="viewer-area">
      <DxfViewer
        ref="viewer"
        :dxfUrl="dxfUrl"
        :fonts="fonts"
        @dxf-loaded="_OnLoaded"
        @dxf-cleared="_OnCleared"
        @dxf-message="_OnMessage"
      />
    </div>

    <!-- Layers sidebar -->
    <div class="layers-panel" v-if="layers !== null">
      <h3>Layers</h3>

      <label class="layer-row">
        <input type="checkbox" v-model="allVisible" @change="_ToggleAll" />
        <em>All layers</em>
      </label>

      <label
        class="layer-row"
        v-for="layer in layers"
        :key="layer.name"
      >
        <span
          class="color-swatch"
          :style="{ background: _LayerCss(layer.color) }"
        ></span>
        <input
          type="checkbox"
          :checked="layer.isVisible"
          @change="e => _ToggleLayer(layer, e.target.checked)"
        />
        {{ layer.displayName }}
      </label>
    </div>
  </div>
</template>

<script>
import DxfViewer from "./DxfViewer.vue"
import { DxfViewer as _DxfViewer } from "dxf-viewer"

// Font imports (webpack resolves these to asset URLs)
import mainFont from "@/assets/fonts/Roboto-LightItalic.ttf"
import aux1Font from "@/assets/fonts/NotoSansDisplay-SemiCondensedLightItalic.ttf"
import aux2Font from "@/assets/fonts/HanaMinA.ttf"
import aux3Font from "@/assets/fonts/NanumGothic-Regular.ttf"

export default {
  name: "ViewerPage",
  components: { DxfViewer },

  props: {
    dxfUrl: { type: String, default: null },
  },

  data() {
    return {
      layers: null,
      allVisible: true,
    }
  },

  created() {
    // Build font stack once at creation time
    this.fonts = [mainFont, aux1Font, aux2Font, aux3Font]
  },

  methods: {
    _OnLoaded() {
      // Fetch all layers (including hidden) after load
      const layers = this.$refs.viewer.GetViewer().GetLayers(true)
      // Add reactive isVisible flag to each layer
      layers.forEach(lyr => { lyr.isVisible = true })
      this.layers = layers
      this.allVisible = true
    },

    _OnCleared() {
      this.layers = null
    },

    _OnMessage(e) {
      const { level, message } = e.detail
      const isError = level === _DxfViewer.MessageLevel.ERROR
      console[isError ? "error" : "warn"]("[dxf-viewer]", message)
      // Replace with your notification library:
      // this.$toast[isError ? 'error' : 'warning'](message)
    },

    _ToggleLayer(layer, newState) {
      layer.isVisible = newState
      this.$refs.viewer.GetViewer().ShowLayer(layer.name, newState)
    },

    _ToggleAll(e) {
      const newState = e.target.checked
      if (this.layers) {
        for (const layer of this.layers) {
          if (layer.isVisible !== newState) {
            this._ToggleLayer(layer, newState)
          }
        }
      }
    },

    _LayerCss(colorInt) {
      return "#" + colorInt.toString(16).padStart(6, "0")
    },
  },
}
</script>

<style scoped>
.viewer-page {
  display: flex;
  width: 100%;
  height: 100%;
}

.viewer-area {
  flex: 1;
  position: relative;
  min-width: 0;
}

.layers-panel {
  width: 260px;
  border-left: 1px solid #ddd;
  overflow-y: auto;
  padding: 8px;
  flex-shrink: 0;
}

.layers-panel h3 {
  margin: 0 0 8px;
  font-size: 14px;
  font-weight: 600;
  color: #555;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.layer-row {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 3px 0;
  cursor: pointer;
  font-size: 13px;
}

.color-swatch {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 2px;
  flex-shrink: 0;
}
</style>
```

### Using ViewerPage in App.vue

```vue
<template>
  <div style="height: 100vh; display: flex; flex-direction: column;">
    <header style="padding: 8px 16px; background: #1976d2; color: #fff;">
      <input type="file" accept=".dxf" @change="_OnFileSelected" />
    </header>
    <div style="flex: 1; overflow: hidden;">
      <ViewerPage :dxfUrl="dxfUrl" />
    </div>
  </div>
</template>

<script>
import ViewerPage from "@/components/ViewerPage"

export default {
  components: { ViewerPage },
  data() {
    return { dxfUrl: null, _blobUrl: null }
  },
  methods: {
    _OnFileSelected(e) {
      const file = e.target.files[0]
      if (!file) return
      if (this._blobUrl) URL.revokeObjectURL(this._blobUrl)
      this._blobUrl = URL.createObjectURL(file)
      this.dxfUrl = this._blobUrl
    },
  },
  beforeDestroy() {
    if (this._blobUrl) URL.revokeObjectURL(this._blobUrl)
  },
}
</script>
```

### vue.config.js for dxf-viewer

```js
// vue.config.js
module.exports = {
  transpileDependencies: [
    // dxf-viewer uses modern JS that must be transpiled for older browsers
    /[\\\/]node_modules[\\\/]dxf-viewer[\\\/]/,
  ],
}
```

---

## React Integration

A React functional component wrapping the viewer follows the same lifecycle rules as any imperative DOM library: create in a `useEffect` with an empty dependency array, destroy in the cleanup function, and respond to prop changes in a separate `useEffect`.

### DxfViewer React component (Vite)

```jsx
// DxfViewerComponent.jsx
import { useEffect, useRef, useState, useCallback } from "react"
import { DxfViewer } from "dxf-viewer"
import * as THREE from "three"

// Import fonts as URLs via Vite
const FONTS = [
  new URL("./fonts/Roboto-LightItalic.ttf",                         import.meta.url).href,
  new URL("./fonts/NotoSansDisplay-SemiCondensedLightItalic.ttf",   import.meta.url).href,
  new URL("./fonts/HanaMinA.ttf",                                   import.meta.url).href,
  new URL("./fonts/NanumGothic-Regular.ttf",                        import.meta.url).href,
]

// Worker factory — wrap in a function so each Load() call gets a fresh worker
const workerFactory = () =>
  new Worker(new URL("./DxfViewerWorker.js", import.meta.url), { type: "module" })

/**
 * Props:
 *   dxfUrl   — string | null   URL to load. Set to null to clear.
 *   onLayers — function(layers) called after loading with the layer list
 */
export function DxfViewerComponent({ dxfUrl, onLayers }) {
  const containerRef = useRef(null)
  const viewerRef    = useRef(null)

  const [loading,      setLoading]      = useState(false)
  const [error,        setError]        = useState(null)
  const [progress,     setProgress]     = useState(null)
  const [progressText, setProgressText] = useState(null)

  // Create/destroy viewer
  useEffect(() => {
    const viewer = new DxfViewer(containerRef.current, {
      clearColor:      new THREE.Color("#ffffff"),
      autoResize:      true,
      colorCorrection: true,
    })
    viewerRef.current = viewer

    viewer.Subscribe("message", (e) => {
      const isError = e.detail.level === DxfViewer.MessageLevel.ERROR
      console[isError ? "error" : "warn"]("[dxf-viewer]", e.detail.message)
    })

    return () => {
      viewer.Destroy()
      viewerRef.current = null
    }
  }, [])

  // Progress callback
  const handleProgress = useCallback((phase, size, totalSize) => {
    const texts = { font: "Fetching fonts...", fetch: "Fetching file...",
                    parse: "Parsing file...", prepare: "Preparing rendering data..." }
    setProgressText(texts[phase] ?? phase)
    setProgress(totalSize === null ? -1 : size / totalSize)
  }, [])

  // Load/clear when dxfUrl changes
  useEffect(() => {
    const viewer = viewerRef.current
    if (!viewer) return

    if (!dxfUrl) {
      viewer.Clear()
      setError(null)
      setLoading(false)
      setProgress(null)
      return
    }

    let cancelled = false
    setLoading(true)
    setError(null)
    setProgress(null)

    viewer.Load({
      url:           dxfUrl,
      fonts:         FONTS,
      workerFactory,
      progressCbk:   handleProgress,
    })
      .then(() => {
        if (cancelled) return
        if (onLayers) onLayers(viewer.GetLayers(true))
      })
      .catch((err) => {
        if (cancelled) return
        setError(err.message)
      })
      .finally(() => {
        if (!cancelled) {
          setLoading(false)
          setProgress(null)
          setProgressText(null)
        }
      })

    return () => { cancelled = true }
  }, [dxfUrl, handleProgress, onLayers])

  return (
    <div ref={containerRef} style={{ position: "relative", width: "100%", height: "100%" }}>
      {loading && (
        <div style={overlayStyle}>
          {progress !== null && (
            <div style={{ width: "80%", marginBottom: 8 }}>
              <div style={{
                height: 4, background: "#1976d2",
                width: progress < 0 ? "100%" : `${progress * 100}%`,
                transition: "width 0.2s",
              }} />
              {progressText && (
                <div style={{ fontSize: 13, color: "#555", textAlign: "center", marginTop: 4 }}>
                  {progressText}
                </div>
              )}
            </div>
          )}
          <div>Loading...</div>
        </div>
      )}
      {error && (
        <div style={{ ...overlayStyle, color: "#c00", background: "rgba(255,255,255,0.85)" }}>
          Error: {error}
        </div>
      )}
    </div>
  )
}

const overlayStyle = {
  position: "absolute", inset: 0, zIndex: 10,
  display: "flex", flexDirection: "column",
  alignItems: "center", justifyContent: "center",
  background: "rgba(255,255,255,0.7)",
}
```

### Full page with layers sidebar (React)

```jsx
// App.jsx
import { useState, useCallback } from "react"
import { DxfViewerComponent } from "./DxfViewerComponent"

function LayerRow({ layer, onToggle }) {
  const color = "#" + layer.color.toString(16).padStart(6, "0")
  return (
    <label style={{ display: "flex", alignItems: "center", gap: 6, padding: "3px 0", cursor: "pointer" }}>
      <span style={{ width: 10, height: 10, background: color, borderRadius: 2, flexShrink: 0 }} />
      <input
        type="checkbox"
        checked={layer.isVisible ?? true}
        onChange={e => onToggle(layer, e.target.checked)}
      />
      <span style={{ fontSize: 13 }}>{layer.displayName}</span>
    </label>
  )
}

export default function App() {
  const [dxfUrl, setDxfUrl]   = useState(null)
  const [blobUrl, setBlobUrl] = useState(null)
  const [layers, setLayers]   = useState(null)
  const [viewerRef]           = useState({})

  function handleFile(e) {
    const file = e.target.files[0]
    if (!file) return
    if (blobUrl) URL.revokeObjectURL(blobUrl)
    const url = URL.createObjectURL(file)
    setBlobUrl(url)
    setDxfUrl(url)
    setLayers(null)
  }

  const handleLayers = useCallback((rawLayers) => {
    setLayers(rawLayers.map(l => ({ ...l, isVisible: true })))
  }, [])

  function toggleLayer(layer, newState) {
    layer.isVisible = newState
    // Access underlying viewer through the component ref if needed
    // viewerRef.current?.ShowLayer(layer.name, newState)
    setLayers(prev => prev.map(l => l.name === layer.name ? { ...l, isVisible: newState } : l))
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", height: "100vh" }}>
      <header style={{ padding: "8px 16px", background: "#1976d2", color: "#fff", flexShrink: 0 }}>
        <input type="file" accept=".dxf" onChange={handleFile} />
      </header>

      <div style={{ flex: 1, display: "flex", overflow: "hidden" }}>
        <div style={{ flex: 1, position: "relative" }}>
          <DxfViewerComponent dxfUrl={dxfUrl} onLayers={handleLayers} />
        </div>

        {layers && (
          <div style={{ width: 240, borderLeft: "1px solid #ddd", overflowY: "auto", padding: 8 }}>
            <div style={{ fontWeight: 600, marginBottom: 8, fontSize: 13, textTransform: "uppercase" }}>
              Layers
            </div>
            {layers.map(layer => (
              <LayerRow key={layer.name} layer={layer} onToggle={toggleLayer} />
            ))}
          </div>
        )}
      </div>
    </div>
  )
}
```

> **Note on layer visibility with React:** Because `ShowLayer()` is a method on the viewer instance, you need access to the `DxfViewer` object. The cleanest approach is to expose a ref from your wrapper component or use a shared ref pattern. The example above shows the state-tracking side; wire it to `viewer.ShowLayer()` via a forwarded ref.

---

## Vanilla JS Integration

A complete, framework-free integration. All you need is an HTML file and a bundler (or native ES modules in a modern browser).

### Project structure

```
my-dxf-app/
  index.html
  main.js
  DxfViewerWorker.js
  fonts/
    Roboto-LightItalic.ttf
```

### DxfViewerWorker.js

```js
import { DxfViewer } from "dxf-viewer"
DxfViewer.SetupWorker()
```

### main.js

```js
import { DxfViewer } from "dxf-viewer"
import * as THREE from "three"

// ── Setup ──────────────────────────────────────────────────────────────────
const container = document.getElementById("viewer")
const statusEl  = document.getElementById("status")
const layersEl  = document.getElementById("layers")

const viewer = new DxfViewer(container, {
  clearColor:      new THREE.Color("#1e1e2e"),
  autoResize:      true,
  colorCorrection: true,
})

// ── Worker factory ─────────────────────────────────────────────────────────
// For Vite/modern bundlers:
const workerFactory = () =>
  new Worker(new URL("./DxfViewerWorker.js", import.meta.url), { type: "module" })

// ── Fonts ─────────────────────────────────────────────────────────────────
const fonts = [
  new URL("./fonts/Roboto-LightItalic.ttf", import.meta.url).href,
]

// ── Events ─────────────────────────────────────────────────────────────────
viewer.Subscribe("loaded", () => {
  renderLayers(viewer.GetLayers(true))
})

viewer.Subscribe("cleared", () => {
  layersEl.innerHTML = ""
})

viewer.Subscribe("message", (e) => {
  const { level, message } = e.detail
  const isError = level === DxfViewer.MessageLevel.ERROR
  console[isError ? "error" : "warn"]("[dxf]", message)
})

// ── File input ─────────────────────────────────────────────────────────────
let currentBlobUrl = null

document.getElementById("fileInput").addEventListener("change", async (e) => {
  const file = e.target.files[0]
  if (!file) return

  // Revoke previous blob URL
  if (currentBlobUrl) {
    URL.revokeObjectURL(currentBlobUrl)
    currentBlobUrl = null
  }

  currentBlobUrl = URL.createObjectURL(file)
  setStatus("Loading...")

  try {
    await viewer.Load({
      url:          currentBlobUrl,
      fonts,
      workerFactory,
      progressCbk:  onProgress,
    })
    setStatus("")
  } catch (err) {
    setStatus("Error: " + err.message)
    console.error(err)
  }
})

// ── Progress ───────────────────────────────────────────────────────────────
const phaseLabels = {
  font:    "Fetching fonts...",
  fetch:   "Fetching file...",
  parse:   "Parsing file...",
  prepare: "Preparing rendering data...",
}

function onProgress(phase, size, totalSize) {
  const label = phaseLabels[phase] ?? phase
  if (totalSize === null) {
    setStatus(label)
  } else {
    const pct = Math.round((size / totalSize) * 100)
    setStatus(`${label} ${pct}%`)
  }
}

function setStatus(text) {
  statusEl.textContent = text
}

// ── Layers ─────────────────────────────────────────────────────────────────
function layerColorToCss(colorInt) {
  return "#" + colorInt.toString(16).padStart(6, "0")
}

function renderLayers(layers) {
  layersEl.innerHTML = ""

  const heading = document.createElement("div")
  heading.textContent = "Layers"
  heading.style.cssText = "font-weight:600;margin-bottom:8px;font-size:13px;text-transform:uppercase;"
  layersEl.appendChild(heading)

  layers.forEach(layer => {
    const row = document.createElement("label")
    row.style.cssText = "display:flex;align-items:center;gap:6px;padding:2px 0;cursor:pointer;"

    const swatch = document.createElement("span")
    swatch.style.cssText = `display:inline-block;width:10px;height:10px;border-radius:2px;background:${layerColorToCss(layer.color)};`

    const cb = document.createElement("input")
    cb.type    = "checkbox"
    cb.checked = true
    cb.addEventListener("change", () => viewer.ShowLayer(layer.name, cb.checked))

    const label = document.createElement("span")
    label.textContent = layer.displayName
    label.style.fontSize = "13px"

    row.appendChild(swatch)
    row.appendChild(cb)
    row.appendChild(label)
    layersEl.appendChild(row)
  })
}

// ── Cleanup ────────────────────────────────────────────────────────────────
window.addEventListener("beforeunload", () => {
  viewer.Destroy()
  if (currentBlobUrl) URL.revokeObjectURL(currentBlobUrl)
})
```

### index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>DXF Viewer</title>
  <style>
    * { box-sizing: border-box; }

    html, body {
      margin: 0;
      padding: 0;
      height: 100%;
      overflow: hidden;
      font-family: system-ui, sans-serif;
    }

    .app {
      display: flex;
      flex-direction: column;
      height: 100%;
    }

    .toolbar {
      display: flex;
      align-items: center;
      gap: 12px;
      padding: 8px 16px;
      background: #1976d2;
      color: #fff;
      flex-shrink: 0;
    }

    #status {
      font-size: 13px;
      opacity: 0.85;
    }

    .main {
      display: flex;
      flex: 1;
      overflow: hidden;
    }

    #viewer {
      flex: 1;
      position: relative;
      min-width: 0;
    }

    #layers {
      width: 240px;
      border-left: 1px solid #ddd;
      overflow-y: auto;
      padding: 8px;
      flex-shrink: 0;
    }
  </style>
</head>
<body>
  <div class="app">
    <div class="toolbar">
      <label>
        <strong>Open DXF:</strong>
        <input id="fileInput" type="file" accept=".dxf" />
      </label>
      <span id="status"></span>
    </div>
    <div class="main">
      <div id="viewer"></div>
      <div id="layers"></div>
    </div>
  </div>

  <script type="module" src="./main.js"></script>
</body>
</html>
```

---

## Common Patterns & Tips

### Always revoke blob URLs

Blob URLs created from `File` objects hold a reference to the file data in memory. You must call `URL.revokeObjectURL()` when the URL is no longer needed. The best place is after `Load()` resolves (or rejects), not before — the viewer fetches the URL asynchronously.

```js
const url = URL.createObjectURL(file)
try {
  await viewer.Load({ url, fonts, workerFactory })
} catch (err) {
  console.error(err)
} finally {
  URL.revokeObjectURL(url)  // always clean up
}
```

If you store the blob URL in state (to pass it as a prop), revoke it when the user selects a new file or clears:

```js
let currentBlobUrl = null

function loadFile(file) {
  if (currentBlobUrl) {
    URL.revokeObjectURL(currentBlobUrl)
  }
  currentBlobUrl = URL.createObjectURL(file)
  return currentBlobUrl
}
```

### Always destroy the viewer on unmount

WebGL contexts are a limited resource. If you create a viewer in a component, always destroy it when the component is removed:

```js
// Vue 2
destroyed() {
  this.dxfViewer.Destroy()
}

// Vue 3
onUnmounted(() => {
  viewer.Destroy()
})

// React
useEffect(() => {
  const viewer = new DxfViewer(container, options)
  return () => viewer.Destroy()   // cleanup on unmount
}, [])
```

### Load from a URL query parameter

A common pattern for shareable DXF links:

```js
// On page load, check for a ?dxfUrl= parameter
const searchParams = new URL(location.href).searchParams
const dxfUrl = searchParams.get("dxfUrl")

if (dxfUrl && URL.canParse(dxfUrl)) {
  await viewer.Load({ url: dxfUrl, fonts, workerFactory })
}
```

To generate a shareable link:

```js
const shareUrl = new URL(location.href)
shareUrl.searchParams.set("dxfUrl", "https://example.com/drawing.dxf")
console.log(shareUrl.toString())
```

### Using a CORS proxy for remote URLs

When loading DXF files from a third-party server that does not send CORS headers, the browser will block the request. Options:

1. **Own proxy** — serve the DXF through your own backend, which fetches and re-serves it.
2. **AllOrigins** (development/demo only) — prepend `https://api.allorigins.win/raw?url=`:

```js
function proxiedUrl(rawUrl) {
  return "https://api.allorigins.win/raw?url=" + encodeURIComponent(rawUrl)
}

await viewer.Load({ url: proxiedUrl(userInputUrl), fonts, workerFactory })
```

> Do not rely on third-party CORS proxies in production. They can go offline, rate-limit, or be blocked.

### Surface viewer warnings in the UI

The `"message"` event fires for non-fatal issues the parser encounters (unsupported entities, missing blocks, etc.). In production, surface these to help users understand incomplete renders:

```js
viewer.Subscribe("message", (e) => {
  const { level, message } = e.detail
  const isError = level === DxfViewer.MessageLevel.ERROR

  // Log to console always
  console[isError ? "error" : "warn"]("[dxf-viewer]", message)

  // Optionally show a toast/snackbar to the user
  showNotification({ type: isError ? "error" : "warning", text: message })
})
```

### Replace a file without destroying the viewer

Call `viewer.Clear()` and then `viewer.Load(...)` again. There is no need to re-create the viewer:

```js
async function replaceFile(newFile) {
  viewer.Clear()

  const url = URL.createObjectURL(newFile)
  try {
    await viewer.Load({ url, fonts, workerFactory })
  } finally {
    URL.revokeObjectURL(url)
  }
}
```

### React: don't create the viewer inside render

Always create the viewer in a `useEffect` (runs after mount), never in the component body or render function — those run before the DOM element is attached.

```js
// CORRECT
useEffect(() => {
  const viewer = new DxfViewer(containerRef.current, options)
  // ...
}, [])

// WRONG — containerRef.current is null during render
const viewer = new DxfViewer(containerRef.current, options)  // DO NOT DO THIS
```

### Controlling background color dynamically

You can change the background after instantiation by passing a different `clearColor` at construction time, or by creating a new viewer. Currently there is no setter for `clearColor` after construction — bake the desired background into your options at startup.

```js
// Light mode
const viewer = new DxfViewer(container, { clearColor: new THREE.Color("#ffffff"), ... })

// Dark mode
const viewer = new DxfViewer(container, { clearColor: new THREE.Color("#1a1a2e"), ... })
```

---

## Troubleshooting

### Text is not showing

**Cause:** `fonts` was not passed to `Load()`, or the font array is empty.

**Fix:** Provide at least one TTF font URL in the `fonts` array. Text rendering is entirely skipped if `fonts` is `null` or `[]`.

```js
await viewer.Load({
  url,
  fonts: ["/fonts/Roboto-LightItalic.ttf"],   // required for text
  workerFactory,
})
```

---

### Canvas is blank / viewer shows nothing

**Likely cause:** The container element has zero height.

The viewer fills its container. If the container has no CSS height, the canvas is 0px tall and nothing renders.

**Fix:** Give the container an explicit height:

```css
/* Option A: fill viewport */
html, body { height: 100%; margin: 0; }
#viewer { width: 100%; height: 100%; }

/* Option B: fixed size */
#viewer { width: 800px; height: 600px; }

/* Option C: flex child */
.layout { display: flex; height: 100vh; }
#viewer { flex: 1; }
```

Check in DevTools that the container's computed height is greater than 0.

---

### Worker not set up / `workerFactory is not a constructor`

**Cause:** `workerFactory` was not passed, was `undefined`, or the worker file is wrong.

**Fix:**

1. Create `DxfViewerWorker.js`:
   ```js
   import { DxfViewer } from "dxf-viewer"
   DxfViewer.SetupWorker()
   ```

2. Import it correctly for your bundler:
   - **Webpack/Vue CLI:** `import DxfViewerWorker from "worker-loader!./DxfViewerWorker.js"`
   - **Vite:** `() => new Worker(new URL("./DxfViewerWorker.js", import.meta.url), { type: "module" })`

3. Pass it to `Load()`:
   ```js
   await viewer.Load({ url, fonts, workerFactory: DxfViewerWorker })
   ```

---

### CORS error when loading a remote DXF

**Error in console:** `Access to fetch at 'https://...' has been blocked by CORS policy`

**Cause:** The remote server does not include the `Access-Control-Allow-Origin` header.

**Fixes (in order of preference):**
1. Host the DXF file on your own domain or a CDN you control and configure CORS headers.
2. Proxy the request through your own backend server.
3. For development/demos only: prepend `https://api.allorigins.win/raw?url=` to the URL.

---

### Webpack build errors / `Unexpected token`

**Cause:** `dxf-viewer` ships untranspiled modern JavaScript. Webpack (especially older Vue CLI 4/5 setups) doesn't transpile `node_modules` by default.

**Fix for Vue CLI (`vue.config.js`):**
```js
module.exports = {
  transpileDependencies: [
    /[\\\/]node_modules[\\\/]dxf-viewer[\\\/]/,
  ],
}
```

**Fix for raw Webpack:**
```js
// webpack.config.js
module: {
  rules: [
    {
      test: /\.js$/,
      include: [
        path.resolve(__dirname, "src"),
        path.resolve(__dirname, "node_modules/dxf-viewer"),
      ],
      use: "babel-loader",
    },
  ],
},
```

---

### WebGL context lost

**Symptom:** Drawing disappears, browser console shows `WebGL: CONTEXT_LOST_WEBGL`.

**Causes:**
- Too many WebGL contexts on the page (browsers limit this to ~8–16).
- GPU driver reset.

**Fix:**
- Call `viewer.Destroy()` whenever you remove the viewer from the page. This releases the context.
- Avoid creating multiple viewer instances simultaneously — one at a time is best.

---

### `viewer.GetLayers()` returns an empty array

**Cause:** Called before the DXF has finished loading.

**Fix:** Only call `GetLayers()` inside a `"loaded"` event handler (or after `await viewer.Load(...)` resolves):

```js
viewer.Subscribe("loaded", () => {
  const layers = viewer.GetLayers(true)   // safe here
  // ...
})

// Or:
await viewer.Load({ url, fonts, workerFactory })
const layers = viewer.GetLayers(true)     // safe after await
```
