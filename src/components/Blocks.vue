<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

// ===== PROPS =====
const props = defineProps({
  maxBloques: { type: Number, default: Infinity },
  maxPorGrupo: { type: Number, default: Infinity },
  bloquesbarra: { type: Number, default: 9 },
  cantidadBl: {type: Number, default: 10},
  gridColumns: {type: Number, default: 20},
  gridRows: {type: Number, default: 15},
  initialBlocks: { type: Array, default: () => [] },
});

// ===== CONFIGURACIÓN =====
const BLOCKS_CONFIG = {
  // UNIDADES (1-9) - Colores SATURADOS fijos
  uno: { value: 1, color: '#EF4444', width: 1, height: 1 },  
  dos: { value: 2, color: '#F59E0B', width: 2, height: 1 },  
  tres: { value: 3, color: '#10B981', width: 3, height: 1},  
  cuatro: { value: 4, color: '#3B82F6', width: 4, height: 1 },
  cinco: { value: 5, color: '#8B5CF6', width: 5, height: 1},  
  seis: { value: 6, color: '#EC4899', width: 6, height: 1 },  
  siete: { value: 7, color: '#06B6D4', width: 7, height: 1},  
  ocho: { value: 8, color: '#F97316', width: 8, height: 1 },  
  nueve: { value: 9, color: '#14B8A6', width: 9, height: 1 },  
  
  // DECENAS (10, 20, 30, 40) - Progresión de saturación
  diez: { value: 10, color: '#3B82F6', width: 10, height: 1},  
  veinte: { value: 20, color: '#2563EB', width: 10, height: 2}, 
  treinta: { value: 30, color: '#1E40AF', width: 10, height: 3 },  
  cuarenta: { value: 40, color: '#1E3A8A', width: 10, height: 4 }  
};

/**
 * Separa dos bloques moviendo uno de ellos un espacio
 */
const separateBlocks = (block1, block2) => {
  
  const subgroup1 = findPhysicalSubgroup(block1.id, block2.id);
  const subgroup2 = findPhysicalSubgroup(block2.id, block1.id);
  

  // Decidir qué subgrupo mover: el más pequeño, o el de la derecha si son iguales
  let subgroupToMove;
  let otherSubgroup;
  
  if (subgroup1.length < subgroup2.length) {
    subgroupToMove = subgroup1;
    otherSubgroup = subgroup2;
  } else if (subgroup2.length < subgroup1.length) {
    subgroupToMove = subgroup2;
    otherSubgroup = subgroup1;
  } else {
    subgroupToMove = subgroup2;
    otherSubgroup = subgroup1;
  }
  
  // Intentar mover el subgrupo en diferentes direcciones
  // PRIORIDAD: Primero lados (izquierda/derecha), luego arriba/abajo
  const sideDirections = [
    { dx: 1, dy: 0, name: 'derecha' },
    { dx: -1, dy: 0, name: 'izquierda' }
  ];
  
  const verticalDirections = [
    { dx: 0, dy: 1, name: 'abajo' },
    { dx: 0, dy: -1, name: 'arriba' }
  ];
  
  // NUEVA LÓGICA: Verificar si el movimiento causaría acoplamiento inmediato
  const tryMoveSubgroup = (subgroup, excludeSubgroup, directions) => {
    for (const dir of directions) {
      let canMoveAll = true;
      const newPositions = [];
      
      for (const blockId of subgroup) {
        const block = shapes.value.find(s => s.id === blockId);
        if (!block) continue;
        
        const newX = block.gridX + dir.dx;
        const newY = block.gridY + dir.dy;
        const dims = getShapeDimensions(block);
        
        // Verificar límites
        if (newX < 0 || newY < 0 || 
            newX + dims.width > GRID_SIZE || 
            newY + dims.height > GRID_SIZE) {
          canMoveAll = false;
          break;
        }
        
        // Verificar disponibilidad (excluyendo el propio subgrupo)
        if (!isPositionAvailableExcluding(newX, newY, dims.width, dims.height, subgroup)) {
          canMoveAll = false;
          break;
        }
        
        newPositions.push({ blockId, newX, newY, dims });
      }
      
      // Si se puede mover, verificar que NO se acople inmediatamente con otros bloques
      if (canMoveAll && newPositions.length > 0) {
        let wouldReconnect = false;
        
        // Verificar si algún bloque del subgrupo movido quedaría adyacente a CUALQUIER otro bloque
        for (const pos of newPositions) {
          // Buscar bloques adyacentes en la nueva posición
          const adjacentPositions = [
            { x: pos.newX + pos.dims.width, y: pos.newY }, // derecha
            { x: pos.newX - 1, y: pos.newY }, // izquierda
            { x: pos.newX, y: pos.newY + pos.dims.height }, // abajo
            { x: pos.newX, y: pos.newY - 1 } // arriba
          ];
          
          for (const adjPos of adjacentPositions) {
            const adjacentBlock = shapes.value.find(s => {
              // Solo NO considerar bloques del subgrupo que se está moviendo
              if (subgroup.includes(s.id)) return false;
              // IMPORTANTE: Verificar con TODOS los demás bloques (incluido el otro subgrupo)
              
              const dims = getShapeDimensions(s);
              return s.gridX === adjPos.x && s.gridY === adjPos.y;
            });
            
            if (adjacentBlock) {
              wouldReconnect = true;
              break;
            }
          }
          
          if (wouldReconnect) break;
        }
        
        // Si NO se reconecta, aplicar el movimiento
        if (!wouldReconnect) {
          newPositions.forEach(pos => {
            const block = shapes.value.find(s => s.id === pos.blockId);
            if (block) {
              block.gridX = pos.newX;
              block.gridY = pos.newY;
            }
          });
          return true;
        }
      }
    }
    return false;
  };
  
  // NUEVA ESTRATEGIA DE SEPARACIÓN:
  // 1. Intentar mover subgrupo pequeño a los LADOS
  if (tryMoveSubgroup(subgroupToMove, otherSubgroup, sideDirections)) {
    return true;
  }
  
  // 2. Si no funciona, intentar mover el OTRO subgrupo a los LADOS
  if (tryMoveSubgroup(otherSubgroup, subgroupToMove, sideDirections)) {
    return true;
  }
  
  // 3. Intentar mover subgrupo pequeño VERTICAL (arriba/abajo)
  if (tryMoveSubgroup(subgroupToMove, otherSubgroup, verticalDirections)) {
    return true;
  }
  
  // 4. Intentar mover el OTRO subgrupo VERTICAL
  if (tryMoveSubgroup(otherSubgroup, subgroupToMove, verticalDirections)) {
    return true;
  }
  
  // 5. Si NADA funciona, crear separación VISUAL sin mover
  // Agregar un pequeño offset visual para mostrar que están separados
  console.log('⚠️ No se puede separar físicamente - aplicando separación visual');
  
  // Marcar bloques como visualmente separados (esto se puede usar en CSS)
  subgroupToMove.forEach(id => {
    const block = shapes.value.find(s => s.id === id);
    if (block) {
      block.visuallySeparated = true;
    }
  });
  
  return true; // Consideramos que la separación fue exitosa (visual)
};

/**
 * Encuentra subgrupo de bloques FÍSICAMENTE ADYACENTES
 */
const findPhysicalSubgroup = (startBlockId, excludeBlockId) => {
  const visited = new Set();
  const subgroup = [];
  const queue = [startBlockId];
  
  while (queue.length > 0) {
    const currentId = queue.shift();
    
    if (visited.has(currentId)) continue;
    
    visited.add(currentId);
    subgroup.push(currentId);
    
    // Encontrar bloques FÍSICAMENTE ADYACENTES en el grid
    const currentBlock = shapes.value.find(s => s.id === currentId);
    if (!currentBlock) continue;
    
    const adjacentIds = findPhysicallyAdjacentBlocks(currentBlock);
    
    adjacentIds.forEach(adjId => {
      if (!visited.has(adjId) && adjId !== excludeBlockId) {
        queue.push(adjId);
      }
    });
  }
  
  return subgroup;
};

/**
 * Encuentra bloques que están FÍSICAMENTE ADYACENTES (tocándose) en el grid
 * Solo horizontalmente (derecha/izquierda)
 */
const findPhysicallyAdjacentBlocks = (block) => {
  const adjacent = [];
  const dims = getShapeDimensions(block);
  
  // Solo buscar adyacencia HORIZONTAL (como en checkIfAdjacent)
  shapes.value.forEach(otherBlock => {
    if (otherBlock.id === block.id) return;
    
    const otherDims = getShapeDimensions(otherBlock);
    
    // Verificar si están tocándose horizontalmente
    const dx = otherBlock.gridX - block.gridX;
    const dy = otherBlock.gridY - block.gridY;
    
    // Adyacente a la derecha: dx = width del bloque actual, mismo Y
    if (dx === dims.width && dy === 0 && dims.height === otherDims.height) {
      adjacent.push(otherBlock.id);
    }
    
    // Adyacente a la izquierda: dx = -width del otro bloque, mismo Y
    if (dx === -otherDims.width && dy === 0 && dims.height === otherDims.height) {
      adjacent.push(otherBlock.id);
    }
  });
  
  return adjacent;
};

/**
 * Verifica si una posición está disponible, excluyendo ciertos bloques
 */
const isPositionAvailableExcluding = (gridX, gridY, width, height, excludeIds) => {
  for (let dy = 0; dy < height; dy++) {
    for (let dx = 0; dx < width; dx++) {
      const checkX = gridX + dx;
      const checkY = gridY + dy;
      
      // Verificar si hay algún bloque en esta celda
      const blockInCell = shapes.value.find(shape => {
        // Excluir bloques que estamos moviendo
        if (excludeIds.includes(shape.id)) return false;
        
        const dims = getShapeDimensions(shape);
        return checkX >= shape.gridX && 
               checkX < shape.gridX + dims.width &&
               checkY >= shape.gridY && 
               checkY < shape.gridY + dims.height;
      });
      
      if (blockInCell) return false;
    }
  }
  
  return true;
};

const ALL_UNIDADES = ['uno', 'dos', 'tres', 'cuatro', 'cinco', 'seis', 'siete', 'ocho', 'nueve'];
const UNIDADES = computed(() => ALL_UNIDADES.slice(0, props.bloquesbarra));
const DECENAS = ['diez', 'veinte', 'treinta', 'cuarenta'];

/**
 * Genera las unidades (cubitos 1x1) de un bloque según su configuración
 */
const generateUnitsForBlock = (type) => {
  const config = BLOCKS_CONFIG[type];
  if (!config) return [];
  
  const units = [];
  const width = config.width;
  const height = config.height;
  
  // Generar unidades para todas las celdas del bloque
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      units.push({
        relX: x,
        relY: y,
        color: config.color,  // Color base inicial
        baseColor: config.color  // Guardar color original
      });
    }
  }
  
  return units;
};

/**
 * Genera unidades para un bloque custom (cortado)
 */
const generateUnitsForCustomBlock = (quadrants, baseColor) => {
  return quadrants.map(q => ({
    relX: q.relX,
    relY: q.relY,
    color: baseColor,
    baseColor: baseColor
  }));
};

// ===== SISTEMA DE COLOR SEGÚN SUMA =====

// Mapa de colores: suma → color
const COLORS_BY_SUM = {
  // Unidades 1-9: Colores fijos
  1: '#EF4444',  // Rojo
  2: '#F59E0B',  // Naranja
  3: '#10B981',  // Verde
  4: '#3B82F6',  // Azul
  5: '#8B5CF6',  // Púrpura
  6: '#EC4899',  // Rosa
  7: '#06B6D4',  // Cyan
  8: '#F97316',  // Naranja oscuro
  9: '#14B8A6',  // Turquesa
  
  // Decenas: Progresión de saturación en azul
  10: '#3B82F6',  // Azul base
  11: '#3B82F6',
  12: '#3B82F6',
  13: '#3B82F6',
  14: '#3B82F6',
  15: '#3B82F6',
  16: '#3B82F6',
  17: '#3B82F6',
  18: '#3B82F6',
  19: '#3B82F6',
  
  20: '#2563EB',  // Azul más oscuro
  21: '#2563EB',
  22: '#2563EB',
  23: '#2563EB',
  24: '#2563EB',
  25: '#2563EB',
  26: '#2563EB',
  27: '#2563EB',
  28: '#2563EB',
  29: '#2563EB',
  
  30: '#1E40AF',  // Azul aún más oscuro
  31: '#1E40AF',
  32: '#1E40AF',
  33: '#1E40AF',
  34: '#1E40AF',
  35: '#1E40AF',
  36: '#1E40AF',
  37: '#1E40AF',
  38: '#1E40AF',
  39: '#1E40AF',
  
  40: '#1E3A8A',  // Azul muy oscuro
};

