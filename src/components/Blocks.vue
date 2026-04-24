<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";

// ===== PROPS =====
const props = defineProps({
  maxBloques: { type: Number, default: Infinity },
  maxPorGrupo: { type: Number, default: Infinity },
  cantidadBl: { type: Number, default: 10 },
  gridColumns: { type: Number, default: 8 },
  gridRows: { type: Number, default: 7 },
  initialBlocks: { type: Array, default: () => [] },
  showToolbar: { type: Boolean, default: true },
  // Array: [{ label, cols, rows, initialBlocks, position }]
  // position: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right' | 'center'
  grids: { type: Array, default: () => [] },
  // inline: renderiza los grids directamente sin overlay fullscreen ni FAB
  inline: { type: Boolean, default: false },
  // inlineColumns: número de columnas CSS para el container inline (default 'auto' = flex-row)
  inlineColumns: { type: Number, default: 0 },
  // storageKey: clave de localStorage (permite múltiples instancias independientes)
  storageKey: { type: String, default: 'grid_blocks_data' },
  // noSnap: desactiva el acoplamiento entre bloques (y el modo corte)
  noSnap: { type: Boolean, default: false },
  // showGridLabels: muestra el número debajo de cada grid (sumandos y respuesta)
  showGridLabels: { type: Boolean, default: true },
});

// ===== GRID DEFINITIONS =====
const POSITIONS = [
  "top-left",
  "top-right",
  "bottom-left",
  "bottom-right",
  "center",
  "left",
  "right",
  "top",
  "bottom",
];

const gridDefs = ref(
  props.grids.length > 0
    ? props.grids.map((g, i) => ({
        id: i,
        label: g.label ?? `Grid ${i + 1}`,
        cols: g.cols ?? props.gridColumns,
        rows: g.rows ?? props.gridRows,
        position: g.position ?? POSITIONS[i] ?? "top-left",
        initialBlocks: g.initialBlocks ?? [],
        isAnswer: g.isAnswer ?? false,
        showLabel: g.showLabel ?? true,
        showCount: g.showCount ?? true,
      }))
    : [
        {
          id: 0,
          label: "Grid 1",
          cols: props.gridColumns,
          rows: props.gridRows,
          position: "top-left",
          initialBlocks: props.initialBlocks,
        },
      ],
);

// ===== GRID COUNT =====
const gridCount = ref(gridDefs.value.length);

// Verdadero después del primer montaje — distingue recarga de página (false) de remount por parámetros (true)
let _hasBeenMounted = false;

// ===== CONFIGURACIÓN =====
const BLOCK_COLOR = "#EF4444";

// ===== COLORES =====
const COLORS_BY_SUM = {
  1: "#EF4444",
  2: "#F59E0B",
  3: "#10B981",
  4: "#3B82F6",
  5: "#8B5CF6",
  6: "#EC4899",
  7: "#06B6D4",
  8: "#F97316",
  9: "#14B8A6",
};
const DECENA_COLORS = ["#1E40AF", "#3B82F6", "#60A5FA", "#93C5FD"];
const getColorBySum = (sum) => COLORS_BY_SUM[sum] ?? "#64748b";

// Colorea las unidades de un grupo según su suma total.
// Suma 1-9: todos los cubitos toman el color de esa suma.
// Suma ≥ 10: cada grupo de 10 cubitos (contando de izquierda a derecha)
// toma un tono de azul distinto; los cubitos del resto final toman
// el color de su valor (ej. resto 3 → verde).
const colorGroupBySum = (group) => {
  let totalSum = 0;
  group.forEach((id) => {
    const b = blocks.value.find((s) => s.id === id);
    if (b) totalSum += b.value;
  });

  if (totalSum < 10) {
    const color = getColorBySum(totalSum);
    group.forEach((id) => {
      const b = blocks.value.find((s) => s.id === id);
      if (b?.units)
        b.units.forEach((u) => {
          u.color = color;
        });
    });
    return;
  }

  // Recolectar unidades ordenadas de izquierda a derecha
  const units = [];
  group.forEach((id) => {
    const b = blocks.value.find((s) => s.id === id);
    if (!b?.units) return;
    b.units.forEach((u, i) => {
      units.push({
        blockId: b.id,
        unitIndex: i,
        x: b.gridX + u.relX,
        baseColor: u.baseColor || u.color,
      });
    });
  });
  units.sort((a, b) => a.x - b.x);

  const total = units.length;
  const resto = total % 10;
  const inicioResto = total - resto;

  units.forEach((u, index) => {
    const b = blocks.value.find((s) => s.id === u.blockId);
    if (!b) return;
    const decena = Math.floor(index / 10);
    const color =
      index >= inicioResto && resto > 0
        ? getColorBySum(resto)
        : (DECENA_COLORS[decena] ?? DECENA_COLORS[DECENA_COLORS.length - 1]);
    b.units[u.unitIndex].color = color;
  });
};

// ===== TAMAÑO DE GRILLA (fallback global) =====
const COLS = props.gridColumns;
const ROWS = props.gridRows;

// ===== CELL SIZE ADAPTATIVO (por grid) =====
const MAX_CELL_SIZE = 45;
const MIN_CELL_SIZE = 16;
const PADDING_H = 40;
const PADDING_V = 80;

const viewportW = ref(window.innerWidth);
const viewportH = ref(window.innerHeight);

// Para modo inline: tamaño del contenedor
const containerW = ref(0);
const containerH = ref(0);
const containerRef = ref(null);

const updateContainerSize = () => {
  if (containerRef.value) {
    containerW.value = containerRef.value.clientWidth;
    containerH.value = containerRef.value.clientHeight;
  }
};

// Helpers por gridId — cada grid usa sus propias dims
const getGridCols = (gridId) =>
  gridDefs.value.find((g) => g.id === gridId)?.cols ?? COLS;
const getGridRows = (gridId) =>
  gridDefs.value.find((g) => g.id === gridId)?.rows ?? ROWS;

const getCellSize = (gridId) => {
  const cols = getGridCols(gridId);
  const rows = getGridRows(gridId);
  if (props.inline) {
    if (props.inlineColumns > 0) {
      // Tamaño fijo basado en el número más grande entre los sumandos
      // Para grids de 1 fila, queremos celdas grandes y proporcionales
      const maxCols = Math.max(...gridDefs.value.map(g => g.cols));

      // Tamaño de la celda en pixeles.
      const cellSize = Math.max(24, Math.min(40, Math.floor(360 / maxCols)));
      return cellSize;
    }
    // Layout flex-row original — usar altura del container
    const availW = containerW.value > 0 ? containerW.value : 300;
    const availH = containerH.value > 0 ? containerH.value : 200;
    const numGrids = gridDefs.value.length || 1;
    const perGridW = Math.floor((availW - 10 * (numGrids + 1)) / numGrids);
    const def = gridDefs.value.find(g => g.id === gridId);
    const titleH = def?.showLabel ? 24 : 0;
    const maxByCols = Math.floor(perGridW / cols);
    const maxByRows = Math.floor((availH - titleH) / rows);
    return Math.max(Math.min(maxByCols, maxByRows, MAX_CELL_SIZE), MIN_CELL_SIZE);
  }
  const maxW = Math.floor((viewportW.value * 0.44) / cols);
  const maxH = Math.floor((viewportH.value * 0.78) / rows);
  return Math.max(Math.min(maxW, maxH, MAX_CELL_SIZE), MIN_CELL_SIZE);
};

const getGridWidth = (gridId) => getGridCols(gridId) * getCellSize(gridId);
const getGridHeight = (gridId) => getGridRows(gridId) * getCellSize(gridId);

// Compatibilidad con código legado
const CELL_SIZE = computed(() => getCellSize(gridDefs.value[0]?.id ?? 0));
const gridWidth = computed(() => getGridWidth(gridDefs.value[0]?.id ?? 0));
const gridHeight = computed(() => getGridHeight(gridDefs.value[0]?.id ?? 0));

// ===== GRID COLLISION CHECK (no longer needed for fixed grids) =====
const doesGridOverlap = () => false;

const onResize = () => {
  viewportW.value = window.innerWidth;
  viewportH.value = window.innerHeight;
  if (props.inline) updateContainerSize();
  borderStyleCache.clear();
};

// ===== FACTORY DE BLOQUE =====
// Todos los bloques son siempre 1×1
const createBlock = (gridX, gridY, id, gridId = 0) => ({
  id,
  type: "uno",
  gridId,
  gridX,
  gridY,
  value: 1,
  color: BLOCK_COLOR,
  isConnected: false,
  width: 1,
  height: 1,
  units: [{ relX: 0, relY: 0, color: BLOCK_COLOR, baseColor: BLOCK_COLOR }],
});

// ===== ESTADO PRINCIPAL =====
const showOverlay = ref(false);
const blocks = ref([]); // ← antes: shapes
const selectedBlockId = ref(null); // ← antes: selectedShapeId
const draggedBlockId = ref(null); // ← antes: draggingId
const draggedBlockGrid = ref(0);
// Posición del mouse para el ghost visual durante drag
const ghostPos = ref({ x: 0, y: 0, visible: false });
const isDragging = ref(false);
const adjacentGroups = ref([]); // ← antes: shapeGroups
const cuttingMode = ref(false);
const connections = ref([]);
const connectionsToCut = ref([]);

// ===== DRAG STATE (antes: dragStart) =====
// Separado en dos objetos con responsabilidades claras:
//   dragState   → coordenadas del drag (posición de ratón y ancla)
//   groupContext → qué bloques se están moviendo y sus posiciones originales
const dragState = ref(null); // { mouseX, mouseY, anchorX, anchorY, lastValidDx, lastValidDy }
const groupContext = ref(null); // { ids: [], initialPositions: [{ id, gridX, gridY }] }

// ===== POSICIÓN FIJA DE CADA GRID =====
// Los grids no son arrastrables — su posición viene de gridDef.position
const panelDrag = ref(null); // mantenido para compatibilidad, no se usa

// Calcula el estilo CSS fixed para cada posición
const getGridPositionStyle = (gridId) => {
  const def = gridDefs.value.find((g) => g.id === gridId);
  const pos = def?.position ?? "top-left";
  const M = "16px";
  const gw = getGridWidth(gridId);
  const gh = getGridHeight(gridId);
  const styles = {
    "top-left": { top: M, left: M, bottom: "auto", right: "auto" },
    "top-right": { top: M, right: M, bottom: "auto", left: "auto" },
    "bottom-left": { bottom: M, left: M, top: "auto", right: "auto" },
    "bottom-right": { bottom: M, right: M, top: "auto", left: "auto" },
    center: {
      top: `calc(50vh - ${gh / 2}px)`,
      left: `calc(50vw - ${gw / 2}px)`,
      bottom: "auto",
      right: "auto",
    },
    left: {
      top: `calc(50vh - ${gh / 2}px)`,
      left: M,
      bottom: "auto",
      right: "auto",
    },
    right: {
      top: `calc(50vh - ${gh / 2}px)`,
      right: M,
      bottom: "auto",
      left: "auto",
    },
    top: {
      top: M,
      left: `calc(50vw - ${gw / 2}px)`,
      bottom: "auto",
      right: "auto",
    },
    bottom: {
      bottom: M,
      left: `calc(50vw - ${gw / 2}px)`,
      top: "auto",
      right: "auto",
    },
  };
  return styles[pos] ?? styles["top-left"];
};

