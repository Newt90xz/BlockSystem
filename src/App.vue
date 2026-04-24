<template>
  <div class="app">

    <!-- INICIO -->
    <div v-if="!modo" class="center">
      <h1>🧱 Bloques</h1>
      <!-- <button @click="modo = 'config'">⚙️ Demo configuración</button> -->
      <button @click="modo = 'ejercicios'">➕ Ejercicios de suma</button>
    </div>

    <!-- CONFIG -->
    <div v-else-if="modo === 'config'" class="page">
      <div class="bar">
        <button @click="modo = null">← Volver</button>
        <span>Demo configuración</span>
        <button @click="modalAbierto = true">⚙️ Configurar grids</button>
      </div>

      <div class="preview">
        <Blocks :key="key" :maxBloques="cfg.maxBloques" :maxPorGrupo="99"
          :showToolbar="cfg.showToolbar" :grids="gridsActivos" />
      </div>
    </div>

    <!-- EJERCICIOS -->
    <div v-else-if="modo === 'ejercicios'" class="page">
      <div class="bar">
        <button @click="modo = null">← Volver</button>
        <button @click="showGridLabels = !showGridLabels">
          {{ showGridLabels ? '🔢 Ocultar números' : '🔢 Mostrar números' }}
        </button>
        <button @click="reiniciar">↺ Reiniciar</button>
      </div>

      <!-- Lista -->
      <div class="form-scroll">
        <div class="form-inner">

          <!-- Una tarjeta por ejercicio -->
          <div
            v-for="(ej, idx) in lista"
            :key="idx"
            class="ej-card"
            :class="{
              'ej-card--ok': resultados[idx] === 'ok',
              'ej-card--mal': resultados[idx] === 'mal',
            }"
          >
            <!-- Número de pregunta + estado -->
            <div class="ej-card-top">
              <span class="ej-num">{{ idx + 1 }}</span>
              <div class="ej-question">{{ ej.texto }}</div>
              <span v-if="resultados[idx] === 'ok'" class="badge badge-ok">✓ Correcto</span>
              <span v-else-if="resultados[idx] === 'mal'" class="badge badge-mal">✗ Intenta de nuevo</span>
            </div>

            <!-- Grids inside exercise -->
            <div class="ej-grids">
              <Blocks
                :key="ejKeys[idx]"
                :maxBloques="ej.nums.reduce((a,b)=>a+b,0) * 2"
                :maxPorGrupo="99"
                :showToolbar="false"
                :grids="getGridsForEj(ej)"
                :inline="true"
                :inlineColumns="2"
                :storageKey="`grid_ej_${idx}_${ejKeys[idx]}`"
                :noSnap="true"
                :showGridLabels="showGridLabels"
              />
            </div>

            <!-- Footer -->
            <div class="ej-footer">
              <button
                class="btn-ok"
                :disabled="resultados[idx] === 'ok'"
                @click="verificar(idx, ej)"
              >
                {{ resultados[idx] === 'ok' ? '¡Correcto! 🎉' : 'Verificar ✓' }}
              </button>
              <button class="btn-reset" @click="resetEj(idx)">↺</button>
            </div>
          </div>

          <!-- Mensaje final si todos correctos -->
          <div v-if="todosCorrecto" class="form-done">
            🏆 ¡Completaste todos los ejercicios!
          </div>

        </div>
      </div>
    </div>

    <!-- MODAL CONFIG -->
    <div v-if="modalAbierto" class="modal-backdrop" @click.self="modalAbierto = false">
      <div class="modal">
        <div class="modal-header">
          <strong>Configurar grids</strong>
          <button @click="modalAbierto = false">✕</button>
        </div>
        <div class="modal-body">
          <label>Toolbar <input type="checkbox" v-model="cfg.showToolbar" /></label>
          <label>Máx. bloques <input type="number" v-model.number="cfg.maxBloques" min="1" max="100" /></label>
          <hr />
          <div v-for="(g, i) in cfg.grids" :key="i" class="grid-card">
            <div class="grid-card-header">
              <b>Grid {{ i + 1 }}</b>
              <button v-if="cfg.grids.length > 1" @click="cfg.grids.splice(i,1)">✕</button>
            </div>
            <label>Label <input v-model="g.label" /></label>
            <div class="row">
              <label>Cols <input type="number" v-model.number="g.cols" min="3" max="20" /></label>
              <label>Rows <input type="number" v-model.number="g.rows" min="3" max="20" /></label>
            </div>
            <label>Posición
              <select v-model="g.position">
                <option value="top-left">↖ Arriba izq</option>
                <option value="top-right">↗ Arriba der</option>
                <option value="bottom-left">↙ Abajo izq</option>
                <option value="bottom-right">↘ Abajo der</option>
                <option value="center">⊙ Centro</option>
                <option value="left">← Izquierda</option>
                <option value="right">→ Derecha</option>
                <option value="top">↑ Arriba</option>
                <option value="bottom">↓ Abajo</option>
              </select>
            </label>
            <label>Bloques iniciales <small>(ej: 3,2)</small>
              <input v-model="g.initialBlocksInput" placeholder="3,2" />
            </label>
          </div>
          <button @click="cfg.grids.push(newGrid(cfg.grids.length))">+ Agregar grid</button>
        </div>
        <div class="modal-footer">
          <button @click="modalAbierto = false">Cancelar</button>
          <button class="btn-ok" @click="aplicar">Aplicar</button>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import Blocks from './components/Blocks.vue';

