<template>
  <div class="app">

    <!-- Start screen -->
    <div v-if="screen === null" class="center">
      <h1>🧱 Blocks</h1>
      <button @click="screen = 'exercises'">Demo de Ejercicios</button>
      <button @click="screen = 'creator'">Crear Ejercicio</button>
    </div>

    <!-- Demo Exercises screen -->
    <div v-else-if="screen === 'exercises'" class="page">
      <div class="bar">
        <button @click="screen = null">Volver</button>
        <button @click="showLabels = !showLabels">{{ showLabels ? 'Ocultar números' : 'Mostrar números' }}</button>
        <button @click="resetAll">Reiniciar</button>
      </div>
      <div class="exercise-scroll">
        <div class="exercise-list">
          <div v-for="(ex, idx) in demoExercises" :key="'demo-' + idx" class="ex-card" :class="{
            'ex-card--ok': demoResults[idx] === 'ok',
            'ex-card--bad': demoResults[idx] === 'bad',
          }">
            <div class="ex-header">
              <span class="ex-index">{{ idx + 1 }}</span>
              <p class="ex-question">{{ ex.text }}</p>
              <span v-if="demoResults[idx] === 'ok'" class="badge badge--ok">✓ Correcto</span>
              <span v-else-if="demoResults[idx] === 'bad'" class="badge badge--bad">✗ Try again</span>
            </div>
            <div class="ex-grids">
              <div class="ex-grids-row">
                <Blocks :ref="el => demoBlocksRefs[idx] = el" :key="demoKeys[idx]" :config="buildConfig(ex, idx)" />
                <div :ref="el => demoDropzoneRefs[idx] = el" class="dropzone" :class="{
                  'dropzone--hover': demoDropzoneHover[idx],
                  'dropzone--ok': demoResults[idx] === 'ok',
                  'dropzone--bad': demoResults[idx] === 'bad',
                }">
                  <span class="dropzone-icon">{{ demoResults[idx] === 'ok' ? '📬' : '📭' }}</span>
                </div>
              </div>
            </div>
            <div class="ex-footer">
              <button class="btn-reset" @click="resetDemoExercise(idx)">↺ Reset</button>
              <button class="btn-reset btn-delete" @click="deleteDemoExercise(idx)">✕</button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- User Exercises screen -->
    <div v-else-if="screen === 'user-exercises'" class="page">
      <div class="bar">
        <button @click="screen = null">Volver</button>
        <button @click="showLabels = !showLabels">{{ showLabels ? 'Ocultar números' : 'Mostrar números' }}</button>
        <button @click="resetAllUser">Reiniciar</button>
        <button @click="screen = 'creator'" style="margin-left:auto">+ Crear</button>
      </div>
      <div class="exercise-scroll">
        <div class="exercise-list">
          <div v-if="!userExercises.length" class="empty-state">No hay ejercicios creados todavía.</div>
          <div v-for="(ex, idx) in userExercises" :key="'user-' + idx" class="ex-card" :class="{
            'ex-card--ok': userResults[idx] === 'ok',
            'ex-card--bad': userResults[idx] === 'bad',
          }">
            <div class="ex-header">
              <span class="ex-index">{{ idx + 1 }}</span>
              <p class="ex-question">{{ ex.text || '\u00A0' }}</p>
              <span v-if="userResults[idx] === 'ok'" class="badge badge--ok">✓ Correcto</span>
              <span v-else-if="userResults[idx] === 'bad'" class="badge badge--bad">✗ Try again</span>
            </div>
            <div class="ex-grids">
              <div class="ex-grids-row">
                <Blocks :ref="el => userBlocksRefs[idx] = el" :key="userKeys[idx]" :config="buildConfig(ex, idx)" />
                <div :ref="el => userDropzoneRefs[idx] = el" class="dropzone" :class="{
                  'dropzone--hover': userDropzoneHover[idx],
                  'dropzone--ok': userResults[idx] === 'ok',
                  'dropzone--bad': userResults[idx] === 'bad',
                }">
                  <span class="dropzone-icon">{{ userResults[idx] === 'ok' ? '📬' : '📭' }}</span>
                </div>
              </div>
            </div>
            <div class="ex-footer">
              <button class="btn-reset" @click="resetUserExercise(idx)">↺ Reset</button>
              <button class="btn-reset btn-delete" @click="deleteUserExercise(idx)">✕</button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Creator screen -->
    <div v-else-if="screen === 'creator'" class="page">
      <div class="bar">
        <button @click="screen = 'user-exercises'">Volver</button>
        <span style="font-weight:600; color:var(--text)">Crear ejercicios</span>
        <button v-if="draft.length" class="btn-load" @click="loadDraft" style="margin-left:auto">
          Cargar {{ draft.length }} ejercicio{{ draft.length > 1 ? 's' : '' }}
        </button>
      </div>

      <div class="creator-scroll">
        <div class="creator-body">

          <!-- Draft list -->
          <div v-if="draft.length" class="draft-list">
            <div v-for="(ex, i) in draft" :key="i" class="draft-item">
              <span class="draft-nums">{{ ex.nums.join(' + ') }} = {{ ex.answer }}</span>
              <span class="draft-text">{{ ex.text || '—' }}</span>
              <button class="btn-icon btn-delete" @click="draft.splice(i, 1)">✕</button>
            </div>
          </div>

          <!-- Form -->
          <div class="creator-form">

            <div class="field">
              <label>Enunciado <span class="hint">(opcional)</span></label>
              <input v-model="form.text" type="text" placeholder="Ej: María tiene 2 manzanas..." />
            </div>

            <div class="field">
              <label>Grids <span class="hint">— un número por casilla</span></label>
              <div class="grids-col">
                <input
                  v-for="(n, i) in form.numsDisplay"
                  :key="i"
                  class="grid-num-input"
                  type="number" min="1" max="20"
                  placeholder=""
                  :value="n > 0 ? n : ''"
                  @input="onGridInput(i, $event)"
                  @blur="onGridBlur(i, $event)"
                />
              </div>
            </div>

            <div class="field" v-if="validNums.length">
              <label>Banca <span class="hint">— calculada según los grids</span></label>
              <span class="banca-dims">{{ bancaShape.cols }} col × {{ bancaShape.rows }} fil</span>
            </div>

            <button class="btn-add-draft" @click="addToDraft" :disabled="validNums.length === 0">+ Agregar a la lista</button>
          </div>

        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import Blocks from './components/Blocks.vue';