// ===== TOAST =====
const toastMessage = ref("");
const showToast = ref(false);
let toastTimer = null;

const showToastNotification = (message) => {
  toastMessage.value = message;
  showToast.value = true;
  if (toastTimer) clearTimeout(toastTimer);
  toastTimer = setTimeout(() => {
    showToast.value = false;
  }, 3000);
};

// ===== UNDO/REDO =====
const history = ref([]);
const currentStep = ref(-1);
const maxHistorySize = 10;

const saveToHistory = () => {
  const state = JSON.parse(JSON.stringify(blocks.value));
  if (currentStep.value < history.value.length - 1)
    history.value = history.value.slice(0, currentStep.value + 1);
  history.value.push(state);
  if (history.value.length > maxHistorySize) {
    history.value.shift();
  } else {
    currentStep.value++;
  }
};

const undo = () => {
  if (currentStep.value > 0) {
    currentStep.value--;
    blocks.value = JSON.parse(JSON.stringify(history.value[currentStep.value]));
    refreshGroupsNow();
    saveToStorage();
  }
};

const redo = () => {
  if (currentStep.value < history.value.length - 1) {
    currentStep.value++;
    blocks.value = JSON.parse(JSON.stringify(history.value[currentStep.value]));
    refreshGroupsNow();
    saveToStorage();
  }
};

// ===== CACHÉ Y TIMERS =====
const borderStyleCache = new Map();
let refreshGroupsTimer = null; // ← antes: checkAdjacentTimer
let nextId = 1;

// ===== PERSISTENCIA =====
const saveToStorage = () => {
  try {
    localStorage.setItem(props.storageKey, JSON.stringify(blocks.value));
  } catch (e) {
    console.error("Error guardando:", e);
  }
};

const loadFromStorage = () => {
  try {
    const saved = localStorage.getItem(props.storageKey);
    if (!saved) return;

    const normalized = JSON.parse(saved).map((b) => ({
      ...b,
      gridId: b.gridId ?? 0,
      width: 1,
      height: 1,
      units: b.units || [
        { relX: 0, relY: 0, color: BLOCK_COLOR, baseColor: BLOCK_COLOR },
      ],
    }));

    const inside = [];
    const outside = [];
    normalized.forEach((b) => {
      const gCols = getGridCols(b.gridId ?? 0);
      const gRows = getGridRows(b.gridId ?? 0);
      const out =
        b.gridX < 0 ||
        b.gridY < 0 ||
        b.gridX + 1 > gCols ||
        b.gridY + 1 > gRows;
      (out ? outside : inside).push(b);
    });

    outside.forEach((b) => {
      const pos = findAvailablePosition(0);
      if (pos) inside.push({ ...b, gridX: pos.x, gridY: pos.y });
    });

    blocks.value = inside;
    if (blocks.value.length > 0)
      nextId = Math.max(...blocks.value.map((s) => s.id), 0) + 1;

    refreshGroups();
  } catch (e) {
    console.error("Error cargando:", e);
  }
};

// ===== UTILIDADES DE GRILLA =====
// Todos los bloques son 1×1 — no se necesita getShapeDimensions
const getOccupiedCells = (block) => [{ x: block.gridX, y: block.gridY }];

const getGridBlocks = (gridId) =>
  blocks.value.filter((b) => b.gridId === gridId);
const getGridBlock = (gridId, id) =>
  blocks.value.find((b) => b.gridId === gridId && b.id === id);
const gridIndices = computed(() => gridDefs.value.map((g) => g.id));

const isPositionAvailable = (gridId, gridX, gridY, excludeId = null) => {
  const cols = getGridCols(gridId);
  const rows = getGridRows(gridId);
  if (gridX >= cols || gridY >= rows || gridX < 0 || gridY < 0) return false;
  return !blocks.value.find((b) => {
    if (b.gridId !== gridId) return false;
    if (excludeId && b.id === excludeId) return false;
    return b.gridX === gridX && b.gridY === gridY;
  });
};

// Busca la primera celda libre con al menos 1 celda de margen alrededor.
// Si no hay ninguna, acepta cualquier celda libre.
const findAvailablePosition = (gridId = 0) => {
  const cols = getGridCols(gridId);
  const rows = getGridRows(gridId);
  // Pasada 1: con margen
  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      if (!isPositionAvailable(gridId, x, y)) continue;
      let hasMargin = true;
      outer: for (let dy = -1; dy <= 1; dy++) {
        for (let dx = -1; dx <= 1; dx++) {
          if (dx === 0 && dy === 0) continue;
          const cx = x + dx,
            cy = y + dy;
          if (cx < 0 || cx >= cols || cy < 0 || cy >= rows) continue;
          if (!isPositionAvailable(gridId, cx, cy)) {
            hasMargin = false;
            break outer;
          }
        }
      }
      if (hasMargin) return { x, y };
    }
  }
  // Pasada 2: cualquier celda libre
  for (let y = 0; y < rows; y++)
    for (let x = 0; x < cols; x++)
      if (isPositionAvailable(gridId, x, y)) return { x, y };
  return null;
};

// ===== BORDES INTELIGENTES =====
// Cada bloque oculta el lado donde tiene un vecino inmediato.
// Como ambos bloques ocultan el lado compartido, no hay borde doble.
const getHiddenBorders = computed(() => {
  const hidden = new Map();
  blocks.value.forEach((block) => {
    const h = { top: false, right: false, bottom: false, left: false };
    blocks.value.forEach((other) => {
      if (other.id === block.id || other.gridId !== block.gridId) return;
      if (other.gridX === block.gridX && other.gridY === block.gridY - 1)
        h.top = true;
      if (other.gridX === block.gridX && other.gridY === block.gridY + 1)
        h.bottom = true;
      if (other.gridX === block.gridX - 1 && other.gridY === block.gridY)
        h.left = true;
      if (other.gridX === block.gridX + 1 && other.gridY === block.gridY)
        h.right = true;
    });
    hidden.set(block.id, h);
  });
  return hidden;
});

const getBorderStyle = (block) => {
  const cs = CELL_SIZE.value;
  const h = getHiddenBorders.value.get(block.id);
  if (!h) return "";

  const cacheKey = `${block.id}-${cs}-${+h.top}-${+h.right}-${+h.bottom}-${+h.left}`;
  if (borderStyleCache.has(cacheKey)) return borderStyleCache.get(cacheKey);

  const parts = [];
  if (!h.top) parts.push(`M 0 0 L ${cs} 0`);
  if (!h.right) parts.push(`M ${cs} 0 L ${cs} ${cs}`);
  if (!h.bottom) parts.push(`M ${cs} ${cs} L 0 ${cs}`);
  if (!h.left) parts.push(`M 0 ${cs} L 0 0`);

  const svg = `data:image/svg+xml,${encodeURIComponent(
    `<svg xmlns="http://www.w3.org/2000/svg" width="${cs}" height="${cs}">
      <path d="${parts.join(" ")}" stroke="#1e293b" stroke-width="3" fill="none" stroke-linecap="square" stroke-linejoin="miter"/>
    </svg>`,
  )}`;

  const result = `url("${svg}")`;
  borderStyleCache.set(cacheKey, result);
  return result;
};

// ===== DETECCIÓN DE ADYACENCIA =====
// Solo horizontal (mismo Y, X separados por exactamente 1)
const areAdjacent = (b1, b2) =>
  b1.gridId === b2.gridId &&
  Math.abs(b1.gridX - b2.gridX) === 1 &&
  b1.gridY === b2.gridY;

// ===== RECÁLCULO DE GRUPOS (antes: checkAdjacentBlocks / checkAdjacentBlocksImmediate) =====
// Con { immediate: true } se ejecuta sin debounce (para Undo/Redo)
const refreshGroups = ({ immediate = false } = {}) => {
  if (!immediate) {
    if (refreshGroupsTimer) clearTimeout(refreshGroupsTimer);
    refreshGroupsTimer = setTimeout(() => refreshGroupsNow(), 50);
  } else {
    refreshGroupsNow();
  }
};

const refreshGroupsNow = () => {
  if (props.noSnap) {
    // Sin acoplamiento: todos los bloques aislados, sin conexiones
    blocks.value.forEach(b => { b.isConnected = false; b.groupSum = null; delete b.visuallySeparated; });
    adjacentGroups.value = [];
    connections.value = [];
    saveToStorage();
    return;
  }

  // Construir grupos por adyacencia
  const groups = [];
  blocks.value.forEach((b1) => {
    blocks.value.forEach((b2) => {
      if (b1.id >= b2.id) return;
      if (b1.gridId !== b2.gridId) return;
      if (!areAdjacent(b1, b2)) return;
      const group = groups.find((g) => g.includes(b1.id) || g.includes(b2.id));
      if (group) {
        if (!group.includes(b1.id)) group.push(b1.id);
        if (!group.includes(b2.id)) group.push(b2.id);
      } else {
        groups.push([b1.id, b2.id]);
      }
    });
  });

  // Consolidar grupos solapados
  let changed = true;
  while (changed) {
    changed = false;
    for (let i = 0; i < groups.length; i++) {
      for (let j = i + 1; j < groups.length; j++) {
        if (groups[i].some((id) => groups[j].includes(id))) {
          groups[i] = [...new Set([...groups[i], ...groups[j]])];
          groups.splice(j, 1);
          changed = true;
          break;
        }
      }
      if (changed) break;
    }
  }

  adjacentGroups.value = groups;

  // Restaurar colores base antes de colorear por grupo
  blocks.value.forEach((b) => {
    if (b.units)
      b.units.forEach((u) => {
        u.color = u.baseColor || u.color;
      });
  });

  // Asignar estado de grupo a cada bloque
  blocks.value.forEach((block) => {
    const group = groups.find((g) => g.includes(block.id));
    if (group) {
      block.isConnected = true;
      block.groupSum = group.reduce((acc, id) => {
        const b = blocks.value.find((x) => x.id === id);
        return acc + (b ? b.value : 0);
      }, 0);
      if (group[0] === block.id) colorGroupBySum(group);
    } else {
      block.isConnected = false;
      block.groupSum = null;
    }
  });

  updateConnections();
  saveToStorage();
  borderStyleCache.clear();
};