/**
 * Obtiene el color según la suma del grupo
 * Si la suma está en el mapa, usa ese color
 * Si no, usa un color por defecto
 */
const getColorBySum = (sum) => {
  if (sum >= 40) return COLORS_BY_SUM[40];
  return COLORS_BY_SUM[sum] || '#64748b'; // Gris por defecto
};

/**
 * Obtiene el color correcto para un bloque según su valor
 * Busca en BLOCKS_CONFIG el color que corresponde a ese valor
 */
const getColorByValue = (value) => {
  // Buscar en unidades 1-9
  const unidadKey = ['uno', 'dos', 'tres', 'cuatro', 'cinco', 'seis', 'siete', 'ocho', 'nueve'][value - 1];
  if (unidadKey && BLOCKS_CONFIG[unidadKey]) {
    return BLOCKS_CONFIG[unidadKey].color;
  }
  
  // Si es 10 o más, usar el sistema de colores por suma
  return getColorBySum(value);
};

/**
 * Colorea las unidades de un grupo según el sistema de decenas
 * - Si suma < 10: Color según la suma (ej: 1+1=2 → naranja)
 * - Si suma >= 10: Primeras 10 unidades color decena, resto según valor
 */
const colorearUnidadesDeGrupo = (grupo) => {
  
  // 1. Calcular suma total del grupo
  let sumaTotal = 0;
  grupo.forEach(bloqueId => {
    const bloque = shapes.value.find(s => s.id === bloqueId);
    if (bloque) sumaTotal += bloque.value;
  });
  
  
  // 2. Si suma < 10: Colorear TODAS las unidades con el color de la suma
  if (sumaTotal < 10) {
    const colorDeLaSuma = getColorByValue(sumaTotal);
    
    grupo.forEach(bloqueId => {
      const bloque = shapes.value.find(s => s.id === bloqueId);
      if (!bloque || !bloque.units) return;
      
      bloque.units.forEach(unit => {
        unit.color = colorDeLaSuma;
      });
    });
    return;
  }
  
  // 3. Si suma >= 10: Sistema de decenas
  // Recolectar TODAS las unidades del grupo (de izquierda a derecha)
  const todasLasUnidades = [];
  
  grupo.forEach(bloqueId => {
    const bloque = shapes.value.find(s => s.id === bloqueId);
    if (!bloque || !bloque.units) return;
    
    bloque.units.forEach((unit, unitIndex) => {
      todasLasUnidades.push({
        bloqueId: bloque.id,
        unitIndex: unitIndex,
        posicionX: bloque.gridX + unit.relX,
        posicionY: bloque.gridY + unit.relY,
        baseColor: unit.baseColor || unit.color
      });
    });
  });
  
  // Ordenar por posición (izquierda a derecha, luego arriba a abajo)
  todasLasUnidades.sort((a, b) => {
    if (a.posicionX !== b.posicionX) {
      return a.posicionX - b.posicionX;
    }
    return a.posicionY - b.posicionY;
  });
  
  const totalUnidades = todasLasUnidades.length;
  const resto = totalUnidades % 10;
  
  
  // 4. Colorear según posición
  todasLasUnidades.forEach((unitData, index) => {
    const bloque = shapes.value.find(s => s.id === unitData.bloqueId);
    if (!bloque) return;
    
    let nuevoColor;
    
    if (index < 10) {
      // Primera decena (0-9): Azul oscuro
      nuevoColor = '#1E40AF';
    } else if (index < 20) {
      // Segunda decena (10-19): Azul medio
      nuevoColor = '#3B82F6';
    } else if (index < 30) {
      // Tercera decena (20-29): Azul claro
      nuevoColor = '#60A5FA';
    } else if (index < 40) {
      // Cuarta decena (30-39): Azul muy claro
      nuevoColor = '#93C5FD';
    } else {
      // Más de 40: Color según el resto
      nuevoColor = resto > 0 ? getColorByValue(resto) : unitData.baseColor;
    }
    
    // IMPORTANTE: Si estamos en el "resto" (después de las decenas completas)
    // Colorear con el color correspondiente al valor del resto
    const decenasCompletas = Math.floor(totalUnidades / 10);
    const primerIndiceResto = decenasCompletas * 10;
    
    if (index >= primerIndiceResto && resto > 0) {
      // Estamos en el resto: usar color según valor del resto
      nuevoColor = getColorByValue(resto);
    } else {
    }
    
    bloque.units[unitData.unitIndex].color = nuevoColor;
  });
  
};

const GRID_SIZE = props.gridColumns; // Ancho máximo (para compatibilidad)
// ===== CELL SIZE ADAPTATIVO =====
const MAX_CELL_SIZE = 45;
const MIN_CELL_SIZE = 16;
const PADDING_H = 40;
const PADDING_V = 80;

const viewportW = ref(window.innerWidth);
const viewportH = ref(window.innerHeight);

const CELL_SIZE = computed(() => {
  const byWidth  = Math.floor((viewportW.value - PADDING_H) / GRID_WIDTH);
  const byHeight = Math.floor((viewportH.value - PADDING_V) / GRID_HEIGHT);
  return Math.max(Math.min(byWidth, byHeight, MAX_CELL_SIZE), MIN_CELL_SIZE);
});

const onResize = () => {
  viewportW.value = window.innerWidth;
  viewportH.value = window.innerHeight;
  borderStyleCache.clear();
};
const GRID_WIDTH = props.gridColumns; 
const GRID_HEIGHT = props.gridRows; 

// ===== CONFIGURACIÓN DE VISUALIZACIÓN =====
// SHOW_COLUMN_NUMBERS eliminado - Ruler innecesario

/**
 * Crea una matriz de cuadrantes para un bloque
 * @param {number} width - Ancho en cuadrantes
 * @param {number} height - Alto en cuadrantes
 * @param {number} offsetX - Posición X relativa al bloque
 * @param {number} offsetY - Posición Y relativa al bloque
 * @returns {Array} Array de cuadrantes con coordenadas relativas
 */
const createQuadrants = (width, height, offsetX = 0, offsetY = 0) => {
  const quadrants = [];
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      quadrants.push({
        relX: x + offsetX,  // Posición relativa X dentro del bloque
        relY: y + offsetY,  // Posición relativa Y dentro del bloque
        value: 1            // Cada cuadrante vale 1
      });
    }
  }
  return quadrants;
};

/**
 * Crea un bloque con sistema de cuadrantes
 * @param {string} type - Tipo de bloque
 * @param {number} gridX - Posición X en el grid
 * @param {number} gridY - Posición Y en el grid
 * @param {number} id - ID único del bloque
 * @returns {Object} Bloque con cuadrantes
 */
const createBlockWithQuadrants = (type, gridX, gridY, id) => {
  const config = BLOCKS_CONFIG[type];
  const quadrants = createQuadrants(config.width, config.height);
  
  // NUEVO: Generar unidades (cubitos 1x1) para el bloque
  const units = generateUnitsForBlock(type, false);
  
  return {
    id,
    type,
    gridX,
    gridY,
    value: config.value,
    color: config.color,
    rotated: false,
    isConnected: false,
    quadrants,  // Array de cuadrantes que componen este bloque
    units,      // NUEVO: Array de unidades (cubitos 1x1)
    // Dimensiones del bloque (para compatibilidad)
    width: config.width,
    height: config.height
  };
};

/**
 * Obtiene las celdas del grid que ocupa un bloque (basado en cuadrantes)
 * @param {Object} shape - Bloque con cuadrantes
 * @returns {Array} Array de {x, y} posiciones absolutas en el grid
 */
const getOccupiedCellsFromQuadrants = (shape) => {
  if (!shape.quadrants) {
    // Fallback para bloques antiguos sin cuadrantes
    return getOccupiedCells(shape);
  }
  
  return shape.quadrants.map(q => ({
    x: shape.gridX + q.relX,
    y: shape.gridY + q.relY
  }));
};

/**
 * Corta un bloque en una posición específica
 * @param {Object} shape - Bloque a cortar
 * @param {Object} cutLine - Línea de corte {x1, y1, x2, y2}
 * @returns {Array} Array de dos nuevos bloques o null si no se puede cortar
 */
// cutBlockByQuadrants ELIMINADO - Solo usamos bloques de 1x1

// ===== ESTADO =====
const blockMode = ref('unidades');
const showOverlay = ref(false);
const shapes = ref([]);
const selectedShapeId = ref(null);
const hoveredShapeId = ref(null); // Para mostrar tooltip en hover
const draggingId = ref(null);
const isDragging = ref(false);

const dragStart = ref(null);
const shapeGroups = ref([]);
const cuttingMode = ref(false);
const connections = ref([]);
const cutDragStart = ref(null);
const cutDragEnd = ref(null);
const isCutDragging = ref(false);
const connectionsToCut = ref([]);
const cutTrailPoints = ref([]); // Rastro de puntos para la línea de corte

// ===== TOAST NOTIFICATION =====
const toastMessage = ref('');
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
const alreadyCutConnections = ref(new Set()); // Rastrear conexiones ya cortadas en este trazo

// Sistema de corte de bloques
// Modo cortar bloques ELIMINADO - Solo usamos bloques de 1x1

// Computed para líneas de corte disponibles - ELIMINADO
// Computed availableCutLines ELIMINADO - No cortamos bloques

// Sistema de Undo/Redo
const history = ref([]);
const currentStep = ref(-1);
const maxHistorySize = 10; // Límite de pasos guardados

// Cache para getBorderStyle
const borderStyleCache = new Map();

// Debounce timer
let checkAdjacentTimer = null;
let nextId = 1;

const availableBlocks = computed(() => {
  return blockMode.value === 'unidades' ? UNIDADES : DECENAS;
});

// ===== PERSISTENCIA =====
const saveToStorage = () => {
  try {
    localStorage.setItem('grid_blocks_data', JSON.stringify(shapes.value));
  } catch (e) {
    console.error('Error guardando:', e);
  }
};

const loadFromStorage = () => {
  try {
    const saved = localStorage.getItem('grid_blocks_data');
    if (saved) {
      const loadedShapes = JSON.parse(saved);
      
      // Migrar bloques antiguos a sistema de cuadrantes
      const migratedShapes = loadedShapes.map(shape => {
        if (!shape.quadrants && shape.type && BLOCKS_CONFIG[shape.type]) {
          // Bloque antiguo sin cuadrantes - crear cuadrantes
          const config = BLOCKS_CONFIG[shape.type];
          return {
            ...shape,
            quadrants: createQuadrants(config.width, config.height),
            width: config.width,
            height: config.height
          };
        }
        return shape;
      });
      
      // NUEVO: Validar que todos los bloques estén dentro del grid actual
      const blocksOutsideGrid = [];
      const blocksInsideGrid = [];
      
      migratedShapes.forEach(shape => {
        const dims = getShapeDimensions(shape);
        const isOutside = 
          shape.gridX < 0 || 
          shape.gridY < 0 || 
          shape.gridX + dims.width > GRID_WIDTH || 
          shape.gridY + dims.height > GRID_HEIGHT;
        
        if (isOutside) {
          blocksOutsideGrid.push(shape);
        } else {
          blocksInsideGrid.push(shape);
        }
      });
      
      // Reposicionar bloques que están fuera del grid
      blocksOutsideGrid.forEach(shape => {
        const dims = getShapeDimensions(shape);
        const pos = findAvailablePosition(dims.width, dims.height);
        
        if (pos) {
          shape.gridX = pos.x;
          shape.gridY = pos.y;
          blocksInsideGrid.push(shape);
        }
        // Si no hay espacio, el bloque se pierde (no se agrega)
      });
      
      shapes.value = blocksInsideGrid;
      
      if (shapes.value.length > 0) {
        nextId = Math.max(...shapes.value.map(s => s.id), 0) + 1;
      }
      
      // Registrar todos los bloques en RelationshipManager
      shapes.value.forEach(shape => {
      });
      
      checkAdjacentBlocks();
    }
  } catch (e) {
    console.error('Error cargando:', e);
  }
};

// ===== UNDO/REDO =====
const saveToHistory = () => {
  // Guardar estado actual en el historial
  const currentState = JSON.parse(JSON.stringify(shapes.value));
  
  // Si estamos en medio del historial, eliminar los pasos futuros
  if (currentStep.value < history.value.length - 1) {
    history.value = history.value.slice(0, currentStep.value + 1);
  }
  
  // Agregar nuevo estado
  history.value.push(currentState);
  
  // Limitar tamaño del historial
  if (history.value.length > maxHistorySize) {
    history.value.shift();
  } else {
    currentStep.value++;
  }
};

