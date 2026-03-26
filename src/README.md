# src

This folder contains the full application source. The core of the project is `components/Blocks.vue`. 

`App.vue` is a working example of how to integrate it.

---

## Blocks.vue

`Blocks.vue` is a self-contained Vue 3 component that renders an interactive grid of draggable unit blocks (1×1). Users can place blocks, drag them freely, snap them together into groups, and cut connections between grouped blocks.

It is intended as an educational tool for exploring unit grouping and place value.

### Features

- Drag-and-drop blocks on a configurable grid
- Blocks snap together when placed adjacent horizontally
- Groups are colored by their total sum
- Cut connections between blocks using scissors mode (✂️)
- Shuffle blocks into random positions (🔀)
- Full undo / redo support (↶ ↷) and keyboard shortcuts (`Ctrl+Z`, `Ctrl+Y`)
- Grid cell size adapts automatically to the viewport
- Grid can be dragged freely within the overlay using the title bar
- State is persisted in `localStorage` across sessions; cleared on full page reload
- Toolbar can be hidden via prop for a read-only or restricted experience

---

### Props

| Prop | Type | Default | Description |
|---|---|---|---|
| `gridColumns` | `Number` | `20` | Number of columns in the grid |
| `gridRows` | `Number` | `15` | Number of rows in the grid |
| `maxBloques` | `Number` | `Infinity` | Maximum number of blocks allowed on the grid |
| `maxPorGrupo` | `Number` | `Infinity` | Maximum units allowed in a single connected group |
| `cantidadBl` | `Number` | `10` | Number of blocks generated when shuffling an empty grid |
| `initialBlocks` | `Array<Number>` | `[]` | Pre-loads blocks on first mount. Each number is a group count placed in a different zone of the grid. See below. |
| `showToolbar` | `Boolean` | `true` | Shows or hides the action toolbar (+1, 🔀, ✂️, ↶, ↷, 🗑️) |

### `initialBlocks` layout

The array defines how many blocks go in each zone, distributed across the grid based on the number of groups:

| Groups | Layout |
|---|---|
| 1 | Full grid |
| 2 | Top / Bottom |
| 3 | Top (full width) / Bottom-left / Bottom-right |
| 4 | Top-left / Top-right / Bottom-left / Bottom-right |
| 5+ | Cycles through the 4-quadrant layout |

```vue
<!-- 3 blocks on top, 5 on the bottom-left, 2 on the bottom-right -->
<Blocks :initialBlocks="[3, 5, 2]" />
```

> If `showToolbar` is `false`, `initialBlocks` must be provided — otherwise there is no way to add blocks to the grid.

---

### Color System

Block color reflects the sum of the connected group:

| Sum | Color |
|---|---|
| 1 | Red |
| 2 | Orange |
| 3 | Green |
| 4 | Blue |
| 5 | Purple |
| 6 | Pink |
| 7 | Cyan |
| 8 | Dark orange |
| 9 | Teal |
| 10–19 | Dark blue (1st ten) |
| 20–29 | Medium blue (2nd ten) |
| 30–39 | Light blue (3rd ten) |
| 40+ | Lighter blue (4th ten) |

When a group has more than 10 units, the first 10 cubes take the color of their decade and the remaining cubes take the color of that remainder value.

---

### Scissors Mode (✂️)

Activating scissors mode shows dashed pink lines between all connected block pairs. To cut a connection, click and drag from one end of a connection line to the opposite end. When the cursor reaches the other vertex, the connection is severed and the groups split.

---

## App.vue

`App.vue` is a reference implementation that wraps `Blocks.vue` with a configuration UI. It exists to demonstrate integration and for testing purposes.

It provides:

- A setup screen where grid dimensions, block limits, initial blocks, and toolbar visibility are configured before starting
- A settings modal (⚙️) that can be opened at any time to adjust parameters and reload the grid without navigating away
- Validation: if the toolbar is disabled, initial blocks must be provided before the session can start. The start / apply button is disabled and a warning is shown until the field is filled.

### Minimal integration example

```vue
<script setup>
import Blocks from './components/Blocks.vue';
</script>

<template>
  <Blocks
    :gridColumns="8"
    :gridRows="7"
    :maxBloques="20"
    :maxPorGrupo="5"
    :initialBlocks="[3, 5, 2]"
    :showToolbar="true"
  />
</template>
```