// ===== CONEXIONES (líneas para modo corte) — por grid =====
// Cada grid solo renderiza sus propias conexiones (filtrado en template por gridId)
const updateConnections = () => {
  const all = [];
  const seen = new Set();

  adjacentGroups.value.forEach((group) => {
    if (group.length < 2) return;
    for (let i = 0; i < group.length; i++) {
      for (let j = i + 1; j < group.length; j++) {
        getConnectionsBetween(group[i], group[j]).forEach((c) => {
          const key = `${Math.min(c.shape1Id, c.shape2Id)}-${Math.max(c.shape1Id, c.shape2Id)}-${c.x1}-${c.y1}-${c.x2}-${c.y2}`;
          if (!seen.has(key)) {
            seen.add(key);
            all.push(c);
          }
        });
      }
    }
  });

  connections.value = all;
};

// Obtiene solo las conexiones que pertenecen a un gridId específico
const getConnectionsForGrid = (gridId) =>
  connections.value.filter((c) => {
    const b1 = blocks.value.find((b) => b.id === c.shape1Id);
    return b1?.gridId === gridId;
  });

const getConnectionsBetween = (id1, id2) => {
  const b1 = blocks.value.find((b) => b.id === id1);
  const b2 = blocks.value.find((b) => b.id === id2);
  if (!b1 || !b2 || b1.gridId !== b2.gridId) return [];

  const cs = CELL_SIZE.value;
  const list = [];

  // Conexión vertical (bloques lado a lado → línea vertical entre ellos)
  if (b1.gridY === b2.gridY && Math.abs(b1.gridX - b2.gridX) === 1) {
    const minX = Math.min(b1.gridX, b2.gridX);
    list.push({
      x1: (minX + 1) * cs,
      y1: b1.gridY * cs,
      x2: (minX + 1) * cs,
      y2: (b1.gridY + 1) * cs,
      horizontal: false,
      shape1Id: id1,
      shape2Id: id2,
    });
  }

  return list;
};

// ===== SEPARACIÓN DE BLOQUES =====
const getConnectedSubgroup = (startId, excludeId) => {
  const visited = new Set();
  const subgroup = [];
  const queue = [startId];

  while (queue.length > 0) {
    const currentId = queue.shift();
    if (visited.has(currentId)) continue;
    visited.add(currentId);
    subgroup.push(currentId);

    const current = blocks.value.find((b) => b.id === currentId);
    if (!current) continue;

    blocks.value.forEach((other) => {
      if (
        visited.has(other.id) ||
        other.id === excludeId ||
        other.gridId !== current.gridId
      )
        return;
      if (areAdjacent(current, other)) queue.push(other.id);
    });
  }

  return subgroup;
};

const isPositionAvailableExcluding = (gridId, gridX, gridY, excludeIds) => {
  const cols = getGridCols(gridId);
  const rows = getGridRows(gridId);
  if (gridX < 0 || gridY < 0 || gridX >= cols || gridY >= rows) return false;
  return !blocks.value.find(
    (b) =>
      b.gridId === gridId &&
      !excludeIds.includes(b.id) &&
      b.gridX === gridX &&
      b.gridY === gridY,
  );
};

const separateBlocks = (block1, block2) => {
  const sub1 = getConnectedSubgroup(block1.id, block2.id);
  const sub2 = getConnectedSubgroup(block2.id, block1.id);

  const subgroupToMove = sub1.length <= sub2.length ? sub1 : sub2;
  const otherSubgroup = sub1.length <= sub2.length ? sub2 : sub1;

  const tryMove = (subgroup, directions) => {
    for (const { dx, dy } of directions) {
      let canMove = true;
      const newPositions = [];

      for (const id of subgroup) {
        const b = blocks.value.find((s) => s.id === id);
        if (!b) continue;
        const nx = b.gridX + dx;
        const ny = b.gridY + dy;
        if (!isPositionAvailableExcluding(block1.gridId, nx, ny, subgroup)) {
          canMove = false;
          break;
        }
        newPositions.push({ id, nx, ny });
      }

      if (canMove && newPositions.length > 0) {
        // Verificar que el movimiento no cause reconexión inmediata
        const wouldReconnect = newPositions.some(({ nx, ny }) =>
          blocks.value.some((b) => {
            if (b.gridId !== block1.gridId) return false;
            if (subgroup.includes(b.id)) return false;
            return Math.abs(b.gridX - nx) === 1 && b.gridY === ny;
          }),
        );

        if (!wouldReconnect) {
          newPositions.forEach(({ id, nx, ny }) => {
            const b = blocks.value.find((s) => s.id === id);
            if (b) {
              b.gridX = nx;
              b.gridY = ny;
            }
          });
          return true;
        }
      }
    }
    return false;
  };

  const sides = [
    { dx: 1, dy: 0 },
    { dx: -1, dy: 0 },
  ];
  const vertical = [
    { dx: 0, dy: 1 },
    { dx: 0, dy: -1 },
  ];

  if (tryMove(subgroupToMove, sides)) return true;
  if (tryMove(otherSubgroup, sides)) return true;
  if (tryMove(subgroupToMove, vertical)) return true;
  if (tryMove(otherSubgroup, vertical)) return true;

  // Fallback: separación visual (sin movimiento físico)
  subgroupToMove.forEach((id) => {
    const b = blocks.value.find((s) => s.id === id);
    if (b) b.visuallySeparated = true;
  });
  return true;
};

// ===== MODO CORTE (SNAP-TO-LINE) =====
const cutState = ref({
  // ← antes: snapToLineState
  active: false,
  currentConnection: null,
  startProgress: null,
  currentProgress: 0,
  startVertex: null,
  snappedPosition: null,
  cutConnections: new Set(),
  reachedOppositeVertex: false,
});

const cutConnection = (connection) => {
  if (!connection.shape1Id || !connection.shape2Id) return;
  const b1 = blocks.value.find((b) => b.id === connection.shape1Id);
  const b2 = blocks.value.find((b) => b.id === connection.shape2Id);
  if (!b1 || !b2) return;
  separateBlocks(b1, b2);
  cutState.value.cutConnections.add(
    `${connection.shape1Id}-${connection.shape2Id}`,
  );
  refreshGroups();
};

const findNearestConnection = (x, y) => {
  let nearest = null;
  let minDist = 40;

  connections.value.forEach((conn) => {
    const dx = conn.x2 - conn.x1;
    const dy = conn.y2 - conn.y1;
    const lenSq = dx * dx + dy * dy;
    if (lenSq === 0) return;

    let t = ((x - conn.x1) * dx + (y - conn.y1) * dy) / lenSq;
    t = Math.max(0, Math.min(1, t));

    const cx = conn.x1 + t * dx;
    const cy = conn.y1 + t * dy;
    const dist = Math.sqrt((x - cx) ** 2 + (y - cy) ** 2);

    if (dist < minDist) {
      minDist = dist;
      nearest = {
        connection: conn,
        distance: dist,
        progress: t,
        snappedPoint: { x: cx, y: cy },
      };
    }
  });

  return nearest;
};

const isConnectionCut = (conn) => {
  const k1 = `${conn.shape1Id}-${conn.shape2Id}`;
  const k2 = `${conn.shape2Id}-${conn.shape1Id}`;
  return (
    cutState.value.cutConnections.has(k1) ||
    cutState.value.cutConnections.has(k2)
  );
};

const getClosestVertex = (progress) => (progress < 0.5 ? "start" : "end");
const reachedOtherVertex = (startVertex, progress) =>
  startVertex === "start" ? progress > 0.9 : progress < 0.1;

const doLinesIntersect = (x1, y1, x2, y2, x3, y3, x4, y4) => {
  const denom = (x1 - x2) * (y3 - y4) - (y1 - y2) * (x3 - x4);
  if (Math.abs(denom) < 0.0001) return false;
  const t = ((x1 - x3) * (y3 - y4) - (y1 - y3) * (x3 - x4)) / denom;
  const u = -((x1 - x2) * (y1 - y3) - (y1 - y2) * (x1 - x3)) / denom;
  return t >= 0 && t <= 1 && u >= 0 && u <= 1;
};

const getGridRelativePos = (e) => {
  const hit = findGridAtPoint(e.clientX, e.clientY);
  if (!hit) return null;
  const gridEl = document.querySelector(
    `.grid-center[data-grid-index='${hit.gridIndex}'] .blocks-layer-transparent`,
  );
  if (!gridEl) return null;
  const rect = gridEl.getBoundingClientRect();
  return { x: e.clientX - rect.left, y: e.clientY - rect.top };
};

const handleSnapCutStart = (e) => {
  if (!cuttingMode.value) return;
  const pos = getGridRelativePos(e);
  if (!pos) return;
  const nearest = findNearestConnection(pos.x, pos.y);
  if (!nearest) return;

  const startVertex = getClosestVertex(nearest.progress);
  cutState.value = {
    active: true,
    currentConnection: nearest.connection,
    startProgress: nearest.progress,
    currentProgress: nearest.progress,
    startVertex,
    snappedPosition: nearest.snappedPoint,
    cutConnections: new Set(),
    reachedOppositeVertex: false,
  };
};

const handleSnapCutMove = (e) => {
  if (!cuttingMode.value || !cutState.value.active) return;
  const pos = getGridRelativePos(e);
  if (!pos) return;
  const nearest = findNearestConnection(pos.x, pos.y);
  if (!nearest) return;

  const prevConn = cutState.value.currentConnection;
  const prevPos = cutState.value.snappedPosition;

  cutState.value.snappedPosition = nearest.snappedPoint;
  cutState.value.currentProgress = nearest.progress;
  cutState.value.currentConnection = nearest.connection;

  // Cortar conexiones que el trazo cruzó
  if (prevPos && prevConn) {
    const dx = nearest.snappedPoint.x - prevPos.x;
    const dy = nearest.snappedPoint.y - prevPos.y;
    if (Math.sqrt(dx * dx + dy * dy) > 20) {
      connections.value.forEach((conn) => {
        if (conn === nearest.connection || conn === prevConn) return;
        if (
          doLinesIntersect(
            prevPos.x,
            prevPos.y,
            nearest.snappedPoint.x,
            nearest.snappedPoint.y,
            conn.x1,
            conn.y1,
            conn.x2,
            conn.y2,
          ) &&
          !isConnectionCut(conn)
        ) {
          cutConnection(conn);
        }
      });
    }
  }

  if (cutState.value.startVertex) {
    if (reachedOtherVertex(cutState.value.startVertex, nearest.progress))
      cutState.value.reachedOppositeVertex = true;
  }

  if (prevConn && prevConn !== nearest.connection) {
    cutState.value.startProgress = nearest.progress;
    cutState.value.startVertex = getClosestVertex(nearest.progress);
    cutState.value.reachedOppositeVertex = false;
  }
};