const undo = () => {
  if (currentStep.value > 0) {
    currentStep.value--;
    
    // Desregistrar todos los bloques actuales del RelationshipManager
    
    // Restaurar estado desde historial
    shapes.value = JSON.parse(JSON.stringify(history.value[currentStep.value]));
    
    // Registrar todos los bloques restaurados
    
    checkAdjacentBlocksImmediate();
    saveToStorage();
  }
};

const redo = () => {
  if (currentStep.value < history.value.length - 1) {
    currentStep.value++;
    
    // Desregistrar todos los bloques actuales del RelationshipManager
    
    // Restaurar estado desde historial
    shapes.value = JSON.parse(JSON.stringify(history.value[currentStep.value]));
    
    // Registrar todos los bloques restaurados
    
    checkAdjacentBlocksImmediate();
    saveToStorage();
  }
};

// ===== UTILIDADES =====
const getShapeDimensions = (shape) => {
  // Si el bloque tiene dimensiones explícitas (bloques cortados), usarlas
  if (shape.width !== undefined && shape.height !== undefined) {
    return { width: shape.width, height: shape.height };
  }
  
  // Si tiene cuadrantes pero no dimensiones, calcularlas
  if (shape.quadrants && shape.quadrants.length > 0) {
    const maxX = Math.max(...shape.quadrants.map(q => q.relX)) + 1;
    const maxY = Math.max(...shape.quadrants.map(q => q.relY)) + 1;
    return { width: maxX, height: maxY };
  }
  
  // Fallback a BLOCKS_CONFIG
  const config = BLOCKS_CONFIG[shape.type];
  if (!config) return { width: 1, height: 1 };
  return { width: config.width, height: config.height };
};

const getOccupiedCells = (shape) => {
  const cells = [];
  const dims = getShapeDimensions(shape);
  for (let dy = 0; dy < dims.height; dy++) {
    for (let dx = 0; dx < dims.width; dx++) {
      cells.push({ x: shape.gridX + dx, y: shape.gridY + dy });
    }
  }
  return cells;
};

const isPositionAvailable = (gridX, gridY, width, height, excludeId = null) => {
  for (let dy = 0; dy < height; dy++) {
    for (let dx = 0; dx < width; dx++) {
      const x = gridX + dx;
      const y = gridY + dy;
      if (x >= GRID_WIDTH || y >= GRID_HEIGHT || x < 0 || y < 0) return false;
      const occupant = shapes.value.find(s => {
        if (s.id === excludeId) return false;
        return getOccupiedCells(s).some(c => c.x === x && c.y === y);
      });
      if (occupant) return false;
    }
  }
  return true;
};

const findAvailablePosition = (width, height) => {
  // Buscar desde arriba-izquierda, dejando espacio alrededor
  for (let y = 0; y < GRID_HEIGHT - height + 1; y++) {
    for (let x = 0; x < GRID_WIDTH - width + 1; x++) {
      if (isPositionAvailable(x, y, width, height, null)) {
        // Verificar que tenga espacio libre alrededor (al menos 1 celda)
        let hasSpaceAround = true;
        
        // Revisar si hay bloques adyacentes inmediatos
        for (let dy = -1; dy <= height; dy++) {
          for (let dx = -1; dx <= width; dx++) {
            // Skip las celdas del propio bloque
            if (dx >= 0 && dx < width && dy >= 0 && dy < height) continue;
            
            const checkX = x + dx;
            const checkY = y + dy;
            
            if (checkX >= 0 && checkX < GRID_WIDTH && checkY >= 0 && checkY < GRID_HEIGHT) {
              const occupant = shapes.value.find(s => {
                return getOccupiedCells(s).some(c => c.x === checkX && c.y === checkY);
              });
              if (occupant) {
                hasSpaceAround = false;
                break;
              }
            }
          }
          if (!hasSpaceAround) break;
        }
        
        if (hasSpaceAround) {
          return { x, y };
        }
      }
    }
  }
  
  // Si no encontró con espacio, buscar cualquier posición disponible
  for (let y = 0; y < GRID_HEIGHT - height + 1; y++) {
    for (let x = 0; x < GRID_WIDTH - width + 1; x++) {
      if (isPositionAvailable(x, y, width, height, null)) {
        return { x, y };
      }
    }
  }
  
  return null;
};

// ===== BORDES INTELIGENTES =====
const getHiddenBorders = computed(() => {
  const hidden = new Map();
  shapes.value.forEach(shape => {
    const shapeHidden = { top: new Set(), right: new Set(), bottom: new Set(), left: new Set() };
    const dims = getShapeDimensions(shape);
    for (let dy = 0; dy < dims.height; dy++) {
      for (let dx = 0; dx < dims.width; dx++) {
        const cellX = shape.gridX + dx;
        const cellY = shape.gridY + dy;
        shapes.value.forEach(otherShape => {
          if (shape.id === otherShape.id) return;
          const otherCells = getOccupiedCells(otherShape);
          otherCells.forEach(otherCell => {
            if (otherCell.x === cellX && otherCell.y === cellY - 1 && dy === 0) shapeHidden.top.add(dx);
            if (otherCell.x === cellX && otherCell.y === cellY + 1 && dy === dims.height - 1) shapeHidden.bottom.add(dx);
            if (otherCell.x === cellX - 1 && otherCell.y === cellY && dx === 0) shapeHidden.left.add(dy);
            if (otherCell.x === cellX + 1 && otherCell.y === cellY && dx === dims.width - 1) shapeHidden.right.add(dy);
          });
        });
      }
    }
    hidden.set(shape.id, shapeHidden);
  });
  return hidden;
});

const getBorderStyle = (shape) => {
  const cs = CELL_SIZE.value;
  const dims = getShapeDimensions(shape);
  const hiddenBorders = getHiddenBorders.value.get(shape.id);
  if (!hiddenBorders) return '';

  const cacheKey = `${shape.id}-${dims.width}-${dims.height}-${cs}-${Array.from(hiddenBorders.top).join(',')}-${Array.from(hiddenBorders.right).join(',')}-${Array.from(hiddenBorders.bottom).join(',')}-${Array.from(hiddenBorders.left).join(',')}`;

  if (borderStyleCache.has(cacheKey)) return borderStyleCache.get(cacheKey);

  const width  = dims.width  * cs;
  const height = dims.height * cs;
  let pathParts = [];

  for (let i = 0; i < dims.width; i++) {
    if (!hiddenBorders.top.has(i))
      pathParts.push(`M ${i * cs} 0 L ${(i + 1) * cs} 0`);
  }
  for (let i = 0; i < dims.height; i++) {
    if (!hiddenBorders.right.has(i))
      pathParts.push(`M ${width} ${i * cs} L ${width} ${(i + 1) * cs}`);
  }
  for (let i = dims.width - 1; i >= 0; i--) {
    if (!hiddenBorders.bottom.has(i))
      pathParts.push(`M ${(i + 1) * cs} ${height} L ${i * cs} ${height}`);
  }
  for (let i = dims.height - 1; i >= 0; i--) {
    if (!hiddenBorders.left.has(i))
      pathParts.push(`M 0 ${(i + 1) * cs} L 0 ${i * cs}`);
  }

  const svgUrl = `data:image/svg+xml,${encodeURIComponent(`
    <svg xmlns="http://www.w3.org/2000/svg" width="${width}" height="${height}">
      <path d="${pathParts.join(' ')}" stroke="#1e293b" stroke-width="3" fill="none" stroke-linecap="square" stroke-linejoin="miter"/>
    </svg>
  `)}`;

  const result = `url("${svgUrl}")`;
  borderStyleCache.set(cacheKey, result);
  return result;
};

// ===== DETECCIÓN DE ADYACENCIA =====
const checkIfAdjacent = (shape1, shape2) => {
  const cells1 = getOccupiedCells(shape1);
  const cells2 = getOccupiedCells(shape2);
  
  for (const c1 of cells1) {
    for (const c2 of cells2) {
      const dx = Math.abs(c1.x - c2.x);
      const dy = Math.abs(c1.y - c2.y);
      
      // SOLO permitir conexiones HORIZONTALES (izquierda/derecha)
      // dx === 1 significa que están uno al lado del otro horizontalmente
      // dy === 0 significa que están en la misma fila (y)
      if (dx === 1 && dy === 0) return true;
    }
  }
  return false;
};

// Versión inmediata (sin debounce) - para Undo/Redo
const checkAdjacentBlocksImmediate = () => {
  // Limpiar marcas de separación visual
  shapes.value.forEach(s => {
    if (s.visuallySeparated) {
      delete s.visuallySeparated;
    }
  });
  
  const groups = [];
  
  shapes.value.forEach(s1 => {
    shapes.value.forEach(s2 => {
      if (s1.id >= s2.id) return;
      if (checkIfAdjacent(s1, s2)) {
        let group = groups.find(g => g.includes(s1.id) || g.includes(s2.id));
        if (group) {
          if (!group.includes(s1.id)) group.push(s1.id);
          if (!group.includes(s2.id)) group.push(s2.id);
        } else {
          groups.push([s1.id, s2.id]);
        }
      }
    });
  });

  // Consolidar grupos que se solapan
  let changed = true;
  while (changed) {
    changed = false;
    for (let i = 0; i < groups.length; i++) {
      for (let j = i + 1; j < groups.length; j++) {
        if (groups[i].some(id => groups[j].includes(id))) {
          groups[i] = [...new Set([...groups[i], ...groups[j]])];
          groups.splice(j, 1);
          changed = true;
          break;
        }
      }
      if (changed) break;
    }
  }

  shapeGroups.value = groups;

  // PRIMERO: Resetear colores de unidades a su color base
  shapes.value.forEach(shape => {
    if (shape.units) {
      shape.units.forEach(unit => {
        unit.color = unit.baseColor || unit.color;
      });
    }
  });

  // SEGUNDO: Procesar cada grupo
  shapes.value.forEach(shape => {
    const group = groups.find(g => g.includes(shape.id));
    if (group) {
      shape.isConnected = true;
      
      // Calcular suma del grupo
      shape.groupSum = group.reduce((acc, id) => {
        const s = shapes.value.find(block => block.id === id);
        return acc + (s ? s.value : 0);
      }, 0);
      
      // NUEVO: Colorear por unidades SIEMPRE que haya grupo
      // Solo colorear una vez por grupo (evitar repetir)
      const isFirstInGroup = group[0] === shape.id;
      if (isFirstInGroup) {
        colorearUnidadesDeGrupo(group);
      }
      
      // Mantener groupColor para compatibilidad
      shape.groupColor = getColorBySum(shape.groupSum);
    } else {
      shape.isConnected = false;
      shape.groupSum = null;
      shape.groupColor = null;
    }
  });
  
  updateConnections();
  saveToStorage();
  
  // Limpiar cache de borders cuando cambian las conexiones
  borderStyleCache.clear();
};

// Versión con debounce (optimizada para uso normal)
const checkAdjacentBlocks = () => {
  // Cancelar timer anterior
  if (checkAdjacentTimer) {
    clearTimeout(checkAdjacentTimer);
  }
  
  // Ejecutar después de 50ms de inactividad
  checkAdjacentTimer = setTimeout(() => {
    checkAdjacentBlocksImmediate();
  }, 50);
};

const getBlocksToShowNumber = computed(() => {
  const toShow = new Set();
  shapes.value.forEach(s => {
    if (!s.isConnected) {
      toShow.add(s.id);
    }
  });

  shapeGroups.value.forEach(group => {
    let bestId = null;
    let minY = Infinity, minX = Infinity;
    group.forEach(id => {
      const s = shapes.value.find(block => block.id === id);
      if (s && (s.gridY < minY || (s.gridY === minY && s.gridX < minX))) {
        minY = s.gridY;
        minX = s.gridX;
        bestId = id;
      }
    });
    if (bestId) toShow.add(bestId);
  });

  return toShow;
});

// ===== CONEXIONES =====
const updateConnections = () => {
  const allConnections = [];
  const connectionSet = new Set(); // Para evitar duplicados
  
  shapeGroups.value.forEach(group => {
    if (group.length < 2) return;
    
    for (let i = 0; i < group.length; i++) {
      for (let j = i + 1; j < group.length; j++) {
        const conns = getConnectionsBetweenBlocks(group[i], group[j]);
        conns.forEach(c => {
          // Crear clave única para esta conexión
          const key = `${Math.min(c.shape1Id, c.shape2Id)}-${Math.max(c.shape1Id, c.shape2Id)}-${c.x1}-${c.y1}-${c.x2}-${c.y2}`;
          
          // Solo agregar si no existe
          if (!connectionSet.has(key)) {
            connectionSet.add(key);
            allConnections.push(c);
          }
        });
      }
    }
  });
  
  connections.value = allConnections;
};

