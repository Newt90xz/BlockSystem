<template>
  <div class="app">

    <!-- Start screen -->
    <div v-if="screen === null" class="center">
      <h1>🧱 Blocks</h1>
      <button @click="screen = 'exercises'"> Demo de Ejercicios</button>
    </div>

    <!-- Exercises screen -->
    <div v-else-if="screen === 'exercises'" class="page">
      <div class="bar">
        <button @click="screen = null"> Volver </button>
        <button @click="showLabels = !showLabels">
          {{ showLabels ? 'Ocultar numeros' : 'Mostrar numeros' }}
        </button>
        <button @click="resetAll">Reiniciar</button>
      </div>

      <div class="exercise-scroll">
        <div class="exercise-list">

          <div v-for="(ex, idx) in exercises" :key="idx" class="ex-card" :class="{
            'ex-card--ok': results[idx] === 'ok',
            'ex-card--bad': results[idx] === 'bad',
          }">
            <!-- Header -->
            <div class="ex-header">
              <span class="ex-index">{{ idx + 1 }}</span>
              <p class="ex-question">{{ ex.text }}</p>
              <span v-if="results[idx] === 'ok'" class="badge badge--ok">✓ Correct</span>
              <span v-else-if="results[idx] === 'bad'" class="badge badge--bad">✗ Try again</span>
            </div>

            <!-- Grids + Dropzone -->
            <div class="ex-grids">
              <div class="ex-grids-row">
                <Blocks :ref="el => blocksRefs[idx] = el" :key="keys[idx]" :config="buildConfig(ex, idx)" />
                <div :ref="el => dropzoneRefs[idx] = el" class="dropzone" :class="{
                  'dropzone--hover': dropzoneHover[idx],
                  'dropzone--ok': results[idx] === 'ok',
                  'dropzone--bad': results[idx] === 'bad',
                }">
                  <span class="dropzone-icon">
                    {{ results[idx] === 'ok' ? '📬' : '📭' }}
                  </span>
                </div>
              </div>
            </div>

            <!-- Footer -->
            <div class="ex-footer">
              <button class="btn-reset" @click="resetExercise(idx)">↺ Reset</button>
            </div>
          </div>

        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import Blocks from './components/Blocks.vue';

// ── SCREEN ──
const screen = ref(null);

// ── EXERCISES ──
const exercises = [
  { text: "María tiene 2 manzanas y Juan tiene 3. ¿Cuántas tienen en total?", nums: [2, 3], answer: 5 },
  { text: "En el patio hay 4 perros y llegan 2 más. ¿Cuántos perros hay ahora?", nums: [4, 2], answer: 6 },
  { text: "Ana juntó 3 piedras en el parque y 4 en la playa. ¿Cuántas piedras tiene?", nums: [3, 4], answer: 7 },
];

const keys = ref(exercises.map((_, i) => i * 100));
const results = ref(exercises.map(() => null));
const showLabels = ref(true);
const blocksRefs = ref(exercises.map(() => null));
const dropzoneRefs = ref(exercises.map(() => null));
const dropzoneHover = ref(exercises.map(() => false));

// Grid shape: odd → n×1, even → 2 cols × n/2 rows.
const gridShape = (n) => n % 2 === 0
  ? { cols: 2, rows: n / 2 }
  : { cols: n, rows: 1 };

// Build the Blocks config for a given exercise.
// Operand grids: noSnap true. Banca: noSnap false (blocks can connect).
const buildConfig = (ex, idx) => ({
  inline: true,
  inlineColumns: 2,
  showToolbar: false,
  showGridLabels: showLabels.value,
  maxBloques: ex.answer * 2,
  storageKey: `ex_${idx}_${keys.value[idx]}`,
  grids: [
    ...ex.nums.map(n => ({
      label: `${n}`,
      ...gridShape(n),
      initialBlocks: [n],
      isAnswer: false,
      showLabel: false,
      showCount: false,
      noSnap: true,
    })),
    {
      label: 'Banca',
      cols: Math.max(ex.answer, Math.max(...ex.nums) * 2),
      rows: 2,
      initialBlocks: [],
      isAnswer: true,
      showLabel: true,
      showCount: false,
      noSnap: false,
    },
  ],
});

const isOverEl = (el, x, y) => {
  if (!el) return false;
  const r = el.getBoundingClientRect();
  return x >= r.left && x <= r.right && y >= r.top && y <= r.bottom;
};