const handleSnapCutEnd = () => {
  if (!cuttingMode.value) return;
  if (
    cutState.value.currentConnection &&
    cutState.value.reachedOppositeVertex &&
    !isConnectionCut(cutState.value.currentConnection)
  ) {
    cutConnection(cutState.value.currentConnection);
  }
  if (cutState.value.cutConnections.size > 0) saveToHistory();

  cutState.value = {
    active: false,
    currentConnection: null,
    startProgress: null,
    currentProgress: 0,
    startVertex: null,
    snappedPosition: null,
    cutConnections: new Set(),
    reachedOppositeVertex: false,
  };
};

// ===== MOVIMIENTO GRUPAL =====
const isGroupMoveLegal = (groupIds, dx, dy, targetGridId) => {
  const cols = getGridCols(targetGridId);
  const rows = getGridRows(targetGridId);
  const idSet = new Set(groupIds);
  for (const id of groupIds) {
    const b = blocks.value.find((s) => s.id === id);
    if (!b) continue;
    const nx = b.gridX + dx;
    const ny = b.gridY + dy;
    if (nx < 0 || ny < 0 || nx >= cols || ny >= rows) return false;
    const occupant = blocks.value.find((s) => {
      if (s.gridId !== targetGridId) return false;
      if (idSet.has(s.id)) return false;
      return s.gridX === nx && s.gridY === ny;
    });
    if (occupant) return false;
  }
  return true;
};

const moveGroup = (groupIds, dx, dy, targetGridId = null) => {
  groupIds.forEach((id) => {
    const b = blocks.value.find((s) => s.id === id);
    if (!b) return;
    b.gridX += dx;
    b.gridY += dy;
    if (targetGridId !== null) b.gridId = targetGridId;
  });
};

const findSnapPosition = (block, targetX, targetY, targetGridId) => {
  if (!dragState.value || !groupContext.value)
    return { x: targetX, y: targetY };
  if (!isPositionAvailable(targetGridId, targetX, targetY, block.id))
    return { x: targetX, y: targetY };

  const idSet = new Set(groupContext.value.ids);
  let best = null;
  let minDist = 3; // snapDistance

  blocks.value.forEach((other) => {
    if (other.gridId !== targetGridId) return;
    if (idSet.has(other.id)) return;
    const candidates = [
      { x: other.gridX + 1, y: other.gridY },
      { x: other.gridX - 1, y: other.gridY },
      { x: other.gridX, y: other.gridY + 1 },
      { x: other.gridX, y: other.gridY - 1 },
    ];
    candidates.forEach((snap) => {
      if (snap.x !== targetX || snap.y !== targetY) return;
      const dist = Math.abs(targetX - snap.x) + Math.abs(targetY - snap.y);
      if (dist < minDist) {
        const snapDx = snap.x - dragState.value.anchorX;
        const snapDy = snap.y - dragState.value.anchorY;
        if (
          isGroupMoveLegal(groupContext.value.ids, snapDx, snapDy, targetGridId)
        ) {
          minDist = dist;
          best = snap;
        }
      }
    });
  });

  return best || { x: targetX, y: targetY };
};

// ===== VERIFICACIÓN DE LÍMITE DE GRUPO =====
const simulateMoveExceedsLimit = (
  movingIds,
  initialPositions,
  dx,
  dy,
  targetGridId,
) => {
  if (!props.maxPorGrupo || props.maxPorGrupo === Infinity)
    return { valid: true, totalUnits: 0, message: "" };

  const simBlocks = blocks.value.map((b) => {
    if (movingIds.includes(b.id)) {
      const orig = initialPositions.find((p) => p.id === b.id);
      if (orig)
        return {
          ...b,
          gridId: targetGridId,
          gridX: orig.gridX + dx,
          gridY: orig.gridY + dy,
        };
    }
    return { ...b };
  });

  const simAdjacent = (a, b) =>
    a.gridId === b.gridId &&
    Math.abs(a.gridX - b.gridX) === 1 &&
    a.gridY === b.gridY;

  const simGroups = [];
  for (let i = 0; i < simBlocks.length; i++) {
    for (let j = i + 1; j < simBlocks.length; j++) {
      if (!simAdjacent(simBlocks[i], simBlocks[j])) continue;
      const idI = simBlocks[i].id,
        idJ = simBlocks[j].id;
      const group = simGroups.find((g) => g.includes(idI) || g.includes(idJ));
      if (group) {
        if (!group.includes(idI)) group.push(idI);
        if (!group.includes(idJ)) group.push(idJ);
      } else {
        simGroups.push([idI, idJ]);
      }
    }
  }

  // Consolidar solapados
  let changed = true;
  while (changed) {
    changed = false;
    for (let i = 0; i < simGroups.length; i++) {
      for (let j = i + 1; j < simGroups.length; j++) {
        if (simGroups[i].some((id) => simGroups[j].includes(id))) {
          simGroups[i] = [...new Set([...simGroups[i], ...simGroups[j]])];
          simGroups.splice(j, 1);
          changed = true;
          break;
        }
      }
      if (changed) break;
    }
  }

  for (const group of simGroups) {
    if (!group.some((id) => movingIds.includes(id))) continue;
    const totalUnits = group.reduce((acc, id) => {
      const b = simBlocks.find((s) => s.id === id);
      return acc + (b?.units?.length || b?.value || 1);
    }, 0);
    if (totalUnits > props.maxPorGrupo)
      return {
        valid: false,
        totalUnits,
        message: `Límite de grupo: máximo ${props.maxPorGrupo} unidades.`,
      };
  }

  return { valid: true, totalUnits: 0, message: "" };
};

// ===== ACCIONES =====
// addGrid solo se usa internamente si se necesita en el futuro
// Los grids se definen desde App.vue via prop :grids

const addBlock = (gridId = 0) => {
  const gridBlocks = getGridBlocks(gridId);
  if (gridBlocks.length >= props.maxBloques) {
    showToastNotification(
      `Límite alcanzado: máximo ${props.maxBloques} bloques`,
    );
    return;
  }
  const pos = findAvailablePosition(gridId);
  if (!pos) {
    showToastNotification("¡No hay espacio disponible!");
    return;
  }
  blocks.value.push(createBlock(pos.x, pos.y, nextId++, gridId));
  refreshGroups();
  saveToHistory();
};

const toggleCuttingMode = () => {
  cuttingMode.value = !cuttingMode.value;
  selectedBlockId.value = null;
};

const clearAll = () => {
  blocks.value = [];
  selectedBlockId.value = null;
  adjacentGroups.value = [];
  connections.value = [];
  saveToStorage();
  saveToHistory();
};

// ===== EVENT HANDLERS =====
const handleBlockMouseDown = (e, blockId, gridId) => {
  if (cuttingMode.value) return;
  e.stopPropagation();
  e.preventDefault();

  selectedBlockId.value = blockId;
  draggedBlockId.value = blockId;
  draggedBlockGrid.value = gridId;
  isDragging.value = false;

  const block = getGridBlock(gridId, blockId);
  if (!block) return;

  const group = adjacentGroups.value.find((g) => g.includes(blockId));
  const ids = group || [blockId];

  groupContext.value = {
    ids,
    initialPositions: ids.map((id) => {
      const b = getGridBlock(gridId, id);
      return {
        id,
        gridId: b?.gridId ?? gridId,
        gridX: b?.gridX ?? 0,
        gridY: b?.gridY ?? 0,
      };
    }),
  };

  dragState.value = {
    mouseX: e.clientX,
    mouseY: e.clientY,
    anchorX: block.gridX,
    anchorY: block.gridY,
    startGridId: gridId,
    currentGridId: gridId,
    lastValidDx: undefined,
    lastValidDy: undefined,
  };
};

const findGridAtPoint = (x, y) => {
  const gridEls = Array.from(document.querySelectorAll(".grid-center"));
  for (const el of gridEls) {
    const rect = el.getBoundingClientRect();
    if (
      x >= rect.left &&
      x <= rect.right &&
      y >= rect.top &&
      y <= rect.bottom
    ) {
      const index = Number(el.dataset.gridIndex);
      return { gridIndex: Number.isNaN(index) ? 0 : index, rect };
    }
  }
  return null;
};

const handleMouseMove = (e) => {
  if (!draggedBlockId.value || !dragState.value || !groupContext.value) return;

  const dxMouse = Math.abs(e.clientX - dragState.value.mouseX);
  const dyMouse = Math.abs(e.clientY - dragState.value.mouseY);
  if (dxMouse > 3 || dyMouse > 3) isDragging.value = true;
  if (!isDragging.value) return;

  const hit = findGridAtPoint(e.clientX, e.clientY);

  // Actualizar ghost: siempre sigue al mouse
  if (props.inline && containerRef.value) {
    const cr = containerRef.value.getBoundingClientRect();
    ghostPos.value = {
      x: e.clientX - cr.left,
      y: e.clientY - cr.top,
      visible: !hit, // solo mostrar cuando está fuera de un grid
    };
  }

  if (!hit) return;

  ghostPos.value.visible = false;

  const { gridIndex, rect } = hit;
  const cs = getCellSize(gridIndex);
  const rawGridX = Math.floor((e.clientX - rect.left) / cs);
  const rawGridY = Math.floor((e.clientY - rect.top) / cs);
  const targetDx = rawGridX - dragState.value.anchorX;
  const targetDy = rawGridY - dragState.value.anchorY;

  groupContext.value.initialPositions.forEach(
    ({ id, gridX, gridY, gridId }) => {
      const b = blocks.value.find((s) => s.id === id);
      if (b) {
        b.gridX = gridX;
        b.gridY = gridY;
        b.gridId = gridId;
      }
    },
  );

  dragState.value.currentGridId = gridIndex;

  if (isGroupMoveLegal(groupContext.value.ids, targetDx, targetDy, gridIndex)) {
    moveGroup(groupContext.value.ids, targetDx, targetDy, gridIndex);
    dragState.value.lastValidDx = targetDx;
    dragState.value.lastValidDy = targetDy;
  } else if (dragState.value.lastValidDx !== undefined) {
    moveGroup(
      groupContext.value.ids,
      dragState.value.lastValidDx,
      dragState.value.lastValidDy,
      gridIndex,
    );
  }
};