const getConnectionsBetweenBlocks = (shape1Id, shape2Id) => {
  const shape1 = shapes.value.find(s => s.id === shape1Id);
  const shape2 = shapes.value.find(s => s.id === shape2Id);
  if (!shape1 || !shape2) return [];
  
  const cells1 = getOccupiedCells(shape1);
  const cells2 = getOccupiedCells(shape2);
  
  // Agrupar conexiones por eje
  const horizontalConnections = [];
  const verticalConnections = [];
  
  cells1.forEach(cell1 => {
    cells2.forEach(cell2 => {
      // Conexión horizontal (bloques arriba/abajo)
      if (cell1.x === cell2.x && Math.abs(cell1.y - cell2.y) === 1) {
        const minY = Math.min(cell1.y, cell2.y);
        const connectionY = minY * CELL_SIZE.value + CELL_SIZE.value;
        horizontalConnections.push({
          x: cell1.x,
          y: connectionY
        });
      }
      // Conexión vertical (bloques izquierda/derecha)
      if (cell1.y === cell2.y && Math.abs(cell1.x - cell2.x) === 1) {
        const minX = Math.min(cell1.x, cell2.x);
        const connectionX = minX * CELL_SIZE.value + CELL_SIZE.value;
        verticalConnections.push({
          x: connectionX,
          y: cell1.y
        });
      }
    });
  });
  
  const connectionsList = [];
  
  // Consolidar conexiones horizontales
  if (horizontalConnections.length > 0) {
    // Agrupar por Y (misma fila)
    const groupedByY = {};
    horizontalConnections.forEach(conn => {
      if (!groupedByY[conn.y]) {
        groupedByY[conn.y] = [];
      }
      groupedByY[conn.y].push(conn.x);
    });
    
    // Crear una línea por grupo
    Object.keys(groupedByY).forEach(y => {
      const xValues = groupedByY[y].sort((a, b) => a - b);
      const minX = Math.min(...xValues);
      const maxX = Math.max(...xValues);
      connectionsList.push({
        x1: minX * CELL_SIZE.value,
        y1: parseFloat(y),
        x2: (maxX + 1) * CELL_SIZE.value,
        y2: parseFloat(y),
        horizontal: true,
        shape1Id,
        shape2Id
      });
    });
  }
  
  // Consolidar conexiones verticales
  if (verticalConnections.length > 0) {
    // Agrupar por X (misma columna)
    const groupedByX = {};
    verticalConnections.forEach(conn => {
      if (!groupedByX[conn.x]) {
        groupedByX[conn.x] = [];
      }
      groupedByX[conn.x].push(conn.y);
    });
    
    // Crear una línea por grupo
    Object.keys(groupedByX).forEach(x => {
      const yValues = groupedByX[x].sort((a, b) => a - b);
      const minY = Math.min(...yValues);
      const maxY = Math.max(...yValues);
      connectionsList.push({
        x1: parseFloat(x),
        y1: minY * CELL_SIZE.value,
        x2: parseFloat(x),
        y2: (maxY + 1) * CELL_SIZE.value,
        horizontal: false,
        shape1Id,
        shape2Id
      });
    });
  }
  
  return connectionsList;
};

const lineIntersectsConnection = (dragStart, dragEnd, connection) => {
  // Aumentar tolerancia para mejor detección
  const tolerance = 15;
  
  // Definir la línea de corte del usuario
  const x1 = dragStart.x;
  const y1 = dragStart.y;
  const x2 = dragEnd.x;
  const y2 = dragEnd.y;
  
  // La línea de conexión ya tiene x1,y1,x2,y2
  const connX1 = connection.x1;
  const connY1 = connection.y1;
  const connX2 = connection.x2;
  const connY2 = connection.y2;
  
  // Calcular intersección real usando determinantes
  const denominator = (x1 - x2) * (connY1 - connY2) - (y1 - y2) * (connX1 - connX2);
  
  // Si las líneas son paralelas, verificar distancia mínima
  if (Math.abs(denominator) < 0.001) {
    // Calcular distancia del punto medio de la línea de corte a la conexión
    const midX = (x1 + x2) / 2;
    const midY = (y1 + y2) / 2;
    const connMidX = (connX1 + connX2) / 2;
    const connMidY = (connY1 + connY2) / 2;
    const dist = Math.sqrt(Math.pow(midX - connMidX, 2) + Math.pow(midY - connMidY, 2));
    return dist < tolerance;
  }
  
  const t = ((x1 - connX1) * (connY1 - connY2) - (y1 - connY1) * (connX1 - connX2)) / denominator;
  const u = -((x1 - x2) * (y1 - connY1) - (y1 - y2) * (x1 - connX1)) / denominator;
  
  // Verificar si la intersección ocurre dentro de ambas líneas
  // Ser más permisivo con los límites
  if (t >= -0.1 && t <= 1.1 && u >= -0.1 && u <= 1.1) {
    return true;
  }
  
  // Verificar también distancia mínima entre las líneas
  // Calcular distancia punto-línea para ambos extremos
  const distStart1 = pointToLineDistance(x1, y1, connX1, connY1, connX2, connY2);
  const distEnd1 = pointToLineDistance(x2, y2, connX1, connY1, connX2, connY2);
  const distStart2 = pointToLineDistance(connX1, connY1, x1, y1, x2, y2);
  const distEnd2 = pointToLineDistance(connX2, connY2, x1, y1, x2, y2);
  
  const minDist = Math.min(distStart1, distEnd1, distStart2, distEnd2);
  return minDist < tolerance;
};

// Función auxiliar para calcular distancia de un punto a una línea
const pointToLineDistance = (px, py, x1, y1, x2, y2) => {
  const A = px - x1;
  const B = py - y1;
  const C = x2 - x1;
  const D = y2 - y1;
  
  const dot = A * C + B * D;
  const lenSq = C * C + D * D;
  let param = -1;
  
  if (lenSq !== 0) {
    param = dot / lenSq;
  }
  
  let xx, yy;
  
  if (param < 0) {
    xx = x1;
    yy = y1;
  } else if (param > 1) {
    xx = x2;
    yy = y2;
  } else {
    xx = x1 + param * C;
    yy = y1 + param * D;
  }
  
  const dx = px - xx;
  const dy = py - yy;
  return Math.sqrt(dx * dx + dy * dy);
};

// ===== SISTEMA DE DESACOPLAMIENTO CON SNAP-TO-LINE (NUEVO) =====

// Estado para el sistema de snap
const snapToLineState = ref({
  active: false,              // Si está activo el snap
  currentConnection: null,    // Conexión actual donde está el cursor
  startProgress: null,        // Progreso donde empezó el trazo (0 a 1)
  currentProgress: 0,         // Progreso actual del cursor
  startVertex: null,          // Vértice de inicio ('start' o 'end')
  snappedPosition: null,      // Posición actual snapeada
  cutConnections: new Set(),  // Conexiones ya cortadas en este trazo
  reachedOppositeVertex: false // Si llegó al vértice opuesto
});

/**
 * Desacopla dos bloques usando RelationshipManager
 * @param {Object} connection - Objeto con shape1Id y shape2Id
 */
const cutConnection = (connection) => {
  if (!connection.shape1Id || !connection.shape2Id) return;
  
  const shape1 = shapes.value.find(s => s.id === connection.shape1Id);
  const shape2 = shapes.value.find(s => s.id === connection.shape2Id);
  
  if (!shape1 || !shape2) return;
  
  // Separar bloques automáticamente
  separateBlocks(shape1, shape2);
  
  // Marcar como cortada
  const connKey = `${connection.shape1Id}-${connection.shape2Id}`;
  snapToLineState.value.cutConnections.add(connKey);
  
  // Recalcular grupos
  checkAdjacentBlocks();
};

/**
 * Encuentra la conexión más cercana al cursor
 * @param {number} x - Coordenada X del cursor
 * @param {number} y - Coordenada Y del cursor
 * @returns {Object|null} - { connection, distance, progress, snappedPoint }
 */
const findNearestConnection = (x, y) => {
  let nearest = null;
  let minDistance = 40; // Radio de snap en píxeles (aumentado)
  
  connections.value.forEach(conn => {
    // Calcular punto más cercano en la línea
    const dx = conn.x2 - conn.x1;
    const dy = conn.y2 - conn.y1;
    const lengthSquared = dx * dx + dy * dy;
    
    if (lengthSquared === 0) return;
    
    // Proyección del cursor sobre la línea
    let t = ((x - conn.x1) * dx + (y - conn.y1) * dy) / lengthSquared;
    t = Math.max(0, Math.min(1, t)); // Clamp entre 0 y 1
    
    // Punto más cercano en la línea
    const closestX = conn.x1 + t * dx;
    const closestY = conn.y1 + t * dy;
    
    // Distancia del cursor al punto más cercano
    const dist = Math.sqrt(Math.pow(x - closestX, 2) + Math.pow(y - closestY, 2));
    
    if (dist < minDistance) {
      minDistance = dist;
      nearest = {
        connection: conn,
        distance: dist,
        progress: t,
        snappedPoint: { x: closestX, y: closestY }
      };
    }
  });
  
  return nearest;
};

/**
 * Verifica si una conexión ya fue cortada
 */
const isConnectionCut = (connection) => {
  const key1 = `${connection.shape1Id}-${connection.shape2Id}`;
  const key2 = `${connection.shape2Id}-${connection.shape1Id}`;
  return snapToLineState.value.cutConnections.has(key1) || 
         snapToLineState.value.cutConnections.has(key2);
};

/**
 * Determina qué vértice está más cerca del progreso dado
 */
const getClosestVertex = (progress) => {
  return progress < 0.5 ? 'start' : 'end';
};

/**
 * Verifica si llegó al vértice opuesto
 */
const hasReachedOppositeVertex = (startVertex, currentProgress) => {
  if (startVertex === 'start') {
    // Empezó en start (0), debe llegar a end (1)
    return currentProgress > 0.9; // 90% = prácticamente en el vértice
  } else {
    // Empezó en end (1), debe llegar a start (0)
    return currentProgress < 0.1; // 10% = prácticamente en el vértice
  }
};

/**
 * Inicia el modo de corte con snap
 */
const handleSnapCutStart = (e) => {
  if (!cuttingMode.value) return;
  
  const gridEl = document.querySelector('.blocks-layer-transparent');
  if (!gridEl) return;
  
  const rect = gridEl.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  
  // Buscar conexión cercana
  const nearest = findNearestConnection(x, y);
  
  if (nearest) {
    const startVertex = getClosestVertex(nearest.progress);
    
    snapToLineState.value = {
      active: true,
      currentConnection: nearest.connection,
      startProgress: nearest.progress,
      currentProgress: nearest.progress,
      startVertex: startVertex,
      snappedPosition: nearest.snappedPoint,
      cutConnections: new Set(),
      reachedOppositeVertex: false
    };
  }
};

/**
 * Encuentra todas las conexiones que intersectan una línea recta
 * Excluye las conexiones que el cursor ya visitó (actual y anterior)
 * @param {number} x1 - Punto inicial X
 * @param {number} y1 - Punto inicial Y
 * @param {number} x2 - Punto final X
 * @param {number} y2 - Punto final Y
 * @param {Object} currentConn - Conexión actual (para excluir)
 * @param {Object} previousConn - Conexión anterior (para excluir)
 * @returns {Array} Array de conexiones que intersectan la línea
 */
const findConnectionsCrossedByLine = (x1, y1, x2, y2, currentConn = null, previousConn = null) => {
  const crossedConnections = [];
  
  connections.value.forEach(conn => {
    // No incluir la conexión actual ni la anterior
    if (conn === currentConn || conn === previousConn) return;
    
    // Verificar si la línea (x1,y1)→(x2,y2) intersecta la conexión
    if (doLinesIntersect(x1, y1, x2, y2, conn.x1, conn.y1, conn.x2, conn.y2)) {
      crossedConnections.push(conn);
    }
  });
  
  return crossedConnections;
};

/**
 * Verifica si dos segmentos de línea se intersectan
 * @returns {boolean} true si se intersectan
 */
const doLinesIntersect = (x1, y1, x2, y2, x3, y3, x4, y4) => {
  const denom = (x1 - x2) * (y3 - y4) - (y1 - y2) * (x3 - x4);
  
  // Líneas paralelas
  if (Math.abs(denom) < 0.0001) return false;
  
  const t = ((x1 - x3) * (y3 - y4) - (y1 - y3) * (x3 - x4)) / denom;
  const u = -((x1 - x2) * (y1 - y3) - (y1 - y2) * (x1 - x3)) / denom;
  
  // Intersectan si ambos parámetros están entre 0 y 1
  return (t >= 0 && t <= 1 && u >= 0 && u <= 1);
};

/**
 * Actualiza la posición durante el drag con snap
 */