// ── SCREEN ──
const screen = ref(null);

// ── HELPERS ──

// Shape for operand grids: even → 2×(n/2), odd → n×1
const gridShape = (n) => n % 2 === 0
  ? { cols: 2, rows: n / 2 }
  : { cols: n, rows: 1 };

// Banca must fit (answer + 1) blocks horizontally across 3 rows.
// cols = ceil((answer + 1) / 3), rows = 3 — ensures the chain always fits.
const bancaGridShape = (answer) => {
  const rows = 3;
  const cols = Math.ceil((answer + 1) / rows);
  return { cols: Math.max(cols, answer + 1), rows };
};

// ── EXERCISE FACTORY ──
// Takes a plain config object and returns a normalised exercise ready for buildConfig.
// { text, nums, answer, bancaCols?, bancaRows? }
const createExercise = ({ text = '', nums, answer, bancaCols, bancaRows }) => {
  const shape = bancaGridShape(answer);
  return {
    text,
    nums,
    answer,
    bancaCols: bancaCols ?? shape.cols,
    bancaRows: bancaRows ?? shape.rows,
  };
};

// ── DEMO EXERCISES — shown only in the demo screen ──
const demoExercises = ref([
  createExercise({ text: "María tiene 2 manzanas y Juan tiene 3. ¿Cuántas tienen en total?", nums: [2, 3], answer: 5 }),
  createExercise({ text: "En el patio hay 4 perros y llegan 2 más. ¿Cuántos perros hay ahora?", nums: [4, 2], answer: 6 }),
  createExercise({ text: "Ana juntó 3 piedras en el parque y 4 en la playa. ¿Cuántas piedras tiene?", nums: [3, 4], answer: 7 }),
]);

// ── USER EXERCISES — shown only in the user exercise screen ──
const userExercises = ref([]);