const handleMouseUp = () => {
  const wasDragging = isDragging.value;
  const savedDragState = dragState.value;
  const savedGroupCtx = groupContext.value;

  ghostPos.value.visible = false;
  draggedBlockId.value = null;
  isDragging.value = false;

  if (wasDragging && savedDragState && savedGroupCtx) {
    const mainBlock = blocks.value.find((b) => b.id === selectedBlockId.value);
    const targetGridId =
      savedDragState.currentGridId ?? savedDragState.startGridId;

    if (mainBlock) {
      const snapPos = findSnapPosition(
        mainBlock,
        mainBlock.gridX,
        mainBlock.gridY,
        targetGridId,
      );

      if (snapPos.x !== mainBlock.gridX || snapPos.y !== mainBlock.gridY) {
        const snapDx = snapPos.x - savedDragState.anchorX;
        const snapDy = snapPos.y - savedDragState.anchorY;

        savedGroupCtx.initialPositions.forEach(
          ({ id, gridX, gridY, gridId }) => {
            const b = blocks.value.find((s) => s.id === id);
            if (b) {
              b.gridX = gridX;
              b.gridY = gridY;
              b.gridId = gridId;
            }
          },
        );

        const limitCheck = simulateMoveExceedsLimit(
          savedGroupCtx.ids,
          savedGroupCtx.initialPositions,
          snapDx,
          snapDy,
          targetGridId,
        );
        if (!limitCheck.valid) {
          showToastNotification(limitCheck.message);
          dragState.value = groupContext.value = null;
          refreshGroups();
          saveToHistory();
          return;
        }

        if (isGroupMoveLegal(savedGroupCtx.ids, snapDx, snapDy, targetGridId)) {
          moveGroup(savedGroupCtx.ids, snapDx, snapDy, targetGridId);
        } else if (savedDragState.lastValidDx !== undefined) {
          const lastCheck = simulateMoveExceedsLimit(
            savedGroupCtx.ids,
            savedGroupCtx.initialPositions,
            savedDragState.lastValidDx,
            savedDragState.lastValidDy,
            targetGridId,
          );
          if (lastCheck.valid)
            moveGroup(
              savedGroupCtx.ids,
              savedDragState.lastValidDx,
              savedDragState.lastValidDy,
              targetGridId,
            );
        }
      } else {
        const { lastValidDx: ldx, lastValidDy: ldy } = savedDragState;
        if (ldx !== undefined && ldy !== undefined) {
          const limitCheck = simulateMoveExceedsLimit(
            savedGroupCtx.ids,
            savedGroupCtx.initialPositions,
            ldx,
            ldy,
            targetGridId,
          );
          if (!limitCheck.valid) {
            savedGroupCtx.initialPositions.forEach(
              ({ id, gridX, gridY, gridId }) => {
                const b = blocks.value.find((s) => s.id === id);
                if (b) {
                  b.gridX = gridX;
                  b.gridY = gridY;
                  b.gridId = gridId;
                }
              },
            );
            showToastNotification(limitCheck.message);
            dragState.value = groupContext.value = null;
            refreshGroups();
            saveToHistory();
            return;
          }
        }
      }
    }

    refreshGroups();
    saveToHistory();
  }

  dragState.value = null;
  groupContext.value = null;
};

const handleGridCanvasClick = (e) => {
  const cls = e.target.classList;
  if (
    cls.contains("grid-cell-transparent") ||
    cls.contains("grid-fullscreen") ||
    cls.contains("grid-center") ||
    cls.contains("grid-transparent")
  ) {
    selectedBlockId.value = null;
  }
};

const handleCutModeMouseDown = (e) => {
  if (cuttingMode.value) {
    e.stopPropagation();
    e.preventDefault();
    handleSnapCutStart(e);
  }
};

const toggleOverlay = () => (showOverlay.value = !showOverlay.value);
const closeOverlay = () => (showOverlay.value = false);

// ===== DRAG DEL PANEL FLOTANTE =====
// Grids son fijos — sin drag
const handlePanelDragStart = () => {};
const handlePanelDragMove = () => {};
const handlePanelDragEnd = () => {};

// ===== POSICIONAMIENTO ALEATORIO =====
const makeOccupancyTracker = (gridId) => {
  const cols = getGridCols(gridId);
  const rows = getGridRows(gridId);
  const occupied = new Set();
  const isFree = (x, y) => {
    for (let dy = -1; dy <= 1; dy++)
      for (let dx = -1; dx <= 1; dx++) {
        const cx = x + dx,
          cy = y + dy;
        if (cx < 0 || cx >= cols || cy < 0 || cy >= rows) continue;
        if (occupied.has(`${cx},${cy}`)) return false;
      }
    return true;
  };
  const mark = (x, y) => {
    for (let dy = -1; dy <= 1; dy++)
      for (let dx = -1; dx <= 1; dx++) {
        const cx = x + dx,
          cy = y + dy;
        if (cx >= 0 && cx < cols && cy >= 0 && cy < rows)
          occupied.add(`${cx},${cy}`);
      }
  };
  return { isFree, mark };
};

// Shuffle solo los bloques del grid activo (el que tiene el foco de la toolbar)
const shuffleGrid = (gridId) => {
  const cols = getGridCols(gridId);
  const rows = getGridRows(gridId);
  const gridBlocks = getGridBlocks(gridId);
  if (gridBlocks.length === 0) return;

  const randomPos = () => ({
    x: Math.floor(Math.random() * cols),
    y: Math.floor(Math.random() * rows),
  });

  const { isFree, mark } = makeOccupancyTracker(gridId);
  const newPositions = [];
  gridBlocks.forEach((block) => {
    let placed = false;
    for (let i = 0; i < 300 && !placed; i++) {
      const { x, y } = randomPos();
      if (isFree(x, y)) {
        newPositions.push({ id: block.id, x, y });
        mark(x, y);
        placed = true;
      }
    }
    if (!placed)
      newPositions.push({ id: block.id, x: block.gridX, y: block.gridY });
  });
  newPositions.forEach(({ id, x, y }) => {
    const b = blocks.value.find((s) => s.id === id);
    if (b) {
      b.gridX = x;
      b.gridY = y;
    }
  });
  refreshGroups();
  saveToHistory();
};

// ===== CREAR BLOQUES INICIALES =====
const createInitialBlocks = () => {
  blocks.value = [];

  gridDefs.value.forEach((gridDef) => {
    if (!gridDef.initialBlocks?.length) return;
    const cols = gridDef.cols;
    const rows = gridDef.rows;
    const midW = Math.floor(cols / 2);
    const midH = Math.floor(rows / 2);
    const gId = gridDef.id;

    const layouts = {
      1: [{ x0: 0, y0: 0, x1: cols, y1: rows }],
      2: [
        { x0: 0, y0: 0, x1: cols, y1: midH },
        { x0: 0, y0: midH, x1: cols, y1: rows },
      ],
      3: [
        { x0: 0, y0: 0, x1: cols, y1: midH },
        { x0: 0, y0: midH, x1: midW, y1: rows },
        { x0: midW, y0: midH, x1: cols, y1: rows },
      ],
      4: [
        { x0: 0, y0: 0, x1: midW, y1: midH },
        { x0: midW, y0: 0, x1: cols, y1: midH },
        { x0: 0, y0: midH, x1: midW, y1: rows },
        { x0: midW, y0: midH, x1: cols, y1: rows },
      ],
    };

    const getQuadrant = (index, total) => {
      if (total <= 4 && layouts[total]) return layouts[total][index];
      return layouts[4][index % 4];
    };

    const placeGroupInQuadrant = (count, q) => {
      const occupied = new Set();
      const candidates = [];
      for (let y = q.y0; y < q.y1; y++)
        for (let x = q.x0; x < q.x1; x++) candidates.push({ x, y });

      for (let i = candidates.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [candidates[i], candidates[j]] = [candidates[j], candidates[i]];
      }

      let placed = 0;
      for (const { x, y } of candidates) {
        if (placed >= count) break;
        if (occupied.has(`${x},${y}`)) continue;
        if (!isPositionAvailable(gId, x, y)) continue;
        blocks.value.push(createBlock(x, y, nextId++, gId));
        occupied.add(`${x},${y}`);
        placed++;
      }
    };

    gridDef.initialBlocks.forEach((count, index) => {
      placeGroupInQuadrant(
        count,
        getQuadrant(index, gridDef.initialBlocks.length),
      );
    });
  });

  refreshGroups();
  saveToHistory();
  saveToStorage();
};

// ===== TECLADO =====
const handleKeyDown = (e) => {
  const inInput = ["INPUT", "TEXTAREA"].includes(
    document.activeElement.tagName,
  );

  if ((e.key === "x" || e.key === "X") && !inInput) {
    if (props.noSnap) return;
    e.preventDefault();
    toggleCuttingMode();
  }
  if (e.key === "1" && props.showToolbar && !inInput) {
    e.preventDefault();
    addBlock();
  }
  if ((e.key === "s" || e.key === "S") && !inInput) {
    e.preventDefault();
    shuffleOrGenerate();
  }
};

// ===== LIFECYCLE =====
onMounted(() => {
  if (props.inline) {
    // Pequeño delay para que el DOM esté renderizado
    setTimeout(updateContainerSize, 0);
  }
  if (_hasBeenMounted) {
    loadFromStorage();
  } else {
    localStorage.removeItem(props.storageKey);
    _hasBeenMounted = true;
  }
  saveToHistory();

  if (gridDefs.value.some((g) => g.initialBlocks?.length > 0))
    createInitialBlocks();

  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", handleMouseUp);
  document.addEventListener("mousemove", handleSnapCutMove);
  document.addEventListener("mouseup", handleSnapCutEnd);
  document.addEventListener("mousemove", handlePanelDragMove);
  document.addEventListener("mouseup", handlePanelDragEnd);
  window.addEventListener("resize", onResize);
  document.addEventListener("keydown", handleKeyDown);
});

onUnmounted(() => {
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", handleMouseUp);
  document.removeEventListener("mousemove", handleSnapCutMove);
  document.removeEventListener("mouseup", handleSnapCutEnd);
  document.removeEventListener("mousemove", handlePanelDragMove);
  document.removeEventListener("mouseup", handlePanelDragEnd);
  document.removeEventListener("keydown", handleKeyDown);
  window.removeEventListener("resize", onResize);
});
</script>

