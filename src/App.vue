<template>
  <div class="app-container">
    <!-- Página de Configuración inicial -->
    <div v-if="!gameStarted" class="config-page">
      <div class="config-card">
        <h1 class="config-title"> Sistema de bloques</h1>
        
        <form @submit.prevent="startGame" class="config-form">
          <div class="form-row">
            <div class="form-group">
              <label>Columnas</label>
              <input 
                type="number" 
                v-model.number="config.gridColumns"
                min="5"
                max="30"
              />
            </div>
            <div class="form-group">
              <label>Filas</label>
              <input 
                type="number" 
                v-model.number="config.gridRows"
                min="5"
                max="30"
              />
            </div>
          </div>

          <div class="form-row">
            <div class="form-group">
              <label>Máx. Bloques</label>
              <input 
                type="number" 
                v-model.number="config.maxBloques"
                min="1"
                max="40"
              />
            </div>
            <div class="form-group">
              <label>Máx. x Grupo</label>
              <input 
                type="number" 
                v-model.number="config.maxUnitsPerGroup"
                min="1"
                max="40"
              />
            </div>
          </div>

          <div class="form-group-full">
            <label>Bloques Iniciales <span class="optional">(opcional)</span></label>
            <input 
              type="text" 
              v-model="initialBlocksInput"
              placeholder="3,5,2"
            />
            <span class="hint">Ej: "3,5,2" = 10 bloques</span>
          </div>

          <button type="submit" class="btn-start">
            Iniciar
          </button>
        </form>
      </div>
    </div>

    <!-- Juego activo -->
    <div v-else class="game-page">
      <!-- Botón para abrir el modal de parámetros -->
      <button class="btn-params" @click="openParamsModal" title="Configuración">
        ⚙️
      </button>

      <!-- Componente Blocks — key fuerza remount al aplicar nuevos parámetros -->
      <Blocks 
        :key="gridKey"
        :gridColumns="activeConfig.gridColumns"
        :gridRows="activeConfig.gridRows"
        :maxBloques="activeConfig.maxBloques"
        :maxPorGrupo="activeConfig.maxUnitsPerGroup"
        :initialBlocks="activeInitialBlocks"
      />
    </div>

    <!-- Modal de Parámetros -->
    <Transition name="modal">
      <div v-if="showParamsModal" class="modal-backdrop" @click.self="closeParamsModal">
        <div class="modal-card">
          <div class="modal-header">
            <h2 class="modal-title">⚙️ Configuración de Grid</h2>
            <button class="modal-close" @click="closeParamsModal">×</button>
          </div>

          <div class="modal-body">
            <div class="form-row">
              <div class="form-group">
                <label>Columnas</label>
                <input 
                  type="number" 
                  v-model.number="draftConfig.gridColumns"
                  min="5"
                  max="30"
                />
              </div>
              <div class="form-group">
                <label>Filas</label>
                <input 
                  type="number" 
                  v-model.number="draftConfig.gridRows"
                  min="5"
                  max="30"
                />
              </div>
            </div>

            <div class="form-row">
              <div class="form-group">
                <label>Maximo de Bloques</label>
                <input 
                  type="number" 
                  v-model.number="draftConfig.maxBloques"
                  min="1"
                  max="40"
                />
              </div>
              <div class="form-group">
                <label>Maximo por grupo</label>
                <input 
                  type="number" 
                  v-model.number="draftConfig.maxUnitsPerGroup"
                  min="1"
                  max="40"
                />
              </div>
            </div>

            <div class="form-group-full">
              <label>Bloques Iniciales</label>
              <input 
                type="text" 
                v-model="draftInitialBlocksInput"
                placeholder="3,5,2"
              />
              <span class="hint">Ej: "3,5,2" = 10 bloques </span>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-cancel" @click="closeParamsModal">Cancelar</button>
            <button class="btn-apply" @click="applyParams">Aplicar y Recargar</button>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import Blocks from './components/Blocks.vue';

const gameStarted = ref(false);
const initialBlocksInput = ref('');
const showParamsModal = ref(false);

// Configuración activa (la que usa el grid)
const config = ref({
  gridColumns: 15,
  gridRows: 10,
  maxBloques: 20,
  maxUnitsPerGroup: 9,
});

const activeConfig = ref({ ...config.value });
const activeInitialBlocks = ref([]);

// Borrador para el modal (se edita sin afectar el juego hasta Aplicar)
const draftConfig = ref({ ...config.value });
const draftInitialBlocksInput = ref('');

// key que fuerza remount de Blocks cuando se aplican nuevos parámetros
const gridKey = ref(0);

const parsedInitialBlocks = computed(() => {
  if (!initialBlocksInput.value.trim()) return [];
  return initialBlocksInput.value
    .split(',')
    .map(n => parseInt(n.trim()))
    .filter(n => !isNaN(n) && n > 0);
});

const parseDraftInitialBlocks = () => {
  if (!draftInitialBlocksInput.value.trim()) return [];
  return draftInitialBlocksInput.value
    .split(',')
    .map(n => parseInt(n.trim()))
    .filter(n => !isNaN(n) && n > 0);
};