// Demo and user have completely separate reactive state so they never interfere.
const demoKeys          = ref(demoExercises.value.map((_, i) => i * 100));
const demoResults       = ref(demoExercises.value.map(() => null));
const demoBlocksRefs    = ref(demoExercises.value.map(() => null));
const demoDropzoneRefs  = ref(demoExercises.value.map(() => null));
const demoDropzoneHover = ref(demoExercises.value.map(() => false));

const userKeys          = ref([]);
const userResults       = ref([]);
const userBlocksRefs    = ref([]);
const userDropzoneRefs  = ref([]);
const userDropzoneHover = ref([]);

// Keep showLabels shared
const showLabels = ref(true);

// Legacy aliases so buildConfig still works with a flat idx
// (demo uses demoKeys, user uses userKeys — handled per-template)
const keys          = demoKeys;   // not used for user cards
const results       = demoResults;
const blocksRefs    = demoBlocksRefs;
const dropzoneRefs  = demoDropzoneRefs;
const dropzoneHover = demoDropzoneHover;

const growUserArrays = (n) => {
  for (let i = 0; i < n; i++) {
    userKeys.value.push(userKeys.value.length * 100 + Date.now() % 1000 + i);
    userResults.value.push(null);
    userBlocksRefs.value.push(null);
    userDropzoneRefs.value.push(null);
    userDropzoneHover.value.push(false);
  }
};

// ── BUILD CONFIG ──
const buildConfig = (ex, idx) => ({
  inline: true,
  inlineColumns: 2,
  showToolbar: false,
  showGridLabels: showLabels.value,
  maxBloques: (ex.answer + 1) * 2,
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
      cols: ex.bancaCols,
      rows: ex.bancaRows,
      initialBlocks: [],
      isAnswer: true,
      showLabel: true,
      showCount: false,
      noSnap: false,
    },
  ],
});

// ── CREATOR ──
// numsDisplay: array of raw input strings, always has one trailing empty slot.
const defaultForm = () => ({
  text: '',
  numsDisplay: [''],  // one slot; more appear as numbers are entered
});
const form  = ref(defaultForm());
const draft = ref([]);

// Valid nums: non-empty, non-zero entries parsed as integers
const validNums = computed(() =>
  form.value.numsDisplay
    .slice(0, -1)  // exclude trailing empty slot
    .map(v => parseInt(v))
    .filter(v => !isNaN(v) && v > 0)
);

const computedAnswer = computed(() =>
  validNums.value.reduce((a, b) => a + b, 0)
);

const bancaShape = computed(() => bancaGridShape(Math.max(1, computedAnswer.value)));

// Called on every keystroke in a grid input
const onGridInput = (i, e) => {
  const val = e.target.value;
  form.value.numsDisplay[i] = val;

  const isLast = i === form.value.numsDisplay.length - 1;
  const hasValue = val !== '' && parseInt(val) > 0;

  // If this is the last slot and got a value, append a new empty slot
  if (isLast && hasValue && form.value.numsDisplay.length < 7) {
    form.value.numsDisplay.push('');
  }
};

// On blur: if value is blank or 0, clear it; collapse trailing empty slots to one
const onGridBlur = (i, e) => {
  const val = e.target.value;
  const parsed = parseInt(val);
  if (!val || isNaN(parsed) || parsed <= 0) {
    form.value.numsDisplay[i] = '';
  }
  // Remove all trailing empty slots, then add exactly one
  const display = form.value.numsDisplay;
  let last = display.length - 1;
  while (last > 0 && (display[last] === '' || display[last] === '0')) last--;
  form.value.numsDisplay = [...display.slice(0, last + 1), ''];
};


const addToDraft = () => {
  if (validNums.value.length === 0) return;
  draft.value.push(createExercise({
    text:   form.value.text,
    nums:   [...validNums.value],
    answer: Math.max(1, computedAnswer.value),
  }));
  form.value = defaultForm();
};

const loadDraft = () => {
  growUserArrays(draft.value.length);
  userExercises.value.push(...draft.value);
  draft.value = [];
  screen.value = 'user-exercises';
};

// ── DELETE ──
const deleteDemoExercise = (idx) => {
  demoExercises.value.splice(idx, 1);
  demoKeys.value.splice(idx, 1);
  demoResults.value.splice(idx, 1);
  demoBlocksRefs.value.splice(idx, 1);
  demoDropzoneRefs.value.splice(idx, 1);
  demoDropzoneHover.value.splice(idx, 1);
};