<template>
  <div>
    <!-- ========== MODO INLINE ========== -->
    <div
      v-if="props.inline"
      ref="containerRef"
      class="inline-grids-container"
      @mousedown="handleGridCanvasClick"
    >
      <!-- Zona izquierda: sumandos apilados, fit-content -->
      <template v-if="props.inlineColumns > 0">
        <div class="inline-zone-left">
          <div
            v-for="(gridDef, gIdx) in gridDefs.filter(g => !g.isAnswer)"
            :key="gridDef.id"
            class="sumando-wrapper"
          >
            <div
              class="grid-center inline-grid inline-grid-dashed"
              :data-grid-index="gridDef.id"
              @mousedown.stop
              @mousedown.capture="handleCutModeMouseDown"
            >
            <div v-if="gridDef.showLabel" class="grid-titlebar">
              <span></span><span></span>
              <span class="grid-titlebar-label">{{ gridDef.label }}</span>
              <span v-if="gridDef.showCount" class="grid-titlebar-count">
                {{ getGridBlocks(gridDef.id).length }} bloque{{
                  getGridBlocks(gridDef.id).length !== 1 ? "s" : ""
                }}
              </span>
            </div>
            <div class="grid-body" :class="{ 'grid-body--no-titlebar': !gridDef.showLabel }">
              <div
                class="grid-transparent"
                :class="{ 'cutting-cursor': cuttingMode }"
                :style="{ width: getGridWidth(gridDef.id) + 'px', height: getGridHeight(gridDef.id) + 'px' }"
                @mousedown="handleGridCanvasClick"
              >
                <div v-for="y in getGridRows(gridDef.id)" :key="`row-${gridDef.id}-${y}`" class="grid-row-transparent">
                  <div
                    v-for="x in getGridCols(gridDef.id)"
                    :key="`cell-${gridDef.id}-${x}-${y}`"
                    class="grid-cell-transparent"
                    :style="{ width: getCellSize(gridDef.id) + 'px', height: getCellSize(gridDef.id) + 'px' }"
                  ></div>
                </div>
              </div>
              <div class="blocks-layer-transparent" :style="{ width: getGridWidth(gridDef.id) + 'px', height: getGridHeight(gridDef.id) + 'px' }">
                <div
                  v-for="block in getGridBlocks(gridDef.id)"
                  :key="block.id"
                  @mousedown="handleBlockMouseDown($event, block.id, gridDef.id)"
                  :class="{ 'block-transparent': true, 'block-dragging': draggedBlockId === block.id, 'block-selected': selectedBlockId === block.id, 'block-connected': block.isConnected, 'no-hover': cuttingMode, 'visually-separated': block.visuallySeparated }"
                  :style="{ left: block.gridX * getCellSize(gridDef.id) + 'px', top: block.gridY * getCellSize(gridDef.id) + 'px', width: getCellSize(gridDef.id) + 'px', height: getCellSize(gridDef.id) + 'px', position: 'absolute', backgroundColor: 'transparent' }"
                >
                  <template v-if="block.units?.length > 0">
                    <div v-for="(unit, i) in block.units" :key="i" class="unit-cube"
                      :style="{ position: 'absolute', left: unit.relX * getCellSize(gridDef.id) + 'px', top: unit.relY * getCellSize(gridDef.id) + 'px', width: getCellSize(gridDef.id) + 'px', height: getCellSize(gridDef.id) + 'px', backgroundColor: unit.color, border: '2px solid rgba(0,0,0,0.3)', boxSizing: 'border-box' }"
                    ></div>
                  </template>
                  <div :style="{ position: 'absolute', inset: 0, backgroundImage: getBorderStyle(block), pointerEvents: 'none' }"></div>
                </div>
              </div>
            </div>
            <div v-if="showToolbar" class="grid-toolbar" @mousedown.stop>
              <button @click="addBlock(gridDef.id)" class="tool-btn tool-btn--add" :style="{ backgroundColor: BLOCK_COLOR }" title="Agregar bloque">+1</button>
              <div class="tool-divider"></div>
              <button @click="shuffleGrid(gridDef.id)" class="tool-btn" title="Mezclar">🔀</button>
            </div>
            </div>
            <!-- Número debajo del grid -->
            <div v-if="props.showGridLabels" class="sumando-num-label">{{ gridDef.label }}</div>
          </div>
        </div>

        <!-- Separador eliminado — espacio en blanco -->
        <div class="inline-spacer"></div>

        <!-- Zona derecha: grid respuesta -->
        <div class="inline-zone-right">
          <div
            v-for="gridDef in gridDefs.filter(g => g.isAnswer)"
            :key="gridDef.id"
            class="sumando-wrapper"
          >
            <div
              class="grid-center inline-grid inline-grid-dashed inline-grid-answer"
              :data-grid-index="gridDef.id"
              @mousedown.stop
              @mousedown.capture="handleCutModeMouseDown"
            >
            <div class="grid-body grid-body--no-titlebar">
              <div
                class="grid-transparent"
                :class="{ 'cutting-cursor': cuttingMode }"
                :style="{ width: getGridWidth(gridDef.id) + 'px', height: getGridHeight(gridDef.id) + 'px' }"
                @mousedown="handleGridCanvasClick"
              >
                <div v-for="y in getGridRows(gridDef.id)" :key="`row-${gridDef.id}-${y}`" class="grid-row-transparent">
                  <div
                    v-for="x in getGridCols(gridDef.id)"
                    :key="`cell-${gridDef.id}-${x}-${y}`"
                    class="grid-cell-transparent"
                    :style="{ width: getCellSize(gridDef.id) + 'px', height: getCellSize(gridDef.id) + 'px' }"
                  ></div>
                </div>
              </div>
              <div class="blocks-layer-transparent" :style="{ width: getGridWidth(gridDef.id) + 'px', height: getGridHeight(gridDef.id) + 'px' }">
                <div
                  v-for="block in getGridBlocks(gridDef.id)"
                  :key="block.id"
                  @mousedown="handleBlockMouseDown($event, block.id, gridDef.id)"
                  :class="{ 'block-transparent': true, 'block-dragging': draggedBlockId === block.id, 'block-selected': selectedBlockId === block.id, 'block-connected': block.isConnected, 'no-hover': cuttingMode, 'visually-separated': block.visuallySeparated }"
                  :style="{ left: block.gridX * getCellSize(gridDef.id) + 'px', top: block.gridY * getCellSize(gridDef.id) + 'px', width: getCellSize(gridDef.id) + 'px', height: getCellSize(gridDef.id) + 'px', position: 'absolute', backgroundColor: 'transparent' }"
                >
                  <template v-if="block.units?.length > 0">
                    <div v-for="(unit, i) in block.units" :key="i" class="unit-cube"
                      :style="{ position: 'absolute', left: unit.relX * getCellSize(gridDef.id) + 'px', top: unit.relY * getCellSize(gridDef.id) + 'px', width: getCellSize(gridDef.id) + 'px', height: getCellSize(gridDef.id) + 'px', backgroundColor: unit.color, border: '2px solid rgba(0,0,0,0.3)', boxSizing: 'border-box' }"
                    ></div>
                  </template>
                  <div :style="{ position: 'absolute', inset: 0, backgroundImage: getBorderStyle(block), pointerEvents: 'none' }"></div>
                </div>
              </div>
            </div>
            <div v-if="showToolbar" class="grid-toolbar" @mousedown.stop>
              <button @click="addBlock(gridDef.id)" class="tool-btn tool-btn--add" :style="{ backgroundColor: BLOCK_COLOR }" title="Agregar bloque">+1</button>
            </div>
            </div>
            <!-- Label debajo del grid de respuesta -->
            <div v-if="props.showGridLabels" class="sumando-num-label sumando-num-label--answer">{{ gridDef.label }}</div>
          </div>
        </div>
      </template>

      <!-- ── Modo flex-row original (inlineColumns === 0) ── -->
      <template v-else>
        <div
          v-for="gridDef in gridDefs"
          :key="gridDef.id"
          class="grid-center inline-grid"
          :data-grid-index="gridDef.id"
          @mousedown.stop
          @mousedown.capture="handleCutModeMouseDown"
        >
        <!-- Titlebar (ocultable) -->
        <div v-if="gridDef.showLabel" class="grid-titlebar">
          <span></span><span></span>
          <span class="grid-titlebar-label">{{ gridDef.label }}</span>
          <span v-if="gridDef.showCount" class="grid-titlebar-count">
            {{ getGridBlocks(gridDef.id).length }} bloque{{
              getGridBlocks(gridDef.id).length !== 1 ? "s" : ""
            }}
          </span>
        </div>

        <!-- Cuerpo del grid -->
        <div class="grid-body" :class="{ 'grid-body--no-titlebar': !gridDef.showLabel }">
          <div
            class="grid-transparent"
            :class="{ 'cutting-cursor': cuttingMode }"
            :style="{
              width: getGridWidth(gridDef.id) + 'px',
              height: getGridHeight(gridDef.id) + 'px',
            }"
            @mousedown="handleGridCanvasClick"
          >
            <div
              v-for="y in getGridRows(gridDef.id)"
              :key="`row-${gridDef.id}-${y}`"
              class="grid-row-transparent"
            >
              <div
                v-for="x in getGridCols(gridDef.id)"
                :key="`cell-${gridDef.id}-${x}-${y}`"
                class="grid-cell-transparent"
                :style="{
                  width: getCellSize(gridDef.id) + 'px',
                  height: getCellSize(gridDef.id) + 'px',
                }"
              ></div>
            </div>
          </div>

          <!-- Capa de bloques -->
          <div
            class="blocks-layer-transparent"
            :style="{
              width: getGridWidth(gridDef.id) + 'px',
              height: getGridHeight(gridDef.id) + 'px',
            }"
          >
            <div
              v-for="block in getGridBlocks(gridDef.id)"
              :key="block.id"
              @mousedown="handleBlockMouseDown($event, block.id, gridDef.id)"
              :class="{
                'block-transparent': true,
                'block-dragging': draggedBlockId === block.id,
                'block-selected': selectedBlockId === block.id,
                'block-connected': block.isConnected,
                'no-hover': cuttingMode,
                'visually-separated': block.visuallySeparated,
              }"
              :style="{
                left: block.gridX * getCellSize(gridDef.id) + 'px',
                top: block.gridY * getCellSize(gridDef.id) + 'px',
                width: getCellSize(gridDef.id) + 'px',
                height: getCellSize(gridDef.id) + 'px',
                position: 'absolute',
                backgroundColor: 'transparent',
              }"
            >
              <template v-if="block.units?.length > 0">
                <div
                  v-for="(unit, i) in block.units"
                  :key="i"
                  class="unit-cube"
                  :style="{
                    position: 'absolute',
                    left: unit.relX * getCellSize(gridDef.id) + 'px',
                    top: unit.relY * getCellSize(gridDef.id) + 'px',
                    width: getCellSize(gridDef.id) + 'px',
                    height: getCellSize(gridDef.id) + 'px',
                    backgroundColor: unit.color,
                    border: '2px solid rgba(0,0,0,0.3)',
                    boxSizing: 'border-box',
                  }"
                ></div>
              </template>
              <div
                :style="{
                  position: 'absolute',
                  inset: 0,
                  backgroundImage: getBorderStyle(block),
                  pointerEvents: 'none',
                }"
              ></div>
            </div>
          </div>

          <!-- SVGs modo corte -->
          <template v-if="cuttingMode">
            <svg
              class="connections-layer"
              :style="{
                position: 'absolute', top: 0, left: 0,
                width: getGridWidth(gridDef.id) + 'px',
                height: getGridHeight(gridDef.id) + 'px',
                overflow: 'visible', pointerEvents: 'none', zIndex: 50,
              }"
            >
              <g
                v-for="(conn, idx) in getConnectionsForGrid(gridDef.id)"
                :key="`conn-${gridDef.id}-${idx}`"
              >
                <line :x1="conn.x1" :y1="conn.y1" :x2="conn.x2" :y2="conn.y2"
                  :stroke="connectionsToCut.includes(conn) ? '#be185d' : '#ec4899'"
                  :stroke-width="connectionsToCut.includes(conn) ? '6' : '4'"
                  stroke-dasharray="8,4" stroke-linecap="round"
                  :class="['cut-line', { 'cut-line-active': connectionsToCut.includes(conn) }]"
                />
              </g>
            </svg>
          </template>
        </div>

        <!-- Toolbar por grid -->
        <div v-if="showToolbar" class="grid-toolbar" @mousedown.stop>
          <button @click="addBlock(gridDef.id)" class="tool-btn tool-btn--add"
            :style="{ backgroundColor: BLOCK_COLOR }" title="Agregar bloque">+1</button>
          <div class="tool-divider"></div>
          <button @click="shuffleGrid(gridDef.id)" class="tool-btn" title="Mezclar">🔀</button>
          <div class="tool-divider"></div>
          <button @click="toggleCuttingMode" :class="['tool-btn', { 'tool-btn--active': cuttingMode }]" title="Cortar">✂️</button>
          <div class="tool-divider"></div>
          <button @click="undo" :disabled="currentStep <= 0" class="tool-btn" title="Deshacer">↶</button>
          <button @click="redo" :disabled="currentStep >= history.length - 1" class="tool-btn" title="Rehacer">↷</button>
          <div class="tool-divider"></div>
          <button @click="clearAll" class="tool-btn tool-btn--danger" title="Borrar todo">🗑️</button>
        </div>
        </div>
      </template>

      <!-- Ghost drag — visible cuando el bloque sale de todos los grids -->
      <div
        v-if="ghostPos.visible && draggedBlockId !== null"
        class="drag-ghost"
        :style="{
          left: ghostPos.x + 'px',
          top: ghostPos.y + 'px',
          width: getCellSize(draggedBlockGrid) + 'px',
          height: getCellSize(draggedBlockGrid) + 'px',
          backgroundColor: blocks.find(b => b.id === draggedBlockId)?.units?.[0]?.color ?? BLOCK_COLOR,
        }"
      ></div>
    </div>

    <!-- ========== MODO OVERLAY (original) ========== -->
    <template v-else>
      <!-- FAB para abrir el overlay -->
      <div v-if="!showOverlay" class="floating-buttons">
        <button @click="toggleOverlay" class="fab">🧱 Abrir grids</button>
      </div>

      <!-- Overlay fullscreen -->
      <div
        v-if="showOverlay"
        class="grid-fullscreen"
        @mousedown="handleGridCanvasClick"
      >
        <!-- Botón X para cerrar -->
        <button
          class="overlay-close"
          @mousedown.stop
          @click="closeOverlay"
          title="Cerrar"
        >✕</button>

        <!-- Grids posicionados fijos -->
        <div
          v-for="gridDef in gridDefs"
          :key="gridDef.id"
          class="grid-center"
          :data-grid-index="gridDef.id"
          :style="getGridPositionStyle(gridDef.id)"
          @mousedown.stop
          @mousedown.capture="handleCutModeMouseDown"
        >
          <!-- Titlebar -->
          <div class="grid-titlebar">
            <span></span><span></span>
            <span class="grid-titlebar-label">{{ gridDef.label }}</span>
            <span v-if="gridDef.showCount" class="grid-titlebar-count">
              {{ getGridBlocks(gridDef.id).length }} bloque{{
                getGridBlocks(gridDef.id).length !== 1 ? "s" : ""
              }}
            </span>
          </div>

          <!-- Cuerpo del grid -->
          <div class="grid-body">
            <div
              class="grid-transparent"
              :class="{ 'cutting-cursor': cuttingMode }"
              :style="{
                width: getGridWidth(gridDef.id) + 'px',
                height: getGridHeight(gridDef.id) + 'px',
              }"
              @mousedown="handleGridCanvasClick"
            >
              <div
                v-for="y in getGridRows(gridDef.id)"
                :key="`row-${gridDef.id}-${y}`"
                class="grid-row-transparent"
              >
                <div
                  v-for="x in getGridCols(gridDef.id)"
                  :key="`cell-${gridDef.id}-${x}-${y}`"
                  class="grid-cell-transparent"
                  :style="{
                    width: getCellSize(gridDef.id) + 'px',
                    height: getCellSize(gridDef.id) + 'px',
                  }"
                ></div>
              </div>
            </div>

            <!-- Capa de bloques -->
            <div
              class="blocks-layer-transparent"
              :style="{
                width: getGridWidth(gridDef.id) + 'px',
                height: getGridHeight(gridDef.id) + 'px',
              }"
            >
              <div
                v-for="block in getGridBlocks(gridDef.id)"
                :key="block.id"
                @mousedown="handleBlockMouseDown($event, block.id, gridDef.id)"
                :class="{
                  'block-transparent': true,
                  'block-dragging': draggedBlockId === block.id,
                  'block-selected': selectedBlockId === block.id,
                  'block-connected': block.isConnected,
                  'no-hover': cuttingMode,
                  'visually-separated': block.visuallySeparated,
                }"
                :style="{
                  left: block.gridX * getCellSize(gridDef.id) + 'px',
                  top: block.gridY * getCellSize(gridDef.id) + 'px',
                  width: getCellSize(gridDef.id) + 'px',
                  height: getCellSize(gridDef.id) + 'px',
                  position: 'absolute',
                  backgroundColor: 'transparent',
                }"
              >
                <template v-if="block.units?.length > 0">
                  <div
                    v-for="(unit, i) in block.units"
                    :key="i"
                    class="unit-cube"
                    :style="{
                      position: 'absolute',
                      left: unit.relX * getCellSize(gridDef.id) + 'px',
                      top: unit.relY * getCellSize(gridDef.id) + 'px',
                      width: getCellSize(gridDef.id) + 'px',
                      height: getCellSize(gridDef.id) + 'px',
                      backgroundColor: unit.color,
                      border: '2px solid rgba(0,0,0,0.3)',
                      boxSizing: 'border-box',
                    }"
                  ></div>
                </template>

                <div
                  :style="{
                    position: 'absolute',
                    inset: 0,
                    backgroundImage: getBorderStyle(block),
                    pointerEvents: 'none',
                  }"
                ></div>
              </div>
            </div>

            <!-- SVGs de modo corte — filtrados por gridId -->
            <template v-if="cuttingMode">
              <svg
                class="connections-layer"
                :style="{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  width: getGridWidth(gridDef.id) + 'px',
                  height: getGridHeight(gridDef.id) + 'px',
                  overflow: 'visible',
                  pointerEvents: 'none',
                  zIndex: 50,
                }"
              >
                <g
                  v-for="(conn, idx) in getConnectionsForGrid(gridDef.id)"
                  :key="`conn-${gridDef.id}-${idx}`"
                >
                  <line
                    :x1="conn.x1"
                    :y1="conn.y1"
                    :x2="conn.x2"
                    :y2="conn.y2"
                    :stroke="
                      connectionsToCut.includes(conn) ? '#be185d' : '#ec4899'
                    "
                    :stroke-width="connectionsToCut.includes(conn) ? '6' : '4'"
                    stroke-dasharray="8,4"
                    stroke-linecap="round"
                    :class="[
                      'cut-line',
                      { 'cut-line-active': connectionsToCut.includes(conn) },
                    ]"
                  />
                  <circle
                    :cx="conn.x1"
                    :cy="conn.y1"
                    :r="connectionsToCut.includes(conn) ? '6' : '4'"
                    :fill="
                      connectionsToCut.includes(conn) ? '#be185d' : '#ec4899'
                    "
                  />
                  <circle
                    :cx="conn.x2"
                    :cy="conn.y2"
                    :r="connectionsToCut.includes(conn) ? '6' : '4'"
                    :fill="
                      connectionsToCut.includes(conn) ? '#be185d' : '#ec4899'
                    "
                  />
                </g>
              </svg>

              <svg
                v-if="cutState.active && cutState.currentConnection"
                class="snap-cut-visualization"
                :style="{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  width: getGridWidth(gridDef.id) + 'px',
                  height: getGridHeight(gridDef.id) + 'px',
                  overflow: 'visible',
                  pointerEvents: 'none',
                  zIndex: 55,
                }"
              >
                <line
                  v-for="conn in getConnectionsForGrid(gridDef.id)"
                  :key="`snap-${conn.shape1Id}-${conn.shape2Id}`"
                  :x1="conn.x1"
                  :y1="conn.y1"
                  :x2="conn.x2"
                  :y2="conn.y2"
                  :stroke="isConnectionCut(conn) ? '#10b981' : '#94a3b8'"
                  :stroke-width="isConnectionCut(conn) ? 6 : 3"
                  stroke-linecap="round"
                  :opacity="isConnectionCut(conn) ? 0.9 : 0.3"
                />
                <line
                  v-if="cutState.currentConnection"
                  :x1="cutState.currentConnection.x1"
                  :y1="cutState.currentConnection.y1"
                  :x2="cutState.currentConnection.x2"
                  :y2="cutState.currentConnection.y2"
                  :stroke="cutState.reachedOppositeVertex ? '#10b981' : '#ec4899'"
                  stroke-width="8"
                  stroke-linecap="round"
                  :opacity="cutState.reachedOppositeVertex ? 0.9 : 0.6"
                />
                <circle
                  v-if="cutState.snappedPosition"
                  :cx="cutState.snappedPosition.x"
                  :cy="cutState.snappedPosition.y"
                  r="10"
                  :fill="cutState.reachedOppositeVertex ? '#10b981' : '#fbbf24'"
                  stroke="#ffffff"
                  stroke-width="3"
                  opacity="1"
                >
                  <animate
                    v-if="cutState.reachedOppositeVertex"
                    attributeName="r"
                    values="10;14;10"
                    dur="0.6s"
                    repeatCount="indefinite"
                  />
                </circle>
              </svg>
            </template>
          </div>

          <!-- Toolbar por grid -->
          <div v-if="showToolbar" class="grid-toolbar" @mousedown.stop>
            <button
              @click="addBlock(gridDef.id)"
              class="tool-btn tool-btn--add"
              :style="{ backgroundColor: BLOCK_COLOR }"
              title="Agregar bloque"
            >
              +1
            </button>

            <div class="tool-divider"></div>
            <button
              @click="shuffleGrid(gridDef.id)"
              class="tool-btn"
              title="Mezclar bloques"
            >
              🔀
            </button>
            <div class="tool-divider"></div>

            <button
              @click="toggleCuttingMode"
              :class="['tool-btn', { 'tool-btn--active': cuttingMode }]"
              title="Desacoplar conexiones"
            >
              ✂️
            </button>

            <div class="tool-divider"></div>
            <button
              @click="undo"
              :disabled="currentStep <= 0"
              class="tool-btn"
              title="Deshacer"
            >
              ↶
            </button>
            <button
              @click="redo"
              :disabled="currentStep >= history.length - 1"
              class="tool-btn"
              title="Rehacer"
            >
              ↷
            </button>
            <div class="tool-divider"></div>
            <button
              @click="clearAll"
              class="tool-btn tool-btn--danger"
              title="Borrar todo"
            >
              🗑️
            </button>
          </div>
        </div>
      </div>
    </template>

    <Transition name="toast">
      <div v-if="showToast" class="toast-notification">{{ toastMessage }}</div>
    </Transition>
  </div>
