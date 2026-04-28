<template>
  <div class="app">

    <!-- Start screen -->
    <div v-if="!modo" class="center">
      <h1>🧱 Bloques</h1>
      <button @click="modo = 'exercise'">Ejercicios de suma</button>
    </div>

    <!-- Exercises -->
    <div v-else-if="modo === 'exercise'" class="page">
      <div class="bar">
        <button @click="modo = null">← Volver</button>
        <button @click="showGridLabels = !showGridLabels">
          {{ showGridLabels ? 'Ocultar números' : 'Mostrar números' }}
        </button>
        <button @click="reiniciar">↺ Reiniciar</button>
      </div>

      <div class="form-scroll">
        <div class="form-inner">

          <div v-for="(ex, idx) in lista" :key="idx" class="ex-card" :class="{
            'ex-card--ok': resultados[idx] === 'ok',
            'ex-card--mal': resultados[idx] === 'mal',
          }">
            <!-- Exercise header -->
            <div class="ex-card-top">
              <span class="ex-num">{{ idx + 1 }}</span>
              <div class="ex-question">{{ ex.texto }}</div>
              <span v-if="resultados[idx] === 'ok'" class="badge badge-ok">✓ Correcto</span>
              <span v-else-if="resultados[idx] === 'mal'" class="badge badge-mal">✗ Intenta de nuevo</span>
            </div>

            <!-- Grids -->
            <div class="ex-grids">
              <Blocks :key="exKeys[idx]" :config="{
                maxBloques: ex.answer * 2,
                showToolbar: false,
                inline: true,
                inlineColumns: 2,
                storageKey: `grid_ex_${idx}_${exKeys[idx]}`,
                noSnap: true,
                showGridLabels: showGridLabels,
                grids: getGridsForExercise(ex),
              }" />
            </div>

            <!-- Footer -->
            <div class="ex-footer">
              <button class="btn-ok" :disabled="resultados[idx] === 'ok'" @click="verify(idx, ex)">
                {{ resultados[idx] === 'ok' ? '¡Correcto! 🎉' : 'Verificar ✓' }}
              </button>
              <button class="btn-reset" @click="resetExercise(idx)">↺</button>
            </div>
          </div>

        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref } from 'vue';
import Blocks from './components/Blocks.vue';

const modo = ref(null);

// ── EXERCISES ──
// Each exercise declares: texto, op, nums, answer.
// op: 'add' | 'subtract' | 'multiply' | 'divide'
// answer: the expected number of blocks in the answer grid.
const statements = [
  { texto: "María tiene 2 manzanas y Juan tiene 3. ¿Cuántas tienen en total?",   op: 'add',      nums: [2, 3],   answer: 5  },
  { texto: "En el patio hay 4 perros y llegan 2 más. ¿Cuántos perros hay ahora?", op: 'add',      nums: [4, 2],   answer: 6  },
  { texto: "Ana juntó 3 piedras en el parque y 4 en la playa. ¿Cuántas piedras tiene?", op: 'add', nums: [3, 4],  answer: 7  },
];

const lista = ref(statements.map(e => ({ ...e })));
const exKeys = ref(statements.map((_, i) => i * 100));
const resultados = ref(statements.map(() => null));
const showGridLabels = ref(true);

// Build the grid config for a given exercise.
// One grid per operand + one answer grid on the right.
const getGridsForExercise = (ex) => {
  const { nums, answer } = ex;
  const maxN = Math.max(...nums);
  return [
    ...nums.map(n => ({
      label: `${n}`,
      cols: n,
      rows: 1,
      initialBlocks: [n],
      isAnswer: false,
      showLabel: false,
      showCount: false,
    })),
    {
      label: 'Respuesta',
      cols: Math.max(answer, maxN * 2),
      rows: 2,
      initialBlocks: [],
      isAnswer: true,
      showLabel: true,
      showCount: false,
    },
  ];
};

const verify = (idx, ex) => {
  const key = `grid_ex_${idx}_${exKeys.value[idx]}`;
  const raw = localStorage.getItem(key);
  const blocks = raw ? JSON.parse(raw) : [];
  const answerGridId = ex.nums.length; // last grid is always the answer
  const inAnswer = blocks.filter(b => b.gridId === answerGridId).length;

  if (inAnswer === ex.answer) {
    resultados.value[idx] = 'ok';
  } else {
    resultados.value[idx] = 'mal';
    setTimeout(() => {
      resultados.value[idx] = null;
      exKeys.value[idx]++;
    }, 1000);
  }
};

const resetExercise = (idx) => {
  resultados.value[idx] = null;
  exKeys.value[idx]++;
};

const reiniciar = () => {
  resultados.value = statements.map(() => null);
  exKeys.value = exKeys.value.map(k => k + 1000);
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
  position: relative;
  z-index: 500;
}

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

.ex-card--mal {
  border-color: #ef4444;
}

.ex-card-top {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.9rem 1.25rem 0;
}

.ex-num {
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

.badge-ok {
  background: #d1fae5;
  color: #065f46;
}

.badge-mal {
  background: #fee2e2;
  color: #991b1b;
}

.ex-grids {
  min-height: 120px;
  padding: 8px 0;
  overflow: visible;
}

.ex-footer {
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

.btn-reset:hover {
  background: #f9fafb;
}

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
  background: #d1fae5;
  border-color: #10b981;
  color: #065f46;
}

.btn-ok {
  background: #10b981;
  border-color: #10b981;
  color: white;
  padding: 0.5rem 1.5rem;
}

.btn-ok:hover:not(:disabled) {
  background: #059669;
}
</style>