const deleteUserExercise = (idx) => {
  userExercises.value.splice(idx, 1);
  userKeys.value.splice(idx, 1);
  userResults.value.splice(idx, 1);
  userBlocksRefs.value.splice(idx, 1);
  userDropzoneRefs.value.splice(idx, 1);
  userDropzoneHover.value.splice(idx, 1);
};

// ── DROPZONE LOGIC ──
const isOverEl = (el, x, y) => {
  if (!el) return false;
  const r = el.getBoundingClientRect();
  return x >= r.left && x <= r.right && y >= r.top && y <= r.bottom;
};

const checkDropzone = (exercises, dzRefs, hoverRef, resultsRef, keysRef, blockRefsArr, e) => {
  exercises.forEach((ex, idx) => {
    hoverRef.value[idx] = false;
    if (!isOverEl(dzRefs.value[idx], e.clientX, e.clientY)) return;
    if (resultsRef.value[idx] === 'ok') return;
    const instance = blockRefsArr.value[idx];
    if (!instance) return;
    const bancaGridId = ex.nums.length;
    const groups = instance.getConnectedGroups();
    const bancaGroups = groups.filter(g => g.gridId === bancaGridId);
    const correct = bancaGroups.length === 1 && bancaGroups[0].count === ex.answer;
    if (correct) {
      resultsRef.value[idx] = 'ok';
    } else {
      resultsRef.value[idx] = 'bad';
      setTimeout(() => { resultsRef.value[idx] = null; keysRef.value[idx]++; }, 1000);
    }
  });
};

const onDocumentMouseUp = (e) => {
  checkDropzone(demoExercises.value, demoDropzoneRefs, demoDropzoneHover, demoResults, demoKeys, demoBlocksRefs, e);
  checkDropzone(userExercises.value, userDropzoneRefs, userDropzoneHover, userResults, userKeys, userBlocksRefs, e);
};