</template>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.floating-buttons {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  z-index: 900;
}
.fab {
  padding: 1rem 1.5rem;
  background: white;
  border: none;
  border-radius: 50px;
  font-size: 1rem;
  font-weight: 600;
  color: #1f2937;
  cursor: pointer;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
  transition: all 0.3s ease;
}
.fab:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.25);
  background: #10b981;
  color: white;
}

.grid-fullscreen {
  position: fixed;
  inset: 0;
  background: transparent;
  z-index: 1000;
  overflow: hidden;
  overscroll-behavior: none;
  user-select: none;
  -webkit-user-select: none;
}

/* Botón X para cerrar el overlay */
.overlay-close {
  position: fixed;
  top: 1rem;
  right: 1rem;
  z-index: 2000;
  width: 36px;
  height: 36px;
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid rgba(0, 0, 0, 0.12);
  border-radius: 50%;
  font-size: 1rem;
  color: #475569;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
  transition: all 0.15s;
}
.overlay-close:hover {
  background: white;
  color: #ef4444;
  transform: scale(1.1);
}

/* Grid fijo — posicionado via inline style con getGridPositionStyle() */
.grid-center {
  position: fixed;
  display: inline-flex;
  flex-direction: column;
  align-items: flex-start;
  background: white;
  border-radius: 12px;
  box-shadow:
    0 0 0 2px rgba(0, 0, 0, 0.15),
    0 8px 32px rgba(0, 0, 0, 0.15);
}

