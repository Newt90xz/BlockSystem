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
          <div
            v-for="(ex, idx) in demoExercises"
            :key="'demo-' + demoKeys[idx]"
            class="ex-card"
            :class="{ 'ex-card--ok': demoResults[idx] === 'ok', 'ex-card--bad': demoResults[idx] === 'bad' }"
          >
            <div class="ex-header">
              <span class="ex-index">{{ idx + 1 }}</span>
              <p class="ex-question">{{ ex.text || '\u00A0' }}</p>
              <span v-if="demoResults[idx] === 'ok'" class="badge badge--ok">✓ Correcto</span>
              <span v-else-if="demoResults[idx] === 'bad'" class="badge badge--bad">✗ Inténtalo</span>
            </div>
            <div class="ex-grids">
              <div class="ex-grids-row">
                <Blocks
                  :ref="el => demoBlocksRefs[idx] = el"
                  :key="demoKeys[idx]"
                  :config="buildConfig(ex, 'demo-' + idx)"
                />
                <div class="dz-col">
                  <div
                    v-for="(dz, dzi) in ex.dropzones"
                    :key="dzi"
                    :ref="el => setDzRef('demo', idx, dzi, el)"
                    class="dropzone"
                    :class="{
                      'dropzone--ok':  demoDzStates[idx]?.[dzi] === 'ok',
                      'dropzone--bad': demoDzStates[idx]?.[dzi] === 'bad',
                      'dropzone--hover': demoDzHover[idx]?.[dzi],
                    }"
                  >
                    <span class="dz-label">{{ dz.label }}</span>
                    <span class="dropzone-icon">{{ demoDzStates[idx]?.[dzi] === 'ok' ? '📬' : '📭' }}</span>
                  </div>
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
          <div
            v-for="(ex, idx) in userExercises"
            :key="'user-' + userKeys[idx]"
            class="ex-card"
            :class="{ 'ex-card--ok': userResults[idx] === 'ok', 'ex-card--bad': userResults[idx] === 'bad' }"
          >
            <div class="ex-header">
              <span class="ex-index">{{ idx + 1 }}</span>
              <p class="ex-question">{{ ex.text || '\u00A0' }}</p>
              <span v-if="userResults[idx] === 'ok'" class="badge badge--ok">✓ Correcto</span>
              <span v-else-if="userResults[idx] === 'bad'" class="badge badge--bad">✗ Inténtalo</span>
            </div>
            <div class="ex-grids">
              <div class="ex-grids-row">
                <Blocks
                  :ref="el => userBlocksRefs[idx] = el"
                  :key="userKeys[idx]"
                  :config="buildConfig(ex, 'user-' + idx)"
                />
                <div class="dz-col">
                  <div
                    v-for="(dz, dzi) in ex.dropzones"
                    :key="dzi"
                    :ref="el => setDzRef('user', idx, dzi, el)"
                    class="dropzone"
                    :class="{
                      'dropzone--ok':  userDzStates[idx]?.[dzi] === 'ok',
                      'dropzone--bad': userDzStates[idx]?.[dzi] === 'bad',
                      'dropzone--hover': userDzHover[idx]?.[dzi],
                    }"
                  >
                    <span class="dz-label">{{ dz.label }}</span>
                    <span class="dropzone-icon">{{ userDzStates[idx]?.[dzi] === 'ok' ? '📬' : '📭' }}</span>
                  </div>
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
        <span style="font-weight:600">Crear ejercicio</span>
        <button v-if="draft.length" class="btn-load" @click="loadDraft" style="margin-left:auto">
          Cargar {{ draft.length }} ejercicio{{ draft.length > 1 ? 's' : '' }}
        </button>
      </div>

      <div class="creator-scroll">
        <div class="creator-layout">

          <!-- LEFT: form -->
          <div class="creator-left">

            <!-- Draft list -->
            <div v-if="draft.length" class="draft-list">
              <div v-for="(ex, i) in draft" :key="i" class="draft-item">
                <span class="draft-nums">{{ ex.grids.map(g => g.value).join(' + ') }}</span>
                <span class="draft-dropzones">→ {{ ex.dropzones.map(d => d.label + ':' + d.answer).join(', ') }}</span>
                <button class="btn-icon btn-delete" @click="draft.splice(i, 1)">✕</button>
              </div>
            </div>

            <!-- Form -->
            <div class="creator-form">

              <div class="field">
                <label>Enunciado <span class="hint">(opcional)</span></label>
                <input v-model="form.text" type="text" placeholder="Ej: Pedro tiene 5 manzanas..." />
              </div>

              <div class="field">
                <label>Grids <span class="hint">— valor de cada grupo de bloques</span></label>
                <div class="grids-col">
                  <div v-for="(g, i) in form.grids" :key="i" class="grid-row">
                    <input
                      class="grid-num-input"
                      type="number" min="1" max="20"
                      placeholder=""
                      :value="g.value > 0 ? g.value : ''"
                      @input="onGridValueInput(i, $event)"
                      @blur="onGridBlur(i, $event)"
                    />
                    <input
                      v-if="g.value > 0"
                      class="grid-label-input"
                      type="text"
                      :placeholder="'Grid ' + (i + 1)"
                      v-model="g.label"
                    />
                  </div>
                </div>
              </div>

              <div class="field" v-if="validGrids.length">
                <label>Dropzones <span class="hint">— respuestas esperadas</span></label>
                <div class="grids-col">
                  <div v-for="(dz, i) in form.dropzones" :key="i" class="grid-row">
                    <input
                      class="grid-num-input"
                      type="number" min="0" max="20"
                      placeholder="0"
                      :value="dz.answer > 0 ? dz.answer : ''"
                      @input="onDzAnswerInput(i, $event)"
                      @blur="onDzBlur(i, $event)"
                    />
                    <input
                      class="grid-label-input"
                      type="text"
                      :placeholder="'Dropzone ' + (i + 1)"
                      v-model="dz.label"
                    />
                    <button v-if="form.dropzones.length > 1" class="btn-icon btn-delete" @click="form.dropzones.splice(i, 1)">✕</button>
                  </div>
                  <button class="btn-icon btn-add" @click="form.dropzones.push(defaultDz())" :disabled="form.dropzones.length >= 6">+ dropzone</button>
                </div>
              </div>

              <div class="field" v-if="validGrids.length">
                <label>Banca <span class="hint">— calculada automáticamente</span></label>
                <span class="banca-dims">{{ bancaShape.cols }} col × {{ bancaShape.rows }} fil</span>
              </div>

              <button class="btn-add-draft" @click="addToDraft" :disabled="!canAdd">
                + Agregar a la lista
              </button>
            </div>
          </div>

          <!-- RIGHT: live preview -->
          <div class="creator-right">
            <div class="preview-label">Vista previa</div>
            <div v-if="!previewExercise" class="preview-empty">
              Ingresa al menos un número para ver el ejercicio.
            </div>
            <div v-else class="preview-card" :class="{ 'preview-card--last': !previewIsLive }">
              <p v-if="!previewIsLive" class="preview-last-label">Último agregado</p>
              <p v-if="previewExercise.text" class="preview-question">{{ previewExercise.text }}</p>
              <div class="preview-grids">
                <div class="preview-blocks-wrap">
                  <Blocks
                    :key="previewKey"
                    :config="buildConfig(previewExercise, 'preview')"
                  />
                </div>
                <div class="dz-col">
                  <div
                    v-for="(dz, i) in previewExercise.dropzones"
                    :key="i"
                    class="dropzone preview-dz"
                  >
                    <span class="dz-label">{{ dz.label }}</span>
                    <span class="dropzone-icon">📭</span>
                  </div>
                </div>
              </div>
            </div>
          </div>

        </div>

        <!-- Existing user exercises shown below for reference -->
        <div v-if="userExercises.length" class="creator-existing">
          <div class="creator-existing-label">Ejercicios ya creados</div>
          <div class="creator-existing-list">
            <div
              v-for="(ex, idx) in userExercises"
              :key="'ce-' + idx"
              class="ce-card"
            >
              <span class="ex-index">{{ idx + 1 }}</span>
              <p class="ce-text">{{ ex.text || '\u00A0' }}</p>
              <span class="ce-summary">
                {{ ex.grids.map(g => g.value).join(' + ') }}
                → {{ ex.dropzones.map(d => d.label + ' ' + d.answer).join(', ') }}
              </span>
              <button class="btn-reset btn-delete" @click="deleteUserExercise(idx)">✕</button>
            </div>
          </div>
        </div>

      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue';
