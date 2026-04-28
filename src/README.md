# src

Esta carpeta contiene el código fuente completo de la aplicación. El núcleo del proyecto es `components/Blocks.vue`.

`App.vue` es un ejemplo funcional de cómo integrar y configurar `Blocks.vue` en una aplicación completa con modos interactivos.

---

## Blocks.vue

`Blocks.vue` es un componente autocontenido de Vue 3 que renderiza una cuadrícula interactiva de bloques unitarios arrastrables (1×1). Los usuarios pueden colocar bloques, arrastrarlos libremente, agruparlos y cortar conexiones entre bloques agrupados.

Está diseñado como herramienta educativa para explorar el agrupamiento de unidades y el valor posicional. Soporta configuraciones de múltiples grids, modos inline para integración en layouts complejos, y personalización avanzada a través de props.

### Características

- Arrastrar y soltar bloques en una cuadrícula configurable
- Los bloques se agrupan automáticamente al colocarse adyacentes horizontalmente
- Los grupos se colorean según su suma total
- Cortar conexiones entre bloques usando el modo tijeras (✂️)
- Barajar bloques en posiciones aleatorias (🔀)
- Soporte completo de deshacer/rehacer (↶ ↷) y atajos de teclado (`Ctrl+Z`, `Ctrl+Y`)
- El tamaño de las celdas de la cuadrícula se adapta automáticamente al viewport
- La cuadrícula se puede arrastrar libremente dentro del overlay usando la barra de título
- El estado se persiste en `localStorage` entre sesiones; se borra en recarga completa de página
- La toolbar se puede ocultar vía prop para experiencias de solo lectura o restringidas
- Soporte para múltiples grids con posiciones y configuraciones personalizadas
- Modo inline para renderizado directo sin overlays y presentación compacta en la demo de ejercicios
- Etiquetas y conteos configurables por grid

---

### Props

| Prop | Type | Default | Description |
|---|---|---|---|
| `gridColumns` | `Number` | `8` | Número de columnas en la cuadrícula |
| `gridRows` | `Number` | `7` | Número de filas en la cuadrícula |
| `maxBloques` | `Number` | `Infinity` | Número máximo de bloques permitidos en la cuadrícula |
| `maxPorGrupo` | `Number` | `Infinity` | Número máximo de unidades permitidas en un grupo conectado |
| `cantidadBl` | `Number` | `10` | Número de bloques generados al barajar una cuadrícula vacía |
| `initialBlocks` | `Array<Number>` | `[]` | Precarga bloques en el primer montaje. Cada número indica cuántos bloques agrupar en cada zona del grid. |
| `showToolbar` | `Boolean` | `true` | Muestra u oculta la barra de herramientas (+1, 🔀, ✂️, ↶, ↷, 🗑️) |
| `grids` | `Array<Object>` | `[]` | Array de configuraciones de grids múltiples. Cada objeto describe un grid independiente (sumando, resultado, etc.) usado especialmente en ejercicios. Puede incluir `label`, `cols`, `rows`, `position`, `initialBlocks`, `isAnswer`, `showLabel`, `showCount`. |
| `inline` | `Boolean` | `false` | Renderiza los grids directamente sin overlay fullscreen ni FAB |
| `inlineColumns` | `Number` | `0` | Número de columnas CSS para el contenedor inline (por defecto 'auto' = flex-row) |
| `storageKey` | `String` | `'grid_blocks_data'` | Clave de localStorage para persistir el estado de los grids |
| `noSnap` | `Boolean` | `false` | Desactiva el acoplamiento entre bloques (y el modo corte) |
| `showGridLabels` | `Boolean` | `true` | Muestra los números debajo de cada grid (sumandos y respuesta) |

### Configuración de Grids Múltiples

Cuando se proporciona el prop `grids`, el componente renderiza múltiples grids en lugar de uno solo. Cada objeto dentro del array describe un grid independiente y permite armar por separado los elementos de cada ejercicio: por ejemplo, un grid para cada sumando y otro grid para la respuesta.

- `label`: Etiqueta personalizada para el grid (por defecto "Grid N")
- `cols` / `rows`: Dimensiones específicas
- `position`: Posición en el overlay ('top-left', 'center', etc.)
- `initialBlocks`: Bloques iniciales para ese grid
- `isAnswer`: Marca el grid como área de respuesta
- `showLabel` / `showCount`: Controla la visibilidad de etiquetas y conteos

En ejercicios de suma, `grids` se usa para generar los grids de cada sumando y un grid final de respuesta. Cada objeto puede representar un sumando distinto o el espacio donde el alumno debe colocar el resultado. El modo `inline` se utiliza en la demo para mostrar los grids de forma compacta en la misma interfaz, con `inlineColumns` controlando la disposición horizontal/vertical y `showGridLabels` para ajustar la presentación del ejercicio.

```vue
<Blocks :grids="[
  { label: 'Sumando 1', cols: 5, rows: 1, initialBlocks: [3], showLabel: false },
  { label: 'Sumando 2', cols: 5, rows: 1, initialBlocks: [4], showLabel: false },
  { label: 'Respuesta', cols: 10, rows: 2, initialBlocks: [], isAnswer: true, showLabel: true }
]" :inline="true" :inlineColumns="1" />
```

> Si `showToolbar` es `false`, `initialBlocks` debe proporcionarse — de lo contrario no hay forma de agregar bloques a la cuadrícula.

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

`App.vue` es una implementación de referencia completa que envuelve `Blocks.vue` con una interfaz de usuario avanzada. Incluye modos interactivos para demostración y aprendizaje, sirviendo tanto para integrar el componente como para fines educativos.

### Características Principales

- **Modo Inicio**: Pantalla de bienvenida con opciones para acceder a la demo de configuración o a los ejercicios de suma.
- **Demo de Configuración**: Permite configurar grids personalizados mediante un modal interactivo, ajustando etiquetas, dimensiones, posiciones y bloques iniciales. Incluye una vista previa en tiempo real.
- **Ejercicios de Suma**: Serie de ejercicios matemáticos donde los usuarios resuelven problemas de suma utilizando los grids. Cada ejercicio incluye:
  - Enunciados en lenguaje natural.
  - Grids inline para sumandos y respuesta.
  - Verificación automática de respuestas.
  - Indicadores visuales de corrección o error.
  - Opción para mostrar/ocultar números en los grids.
  - Reinicio individual o global de ejercicios.

La aplicación valida configuraciones (por ejemplo, requiere bloques iniciales si la toolbar está oculta) y persiste el estado de los ejercicios en `localStorage`.

### Ejemplo de Integración Mínima

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
