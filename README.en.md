# vue3-image-annotation

**English** | [简体中文](./README.md)

[![npm version](https://img.shields.io/npm/v/vue3-image-annotation.svg)](https://www.npmjs.com/package/vue3-image-annotation)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![test](https://img.shields.io/badge/test-vitest-green.svg)]()

A Vue 3 image annotation component based on [fabric.js](https://www.fabricjs.com/) — draw rect / circle / polygon annotations on images, with zoom & pan, undo/redo, data echo and PNG export.

It provides an `<ImageCaption />` component consisting of a thumbnail panel, a main annotation area and a toolbar. It supports rectangle, circle and polygon annotations, selection & dragging, deletion, undo/redo, label management, as well as annotation data echo and retrieval.

A React version is also available: [react-images-annotation](https://github.com/Jamestoms/react-images-annotation) — identical features, interactions and annotation data format.

## Features

- Rectangle / circle drawing by dragging, polygon drawing by clicking point by point (double-click or Enter to finish, ESC to cancel)
- Click to select an existing annotation, hold and drag to move it (automatically constrained within the image; annotation coordinates are recalculated on release)
- Delete annotations (toolbar button or Delete / Backspace key)
- Undo / redo (Ctrl+Z / Ctrl+Shift+Z / Ctrl+Y, recorded independently per image)
- Each annotation can be associated with a label — free input or picking from a preset label list
- Mouse wheel zooming (centered on the cursor), left-button dragging on empty space to pan, fit view
- Annotation coordinates are stored in the image's original pixels, so they never drift at any zoom / pan level
- Responsive to screen size changes
- Echo existing annotation data and continue editing
- TypeScript type declarations provided, while staying compatible with non-TS projects
- Zero UI framework dependency (buttons / dropdowns / popovers are all custom-built — no element-plus or any other component library is introduced, so it never conflicts with the host project's UI framework)

## Screenshots

![demo1 Overall UI and annotation result](https://raw.githubusercontent.com/Jamestoms/image-annotation/HEAD/demo1.png)

![demo2 Annotation editing interactions](https://raw.githubusercontent.com/Jamestoms/image-annotation/HEAD/demo2.png)

## Requirements

- Vue `^3.2.0`
- Node `>= 18` (only needed for local development / build)

`fabric` is the only runtime dependency (dependencies) of the component library — it is installed automatically when you install this component, no manual setup required.

## Installation

```bash
pnpm add vue3-image-annotation
# or
npm install vue3-image-annotation
# or
yarn add vue3-image-annotation
```

## Usage

### Option 1: Global registration (recommended, app.use style)

```ts
// main.ts
import { createApp } from 'vue'
import ImageCaption from 'vue3-image-annotation'
import 'vue3-image-annotation/style.css'
import App from './App.vue'

const app = createApp(App)
app.use(ImageCaption) // register the <ImageCaption /> component globally
app.mount('#app')
```

```vue
<!-- use it directly in any component -->
<template>
  <ImageCaption
    :images="images"
    :labels="['Pedestrian', 'Vehicle', 'Building']"
    style="height: 640px"
  >
    <!-- optional: custom action area (right side of the toolbar), e.g. a save button -->
    <template #actions>
      <button class="ic-btn ic-btn--primary" @click="handleSave">Save</button>
    </template>
  </ImageCaption>
</template>
```

### Option 2: Import on demand

```vue
<script setup lang="ts">
import { ImageCaption } from 'vue3-image-annotation'
import 'vue3-image-annotation/style.css'
</script>

<template>
  <ImageCaption :images="images" />
</template>
```

> In non-TypeScript projects, just use it as a normal component (the component is compiled to JS; type declarations are optional).

## Component API

### Props

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `images` | `ImageItem[]` | Yes | `[]` | Image list; if an item contains `annotations`, they are echoed after loading |
| `labels` | `string[]` | No | `[]` | Preset label list, selectable while editing a label (free input is also allowed) |
| `downloadable` | `boolean` | No | `false` | Whether to enable the download feature (a download button appears in the toolbar, exporting a PNG image with annotations) |

### Events

| Name | Payload | Description |
| --- | --- | --- |
| `change` | `(imageId: string, annotations: AnnotationData[])` | Fired when annotation data changes (add / delete / move / label edit / undo / redo / clear) |
| `download` | `(imageId: string, filename: string)` | Fired after a successful export triggered by the download button (requires `downloadable`) |

### Slots

| Name | Description |
| --- | --- |
| `actions` | Custom action area rendered at the far right of the toolbar, suitable for host business buttons such as save / submit / next step. You can reuse the built-in button style classes `ic-btn` (default), `ic-btn--primary` (primary), `ic-btn--danger-plain` (danger), or fully customize your own |

### Best Practices for Saving Data

The component does not ship a built-in save button (when and how to persist data is up to the host). Two common patterns are provided:

**Pattern 1: Real-time saving** — listen to the `change` event and submit automatically on every change:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import type { AnnotationData } from 'vue3-image-annotation'

const images = ref([
  { id: 'img-1', url: 'https://example.com/a.jpg' },
])

async function handleChange(imageId: string, annotations: AnnotationData[]) {
  await fetch(`/api/annotations/${imageId}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(annotations),
  })
}
</script>

<template>
  <ImageCaption :images="images" @change="handleChange" style="height: 640px" />
</template>
```

**Pattern 2: Manual saving** — put your own button in the `actions` slot and read the data from a ref on click (it can be submitted along with other form fields on the page):

```vue
<script setup lang="ts">
import { ref } from 'vue'

const captionRef = ref()

async function handleSave() {
  const imageId = captionRef.value.currentImageId
  const annotations = captionRef.value.getAnnotations()
  await fetch(`/api/annotations/${imageId}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(annotations),
  })
}
</script>

<template>
  <ImageCaption ref="captionRef" :images="images" style="height: 640px">
    <template #actions>
      <button class="ic-btn ic-btn--primary" @click="handleSave">Save</button>
    </template>
  </ImageCaption>
</template>
```

### Component Methods (via ref)

```ts
import { ref } from 'vue'

const captionRef = ref()

// Get annotations of the current image
captionRef.value.getAnnotations()
// Get annotations of a specific image
captionRef.value.getAnnotations('img-1')
// Get annotations of all images: { [imageId]: AnnotationData[] }
captionRef.value.getAllAnnotations()
// Undo / redo
captionRef.value.undo()
captionRef.value.redo()
// Delete the currently selected annotation
captionRef.value.deleteSelected()
// Clear annotations of the current image
captionRef.value.clearCurrent()
// View control
captionRef.value.zoomIn()
captionRef.value.zoomOut()
captionRef.value.fitView()
// Export a PNG dataURL of the current image (with annotations) at the image's original pixel size — useful for custom upload scenarios
const dataUrl = captionRef.value.exportImage()
// Current image id (use together with getAnnotations for manual saving)
const imageId = captionRef.value.currentImageId
// Switch image
captionRef.value.switchImage('img-2')
```

### Data Types

```ts
type AnnotationType = 'rect' | 'circle' | 'polygon'

interface Point {
  x: number
  y: number
}

interface AnnotationData {
  id: string
  type: AnnotationType
  label: string
  /** rect / circle: top-left corner of the bounding box, plus width and height */
  x?: number
  y?: number
  width?: number
  height?: number
  /** polygon: vertex list */
  points?: Point[]
}

interface ImageItem {
  id: string
  url: string
  name?: string
  /** existing annotations, echoed after loading */
  annotations?: AnnotationData[]
}
```

> All coordinates are **in the image's original pixel space** (independent of canvas zoom), ready to be stored and converted by your business logic.

## Interactions

| Action | Behavior |
| --- | --- |
| Mouse wheel | Zoom in / out centered on the cursor |
| Left-button drag on empty space | Pan the image |
| Click an annotation | Select it (a label editing popover appears — rename the label or delete the annotation) |
| Drag a selected annotation | Move it (constrained within the image); coordinates are updated automatically on release |
| Delete / Backspace | Delete the selected annotation |
| Ctrl+Z / Ctrl+Shift+Z (or Ctrl+Y) | Undo / redo |
| Clear button | First click enters a red confirmation state; clicking again within 3 seconds clears the annotations, otherwise it resets automatically |
| ESC | Cancel drawing / deselect / close the popover |
| Enter | Finish polygon drawing |
| Double-click / click the starting vertex | Finish polygon drawing |

> Annotations do not support control-handle scaling (to avoid coordinate conversion errors) — delete and redraw, or undo, if you make a mistake.

## Related Versions

- [react-images-annotation](https://github.com/Jamestoms/react-images-annotation): the React version of this component (npm package name `react-images-annotation`, supports React 18 / 19) — identical features, interactions and UI, with fully compatible annotation data (`AnnotationData` / `ImageItem`), so the same annotation data can be used directly across Vue3 / React projects.

## Local Development

This project uses **pnpm** as its package manager.

```bash
# Install dependencies
pnpm install

# Start the demo page (examples/)
pnpm dev

# Build the library (output in dist/)
pnpm build

# Type checking
pnpm typecheck
```

## License

MIT