import Blocks from './components/Blocks.vue';

// ── SCREEN ──
const screen    = ref(null);
const showLabels = ref(true);

// ── HELPERS ──
const gridShape = (n) => n % 2 === 0
  ? { cols: 2, rows: n / 2 }
  : { cols: n, rows: 1 };

// Banca fits (maxAnswer + 1) with 3 rows
const bancaGridShape = (dropzones) => {
  const maxAnswer = Math.max(...dropzones.map(d => d.answer), 1);
  const total = maxAnswer + 1;
  const rows = 3;
  const cols = Math.max(Math.ceil(total / rows), total);
  return { cols, rows };
};

// ── EXERCISE FACTORY ──
const createExercise = ({ text = '', grids, dropzones }) => {
  const shape = bancaGridShape(dropzones);
  return {
    text,
    grids,
    dropzones,
    bancaCols: shape.cols,
    bancaRows: shape.rows,
    totalBlocks: grids.reduce((s, g) => s + g.value, 0),
  };
};

// ── BUILD CONFIG ──
const buildConfig = (ex, key) => ({
  inline: true,
  inlineColumns: 2,
  showToolbar: false,
  showGridLabels: showLabels.value,
  maxBloques: (ex.totalBlocks ?? 0) * 2 + 4,
  storageKey: `ex_${key}`,
  grids: [
    ...ex.grids.map(g => ({
      label: g.label || String(g.value),
      ...gridShape(g.value),
      initialBlocks: [g.value],
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

// ── DEMO EXERCISES ──
const mkDemo = ({ text, nums, answer }) => {
  const grids     = nums.map(n => ({ value: n, label: String(n) }));
  const dropzones = [{ label: 'Respuesta', answer }];
  const shape     = bancaGridShape(dropzones);
  return {
    text, grids, dropzones,
    bancaCols: shape.cols, bancaRows: shape.rows,
    totalBlocks: nums.reduce((a, b) => a + b, 0),
  };
};

const demoExercises = ref([
  mkDemo({ text: "María tiene 2 manzanas y Juan tiene 3. ¿Cuántas tienen en total?", nums: [2, 3], answer: 5 }),
  mkDemo({ text: "En el patio hay 4 perros y llegan 2 más. ¿Cuántos perros hay ahora?",           nums: [4, 2], answer: 6 }),
  mkDemo({ text: "Ana juntó 3 piedras en el parque y 4 en la playa. ¿Cuántas piedras tiene?",     nums: [3, 4], answer: 7 }),
]);

// ── USER EXERCISES ──
const userExercises = ref([]);

// ── REACTIVE STATE ──
const demoKeys     = ref(demoExercises.value.map((_, i) => i * 100));
const demoResults  = ref(demoExercises.value.map(() => null));
const demoBlocksRefs = ref(demoExercises.value.map(() => null));
// dzStates[exIdx][dzi] = null | 'ok' | 'bad'
const demoDzStates = ref(demoExercises.value.map(ex => ex.dropzones.map(() => null)));
const demoDzHover  = ref(demoExercises.value.map(ex => ex.dropzones.map(() => false)));

const userKeys     = ref([]);
const userResults  = ref([]);
const userBlocksRefs = ref([]);
const userDzStates = ref([]);
const userDzHover  = ref([]);

const growUserArrays = (exercises) => {
  exercises.forEach(ex => {
    userKeys.value.push(userKeys.value.length * 100 + Date.now() % 1000);
    userResults.value.push(null);
    userBlocksRefs.value.push(null);
    userDzStates.value.push(ex.dropzones.map(() => null));
    userDzHover.value.push(ex.dropzones.map(() => false));
  });
};

// ── DROPZONE DOM REFS (Map keyed by "demo|user-exIdx-dzi") ──
const dzRefMap = new Map();
const setDzRef = (group, exIdx, dzi, el) => {
  dzRefMap.set(`${group}-${exIdx}-${dzi}`, el);
};
const getDzEl = (group, exIdx, dzi) => dzRefMap.get(`${group}-${exIdx}-${dzi}`) ?? null;

// ── DROPZONE LOGIC ──
const isOverEl = (el, x, y) => {
  if (!el) return false;
  const r = el.getBoundingClientRect();
  return x >= r.left && x <= r.right && y >= r.top && y <= r.bottom;
};

const checkGroup = (group, exercises, blocksRefs, dzStatesRef, resultsRef, e) => {
  exercises.forEach((ex, idx) => {
    const instance = blocksRefs.value[idx];
    if (!instance) return;

    const bancaGridId = ex.grids.length;
    const groups      = instance.getConnectedGroups();
    const bancaGroups = groups.filter(g => g.gridId === bancaGridId);

    ex.dropzones.forEach((dz, dzi) => {
      if (dzStatesRef.value[idx]?.[dzi] === 'ok') return;
      const el = getDzEl(group, idx, dzi);
      if (!isOverEl(el, e.clientX, e.clientY)) return;

      const match = bancaGroups.find(g => g.count === dz.answer);
      if (match) {
        dzStatesRef.value[idx][dzi] = 'ok';
      } else {
        dzStatesRef.value[idx][dzi] = 'bad';
        setTimeout(() => {
          if (dzStatesRef.value[idx]) dzStatesRef.value[idx][dzi] = null;
        }, 900);
      }
    });

    // All dropzones correct → exercise ok
    if (ex.dropzones.every((_, dzi) => dzStatesRef.value[idx]?.[dzi] === 'ok'))
      resultsRef.value[idx] = 'ok';
  });
};

const onDocumentMouseMove = (e) => {
  [
    ['demo', demoExercises.value, demoDzStates, demoDzHover],
    ['user', userExercises.value, userDzStates, userDzHover],
  ].forEach(([group, exs, dzStates, dzHover]) => {
    exs.forEach((ex, idx) => {
      ex.dropzones.forEach((_, dzi) => {
        if (dzStates.value[idx]?.[dzi] === 'ok') return;
        const el   = getDzEl(group, idx, dzi);
        const over = isOverEl(el, e.clientX, e.clientY);
        if (dzHover.value[idx]) dzHover.value[idx][dzi] = over;
      });
    });
  });
};

const onDocumentMouseUp = (e) => {
  checkGroup('demo', demoExercises.value, demoBlocksRefs, demoDzStates, demoResults, e);
  checkGroup('user', userExercises.value, userBlocksRefs, userDzStates, userResults, e);
};

onMounted(() => {
  document.addEventListener('mouseup', onDocumentMouseUp);
  document.addEventListener('mousemove', onDocumentMouseMove);
});
onUnmounted(() => {
  document.removeEventListener('mouseup', onDocumentMouseUp);
  document.removeEventListener('mousemove', onDocumentMouseMove);
});

// ── RESET / DELETE ──
const resetDemoExercise = (idx) => {
  demoResults.value[idx] = null;
  demoKeys.value[idx]++;
  demoDzStates.value[idx] = demoExercises.value[idx].dropzones.map(() => null);
  demoDzHover.value[idx]  = demoExercises.value[idx].dropzones.map(() => false);
};
const resetUserExercise = (idx) => {
  userResults.value[idx] = null;
  userKeys.value[idx]++;
  userDzStates.value[idx] = userExercises.value[idx].dropzones.map(() => null);
  userDzHover.value[idx]  = userExercises.value[idx].dropzones.map(() => false);
};
const resetAll = () => {
  demoResults.value  = demoExercises.value.map(() => null);
  demoKeys.value     = demoKeys.value.map(k => k + 1000);
  demoDzStates.value = demoExercises.value.map(ex => ex.dropzones.map(() => null));
  demoDzHover.value  = demoExercises.value.map(ex => ex.dropzones.map(() => false));
};
const resetAllUser = () => {
  userResults.value  = userExercises.value.map(() => null);
  userKeys.value     = userKeys.value.map(k => k + 1000);
  userDzStates.value = userExercises.value.map(ex => ex.dropzones.map(() => null));
  userDzHover.value  = userExercises.value.map(ex => ex.dropzones.map(() => false));
};
const deleteDemoExercise = (idx) => {
  [demoExercises, demoKeys, demoResults, demoBlocksRefs, demoDzStates, demoDzHover]
    .forEach(r => r.value.splice(idx, 1));
};
const deleteUserExercise = (idx) => {
  [userExercises, userKeys, userResults, userBlocksRefs, userDzStates, userDzHover]
    .forEach(r => r.value.splice(idx, 1));
};

// ── CREATOR ──
const defaultGrid = () => ({ value: '', label: '' });
const defaultDz   = () => ({ label: '', answer: '' });
const defaultForm = () => ({ text: '', grids: [defaultGrid()], dropzones: [defaultDz()] });

const form      = ref(defaultForm());
const draft     = ref([]);
const lastAdded = ref(null);

const validGrids = computed(() =>
  form.value.grids.filter(g => parseInt(g.value) > 0)
);
const totalValue = computed(() =>
  validGrids.value.reduce((s, g) => s + parseInt(g.value), 0)
);
const bancaShape = computed(() => {
  const dzs = form.value.dropzones.filter(d => parseInt(d.answer) > 0);
  return bancaGridShape(
    dzs.length ? dzs.map(d => ({ answer: parseInt(d.answer) }))
               : [{ answer: totalValue.value }]
  );
});
const canAdd = computed(() =>
  validGrids.value.length > 0 &&
  form.value.dropzones.some(d => parseInt(d.answer) > 0)
);

// Grid inputs — cascade: fill → next slot appears
const onGridValueInput = (i, e) => {
  const raw = parseInt(e.target.value);
  const val = isNaN(raw) ? '' : Math.min(20, Math.max(1, raw));
  e.target.value = val === '' ? '' : val;
  form.value.grids[i].value = val;
  if (i === form.value.grids.length - 1 && val > 0 && form.value.grids.length < 7)
    form.value.grids.push(defaultGrid());
};
const onGridBlur = (i, e) => {
  const parsed = parseInt(e.target.value);
  if (!e.target.value || isNaN(parsed) || parsed <= 0) form.value.grids[i].value = '';
  const gs = form.value.grids;
  let last = gs.length - 1;
  while (last > 0 && (!gs[last].value || parseInt(gs[last].value) <= 0)) last--;
  form.value.grids = [...gs.slice(0, last + 1), defaultGrid()];
};

// Dropzone inputs
const onDzAnswerInput = (i, e) => {
  const raw = parseInt(e.target.value);
  const val = isNaN(raw) ? '' : Math.min(20, Math.max(0, raw));
  e.target.value = val === '' ? '' : val;
  form.value.dropzones[i].answer = val;
};
const onDzBlur = (i, e) => {
  const parsed = parseInt(e.target.value);
  if (!e.target.value || isNaN(parsed) || parsed <= 0) form.value.dropzones[i].answer = '';
};

// Live preview
const previewKey = ref(0);
watch(() => [form.value.grids, form.value.dropzones], () => { previewKey.value++; }, { deep: true });

const previewIsLive = computed(() => validGrids.value.length > 0);

const previewExercise = computed(() => {
  if (previewIsLive.value) {
    const grids = validGrids.value.map(g => ({
      value: parseInt(g.value),
      label: g.label || String(g.value),
    }));
    const dzs = form.value.dropzones
      .filter(d => parseInt(d.answer) > 0)
      .map(d => ({ label: d.label || 'Dropzone', answer: parseInt(d.answer) }));
    const dz    = dzs.length ? dzs : [{ label: 'Respuesta', answer: totalValue.value }];
    const shape = bancaGridShape(dz);
    return {
      text: form.value.text,
      grids,
      dropzones: dz,
      bancaCols: shape.cols,
      bancaRows: shape.rows,
      totalBlocks: grids.reduce((s, g) => s + g.value, 0),
    };
  }
  return lastAdded.value;
});

const addToDraft = () => {
  if (!canAdd.value) return;
  const grids = validGrids.value.map(g => ({
    value: parseInt(g.value),
    label: g.label || String(g.value),
  }));
  const dropzones = form.value.dropzones
    .filter(d => parseInt(d.answer) > 0)
    .map(d => ({ label: d.label || 'Dropzone', answer: parseInt(d.answer) }));
  const ex = createExercise({ text: form.value.text, grids, dropzones });
  draft.value.push(ex);
  lastAdded.value = ex;
  form.value = defaultForm();
};

const loadDraft = () => {
  growUserArrays(draft.value);
  userExercises.value.push(...draft.value);
  draft.value     = [];
  lastAdded.value = null;
  screen.value    = 'user-exercises';
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

/* ── Start ── */
.center {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
  gap: 0.75rem;
}
.center h1 { font-size: 1.5rem; margin-bottom: 0.5rem; }

/* ── Page ── */
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

/* ── Exercise list ── */
.exercise-scroll { flex: 1; overflow-y: auto; padding: 1.5rem 1rem 3rem; }
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

/* Dropzone column — works for 1 or many */
.dz-col {
  display: flex;
  flex-direction: column;
  gap: 8px;
  align-items: center;
  flex-shrink: 0;
}

.dropzone {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 58px;
  min-height: 52px;
  padding: 4px 2px;
  border: 2px solid #94a3b8;
  border-radius: 10px;
  flex-shrink: 0;
  transition: transform 0.15s ease, border-color 0.15s, background 0.15s;
  gap: 2px;
  cursor: default;
}
.dropzone--hover { border-color: #6366f1; background: rgba(99,102,241,0.06); transform: scale(1.08); }
.dropzone--ok    { border-color: #10b981; background: rgba(16,185,129,0.06); }
.dropzone--bad   { border-color: #ef4444; background: rgba(239,68,68,0.06); }
.dropzone-icon   { font-size: 1.3rem; pointer-events: none; line-height: 1; user-select: none; }
.dz-label {
  font-size: 0.65rem;
  font-weight: 700;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  pointer-events: none;
  text-align: center;
  max-width: 54px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.ex-footer {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.25rem;
  border-top: 1px solid #f3f4f6;
}

/* ── Creator ── */
.creator-scroll { flex: 1; overflow-y: auto; padding: 1.25rem 1rem 3rem; }
.creator-layout {
  max-width: 920px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: 340px 1fr;
  gap: 1.25rem;
  align-items: start;
}
.creator-left {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

/* Draft list */
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
  gap: 0.6rem;
  padding: 0.3rem 0;
  border-bottom: 1px solid #f3f4f6;
  font-size: 0.82rem;
}
.draft-item:last-child { border-bottom: none; }
.draft-nums      { font-weight: 700; color: #6366f1; white-space: nowrap; }
.draft-dropzones { flex: 1; color: #555; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

/* Form */
.creator-form {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 1rem 1.1rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.field              { display: flex; flex-direction: column; gap: 0.35rem; }
.field > label      { font-weight: 600; color: #333; font-size: 0.875rem; }
.hint               { font-weight: 400; color: #888; font-size: 0.8rem; }
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
.grids-col { display: flex; flex-direction: column; gap: 0.35rem; }
.grid-row  { display: flex; align-items: center; gap: 0.4rem; }
.grid-num-input {
  width: 58px;
  padding: 0.35rem 0.4rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.875rem;
  font-family: inherit;
  background: white;
  color: #333;
  text-align: center;
}
.grid-num-input:focus { outline: 2px solid #6366f1; outline-offset: -1px; }
.grid-label-input {
  flex: 1;
  padding: 0.35rem 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.8rem;
  font-family: inherit;
  background: white;
  color: #555;
  max-width: 130px;
}
.grid-label-input:focus { outline: 2px solid #6366f1; outline-offset: -1px; }
.banca-dims { font-size: 0.82rem; color: #818cf8; font-weight: 600; }

/* ── Preview ── */
.creator-right {
  position: sticky;
  top: 0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.preview-label {
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  color: #94a3b8;
}
.preview-empty {
  background: white;
  border: 1px dashed #d1d5db;
  border-radius: 8px;
  padding: 2rem 1rem;
  text-align: center;
  color: #9ca3af;
  font-size: 0.85rem;
}
.preview-card {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 0.75rem;
  overflow: visible;
}
.preview-card--last {
  border-color: #c7d2fe;
  background: #fafbff;
}
.preview-last-label {
  font-size: 0.7rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #818cf8;
  margin-bottom: 0.4rem;
}
.preview-question {
  font-size: 0.9rem;
  font-weight: 600;
  color: #1e1b4b;
  margin-bottom: 0.5rem;
  line-height: 1.4;
}
.preview-grids {
  display: flex;
  align-items: center;
  gap: 12px;
  overflow: visible;
}
.preview-blocks-wrap {
  pointer-events: none;
  user-select: none;
}
.preview-dz {
  opacity: 0.65;
  pointer-events: none;
}

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
.btn-icon  { padding: 0.2rem 0.45rem; font-size: 0.78rem; border-color: #e5e7eb; color: #6b7280; }
.btn-add   { color: #6366f1; border-color: #c7d2fe; font-size: 0.78rem; padding: 0.2rem 0.55rem; }
.btn-add:hover { background: #eef2ff; }
.btn-add-draft       { align-self: flex-start; border-color: #d1d5db; color: #333; }
.btn-add-draft:hover { background: #f0f0f0; }
.btn-load       { background: #6366f1; color: white; border-color: #6366f1; }
.btn-load:hover { background: #4f46e5; }

.empty-state {
  text-align: center;
  color: #6b7280;
  padding: 2rem 0;
  font-size: 0.9rem;
}

/* ── Creator: existing exercises ── */
.creator-existing {
  max-width: 920px;
  margin: 1.5rem auto 0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.creator-existing-label {
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  color: #6366f1;
  padding-bottom: 0.3rem;
  border-bottom: 1px solid #c7d2fe;
}

.creator-existing-list {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.ce-card {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 0.55rem 0.9rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.85rem;
}

.ce-text {
  flex: 1;
  color: #374151;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  min-width: 0;
}

.ce-summary {
  color: #6b7280;
  font-size: 0.8rem;
  white-space: nowrap;
  flex-shrink: 0;
}
</style>