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
          <Blocks
            v-for="(ex, idx) in demoExercises"
            :key="'demo-' + demoKeys[idx]"
            :exercise="ex"
            @complete="r => onComplete('demo', idx, r)"
          />
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
          <Blocks
            v-for="(ex, idx) in userExercises"
            :key="'user-' + userKeys[idx]"
            :exercise="ex"
            @complete="r => onComplete('user', idx, r)"
          />
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
              <div
                v-for="(ex, i) in draft" :key="i"
                class="draft-item"
                :class="{ 'draft-item--pinned': pinnedPreview === ex }"
                @click="pinnedPreview = pinnedPreview === ex ? null : ex"
              >
                <span class="draft-nums">{{ ex.grids.map(g => g.label || g.value).join(' + ') }}</span>
                <span class="draft-dropzones">→ {{ (ex.answers ?? []).map(a => prettyAnswer(ex, a)).join(', ') || 'banca' }}</span>
                <button class="btn-icon btn-delete" @click.stop="draft.splice(i, 1); if(pinnedPreview === ex) pinnedPreview = null">✕</button>
              </div>
            </div>

            <!-- Form -->
            <div class="creator-form">

              <!-- Mode: primary config -->
              <div class="field">
                <label>Modo</label>
                <div class="mode-toggle">
                  <button
                    class="mode-btn"
                    :class="{ 'mode-btn--active': !form.shuffle }"
                    @click="form.shuffle = false"
                  >Grids separados</button>
                  <button
                    class="mode-btn"
                    :class="{ 'mode-btn--active': form.shuffle }"
                    @click="form.shuffle = true"
                  >Modo banca</button>
                </div>
              </div>

              <div class="field">
                <label>Enunciado <span class="hint">(opcional)</span></label>
                <input v-model="form.text" type="text" placeholder="Ej: Pedro tiene 5 manzanas..." />
              </div>

              <div class="field">
                <label>Grids <span class="hint">— bloques por persona/grupo</span></label>
                <div class="grids-col">
                  <div v-for="(g, i) in form.grids" :key="i" class="grid-entry">
                    <div class="grid-row">
                      <span class="grid-idx">{{ i + 1 }}</span>
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
                        :placeholder="'Nombre ' + (i + 1)"
                        v-model="g.label"
                      />
                    </div>
                  </div>
                </div>
              </div>

              <div class="field" v-if="validGrids.length">
                <label>Banca <span class="hint">— nombre y respuesta esperada</span></label>
                <div class="grid-row" style="margin-bottom:0.5rem">
                  <input
                    class="grid-label-input"
                    type="text"
                    placeholder="Banca de respuesta"
                    v-model="form.bancaLabel"
                    style="max-width:180px"
                  />
                  <span class="banca-dims">{{ bancaShape.cols }} col × {{ bancaShape.rows }} fil</span>
                </div>
                <div class="needs-table">
                  <div v-for="(g, i) in validGrids" :key="i" class="needs-row">
                    <span class="needs-grid-idx">{{ form.grids.indexOf(g) + 1 }}</span>
                    <span class="needs-label">{{ g.label || ('Grid ' + (form.grids.indexOf(g) + 1)) }}</span>
                    <span class="needs-arrow">→</span>
                    <input
                      class="grid-num-input grid-num-input--sm"
                      type="number" min="0" max="20"
                      placeholder="0"
                      :value="g.needs > 0 ? g.needs : ''"
                      @input="onGridNeedsInput(form.grids.indexOf(g), $event)"
                    />
                    <span class="needs-hint">bloques</span>
                  </div>
                </div>
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
                    :exercise="previewExercise"
                  />
                </div>

              </div>
            </div>
          </div>

        </div>



      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import Blocks from './components/Blocks.vue';

// ── SCREEN ──
const screen    = ref(null);
const showLabels = ref(true);

// ── HELPERS ──


// buildConfig removed — Blocks.vue now handles this internally via :exercise prop.