const handleSnapCutMove = (e) => {
  if (!cuttingMode.value || !snapToLineState.value.active) return;
  
  const gridEl = document.querySelector('.blocks-layer-transparent');
  if (!gridEl) return;
  
  const rect = gridEl.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  
  // Buscar conexión más cercana al cursor
  const nearest = findNearestConnection(x, y);
  
  if (!nearest) {
    // Sin conexión cercana, mantener estado
    return;
  }
  
  const prevConnection = snapToLineState.value.currentConnection;
  const prevPosition = snapToLineState.value.snappedPosition;
  
  // Actualizar posición actual
  snapToLineState.value.snappedPosition = nearest.snappedPoint;
  snapToLineState.value.currentProgress = nearest.progress;
  snapToLineState.value.currentConnection = nearest.connection;
  
  // Si hay una posición previa, verificar camino
  if (prevPosition && prevConnection) {
    
    // Calcular línea directa desde posición anterior a posición actual
    const dx = nearest.snappedPoint.x - prevPosition.x;
    const dy = nearest.snappedPoint.y - prevPosition.y;
    const distance = Math.sqrt(dx * dx + dy * dy);
    
    // Solo buscar camino si el movimiento fue significativo (>20px)
    if (distance > 20) {
      
      // Encontrar conexiones que cruza la línea directa
      // IMPORTANTE: Excluir conexión actual y anterior para evitar cortes en cruces
      const crossedConnections = findConnectionsCrossedByLine(
        prevPosition.x,
        prevPosition.y,
        nearest.snappedPoint.x,
        nearest.snappedPoint.y,
        nearest.connection,  // Excluir conexión actual
        prevConnection       // Excluir conexión anterior
      );
      
      
      // Cortar todas las conexiones cruzadas que no estén ya cortadas
      crossedConnections.forEach(conn => {
        if (!isConnectionCut(conn)) {
          cutConnection(conn);
        }
      });
    }
  }
  
  // Verificar si llegamos al vértice opuesto en la conexión actual
  if (snapToLineState.value.startVertex) {
    if (hasReachedOppositeVertex(snapToLineState.value.startVertex, nearest.progress)) {
      if (!snapToLineState.value.reachedOppositeVertex) {
        snapToLineState.value.reachedOppositeVertex = true;
      }
    }
  }
  
  // Si cambiamos de conexión, reiniciar estado de vértice
  if (prevConnection && prevConnection !== nearest.connection) {
    snapToLineState.value.startProgress = nearest.progress;
    snapToLineState.value.startVertex = getClosestVertex(nearest.progress);
    snapToLineState.value.reachedOppositeVertex = false;
  }
};

/**
 * Finaliza el corte con snap
 */
const handleSnapCutEnd = () => {
  if (!cuttingMode.value) return;
  
  // Si llegamos al vértice opuesto, cortar
  if (snapToLineState.value.currentConnection && 
      snapToLineState.value.reachedOppositeVertex &&
      !isConnectionCut(snapToLineState.value.currentConnection)) {
    cutConnection(snapToLineState.value.currentConnection);
  }
  
  // Guardar en historial si se cortó algo
  if (snapToLineState.value.cutConnections.size > 0) {
    saveToHistory();
  }
  
  // Resetear estado
  snapToLineState.value = {
    active: false,
    currentConnection: null,
    startProgress: null,
    currentProgress: 0,
    startVertex: null,
    snappedPosition: null,
    cutConnections: new Set(),
    reachedOppositeVertex: false
  };
};

const handleCutDragStart = (e, connection) => {
  if (!cuttingMode.value) return;
  e.stopPropagation();
  e.preventDefault();
  
  const gridEl = document.querySelector('.blocks-layer-transparent');
  if (!gridEl) return;
  
  const rect = gridEl.getBoundingClientRect();
  
  cutDragStart.value = { 
    x: e.clientX - rect.left, 
    y: e.clientY - rect.top, 
    connection,
    // Guardar qué vértice se agarró
    startVertex: getClosestVertex(e.clientX - rect.left, e.clientY - rect.top, connection)
  };
  isCutDragging.value = false;
};

const handleCutDragMove = (e) => {
  if (!cutDragStart.value) return;
  
  const gridEl = document.querySelector('.blocks-layer-transparent');
  if (!gridEl) return;
  
  const rect = gridEl.getBoundingClientRect();
  const currentX = e.clientX - rect.left;
  const currentY = e.clientY - rect.top;
  
  isCutDragging.value = true;
  cutDragEnd.value = { x: currentX, y: currentY };
  
  // Determinar el vértice objetivo (el opuesto al que agarraste)
  const connection = cutDragStart.value.connection;
  const targetVertex = cutDragStart.value.startVertex === 'start' 
    ? { x: connection.x2, y: connection.y2 }
    : { x: connection.x1, y: connection.y1 };
  
  // Verificar si el cursor está cerca del vértice objetivo
  const distToTarget = Math.sqrt(
    Math.pow(currentX - targetVertex.x, 2) + 
    Math.pow(currentY - targetVertex.y, 2)
  );
  
  // Si llegaste al vértice opuesto (dentro de 20px), marcar para cortar
  if (distToTarget < 20) {
    connectionsToCut.value = [connection];
  } else {
    connectionsToCut.value = [];
  }
};

const handleCutDragEnd = () => {
  // Solo cortar si llegaste al vértice opuesto
  if (isCutDragging.value && connectionsToCut.value.length > 0) {
    connectionsToCut.value.forEach(conn => {
      cutConnection(conn);
    });
  }
  
  cutDragStart.value = null;
  cutDragEnd.value = null;
  isCutDragging.value = false;
  connectionsToCut.value = [];
  cutTrailPoints.value = [];
  alreadyCutConnections.value.clear();
};

// ===== MOVIMIENTO GRUPAL =====
const canMoveGroup = (groupIds, deltaX, deltaY) => {
  const groupIdSet = new Set(groupIds);
  
  for (const id of groupIds) {
    const shape = shapes.value.find(s => s.id === id);
    if (!shape) continue;
    
    const dims = getShapeDimensions(shape);
    const newX = shape.gridX + deltaX;
    const newY = shape.gridY + deltaY;
    
    for (let dy = 0; dy < dims.height; dy++) {
      for (let dx = 0; dx < dims.width; dx++) {
        const x = newX + dx;
        const y = newY + dy;
        
        if (x >= GRID_WIDTH || y >= GRID_HEIGHT || x < 0 || y < 0) return false;
        
        const occupant = shapes.value.find(s => {
          if (groupIdSet.has(s.id)) return false;
          const cells = getOccupiedCells(s);
          return cells.some(c => c.x === x && c.y === y);
        });
        
        if (occupant) return false;
      }
    }
  }
  
  return true;
};

const moveGroup = (groupIds, deltaX, deltaY) => {
  groupIds.forEach(id => {
    const shape = shapes.value.find(s => s.id === id);
    if (shape) {
      shape.gridX += deltaX;
      shape.gridY += deltaY;
    }
  });
};

const findSnapPosition = (shape, targetX, targetY) => {
  if (!dragStart.value) return { x: targetX, y: targetY };
  
  const dims = getShapeDimensions(shape);
  const snapDistance = 2;
  
  if (!isPositionAvailable(targetX, targetY, dims.width, dims.height, shape.id)) {
    return { x: targetX, y: targetY };
  }
  
  let bestSnap = null;
  let minDistance = snapDistance + 1;
  const groupIdSet = new Set(dragStart.value.groupIds);
  
  shapes.value.forEach(otherShape => {
    if (groupIdSet.has(otherShape.id)) return;
    
    const otherCells = getOccupiedCells(otherShape);
    
    for (let dy = 0; dy < dims.height; dy++) {
      for (let dx = 0; dx < dims.width; dx++) {
        const cellX = targetX + dx;
        const cellY = targetY + dy;
        
        otherCells.forEach(otherCell => {
          const snapPositions = [
            { x: otherCell.x + 1, y: otherCell.y, offsetX: otherCell.x + 1 - dx, offsetY: otherCell.y - dy },
            { x: otherCell.x - 1, y: otherCell.y, offsetX: otherCell.x - 1 - dx, offsetY: otherCell.y - dy },
            { x: otherCell.x, y: otherCell.y + 1, offsetX: otherCell.x - dx, offsetY: otherCell.y + 1 - dy },
            { x: otherCell.x, y: otherCell.y - 1, offsetX: otherCell.x - dx, offsetY: otherCell.y - 1 - dy }
          ];
          
          snapPositions.forEach(snapPos => {
            if (snapPos.x === cellX && snapPos.y === cellY) {
              const distance = Math.abs(targetX - snapPos.offsetX) + Math.abs(targetY - snapPos.offsetY);
              
              if (distance < minDistance && distance <= snapDistance) {
                const deltaX = snapPos.offsetX - dragStart.value.anchorX;
                const deltaY = snapPos.offsetY - dragStart.value.anchorY;
                
                if (canMoveGroup(dragStart.value.groupIds, deltaX, deltaY)) {
                  minDistance = distance;
                  bestSnap = { x: snapPos.offsetX, y: snapPos.offsetY };
                }
              }
            }
          });
        });
      }
    }
  });
  
  return bestSnap || { x: targetX, y: targetY };
};

// ===== ACCIONES =====
const addBlock = (blockType) => {
  // Verificar límite máximo de bloques
  if (shapes.value.length >= props.maxBloques) {
    showToastNotification(`Límite alcanzado: máximo ${props.maxBloques} bloques`);
    return;
  }
  
  const config = BLOCKS_CONFIG[blockType];
  const pos = findAvailablePosition(config.width, config.height);
  
  if (!pos) {
    showToastNotification('¡No hay espacio disponible!');
    return;
  }
  
  // Crear bloque con sistema de cuadrantes
  const newBlock = createBlockWithQuadrants(blockType, pos.x, pos.y, nextId++);
  shapes.value.push(newBlock);
  
  // Registrar en RelationshipManager
  
  checkAdjacentBlocks();
  saveToHistory(); // Guardar en historial
};

const toggleCuttingMode = () => {
  cuttingMode.value = !cuttingMode.value;
  selectedShapeId.value = null;
  
  // Modo cortar bloques ELIMINADO
};

// ===== CORTE DE BLOQUES =====
// getShapeAtPosition ELIMINADO

// handleBlockCutStart, handleBlockCutMove, handleBlockCutEnd ELIMINADOS

// lineCrossesBlockInterior y lineIntersectsSegment ELIMINADOS

// toggleBlockCuttingMode ELIMINADO

const deleteShape = (id) => {
  // Desregistrar del RelationshipManager
  
  shapes.value = shapes.value.filter(s => s.id !== id);
  selectedShapeId.value = null;
  checkAdjacentBlocks();
  saveToHistory(); // Guardar en historial
};

const clearAll = () => {
  // Limpiar RelationshipManager
  
  shapes.value = [];
  selectedShapeId.value = null;
  shapeGroups.value = [];
  connections.value = [];
  saveToStorage();
  saveToHistory(); // Guardar en historial
};

// ===== EVENT HANDLERS =====
const handleBlockMouseDown = (e, shapeId) => {
  // Si está en modo cortar conexiones, ignorar
  if (cuttingMode.value) return;
  
  // Modo cortar bloques ELIMINADO
  
  e.stopPropagation();
  e.preventDefault(); // Evitar selección de texto
  selectedShapeId.value = shapeId;
  draggingId.value = shapeId;
  isDragging.value = false;
  
  const shape = shapes.value.find(s => s.id === shapeId);
  if (!shape) return;
  
  const group = shapeGroups.value.find(g => g.includes(shapeId));
  
  dragStart.value = {
    groupIds: group || [shapeId],
    initialPositions: (group || [shapeId]).map(id => {
      const s = shapes.value.find(sh => sh.id === id);
      return { id, gridX: s.gridX, gridY: s.gridY };
    }),
    mouseX: e.clientX,
    mouseY: e.clientY,
    anchorX: shape.gridX,
    anchorY: shape.gridY
  };
};

