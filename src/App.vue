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

            <!-- Grids -->
            <div class="ex-grids">
              <Blocks :key="keys[idx]" :config="buildConfig(ex, idx)" />
            </div>

            <!-- Footer -->
            <div class="ex-footer">
              <button class="btn-verify" :disabled="results[idx] === 'ok'" @click="verify(idx, ex)">
                {{ results[idx] === 'ok' ? 'Correcto!' : 'Check' }}
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

// ── SCREEN ──
const screen = ref(null);

// ── EXERCISES ──
// Each exercise declares: text, nums (operands), answer (expected block count).
const exercises = [
  { text: "María tiene 2 manzanas y Juan tiene 3. ¿Cuántas tienen en total?", nums: [2, 3], answer: 5 },
  { text: "En el patio hay 4 perros y llegan 2 más. ¿Cuántos perros hay ahora?", nums: [4, 2], answer: 6 },
  { text: "Ana juntó 3 piedras en el parque y 4 en la playa. ¿Cuántas piedras tiene?", nums: [3, 4], answer: 7 },
];

const keys = ref(exercises.map((_, i) => i * 100));
const results = ref(exercises.map(() => null));
const showLabels = ref(true);

// Build the Blocks config for a given exercise.
// One grid per operand (pre-filled) + one empty answer grid.
const buildConfig = (ex, idx) => ({
  inline: true,
  inlineColumns: 2,
  noSnap: true,
  showToolbar: false,
  showGridLabels: showLabels.value,
  maxBloques: ex.answer * 2,
  storageKey: `ex_${idx}_${keys.value[idx]}`,
  grids: [
    ...ex.nums.map(n => ({
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
      cols: Math.max(ex.answer, Math.max(...ex.nums) * 2),
      rows: 2,
      initialBlocks: [],
      isAnswer: true,
      showLabel: true,
      showCount: false,
    },
  ],
});

// Count blocks in the answer grid and compare to expected answer.
const verify = (idx, ex) => {
  const raw = localStorage.getItem(`ex_${idx}_${keys.value[idx]}`);
  const blocks = raw ? JSON.parse(raw) : [];
  const answerGridId = ex.nums.length; // answer grid is always the last one
  const inAnswer = blocks.filter(b => b.gridId === answerGridId).length;

  if (inAnswer === ex.answer) {
    results.value[idx] = 'ok';
  } else {
    results.value[idx] = 'bad';
    setTimeout(() => {
      results.value[idx] = null;
      keys.value[idx]++;
    }, 1000);
  }
};

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

.btn-verify {
  background: #10b981;
  border-color: #10b981;
  color: white;
  padding: 0.5rem 1.5rem;
}

.btn-verify:hover:not(:disabled) {
  background: #059669;
}

.btn-verify:disabled {
  background: #d1fae5;
  border-color: #10b981;
  color: #065f46;
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