// ── DEMO EXERCISES ──
const demoExercises = ref([
  // Origin-based exercises
  {
    text: "Pedro tiene 3 manzanas y Alicia tiene 2. Pedro le da 2 a Alicia. ¿Cuántas tiene Alicia al final?",
    grids:   [{ value: 3, label: '3' }, { value: 2, label: '2' }],
    bancas:  [{ label: 'Alicia' }],
    answers: [
      { bancaIdx: 0, gridIdx: 0, gridLabel: '3',  needs: 2 },
      { bancaIdx: 0, gridIdx: 1, gridLabel: '2', needs: 2 },
    ],
  },
  {
    text: "Pedro tiene 5 naranjas. Le regala 3 a Juan. ¿Cuántas le quedan a Pedro?",
    grids:   [{ value: 5, label: '5' }],
    bancas:  [{ label: 'Pedro' }],
    answers: [{ bancaIdx: 0, gridIdx: 0, gridLabel: '5', needs: 2 }],
  },
]);

// ── USER EXERCISES ──
const userExercises = ref([]);

// ── REACTIVE STATE ──
const demoKeys     = ref(demoExercises.value.map((_, i) => i * 100));
// dzStates[exIdx][dzi] = null | 'ok' | 'bad'

const userKeys     = ref([]);

const growUserArrays = (exercises) => {
  exercises.forEach(ex => {
    userKeys.value.push(userKeys.value.length * 100 + Date.now() % 1000);
  });
};


// ── COMPLETE ──
// Blocks handles result state internally — App just listens.
const onComplete = (group, idx, { correct }) => {
  // Can hook into analytics or parent state here if needed.
};

// ── RESET / DELETE ──
const resetDemoExercise = (idx) => {
  demoKeys.value[idx]++;
};
const resetUserExercise = (idx) => {
  userKeys.value[idx]++;
};
const resetAll = () => {
  demoKeys.value     = demoKeys.value.map(k => k + 1000);
};
const resetAllUser = () => {
  userKeys.value     = userKeys.value.map(k => k + 1000);
};
const deleteDemoExercise = (idx) => {
  [demoExercises, demoKeys]
    .forEach(r => r.value.splice(idx, 1));
};
const deleteUserExercise = (idx) => {
  [userExercises, userKeys]
    .forEach(r => r.value.splice(idx, 1));
};

// ── CREATOR ──
const defaultGrid = () => ({ value: '', label: '', needs: '' });
const defaultForm = () => ({ text: '', grids: [defaultGrid()], bancaLabel: '', shuffle: false });

const form      = ref(defaultForm());
const draft     = ref([]);
const lastAdded     = ref(null);
const pinnedPreview  = ref(null);
const expandedExIdx  = ref(null); // set when user taps a draft/existing item