.grid-titlebar {
  height: 24px;
  align-self: stretch;
  background: #f1f5f9;
  border-radius: 12px 12px 0 0;
  border-bottom: 1px solid rgba(0, 0, 0, 0.08);
  cursor: default;
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 0 8px;
  box-sizing: border-box;
}

.grid-titlebar-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #475569;
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.grid-titlebar-count {
  font-size: 0.7rem;
  font-weight: 500;
  color: #94a3b8;
  background: #e2e8f0;
  border-radius: 10px;
  padding: 1px 7px;
  white-space: nowrap;
  flex-shrink: 0;
}

.grid-toolbar {
  align-self: stretch;
  display: flex;
  align-items: center;
  gap: 0.25rem;
  padding: 0.3rem 0.4rem;
  background: #f1f5f9;
  border-top: 2px solid rgba(0, 0, 0, 0.1);
  border-radius: 0 0 12px 12px;
  box-sizing: border-box;
  overflow-x: auto;
}

.grid-body {
  position: relative;
  display: block;
}

.grid-transparent {
  position: relative;
  z-index: 1;
  background: transparent;
  overflow: hidden;
}
.cutting-cursor {
  cursor: crosshair !important;
}

.grid-row-transparent {
  display: flex;
}

.grid-cell-transparent {
  border: 1px solid rgba(0, 0, 0, 0.12);
  background-color: transparent;
  position: relative;
}

.blocks-layer-transparent {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 10;
  pointer-events: none;
}

.block-transparent {
  position: absolute;
  border-radius: 0;
  display: flex;
  cursor: grab;
  transition:
    transform 0.15s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.15s ease,
    background-color 0.2s ease,
    box-shadow 0.2s ease;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  pointer-events: auto;
  opacity: 0.95;
  will-change: transform;
  user-select: none;
  -webkit-user-select: none;
}

.unit-cube {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.7rem;
  color: white;
  font-weight: 600;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
  transition: background-color 0.2s ease;
}
.unit-cube::after {
  content: "";
  position: absolute;
  inset: 3px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  pointer-events: none;
  border-radius: 1px;
}

.block-transparent.block-connected {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.block-transparent.visually-separated {
  outline: 3px dashed rgba(239, 68, 68, 0.6);
  outline-offset: 2px;
  animation: visualSeparationPulse 2s ease-in-out infinite;
}
@keyframes visualSeparationPulse {
  0%,
  100% {
    outline-color: rgba(239, 68, 68, 0.6);
  }
  50% {
    outline-color: rgba(239, 68, 68, 0.9);
  }
}

.block-transparent:hover {
  filter: brightness(1.1);
  transform: scale(1.02);
  opacity: 1;
  box-shadow:
    0 4px 12px rgba(0, 0, 0, 0.15),
    0 0 0 3px rgba(59, 130, 246, 0.6),
    0 0 12px rgba(59, 130, 246, 0.4);
  z-index: 50;
}
.block-transparent.no-hover:hover {
  filter: none;
  transform: none;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: auto;
}
.block-transparent.block-dragging {
  cursor: grabbing;
  z-index: 100;
  opacity: 0.85;
  transform: scale(1.03);
  transition:
    opacity 0.1s ease,
    transform 0.05s ease-out;
  box-shadow:
    0 8px 24px rgba(0, 0, 0, 0.3),
    0 0 0 2px rgba(59, 130, 246, 0.4);
}
.block-transparent.block-selected {
  z-index: 50;
  box-shadow: 0 0 0 4px rgba(59, 130, 246, 0.6);
  border-radius: 0;
}

.cut-line {
  animation: dash 1.5s linear infinite;
}
.cut-line-active {
  stroke-width: 7 !important;
  animation: dash 0.8s linear infinite !important;
}
@keyframes dash {
  to {
    stroke-dashoffset: -24;
  }
}

.tool-btn {
  width: 34px;
  height: 34px;
  background: transparent;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition:
    background 0.15s,
    transform 0.15s;
  color: #374151;
  flex-shrink: 0;
}
.tool-btn:hover:not(:disabled) {
  background: #f3f4f6;
  transform: scale(1.08);
}
.tool-btn:active:not(:disabled) {
  transform: scale(0.95);
}
.tool-btn:disabled {
  opacity: 0.25;
  cursor: not-allowed;
}

.tool-btn--add {
  color: white !important;
  font-size: 0.85rem;
  font-weight: 700;
  border-radius: 10px;
}
.tool-btn--add:hover:not(:disabled) {
  filter: brightness(1.1);
  transform: scale(1.08);
}
.tool-btn--active {
  background: #fce7f3 !important;
  color: #be185d;
}
.tool-btn--active:hover:not(:disabled) {
  background: #fbcfe8 !important;
}
.tool-btn--danger:hover:not(:disabled) {
  background: #fef2f2 !important;
  color: #dc2626;
}

.tool-divider {
  width: 1px;
  height: 24px;
  background: #e5e7eb;
  margin: 0 0.15rem;
  flex-shrink: 0;
}

.toast-notification {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(30, 41, 59, 0.95);
  color: white;
  padding: 1rem 2rem;
  border-radius: 12px;
  font-size: 0.95rem;
  font-weight: 500;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  z-index: 10000;
  max-width: 90%;
  text-align: center;
  border: 1px solid rgba(255, 255, 255, 0.1);
  pointer-events: none;
}
.toast-enter-active {
  transition: all 0.3s ease;
}
.toast-leave-active {
  transition: all 0.3s ease 2.7s;
}
.toast-enter-from {
  opacity: 0;
  transform: translateX(-50%) translateY(20px);
}
.toast-leave-to {
  opacity: 0;
}

/* ===== MODO INLINE ===== */
.inline-grids-container {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
  gap: 16px;
  width: 100%;
  height: 100%;
  padding: 16px;
  box-sizing: border-box;
  position: relative;
  overflow: visible;
}

/* Zona izquierda: sumandos apilados, fit-content */
.inline-zone-left {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  justify-content: center;
  gap: 12px;
  flex-shrink: 0;
}

/* Wrapper de cada sumando: grid + número abajo */
.sumando-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
}

.sumando-num-label {
  font-size: 1.1rem;
  font-weight: 700;
  color: #374151;
  text-align: center;
  line-height: 1;
}

.sumando-num-label--answer {
  font-size: 0.8rem;
  font-weight: 600;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

/* Espacio entre sumandos y respuesta */
.inline-spacer {
  width: 24px;
  flex-shrink: 0;
}

/* Zona derecha: respuesta */
.inline-zone-right {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  flex-shrink: 0;
}

/* Los grids en modo inline son fit-content — sin estirarse */
.inline-grid {
  position: relative !important;
  top: auto !important;
  left: auto !important;
  right: auto !important;
  bottom: auto !important;
  display: inline-flex !important;
  flex: none !important;
  width: fit-content !important;
}

/* Grids de sumandos: contorno punteado, sin fondo sólido */
.inline-grid-dashed {
  background: transparent !important;
  box-shadow: none !important;
  border: 2px dashed #94a3b8 !important;
  border-radius: 10px !important;
}

/* Sin titlebar: border-radius completo */
.grid-body--no-titlebar {
  border-radius: 10px 10px 10px 10px !important;
}
.inline-grid:has(.grid-body--no-titlebar) {
  border-radius: 10px !important;
}

/* Ghost visual durante drag fuera del grid */
.drag-ghost {
  position: absolute;
  pointer-events: none;
  border-radius: 4px;
  border: 2px solid rgba(0,0,0,0.3);
  opacity: 0.75;
  z-index: 9999;
  transform: translate(-50%, -50%);
  box-shadow: 0 4px 12px rgba(0,0,0,0.25);
}
</style>