// Fired on every document mouseup — checks if mouse released over a dropzone.
const onDocumentMouseUp = (e) => {
  exercises.forEach((ex, idx) => {
    dropzoneHover.value[idx] = false;
    if (!isOverEl(dropzoneRefs.value[idx], e.clientX, e.clientY)) return;
    if (results.value[idx] === 'ok') return;

    const instance = blocksRefs.value[idx];
    if (!instance) return;

    const bancaGridId = ex.nums.length;
    const groups = instance.getConnectedGroups();
    const bancaGroups = groups.filter(g => g.gridId === bancaGridId);
    const correct = bancaGroups.length === 1 && bancaGroups[0].count === ex.answer;

    if (correct) {
      results.value[idx] = 'ok';
    } else {
      results.value[idx] = 'bad';
      setTimeout(() => {
        results.value[idx] = null;
        keys.value[idx]++;
      }, 1000);
    }
  });
};

// Update dropzone hover highlight during drag.
const onDocumentMouseMove = (e) => {
  exercises.forEach((_, idx) => {
    dropzoneHover.value[idx] = isOverEl(dropzoneRefs.value[idx], e.clientX, e.clientY);
  });
};

onMounted(() => {
  document.addEventListener('mouseup', onDocumentMouseUp);
  document.addEventListener('mousemove', onDocumentMouseMove);
});

onUnmounted(() => {
  document.removeEventListener('mouseup', onDocumentMouseUp);
  document.removeEventListener('mousemove', onDocumentMouseMove);
});

const resetExercise = (idx) => {
  results.value[idx] = null;
  keys.value[idx]++;
};

const resetAll = () => {
  results.value = exercises.map(() => null);
  keys.value = keys.value.map(k => k + 1000);
};
</script>

<style scoped>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.app {
  width: 100%;
  height: 100vh;
  font-family: system-ui, sans-serif;
  font-size: 14px;
  background: #f0f4f8;
  display: flex;
  flex-direction: column;
}

/* ── Start screen ── */
.center {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
  gap: 1rem;
}

.center h1 {
  font-size: 1.5rem;
}

/* ── Exercises screen ── */
.page {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.bar {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 0.75rem;
  background: white;
  border-bottom: 1px solid #ddd;
  font-weight: 600;
  color: #333;
  flex-shrink: 0;
  z-index: 500;
}

.exercise-scroll {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem 1rem 3rem;
}

.exercise-list {
  max-width: 860px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

/* ── Exercise card ── */
.ex-card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 1px 6px rgba(0, 0, 0, 0.08);
  overflow: hidden;
  border: 2px solid transparent;
  transition: border-color 0.25s;
}

.ex-card--ok {
  border-color: #10b981;
}

.ex-card--bad {
  border-color: #ef4444;
}

.ex-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.9rem 1.25rem 0;
}

.ex-index {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #6366f1;
  color: white;
  font-size: 0.8rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.ex-question {
  font-size: 1rem;
  font-weight: 600;
  color: #1e1b4b;
  flex: 1;
  line-height: 1.4;
}

.badge {
  font-size: 0.75rem;
  font-weight: 700;
  padding: 0.2rem 0.6rem;
  border-radius: 20px;
  white-space: nowrap;
}

.badge--ok {
  background: #d1fae5;
  color: #065f46;
}

.badge--bad {
  background: #fee2e2;
  color: #991b1b;
}

.ex-grids {
  padding: 8px 0;
  overflow: visible;
}

.ex-grids-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 0 12px;
}

/* Dropzone */
.dropzone {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 48px;
  height: 48px;
  border: 2px solid #94a3b8;
  border-radius: 10px;
  flex-shrink: 0;
  align-self: center;
  transition: transform 0.15s ease, border-color 0.15s, background 0.15s;
}

.dropzone--hover {
  border-color: #6366f1;
  background: rgba(99, 102, 241, 0.06);
  transform: scale(1.12);
}

.dropzone--ok {
  border-color: #10b981;
  background: rgba(16, 185, 129, 0.06);
}

.dropzone--bad {
  border-color: #ef4444;
  background: rgba(239, 68, 68, 0.06);
}

.dropzone-icon {
  font-size: 1.6rem;
  pointer-events: none;
  line-height: 1;
  user-select: none;
}

.ex-footer {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.25rem;
  border-top: 1px solid #f3f4f6;
}

/* ── Buttons ── */
button {
  padding: 0.4rem 0.9rem;
  background: white;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  font-family: inherit;
  color: #333;
}

button:hover {
  background: #f0f0f0;
}

button:disabled {
  opacity: 0.6;
  cursor: default;
}


.btn-reset {
  padding: 0.4rem 0.7rem;
  border-color: #e5e7eb;
  color: #6b7280;
}

.btn-reset:hover {
  background: #f9fafb;
}
</style>