const handleMouseMove = (e) => {
  if (!draggingId.value || !dragStart.value) return;
  
  const deltaX = Math.abs(e.clientX - dragStart.value.mouseX);
  const deltaY = Math.abs(e.clientY - dragStart.value.mouseY);
  
  if (deltaX > 3 || deltaY > 3) isDragging.value = true;
  if (!isDragging.value) return;
  
  const gridEl = document.querySelector('.grid-transparent');
  if (!gridEl) return;
  
  const rect = gridEl.getBoundingClientRect();
  
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  const rawGridX = Math.floor(x / CELL_SIZE.value);
  const rawGridY = Math.floor(y / CELL_SIZE.value);
  
  const targetDeltaX = rawGridX - dragStart.value.anchorX;
  const targetDeltaY = rawGridY - dragStart.value.anchorY;
  
  // Restaurar posiciones iniciales
  dragStart.value.initialPositions.forEach(pos => {
    const shape = shapes.value.find(s => s.id === pos.id);
    if (shape) {
      shape.gridX = pos.gridX;
      shape.gridY = pos.gridY;
    }
  });
  
  // Solo mover si la posición es válida (validación eficiente)
  if (canMoveGroup(dragStart.value.groupIds, targetDeltaX, targetDeltaY)) {
    moveGroup(dragStart.value.groupIds, targetDeltaX, targetDeltaY);
    // Guardar el último movimiento válido
    dragStart.value.lastValidDeltaX = targetDeltaX;
    dragStart.value.lastValidDeltaY = targetDeltaY;
  } else if (dragStart.value.lastValidDeltaX !== undefined) {
    // Si no es válido, mantener la última posición válida
    moveGroup(dragStart.value.groupIds, dragStart.value.lastValidDeltaX, dragStart.value.lastValidDeltaY);
  }
};

/**
 * Verifica si mover un grupo con el delta dado causaría superar el límite de unidades.
 * Recibe initialPositions para construir la simulación limpia.
 */
const wouldExceedUnitLimitByDelta = (movingBlockIds, initialPositions, deltaX, deltaY) => {
  if (!props.maxPorGrupo || props.maxPorGrupo === Infinity) {
    return { valid: true, totalUnits: 0, message: '' };
  }

  // Copia de bloques con posiciones simuladas desde initialPositions
  const simulatedBlocks = shapes.value.map(shape => {
    if (movingBlockIds.includes(shape.id)) {
      const orig = initialPositions.find(p => p.id === shape.id);
      if (orig) {
        return { ...shape, gridX: orig.gridX + deltaX, gridY: orig.gridY + deltaY };
      }
    }
    return { ...shape };
  });

  // Detección de adyacencia local sobre simulatedBlocks (solo horizontal)
  const simAdjacent = (a, b) => {
    const dimsA = getShapeDimensions(a);
    const dimsB = getShapeDimensions(b);
    for (let dy = 0; dy < dimsA.height; dy++) {
      for (let dx = 0; dx < dimsA.width; dx++) {
        const cx = a.gridX + dx;
        const cy = a.gridY + dy;
        for (let ey = 0; ey < dimsB.height; ey++) {
          for (let ex = 0; ex < dimsB.width; ex++) {
            if (Math.abs(cx - (b.gridX + ex)) === 1 && cy === b.gridY + ey) return true;
          }
        }
      }
    }
    return false;
  };

  // Construir grupos sobre simulatedBlocks
  const simGroups = [];
  for (let i = 0; i < simulatedBlocks.length; i++) {
    for (let j = i + 1; j < simulatedBlocks.length; j++) {
      if (simAdjacent(simulatedBlocks[i], simulatedBlocks[j])) {
        const idI = simulatedBlocks[i].id;
        const idJ = simulatedBlocks[j].id;
        let group = simGroups.find(g => g.includes(idI) || g.includes(idJ));
        if (group) {
          if (!group.includes(idI)) group.push(idI);
          if (!group.includes(idJ)) group.push(idJ);
        } else {
          simGroups.push([idI, idJ]);
        }
      }
    }
  }

  // Consolidar grupos solapados
  let changed = true;
  while (changed) {
    changed = false;
    for (let i = 0; i < simGroups.length; i++) {
      for (let j = i + 1; j < simGroups.length; j++) {
        if (simGroups[i].some(id => simGroups[j].includes(id))) {
          simGroups[i] = [...new Set([...simGroups[i], ...simGroups[j]])];
          simGroups.splice(j, 1);
          changed = true;
          break;
        }
      }
      if (changed) break;
    }
  }

  // Verificar cada grupo que contenga al menos un bloque movido
  for (const group of simGroups) {
    const hasMovingBlock = group.some(id => movingBlockIds.includes(id));
    if (!hasMovingBlock) continue;

    let totalUnits = 0;
    for (const blockId of group) {
      const block = simulatedBlocks.find(s => s.id === blockId);
      if (block) {
        totalUnits += (block.units && block.units.length > 0) ? block.units.length : (block.value || 1);
      }
    }
    if (totalUnits > props.maxPorGrupo) {
      return {
        valid: false,
        totalUnits,
        message: `Límite de grupo: máximo ${props.maxPorGrupo} unidades.`
      };
    }
  }

  return { valid: true, totalUnits: 0, message: '' };
};

// Alias para compatibilidad con llamadas anteriores
const wouldExceedUnitLimit = (movingBlockIds, snapX, snapY) => {
  const currentDragStart = dragStart.value;
  if (!currentDragStart) return { valid: true, totalUnits: 0, message: '' };
  const deltaX = snapX - currentDragStart.anchorX;
  const deltaY = snapY - currentDragStart.anchorY;
  return wouldExceedUnitLimitByDelta(movingBlockIds, currentDragStart.initialPositions, deltaX, deltaY);
};

const handleMouseUp = () => {
  const wasDrawing = isDragging.value;
  const currentDragStart = dragStart.value;
  
  draggingId.value = null;
  isDragging.value = false;
  
  if (wasDrawing && currentDragStart) {
    const mainShape = shapes.value.find(s => s.id === selectedShapeId.value);
    
    if (mainShape) {
      // Intentar snap a bloques adyacentes
      const snapPos = findSnapPosition(mainShape, mainShape.gridX, mainShape.gridY);
      
      if (snapPos.x !== mainShape.gridX || snapPos.y !== mainShape.gridY) {
        const snapDeltaX = snapPos.x - currentDragStart.anchorX;
        const snapDeltaY = snapPos.y - currentDragStart.anchorY;
        
        // Restaurar posiciones iniciales antes de cualquier verificación
        currentDragStart.initialPositions.forEach(pos => {
          const shape = shapes.value.find(s => s.id === pos.id);
          if (shape) {
            shape.gridX = pos.gridX;
            shape.gridY = pos.gridY;
          }
        });

        // VERIFICAR LÍMITE antes de aplicar el snap
        const limitCheck = wouldExceedUnitLimitByDelta(
          currentDragStart.groupIds,
          currentDragStart.initialPositions,
          snapDeltaX,
          snapDeltaY
        );

        if (!limitCheck.valid) {
          showToastNotification(limitCheck.message);
          dragStart.value = null;
          checkAdjacentBlocks();
          saveToHistory();
          return;
        }
        
        // Aplicar snap si es válido
        if (canMoveGroup(currentDragStart.groupIds, snapDeltaX, snapDeltaY)) {
          moveGroup(currentDragStart.groupIds, snapDeltaX, snapDeltaY);
        } else if (currentDragStart.lastValidDeltaX !== undefined) {
          // Si el snap no es válido, verificar la última posición válida también
          const lastLimitCheck = wouldExceedUnitLimitByDelta(
            currentDragStart.groupIds,
            currentDragStart.initialPositions,
            currentDragStart.lastValidDeltaX,
            currentDragStart.lastValidDeltaY
          );
          if (lastLimitCheck.valid) {
            moveGroup(currentDragStart.groupIds, currentDragStart.lastValidDeltaX, currentDragStart.lastValidDeltaY);
          }
          // Si tampoco es válido, los bloques ya están en initialPositions (restaurados arriba)
        }
      } else {
        // Sin snap: verificar la posición donde quedó el drag
        const dragDeltaX = currentDragStart.lastValidDeltaX;
        const dragDeltaY = currentDragStart.lastValidDeltaY;

        if (dragDeltaX !== undefined && dragDeltaY !== undefined) {
          const limitCheck = wouldExceedUnitLimitByDelta(
            currentDragStart.groupIds,
            currentDragStart.initialPositions,
            dragDeltaX,
            dragDeltaY
          );

          if (!limitCheck.valid) {
            currentDragStart.initialPositions.forEach(pos => {
              const shape = shapes.value.find(s => s.id === pos.id);
              if (shape) {
                shape.gridX = pos.gridX;
                shape.gridY = pos.gridY;
              }
            });
            showToastNotification(limitCheck.message);
            dragStart.value = null;
            checkAdjacentBlocks();
            saveToHistory();
            return;
          }
        }
      }
    }
    
    checkAdjacentBlocks();
    saveToHistory();
  }
  
  dragStart.value = null;
};

const handleGridCanvasClick = (e) => {
  if (e.target.classList.contains('grid-cell-transparent') || 
      e.target.classList.contains('grid-fullscreen') ||
      e.target.classList.contains('grid-center') ||
      e.target.classList.contains('grid-transparent')) {
    selectedShapeId.value = null;
  }
};

const handleCutModeMouseDown = (e) => {
  // Si está en modo cortar conexiones (con snap)
  if (cuttingMode.value) {
    e.stopPropagation();
    e.preventDefault();
    handleSnapCutStart(e);
    return;
  }
  
  // Modo corte de bloques ELIMINADO
};

const toggleOverlay = () => showOverlay.value = !showOverlay.value;
const closeOverlay = () => showOverlay.value = false;
const selectTool = (tool) => addBlock(tool);

// NUEVO: Dispersar bloques aleatorios en el grid
const spawnRandomBlocks = () => {
  
  // 1. Verificar si hay bloques
  const hasBlocks = shapes.value.length > 0;
  
  if (hasBlocks) {
    // CASO A: HAY BLOQUES → Mezclar posiciones sin cambiar estructura
    
    const blocksToShuffle = [...shapes.value];
    const shuffledPositions = [];
    
    // Crear grid temporal para tracking de ocupación
    const occupied = new Set();
    
    // Función helper para verificar si una posición está libre (CON BUFFER de 1 celda)
    const isPositionFree = (x, y, width, height) => {
      // Verificar no solo el bloque, sino también 1 celda alrededor (buffer)
      for (let dy = -1; dy <= height; dy++) {
        for (let dx = -1; dx <= width; dx++) {
          const checkX = x + dx;
          const checkY = y + dy;
          
          // Verificar límites del grid
          if (checkX < 0 || checkX >= GRID_WIDTH || checkY < 0 || checkY >= GRID_HEIGHT) {
            // Si el buffer sale del grid, no es problema para las celdas fuera
            // Solo verificar que el bloque mismo esté dentro
            if (dx >= 0 && dx < width && dy >= 0 && dy < height) {
              return false; // El bloque sale del grid
            }
            continue;
          }
          
          const key = `${checkX},${checkY}`;
          if (occupied.has(key)) return false; // Hay algo en el buffer
        }
      }
      return true;
    };
    
    // Función helper para marcar posición como ocupada (CON BUFFER de 1 celda)
    const markOccupied = (x, y, width, height) => {
      // Marcar el bloque Y 1 celda alrededor
      for (let dy = -1; dy <= height; dy++) {
        for (let dx = -1; dx <= width; dx++) {
          const markX = x + dx;
          const markY = y + dy;
          
          // Solo marcar dentro del grid
          if (markX >= 0 && markX < GRID_WIDTH && markY >= 0 && markY < GRID_HEIGHT) {
            occupied.add(`${markX},${markY}`);
          }
        }
      }
    };
    
    // Generar posiciones aleatorias únicas para cada bloque
    blocksToShuffle.forEach(block => {
      const dims = getShapeDimensions(block);
      let placed = false;
      let attempts = 0;
      const maxAttempts = 200;
      
      while (!placed && attempts < maxAttempts) {
        attempts++;
        
        // Posición aleatoria
        const gridX = Math.floor(Math.random() * (GRID_WIDTH - dims.width + 1));
        const gridY = Math.floor(Math.random() * (GRID_HEIGHT - dims.height + 1));
        
        // Verificar que no colisione con bloques ya colocados
        if (isPositionFree(gridX, gridY, dims.width, dims.height)) {
          shuffledPositions.push({ blockId: block.id, gridX, gridY });
          markOccupied(gridX, gridY, dims.width, dims.height);
          placed = true;
        }
      }
      
      if (!placed) {
        // Si no encuentra espacio, mantener posición original
        shuffledPositions.push({ blockId: block.id, gridX: block.gridX, gridY: block.gridY });
      }
    });
    
    // Aplicar nuevas posiciones
    shuffledPositions.forEach(pos => {
      const block = shapes.value.find(s => s.id === pos.blockId);
      if (block) {
        block.gridX = pos.gridX;
        block.gridY = pos.gridY;
      }
    });
    
    
  } else {
    // CASO B: VACÍO → Generar bloques nuevos aleatorios
    const desiredBlocks = props.cantidadBl
    const numBlocks = Math.min(desiredBlocks, props.maxBloques); // Respetar límite
    const maxValue = Math.min(1, props.bloquesbarra); // Respetar bloques disponibles
    
    
    // Crear grid temporal para tracking de ocupación (con buffer)
    const occupied = new Set();
    
    // Función helper para verificar si una posición está libre (CON BUFFER de 1 celda)
    const isPositionFree = (x, y, width, height) => {
      // Verificar no solo el bloque, sino también 1 celda alrededor (buffer)
      for (let dy = -1; dy <= height; dy++) {
        for (let dx = -1; dx <= width; dx++) {
          const checkX = x + dx;
          const checkY = y + dy;
          
          // Verificar límites del grid
          if (checkX < 0 || checkX >= GRID_WIDTH || checkY < 0 || checkY >= GRID_HEIGHT) {
            // Si el buffer sale del grid, no es problema para las celdas fuera
            // Solo verificar que el bloque mismo esté dentro
            if (dx >= 0 && dx < width && dy >= 0 && dy < height) {
              return false; // El bloque sale del grid
            }
            continue;
          }
          
          const key = `${checkX},${checkY}`;
          if (occupied.has(key)) return false; // Hay algo en el buffer
        }
      }
      return true;
    };
    
    // Función helper para marcar posición como ocupada (CON BUFFER de 1 celda)
    const markOccupied = (x, y, width, height) => {
      // Marcar el bloque Y 1 celda alrededor
      for (let dy = -1; dy <= height; dy++) {
        for (let dx = -1; dx <= width; dx++) {
          const markX = x + dx;
          const markY = y + dy;
          
          // Solo marcar dentro del grid
          if (markX >= 0 && markX < GRID_WIDTH && markY >= 0 && markY < GRID_HEIGHT) {
            occupied.add(`${markX},${markY}`);
          }
        }
      }
    };
    
    let created = 0;
    let attempts = 0;
    const maxAttempts = numBlocks * 50;
    
    while (created < numBlocks && attempts < maxAttempts) {
      attempts++;
      
      const value = 1 + Math.floor(Math.random() * maxValue);
      const blockTypes = ['uno', 'dos', 'tres', 'cuatro', 'cinco', 'seis'];
      const blockType = blockTypes[value - 1];
      
      if (!BLOCKS_CONFIG[blockType]) continue;
      
      const config = BLOCKS_CONFIG[blockType];
      
      // Posición aleatoria
      const gridX = Math.floor(Math.random() * (GRID_WIDTH - config.width + 1));
      const gridY = Math.floor(Math.random() * (GRID_HEIGHT - config.height + 1));
      
      // Verificar que esté disponible (con buffer)
      if (!isPositionFree(gridX, gridY, config.width, config.height)) {
        continue;
      }
      
      // Crear bloque usando la función existente
      const newBlock = createBlockWithQuadrants(blockType, gridX, gridY, nextId++);
      
      shapes.value.push(newBlock);
      markOccupied(gridX, gridY, config.width, config.height);
      created++;
    }
    
  }
  
  // Verificar acoplamientos
  checkAdjacentBlocks();
  saveToHistory();
};