const validGrids = computed(() =>
  form.value.grids.filter(g => parseInt(g.value) > 0)
);
const totalValue = computed(() =>
  validGrids.value.reduce((s, g) => s + parseInt(g.value), 0)
);
const bancaShape = computed(() => {
  const totalNeeds = validGrids.value.reduce((s, g) => s + (parseInt(g.needs) || 0), 0);
  // cols = totalNeeds + 1 so blocks can chain horizontally
  const cols = Math.max(totalNeeds + 1, 3);
  return { cols, rows: 3 };
});
const canAdd = computed(() =>
  validGrids.value.length > 0 &&
  validGrids.value.some(g => parseInt(g.needs) > 0)
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

const onGridNeedsInput = (i, e) => {
  const raw = parseInt(e.target.value);
  const val = isNaN(raw) ? '' : Math.min(20, Math.max(0, raw));
  e.target.value = val === '' ? '' : val;
  form.value.grids[i].needs = val;
};

// Dropzone inputs (kept for backward compat, no longer used in form)

// Live preview
const previewKey = ref(0);

const previewIsLive = computed(() => validGrids.value.length > 0 && !pinnedPreview.value);

const previewExercise = computed(() => {
  // Pinned takes priority (user clicked a draft/existing item)
  if (pinnedPreview.value) return pinnedPreview.value;
  // Live form preview
  if (validGrids.value.length > 0) {
    const rawGrids = validGrids.value;
    const grids = rawGrids.map(g => ({
      value: parseInt(g.value),
      label: g.label || String(g.value),
    }));
    // Build answers from raw validGrids (which have needs field)
    const answers = rawGrids.reduce((arr, g, gridIdx) => {
      if (parseInt(g.needs) > 0) {
        arr.push({
          bancaIdx: 0,
          gridIdx,
          gridLabel: g.label || String(g.value), // kept for display/backward compat
          needs: parseInt(g.needs),
        });
      }
      return arr;
    }, []);
    return {
      text:   form.value.text,
      grids,
      bancas: [{ label: form.value.bancaLabel || 'Banca de respuesta', shuffle: form.value.shuffle }],
      answers,
    };
  }
  return lastAdded.value;
});

// Clear pin when user starts editing the form again
watch(() => JSON.stringify(form.value), () => {
  if (pinnedPreview.value && validGrids.value.length > 0) pinnedPreview.value = null;
}, { deep: true });

const addToDraft = () => {
  if (!canAdd.value) return;
  const grids = validGrids.value.map(g => ({
    value: parseInt(g.value),
    label: g.label || String(g.value),
  }));
  const answers = validGrids.value.reduce((arr, g, gridIdx) => {
    if (parseInt(g.needs) > 0) {
      arr.push({
        bancaIdx: 0,
        gridIdx,
        gridLabel: g.label || String(g.value), // kept for display/backward compat
        needs: parseInt(g.needs),
      });
    }
    return arr;
  }, []);
  const ex = {
    text:   form.value.text,
    grids,
    bancas: [{ label: form.value.bancaLabel || 'Banca de respuesta', shuffle: form.value.shuffle }],
    answers,
  };
  draft.value.push(ex);
  lastAdded.value = ex;
  form.value = defaultForm();
};

// Helper for draft list rendering (supports new + old answer shapes)
const prettyAnswer = (ex, a) => {
  if (a?.gridIdx != null) {
    const label = ex?.grids?.[a.gridIdx]?.label ?? `Grid ${a.gridIdx + 1}`;
    return `${label}:${a.needs}`;
  }
  return `${a?.gridLabel ?? 'Grid'}:${a?.needs ?? ''}`;
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
.grid-entry { display: flex; flex-direction: column; gap: 0.25rem; }
.grid-row  { display: flex; align-items: center; gap: 0.4rem; }

.grid-idx {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #e0e7ff;
  color: #6366f1;
  font-size: 0.7rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.needs-table { display: flex; flex-direction: column; gap: 0.3rem; }

.needs-row {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.82rem;
}

.needs-grid-idx {
  width: 18px; height: 18px;
  border-radius: 50%;
  background: #e0e7ff;
  color: #6366f1;
  font-size: 0.65rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.needs-label { color: #374151; font-weight: 600; min-width: 60px; }
.needs-arrow { color: #9ca3af; }
.needs-hint  { color: #9ca3af; font-size: 0.75rem; }

.grid-num-input--sm { width: 46px; }
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


.btn-icon  { padding: 0.2rem 0.45rem; font-size: 0.78rem; border-color: #e5e7eb; color: #6b7280; }
.btn-add   { color: #6366f1; border-color: #c7d2fe; font-size: 0.78rem; padding: 0.2rem 0.55rem; }
.btn-add:hover { background: #eef2ff; }
.btn-add-draft       { align-self: flex-start; border-color: #d1d5db; color: #333; }
.btn-add-draft:hover { background: #f0f0f0; }
.btn-load       { background: #6366f1; color: white; border-color: #6366f1; }
.btn-load:hover { background: #4f46e5; }

.mode-toggle {
  display: flex;
  gap: 0;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  overflow: hidden;
  width: fit-content;
}

.mode-btn {
  padding: 0.3rem 0.75rem;
  border: none;
  border-radius: 0;
  font-size: 0.8rem;
  font-weight: 500;
  background: white;
  color: #6b7280;
  cursor: pointer;
}
.mode-btn:hover { background: #f0f0f0; }
.mode-btn--active {
  background: #6366f1;
  color: white;
}
.mode-btn--active:hover { background: #4f46e5; }

.empty-state {
  text-align: center;
  color: #6b7280;
  padding: 2rem 0;
  font-size: 0.9rem;
}

.draft-item { cursor: pointer; }
.draft-item:hover { background: #fafbff; }
.draft-item--pinned { background: #eef2ff; border-radius: 4px; }

.preview-last-label--pinned {
  color: #6366f1;
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin-bottom: 0.35rem;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