const modo = ref(null);
const modalAbierto = ref(false);

// ── CONFIG ──
const key = ref(0);

const newGrid = (i = 0) => ({
  label: `Grid ${i + 1}`,
  cols: 8, rows: 5,
  position: ['left','right','top','bottom'][i] ?? 'center',
  initialBlocksInput: '',
});

const cfg = ref({ showToolbar: true, maxBloques: 20, grids: [newGrid(0)] });

const parseBlocks = (s) =>
  (s||'').split(',').map(n=>parseInt(n.trim())).filter(n=>!isNaN(n)&&n>0);

const gridsActivos = ref([{ label:'Grid 1', cols:8, rows:5, position:'left', initialBlocks:[] }]);

const aplicar = () => {
  gridsActivos.value = cfg.value.grids.map(g => ({
    ...g, initialBlocks: parseBlocks(g.initialBlocksInput)
  }));
  key.value++;
  modalAbierto.value = false;
};

// ── EJERCICIOS ──
// Enunciados en palabras — los números se extraen automáticamente
const LISTA = [
  { texto: "María tiene 2 manzanas y Juan tiene 3. ¿Cuántas tienen en total?" },
  { texto: "En el patio hay 4 perros y llegan 2 más. ¿Cuántos perros hay ahora?" },
  { texto: "Ana juntó 3 piedras en el parque y 4 en la playa. ¿Cuántas piedras tiene?" },
];

// Extrae los números de un enunciado en orden de aparición
const parseNums = (texto) =>
  [...texto.matchAll(/\d+/g)].map(m => parseInt(m[0])).filter(n => n > 0 && n <= 20);

const lista     = ref(LISTA.map(e => ({ ...e, nums: parseNums(e.texto) })));
const ejKeys    = ref(LISTA.map((_, i) => i * 100));
const resultados = ref(LISTA.map(() => null));
const showGridLabels = ref(true);

const todosCorrecto = computed(() => resultados.value.every(r => r === 'ok'));

const getGridsForEj = (ej) => {
  const nums = ej.nums;
  const total = nums.reduce((a, b) => a + b, 0);
  const maxN = Math.max(...nums);
  return [
    // Un grid por cada número del enunciado
    ...nums.map(n => ({
      label: `${n}`,
      cols: n,
      rows: 1,
      position: 'left',
      initialBlocks: [n],
      isAnswer: false,
      showLabel: false,
      showCount: false,
    })),
    // Grid de respuesta
    {
      label: 'Respuesta',
      cols: Math.max(total, maxN * 2),
      rows: 2,
      position: 'right',
      initialBlocks: [],
      isAnswer: true,
      showLabel: true,
      showCount: false,
    },
  ];
};

const verificar = (idx, ej) => {
  const storageKey = `grid_ej_${idx}_${ejKeys.value[idx]}`;
  const raw = localStorage.getItem(storageKey);
  const bloques = raw ? JSON.parse(raw) : [];
  const answerGridId = ej.nums.length; // último grid = respuesta
  const enRespuesta = bloques.filter(b => b.gridId === answerGridId).length;
  const total = ej.nums.reduce((a, b) => a + b, 0);

  if (enRespuesta === total) {
    resultados.value[idx] = 'ok';
  } else {
    resultados.value[idx] = 'mal';
    setTimeout(() => {
      resultados.value[idx] = null;
      ejKeys.value[idx]++;
    }, 1000);
  }
};

const resetEj = (idx) => {
  resultados.value[idx] = null;
  ejKeys.value[idx]++;
};

const reiniciar = () => {
  resultados.value = LISTA.map(() => null);
  ejKeys.value = ejKeys.value.map(k => k + 1000);
};
</script>

<style scoped>
* { box-sizing: border-box; margin: 0; padding: 0; }

.app {
  width: 100%; height: 100vh;
  font-family: system-ui, sans-serif;
  font-size: 14px;
  background: #f0f4f8;
  display: flex; flex-direction: column;
}