// ===== LIFECYCLE =====
const handleKeyDown = (e) => {
  // Ctrl+Z o Cmd+Z para Undo
  if ((e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey) {
    e.preventDefault();
    undo();
  }
  // Ctrl+Y o Ctrl+Shift+Z o Cmd+Shift+Z para Redo
  if ((e.ctrlKey || e.metaKey) && (e.key === 'y' || (e.key === 'z' && e.shiftKey))) {
    e.preventDefault();
    redo();
  }
};

// ===== CREAR BLOQUES INICIALES =====
const createInitialBlocks = () => {
  shapes.value = [];

  const W = GRID_WIDTH;
  const H = GRID_HEIGHT;
  const midW = Math.floor(W / 2);
  const midH = Math.floor(H / 2);

  // Layouts según cantidad de grupos
  // Cada cuadrante: { x0, y0, x1, y1 } (x1/y1 exclusivos)
  const layouts = {
    1: [
      { x0: 0,    y0: 0,    x1: W,    y1: H    }, // centro completo
    ],
    2: [
      { x0: 0,    y0: 0,    x1: W,    y1: midH }, // arriba
      { x0: 0,    y0: midH, x1: W,    y1: H    }, // abajo
    ],
    3: [
      { x0: 0,    y0: 0,    x1: W,    y1: midH }, // arriba (ancho completo)
      { x0: 0,    y0: midH, x1: midW, y1: H    }, // abajo izquierda
      { x0: midW, y0: midH, x1: W,    y1: H    }, // abajo derecha
    ],
    4: [
      { x0: 0,    y0: 0,    x1: midW, y1: midH }, // arriba izquierda
      { x0: midW, y0: 0,    x1: W,    y1: midH }, // arriba derecha
      { x0: 0,    y0: midH, x1: midW, y1: H    }, // abajo izquierda
      { x0: midW, y0: midH, x1: W,    y1: H    }, // abajo derecha
    ],
  };

  // Para 5+ grupos: cuadrantes de 4 ciclando
  const getQuadrant = (index, total) => {
    if (total <= 4 && layouts[total]) {
      return layouts[total][index];
    }
    // 5+: usar cuadrantes de 4 ciclando
    return layouts[4][index % 4];
  };

  const placeGroupInQuadrant = (count, q) => {
    const occupied = new Set();

    const markCell = (x, y) => {
      for (let dy = -1; dy <= 1; dy++) {
        for (let dx = -1; dx <= 1; dx++) {
          occupied.add(`${x + dx},${y + dy}`);
        }
      }
    };

    // Candidatos dentro del cuadrante, mezclados aleatoriamente
    const candidates = [];
    for (let y = q.y0; y < q.y1; y++) {
      for (let x = q.x0; x < q.x1; x++) {
        candidates.push({ x, y });
      }
    }
    for (let i = candidates.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [candidates[i], candidates[j]] = [candidates[j], candidates[i]];
    }

    let placed = 0;
    for (const { x, y } of candidates) {
      if (placed >= count) break;
      if (occupied.has(`${x},${y}`)) continue;
      if (!isPositionAvailable(x, y, 1, 1, null)) continue;

      const newBlock = createBlockWithQuadrants('uno', x, y, nextId++);
      shapes.value.push(newBlock);
      markCell(x, y);
      placed++;
    }

    if (placed < count) {
      console.warn(`Cuadrante lleno: solo se pudieron colocar ${placed}/${count} bloques`);
    }
  };

  const total = props.initialBlocks.length;
  props.initialBlocks.forEach((count, index) => {
    const q = getQuadrant(index, total);
    placeGroupInQuadrant(count, q);
  });

  checkAdjacentBlocks();
  saveToHistory();
  saveToStorage();
};

onMounted(() => {
  loadFromStorage();
  saveToHistory(); // Guardar estado inicial
  
  // NUEVO: Crear bloques iniciales si se proporciona initialBlocks
  if (props.initialBlocks && props.initialBlocks.length > 0) {
    createInitialBlocks();
  }
  
  // Event listeners para drag & drop de bloques
  document.addEventListener('mousemove', handleMouseMove);
  document.addEventListener('mouseup', handleMouseUp);
  
  // Event listeners para cortar conexiones (modo ✂️)
  document.addEventListener('mousemove', handleSnapCutMove);
  document.addEventListener('mouseup', handleSnapCutEnd);
  
  // Resize adaptativo
  window.addEventListener('resize', onResize);
  
  // Event listeners para cortar bloques ELIMINADO
  
  // Keyboard shortcuts
  document.addEventListener('keydown', handleKeyDown);
});

onUnmounted(() => {
  document.removeEventListener('mousemove', handleMouseMove);
  document.removeEventListener('mouseup', handleMouseUp);
  document.removeEventListener('mousemove', handleSnapCutMove);
  document.removeEventListener('mouseup', handleSnapCutEnd);
  document.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('resize', onResize);
});
</script>

<template>
  <div>
    <div v-if="!showOverlay" class="floating-buttons">
      <button @click="toggleOverlay" class="fab">🧱 Bloques</button>
    </div>

    <div v-if="showOverlay" class="grid-fullscreen" @click="handleGridCanvasClick">
      <!-- Barra flotante de herramientas -->
      <div class="floating-toolbar" @click.stop>
        <button
          @click="selectTool('uno')"
          class="tool-btn tool-btn--add"
          :style="{ backgroundColor: BLOCKS_CONFIG['uno'].color }"
          title="Agregar bloque"
        >+1</button>

        <div class="tool-divider"></div>

        <button
          @click="spawnRandomBlocks"
          class="tool-btn"
          title="Mezclar bloques"
        >🔀</button>

        <div class="tool-divider"></div>

        <button
          @click="toggleCuttingMode"
          :class="['tool-btn', { 'tool-btn--active': cuttingMode }]"
          title="Desacoplar conexiones"
        >✂️</button>

        <div class="tool-divider"></div>

        <button
          @click="undo"
          :disabled="currentStep <= 0"
          class="tool-btn"
          title="Deshacer"
        >↶</button>

        <button
          @click="redo"
          :disabled="currentStep >= history.length - 1"
          class="tool-btn"
          title="Rehacer"
        >↷</button>

        <div class="tool-divider"></div>

        <button
          @click="clearAll"
          class="tool-btn tool-btn--danger"
          title="Borrar todo"
        >🗑️</button>
      </div>

      <div class="grid-container">
        <!-- Ruler ELIMINADO -->
        
        <div class="grid-center" @click.stop @mousedown.capture="handleCutModeMouseDown">
          <div 
            class="grid-transparent" 
            :class="{ 'cutting-cursor': cuttingMode }"
            :style="{ 
              width: (GRID_WIDTH * CELL_SIZE) + 'px', 
              height: (GRID_HEIGHT * CELL_SIZE) + 'px' 
            }"
            @click="handleGridCanvasClick"
          >
            <div v-for="y in GRID_HEIGHT" :key="`row-${y}`" class="grid-row-transparent">
              <div
                v-for="x in GRID_WIDTH"
                :key="`cell-${x}-${y}`"
                class="grid-cell-transparent"
                :style="{ width: CELL_SIZE + 'px', height: CELL_SIZE + 'px' }"
              ></div>
            </div>
          </div>

          <div class="blocks-layer-transparent">
            <div
              v-for="shape in shapes"
              :key="shape.id"
              @mousedown="handleBlockMouseDown($event, shape.id)"
              @mouseenter="!cuttingMode && (hoveredShapeId = shape.id)"
              @mouseleave="hoveredShapeId = null"
              :class="{
                'block-transparent': true,
                'block-dragging': draggingId === shape.id,
                'block-selected': selectedShapeId === shape.id,
                'block-connected': shape.isConnected,
                'no-hover': cuttingMode,
                'visually-separated': shape.visuallySeparated
              }"
              :style="{
                left: (shape.gridX * CELL_SIZE) + 'px',
                top: (shape.gridY * CELL_SIZE) + 'px',
                width: (getShapeDimensions(shape).width * CELL_SIZE) + 'px',
                height: (getShapeDimensions(shape).height * CELL_SIZE) + 'px',
                position: 'absolute',
                backgroundColor: !shape.units ? (shape.isConnected ? (shape.groupColor || '#64748b') : (shape.color || '#64748b')) : 'transparent'
              }"
            >
              <!-- Tooltip ELIMINADO - Innecesario -->
              
              <!-- NUEVO: Renderizar cubitos individuales SI tiene units -->
              <template v-if="shape.units && shape.units.length > 0">
                <div
                  v-for="(unit, unitIndex) in shape.units"
                  :key="unitIndex"
                  class="unit-cube"
                  :style="{
                    position: 'absolute',
                    left: (unit.relX * CELL_SIZE) + 'px',
                    top: (unit.relY * CELL_SIZE) + 'px',
                    width: CELL_SIZE + 'px',
                    height: CELL_SIZE + 'px',
                    backgroundColor: unit.color,
                    border: '2px solid rgba(0,0,0,0.3)',
                    boxSizing: 'border-box'
                  }"
                ></div>
              </template>
              
              <!-- Bordes del bloque completo -->
              <div
                :style="{
                  position: 'absolute',
                  inset: 0,
                  backgroundImage: getBorderStyle(shape),
                  pointerEvents: 'none'
                }"
              ></div>
              
              <!-- Botón eliminar ELIMINADO -->
            </div>

            <svg
              v-if="cuttingMode"
              class="connections-layer"
              :style="{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: '100%',
                pointerEvents: 'none',
                zIndex: 15
              }"
            >
              <defs>
                <filter id="glow">
                  <feGaussianBlur stdDeviation="3" result="coloredBlur"/>
                  <feMerge>
                    <feMergeNode in="coloredBlur"/>
                    <feMergeNode in="SourceGraphic"/>
                  </feMerge>
                </filter>
              </defs>
              
              <g v-for="(conn, idx) in connections" :key="`conn-${idx}`">
                <line
                  :x1="conn.x1"
                  :y1="conn.y1"
                  :x2="conn.x2"
                  :y2="conn.y2"
                  stroke="rgba(236, 72, 153, 0.3)"
                  stroke-width="8"
                  stroke-linecap="round"
                  style="pointer-events: stroke; cursor: crosshair;"
                  @mousedown="handleCutDragStart($event, conn)"
                />
                
                <line
                  :x1="conn.x1"
                  :y1="conn.y1"
                  :x2="conn.x2"
                  :y2="conn.y2"
                  :stroke="connectionsToCut.includes(conn) ? '#be185d' : '#ec4899'"
                  :stroke-width="connectionsToCut.includes(conn) ? '6' : '4'"
                  stroke-dasharray="8,4"
                  stroke-linecap="round"
                  :class="['cut-line', { 'cut-line-active': connectionsToCut.includes(conn) }]"
                  :filter="connectionsToCut.includes(conn) ? 'url(#glow)' : 'none'"
                  style="pointer-events: stroke; cursor: crosshair;"
                  @mousedown="handleCutDragStart($event, conn)"
                />
                
                <circle
                  :cx="conn.x1"
                  :cy="conn.y1"
                  :r="connectionsToCut.includes(conn) ? '6' : '4'"
                  :fill="connectionsToCut.includes(conn) ? '#be185d' : '#ec4899'"
                  :class="['cut-circle', { 'cut-circle-active': connectionsToCut.includes(conn) }]"
                  :filter="connectionsToCut.includes(conn) ? 'url(#glow)' : 'none'"
                  style="pointer-events: all; cursor: pointer;"
                  @mousedown="handleCutDragStart($event, conn)"
                />
                
                <circle
                  :cx="conn.x2"
                  :cy="conn.y2"
                  :r="connectionsToCut.includes(conn) ? '6' : '4'"
                  :fill="connectionsToCut.includes(conn) ? '#be185d' : '#ec4899'"
                  :class="['cut-circle', { 'cut-circle-active': connectionsToCut.includes(conn) }]"
                  :filter="connectionsToCut.includes(conn) ? 'url(#glow)' : 'none'"
                  style="pointer-events: all; cursor: pointer;"
                  @mousedown="handleCutDragStart($event, conn)"
                />
              </g>
            </svg>

            <!-- Línea visual de vértice a vértice -->
            <svg
              v-if="isCutDragging && cutDragStart && cutDragEnd"
              class="cut-drag-line"
              :style="{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: '100%',
                pointerEvents: 'none',
                zIndex: 30
              }"
            >
              <defs>
                <filter id="dragGlow">
                  <feGaussianBlur stdDeviation="4" result="coloredBlur"/>
                  <feMerge>
                    <feMergeNode in="coloredBlur"/>
                    <feMergeNode in="SourceGraphic"/>
                  </feMerge>
                </filter>
              </defs>
              
              <!-- Línea desde vértice inicial hasta cursor -->
              <line
                :x1="cutDragStart.x"
                :y1="cutDragStart.y"
                :x2="cutDragEnd.x"
                :y2="cutDragEnd.y"
                stroke="rgba(190, 24, 93, 0.3)"
                stroke-width="8"
                stroke-linecap="round"
              />
              
              <line
                :x1="cutDragStart.x"
                :y1="cutDragStart.y"
                :x2="cutDragEnd.x"
                :y2="cutDragEnd.y"
                :stroke="connectionsToCut.length > 0 ? '#10b981' : '#be185d'"
                stroke-width="4"
                stroke-dasharray="10,5"
                stroke-linecap="round"
                :filter="connectionsToCut.length > 0 ? 'url(#dragGlow)' : 'none'"
                class="drag-line-animated"
              />
            </svg>

            <!-- SVG para snap-to-line (cortar conexiones con seguimiento) -->
            <svg
              v-if="snapToLineState.active && snapToLineState.currentConnection"
              class="snap-cut-visualization"
              :style="{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: '100%',
                pointerEvents: 'none',
                zIndex: 45
              }"
            >
              <!-- Conexión completa (gris = no cortada, verde = cortada) -->
              <line
                v-for="conn in connections"
                :key="`conn-${conn.shape1Id}-${conn.shape2Id}`"
                :x1="conn.x1"
                :y1="conn.y1"
                :x2="conn.x2"
                :y2="conn.y2"
                :stroke="isConnectionCut(conn) ? '#10b981' : '#94a3b8'"
                :stroke-width="isConnectionCut(conn) ? 6 : 3"
                stroke-linecap="round"
                :opacity="isConnectionCut(conn) ? 0.9 : 0.3"
              />
              
              <!-- Conexión activa (resaltada - cambia a verde si llegó al vértice) -->
              <line
                v-if="snapToLineState.currentConnection"
                :x1="snapToLineState.currentConnection.x1"
                :y1="snapToLineState.currentConnection.y1"
                :x2="snapToLineState.currentConnection.x2"
                :y2="snapToLineState.currentConnection.y2"
                :stroke="snapToLineState.reachedOppositeVertex ? '#10b981' : '#ec4899'"
                stroke-width="8"
                stroke-linecap="round"
                :opacity="snapToLineState.reachedOppositeVertex ? 0.9 : 0.6"
              />
              
              
              <!-- Cursor snapeado (círculo - verde si llegó al vértice) -->
              <circle
                v-if="snapToLineState.snappedPosition"
                :cx="snapToLineState.snappedPosition.x"
                :cy="snapToLineState.snappedPosition.y"
                r="10"
                :fill="snapToLineState.reachedOppositeVertex ? '#10b981' : '#fbbf24'"
                stroke="#ffffff"
                stroke-width="3"
                opacity="1"
              >
                <animate
                  v-if="snapToLineState.reachedOppositeVertex"
                  attributeName="r"
                  values="10;14;10"
                  dur="0.6s"
                  repeatCount="indefinite"
                />
              </circle>
              
            </svg>

            <!-- SVG de corte de bloques ELIMINADO - Solo usamos bloques de 1x1 -->
          </div>
        </div>
      </div>
    </div>
    
    <!-- Toast Notification -->
    <Transition name="toast">
      <div v-if="showToast" class="toast-notification">
        {{ toastMessage }}
      </div>
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
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: transparent;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  animation: fadeIn 0.3s ease;
  overflow: hidden;
  padding: 20px 0;
  overscroll-behavior: none;
  /* Evitar selección de texto al arrastrar */
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.grid-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  overflow: visible;
  gap: 4px;
}