const onDocumentMouseMove = (e) => {
  demoExercises.value.forEach((_, idx) => {
    demoDropzoneHover.value[idx] = isOverEl(demoDropzoneRefs.value[idx], e.clientX, e.clientY);
  });
  userExercises.value.forEach((_, idx) => {
    userDropzoneHover.value[idx] = isOverEl(userDropzoneRefs.value[idx], e.clientX, e.clientY);
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

const resetDemoExercise = (idx) => { demoResults.value[idx] = null; demoKeys.value[idx]++; };
const resetUserExercise = (idx) => { userResults.value[idx] = null; userKeys.value[idx]++; };
const resetAll = () => {
  demoResults.value = demoExercises.value.map(() => null);
  demoKeys.value = demoKeys.value.map(k => k + 1000);
};
const resetAllUser = () => {
  userResults.value = userExercises.value.map(() => null);
  userKeys.value = userKeys.value.map(k => k + 1000);
};
</script>

<style scoped>
* { box-sizing: border-box; margin: 0; padding: 0; }



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
  gap: 0.75rem;
}
.center h1 { font-size: 1.5rem; margin-bottom: 0.5rem; }

/* ── Shared page ── */
.page { height: 100vh; display: flex; flex-direction: column; }

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

/* ── Exercises screen ── */
.exercise-scroll { flex: 1; overflow-y: auto; padding: 1.5rem 1rem 3rem; }

.exercise-list {
  max-width: 860px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.section-label {
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  color: #94a3b8;
  padding-bottom: 0.25rem;
  border-bottom: 1px solid #e5e7eb;
  margin-bottom: -0.5rem;
}
.section-label--user { color: #818cf8; border-color: #3730a3; }

/* ── Exercise card ── */
.ex-card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 1px 6px rgba(0,0,0,0.08);
  overflow: hidden;
  border: 2px solid transparent;
  transition: border-color 0.25s;
}
.ex-card--ok  { border-color: #10b981; }
.ex-card--bad { border-color: #ef4444; }

.ex-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.9rem 1.25rem 0;
}

.ex-index {
  width: 28px; height: 28px;
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

.ex-question { font-size: 1rem; font-weight: 600; color: #1e1b4b; flex: 1; line-height: 1.4; }

.badge { font-size: 0.75rem; font-weight: 700; padding: 0.2rem 0.6rem; border-radius: 20px; white-space: nowrap; }
.badge--ok  { background: #d1fae5; color: #065f46; }
.badge--bad { background: #fee2e2; color: #991b1b; }

.ex-grids { padding: 8px 0; overflow: visible; }
.ex-grids-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 0 12px;
}

/* Dropzone — pixel-for-pixel from original */
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
.dropzone--hover { border-color: #6366f1; background: rgba(99, 102, 241, 0.06); transform: scale(1.12); }
.dropzone--ok    { border-color: #10b981; background: rgba(16, 185, 129, 0.06); }
.dropzone--bad   { border-color: #ef4444; background: rgba(239, 68, 68, 0.06); }
.dropzone-icon   { font-size: 1.6rem; pointer-events: none; line-height: 1; user-select: none; }

.ex-footer {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.25rem;
  border-top: 1px solid #f3f4f6;
}

/* ── Creator screen ── */
.creator-scroll { flex: 1; overflow-y: auto; padding: 1.25rem 1rem 3rem; }

.creator-body {
  max-width: 480px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.draft-list {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 0.5rem 0.75rem;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.draft-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.3rem 0;
  border-bottom: 1px solid #f3f4f6;
  font-size: 0.85rem;
}
.draft-item:last-child { border-bottom: none; }
.draft-nums { font-weight: 700; color: #6366f1; white-space: nowrap; }
.draft-text { flex: 1; color: #555; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

.creator-form {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 1rem 1.1rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.field { display: flex; flex-direction: column; gap: 0.35rem; }
.field > label { font-weight: 600; color: #333; font-size: 0.875rem; }
.hint { font-weight: 400; color: #888; font-size: 0.8rem; }

.field input[type="text"] {
  width: 100%;
  padding: 0.38rem 0.6rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.875rem;
  font-family: inherit;
  background: white;
  color: #333;
}

.field input[type="number"] {
  width: 58px;
  padding: 0.38rem 0.4rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.875rem;
  font-family: inherit;
  text-align: center;
  background: white;
  color: #333;
}

.nums-row   { display: flex; align-items: center; gap: 0.45rem; flex-wrap: wrap; }
.num-entry  { display: flex; align-items: center; gap: 0.2rem; }
.answer-row { display: flex; align-items: center; gap: 0.5rem; }
.sublabel   { font-size: 0.8rem; color: #6b7280; }
.banca-preview { font-size: 0.85rem; color: #6366f1; font-weight: 600; }

/* ── Buttons ── */
button {
  padding: 0.4rem 0.9rem;
  background: white;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  font-family: inherit;
  color: #333;
}
button:hover    { background: #f0f0f0; }
button:disabled { opacity: 0.5; cursor: default; }

.btn-reset       { padding: 0.4rem 0.7rem; border-color: #e5e7eb; color: #6b7280; }
.btn-reset:hover { background: #f0f0f0; }

.btn-delete       { color: #ef4444; border-color: #fecaca; }
.btn-delete:hover { background: #fff5f5; }

.btn-icon       { padding: 0.2rem 0.45rem; font-size: 0.78rem; border-color: #e5e7eb; color: #6b7280; }
.btn-add        { color: #6366f1; border-color: #c7d2fe; }
.btn-add:hover  { background: #eef2ff; }

.btn-add-draft       { align-self: flex-start; border-color: #d1d5db; color: #333; }
.btn-add-draft:hover { background: #f0f0f0; }

.btn-load       { background: #6366f1; color: white; border-color: #6366f1; }
.btn-load:hover { background: #4f46e5; }


/* ── Creator: vertical grid inputs ── */
.grids-col {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  max-width: 100px;
}

.grid-num-input {
  width: 80px;
  padding: 0.38rem 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.9rem;
  font-family: inherit;
  background: white;
  color: #333;
  text-align: center;
}

.grid-num-input:focus { outline: 2px solid #6366f1; outline-offset: -1px; }

.banca-row {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}

.banca-dims {
  font-size: 0.8rem;
  color: #818cf8;
  font-weight: 600;
}

.empty-state {
  text-align: center;
  color: #6b7280;
  padding: 2rem 0;
  font-size: 0.9rem;
}
</style>