.center {
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  height: 100vh; gap: 1rem;
}
.center h1, .center h2 { font-size: 1.5rem; }

.page { height: 100vh; display: flex; flex-direction: column; }

.bar {
  display: flex; align-items: center; gap: 0.5rem;
  padding: 0.5rem 0.75rem;
  background: white; border-bottom: 1px solid #ddd;
  font-weight: 600; color: #333; flex-shrink: 0;
  position: relative; z-index: 500;
}
.bar span { flex: 1; text-align: center; }

.preview { flex: 1; position: relative; overflow: hidden; }

/* ── FORM SCROLL ── */
.form-scroll {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem 1rem 3rem;
}

.form-inner {
  max-width: 860px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.form-header {
  background: white;
  border-radius: 12px;
  padding: 1.25rem 1.5rem;
  border-top: 6px solid #6366f1;
  box-shadow: 0 1px 6px rgba(0,0,0,0.08);
}
.form-header h2 { font-size: 1.25rem; color: #1e1b4b; margin-bottom: 0.3rem; }
.form-header p  { color: #666; font-size: 0.85rem; }

/* ── TARJETA DE EJERCICIO ── */
.ej-card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 1px 6px rgba(0,0,0,0.08);
  overflow: hidden;
  border: 2px solid transparent;
  transition: border-color 0.25s;
}
.ej-card--ok  { border-color: #10b981; }
.ej-card--mal { border-color: #ef4444; }

.ej-card-top {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.9rem 1.25rem 0;
}

.ej-num {
  width: 28px; height: 28px;
  border-radius: 50%;
  background: #6366f1;
  color: white;
  font-size: 0.8rem;
  font-weight: 700;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0;
}

.ej-question {
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
.badge-ok  { background: #d1fae5; color: #065f46; }
.badge-mal { background: #fee2e2; color: #991b1b; }


.ej-grids {
  min-height: 120px;
  padding: 8px 0;
  overflow: visible;
}

.ej-footer {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.25rem;
  border-top: 1px solid #f3f4f6;
}

.btn-reset {
  padding: 0.4rem 0.7rem;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  color: #6b7280;
}
.btn-reset:hover { background: #f9fafb; }

.form-done {
  text-align: center;
  font-size: 1.1rem;
  font-weight: 700;
  color: #065f46;
  background: #d1fae5;
  border-radius: 12px;
  padding: 1.5rem;
}

/* modal */
.modal-backdrop {
  position: fixed; inset: 0;
  background: rgba(0,0,0,0.4);
  display: flex; align-items: center; justify-content: center;
  z-index: 2000;
}
.modal {
  background: white; border-radius: 10px;
  width: 340px; max-height: 80vh;
  display: flex; flex-direction: column;
  box-shadow: 0 8px 32px rgba(0,0,0,0.2);
  overflow: hidden;
}
.modal-header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 0.75rem 1rem; border-bottom: 1px solid #eee;
}
.modal-body {
  padding: 0.75rem 1rem; overflow-y: auto; flex: 1;
  display: flex; flex-direction: column; gap: 0.5rem;
}
.modal-footer {
  display: flex; justify-content: flex-end; gap: 0.5rem;
  padding: 0.75rem 1rem; border-top: 1px solid #eee;
}
.modal-body label {
  display: flex; flex-direction: column; gap: 2px;
  font-size: 0.8rem; color: #555;
}
.modal-body label input, .modal-body label select {
  padding: 3px 6px; border: 1px solid #ccc; border-radius: 4px;
  font-size: 0.85rem; font-family: inherit;
}
.modal-body small { color: #999; font-size: 0.7rem; }
.modal-body hr { border: none; border-top: 1px solid #eee; }
.modal-body .row { display: grid; grid-template-columns: 1fr 1fr; gap: 0.5rem; }
.grid-card {
  border: 1px solid #e0e0e0; border-radius: 6px;
  padding: 0.5rem; display: flex; flex-direction: column; gap: 0.4rem;
  background: #fafafa;
}
.grid-card-header {
  display: flex; align-items: center; justify-content: space-between;
}
.grid-card-header b { font-size: 0.85rem; }

/* botones */
button {
  padding: 0.4rem 0.9rem;
  background: white; border: 1px solid #ccc; border-radius: 6px;
  font-size: 0.85rem; font-weight: 600; cursor: pointer;
  font-family: inherit; color: #333;
}
button:hover { background: #f0f0f0; }
button:disabled { opacity: 0.6; cursor: default; background: #d1fae5; border-color: #10b981; color: #065f46; }
.btn-ok { background: #10b981; border-color: #10b981; color: white; padding: 0.5rem 1.5rem; }
.btn-ok:hover:not(:disabled) { background: #059669; }
</style>