/* ===== REGLA NUMÉRICA ELIMINADA ===== */

.grid-center {
  position: relative;
  display: inline-block;
}

.grid-transparent {
  position: relative;
  z-index: 1;
  background: transparent;
  border-radius: 8px;
  padding: 0;
  box-shadow: 0 0 0 1px rgba(148, 163, 184, 0.15);
  /* width y height dinámicos via :style en template */
  overflow: hidden;
}

.cutting-cursor {
  cursor: crosshair !important;
}

.grid-row-transparent {
  display: flex;
}

.grid-cell-transparent {
  border: 1px solid rgba(148, 163, 184, 0.25);
  background-color: transparent;
  position: relative;
}

.grid-cell-transparent::before {
  content: '';
  position: absolute;
  inset: 0;
  background: radial-gradient(circle at center, rgba(59, 130, 246, 0.04) 0%, transparent 70%);
  pointer-events: none;
}

.blocks-layer-transparent {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 10;
  pointer-events: none;
  width: 900px;
  height: 675px;
}

.block-transparent {
  position: absolute;
  border-radius: 0;
  display: flex;
  cursor: grab;
  transition: transform 0.15s cubic-bezier(0.4, 0, 0.2, 1), 
              opacity 0.15s ease, 
              background-color 0.2s ease,
              box-shadow 0.2s ease;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15); 
  pointer-events: auto;
  opacity: 0.95;
  will-change: transform;
  /* Evitar selección de texto */
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}

/* Tooltip con suma de unidades al hacer hover */
.block-tooltip {
  position: absolute;
  top: -32px;
  left: 50%;
  transform: translateX(-50%);
  background: linear-gradient(135deg, #1e293b 0%, #334155 100%);
  color: white;
  padding: 4px 12px;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 700;
  white-space: nowrap;
  z-index: 1000;
  pointer-events: none;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  border: 2px solid rgba(255, 255, 255, 0.2);
  animation: tooltipFadeIn 0.2s ease;
}

.block-tooltip::after {
  content: '';
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 6px solid transparent;
  border-right: 6px solid transparent;
  border-top: 6px solid #334155;
}

@keyframes tooltipFadeIn {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(-4px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

/* NUEVO: Estilos para cubitos unitarios */
.unit-cube {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.7rem;
  color: white;
  font-weight: 600;
  text-shadow: 0 1px 2px rgba(0,0,0,0.3);
  transition: background-color 0.2s ease;
}

/* Borde interno sutil en cada cubito */
.unit-cube::after {
  content: '';
  position: absolute;
  inset: 3px;
  border: 1px solid rgba(255,255,255,0.15);
  pointer-events: none;
  border-radius: 1px;
}

.block-transparent.block-connected {
  /* Sin brillo verde - borde sutil neutral */
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

/* Bloques que no se pudieron separar físicamente - separación visual */
.block-transparent.visually-separated {
  outline: 3px dashed rgba(239, 68, 68, 0.6);
  outline-offset: 2px;
  animation: visualSeparationPulse 2s ease-in-out infinite;
}

@keyframes visualSeparationPulse {
  0%, 100% {
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
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15), 
              0 0 0 3px rgba(59, 130, 246, 0.6),
              0 0 12px rgba(59, 130, 246, 0.4);
  z-index: 50;
}

/* Desactivar hover en modo corte */
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
  transition: opacity 0.1s ease, transform 0.05s ease-out;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.3), 0 0 0 2px rgba(59, 130, 246, 0.4);
}

.block-transparent.block-selected {
  z-index: 50;
  box-shadow: 0 0 0 4px rgba(59, 130, 246, 0.6);
  border-radius: 0; 
}

.btn-rotate {
  position: absolute;
  top: -12px;
  right: -12px;
  font-size: 18px;
  background: white;
  border: 2px solid #3b82f6;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  transition: all 0.2s;
  z-index: 20;
  pointer-events: auto;
}

.btn-rotate:hover {
  transform: rotate(90deg) scale(1.1);
  background: #3b82f6;
  color: white;
}

.btn-rotate:active {
  transform: rotate(180deg) scale(1.05);
}

.btn-delete-transparent {
  position: absolute;
  bottom: -12px;
  right: -12px;
  width: 28px;
  height: 28px;
  background: #ef4444;
  color: white;
  border: 2px solid white;
  border-radius: 50%;
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
  transition: all 0.2s;
  z-index: 20;
  pointer-events: auto;
}

.btn-delete-transparent:hover {
  background: #dc2626;
  transform: scale(1.15);
}

.cut-line {
  transition: all 0.2s ease;
  animation: dash 1.5s linear infinite;
}

.cut-line:hover {
  stroke-width: 5 !important;
  stroke: #f472b6 !important;
}

.cut-circle {
  transition: all 0.2s ease;
}

.cut-circle:hover {
  fill: #f472b6;
  opacity: 1;
}

.cut-circle-active {
  animation: pulse-circle 0.6s ease-in-out infinite;
}

.drag-line-animated {
  animation: dash 1s linear infinite;
}

@keyframes dash {
  to {
    stroke-dashoffset: -24;
  }
}

@keyframes pulse-circle {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.6;
  }
}

.cut-line-active {
  stroke-width: 7 !important;
  animation: dash 0.8s linear infinite, pulse-line 0.6s ease-in-out infinite !important;
}

@keyframes pulse-line {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.7;
  }
}

/* ===== BARRA FLOTANTE DE HERRAMIENTAS ===== */
.floating-toolbar {
  position: fixed;
  bottom: 1.5rem;
  right: 1.5rem;
  z-index: 2000;
  display: flex;
  align-items: center;
  gap: 0.25rem;
  background: white;
  border-radius: 16px;
  padding: 0.45rem 0.55rem;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.13), 0 1px 4px rgba(0,0,0,0.07);
  border: 1px solid rgba(0,0,0,0.07);
}

.tool-btn {
  width: 40px;
  height: 40px;
  background: transparent;
  border: none;
  border-radius: 10px;
  font-size: 1.1rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s, transform 0.15s;
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

@media (max-width: 768px) {
  .floating-toolbar {
    bottom: 1rem;
    right: 50%;
    transform: translateX(50%);
    gap: 0.15rem;
    padding: 0.4rem 0.5rem;
  }
}

/* ===== TOAST NOTIFICATION ===== */
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
</style>