const startGame = () => {
  activeConfig.value = { ...config.value };
  activeInitialBlocks.value = parsedInitialBlocks.value;
  gameStarted.value = true;
};

const openParamsModal = () => {
  // Cargar valores actuales en el borrador
  draftConfig.value = { ...activeConfig.value };
  draftInitialBlocksInput.value = '';
  showParamsModal.value = true;
};

const closeParamsModal = () => {
  showParamsModal.value = false;
};

const applyParams = () => {
  activeConfig.value = { ...draftConfig.value };
  activeInitialBlocks.value = parseDraftInitialBlocks();
  showParamsModal.value = false;
  // Forzar remount del grid con la nueva configuración
  gridKey.value++;
};
</script>

<style scoped>
* {
  box-sizing: border-box;
}

.app-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
}

/* ===== PÁGINA DE CONFIGURACIÓN INICIAL ===== */
.config-page {
  width: 100%;
  height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #4e66bc 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}

.game-page {
  width: 100%;
  height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #4e66bc 100%);
  position: relative;
}

.config-card {
  background: white;
  border-radius: 16px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
  padding: 2rem;
  max-width: 400px;
  width: 100%;
}

.config-title {
  font-size: 1.75rem;
  font-weight: 700;
  color: #1e293b;
  margin: 0 0 1.5rem 0;
  text-align: center;
}

.config-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.75rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.form-group-full {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.form-group label,
.form-group-full label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
}

.optional {
  font-size: 0.75rem;
  font-weight: 400;
  color: #94a3b8;
}

.form-group input,
.form-group-full input {
  padding: 0.625rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  transition: border-color 0.2s;
  font-family: inherit;
}

.form-group input:focus,
.form-group-full input:focus {
  outline: none;
  border-color: #667eea;
}

.hint {
  font-size: 0.75rem;
  color: #94a3b8;
}

.btn-start {
  padding: 0.875rem;
  background: linear-gradient(135deg, #69ae60 0%, #69ae60 100%);
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  margin-top: 0.5rem;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
}

.btn-start:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(102, 126, 234, 0.4);
}

.btn-start:active {
  transform: translateY(0);
}

/* ===== BOTÓN FLOTANTE DE PARÁMETROS ===== */
.btn-params {
  position: fixed;
  top: 1rem;
  left: 1rem;
  z-index: 3000;
  width: 42px;
  height: 42px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  font-size: 1.2rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.12);
  transition: all 0.15s;
}

.btn-params:hover {
  background: #f3f4f6;
  transform: scale(1.08);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
}

/* ===== MODAL ===== */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.55);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 5000;
  padding: 1rem;
  backdrop-filter: blur(2px);
}

.modal-card {
  background: white;
  border-radius: 16px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.25);
  width: 100%;
  max-width: 400px;
  overflow: hidden;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.25rem 1.5rem 0.75rem;
  border-bottom: 1px solid #f1f5f9;
}

.modal-title {
  font-size: 1.15rem;
  font-weight: 700;
  color: #1e293b;
  margin: 0;
}

.modal-close {
  width: 28px;
  height: 28px;
  background: #f1f5f9;
  border: none;
  border-radius: 6px;
  font-size: 1.1rem;
  color: #64748b;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s;
  line-height: 1;
}

.modal-close:hover {
  background: #e2e8f0;
  color: #1e293b;
}

.modal-body {
  padding: 1.25rem 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.modal-footer {
  display: flex;
  gap: 0.75rem;
  padding: 0.75rem 1.5rem 1.25rem;
  border-top: 1px solid #f1f5f9;
}

.btn-cancel {
  flex: 1;
  padding: 0.75rem;
  background: #f1f5f9;
  border: none;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: 600;
  color: #64748b;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-cancel:hover {
  background: #e2e8f0;
}

.btn-apply {
  flex: 2;
  padding: 0.75rem;
  background: linear-gradient(135deg, #667eea 0%, #4e66bc 100%);
  border: none;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: 600;
  color: white;
  cursor: pointer;
  transition: all 0.15s;
  box-shadow: 0 3px 10px rgba(102, 126, 234, 0.3);
}

.btn-apply:hover {
  transform: translateY(-1px);
  box-shadow: 0 5px 14px rgba(102, 126, 234, 0.4);
}

.btn-apply:active {
  transform: translateY(0);
}

/* ===== TRANSICIÓN MODAL ===== */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-active .modal-card,
.modal-leave-active .modal-card {
  transition: transform 0.2s ease, opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-from .modal-card,
.modal-leave-to .modal-card {
  transform: scale(0.94) translateY(-8px);
  opacity: 0;
}

/* ===== RESPONSIVE ===== */
@media (max-width: 480px) {
  .config-card {
    padding: 1.5rem;
  }
  
  .config-title {
    font-size: 1.5rem;
  }

  .modal-card {
    max-width: 100%;
  }
}
</style>