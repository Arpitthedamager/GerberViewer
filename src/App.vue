<template>
  <div class="app-container">
    <!-- Top Section with Upload or Canvas -->
    <div class="top-section">
      <!-- Upload Area -->
      <div v-if="!layers.length" class="upload-area">
        <div class="upload-box">
          <h3>Choose Gerber/file or drag & drop here</h3>
          <p>File format should be</p>
          <p class="file-format">only accepted .zip or .rar file</p>
          <a-upload
            :custom-request="loadGerber"
            :show-upload-list="false"
            accept=".zip,.gbr,.GTL,.GBL,.GTO,.GBO,.GTS,.GBS,.GTP,.GBP,.DRL,.TXT,.GKO"
            class="upload-trigger">
            <a-button type="primary" :loading="loading">
              <template #icon><upload-outlined /></template>
              Open Gerber File
            </a-button>
          </a-upload>
        </div>
      </div>

      <!-- PCB Preview (shows after upload) -->
      <div v-else class="preview-container">
        <div class="canvas-wrapper">
          <gerber-view 
            :layers="layers" 
            :render="render"
            class="gerber-canvas" 
          />
        </div>
      </div>
    </div>

    <!-- Layer Controls (below canvas) -->
    <div v-if="layers.length" class="layer-controls-container">
      <div class="layer-controls-wrapper">
        <a-button 
          type="primary" 
          :class="['both-sides-button', { 'active': render.showBothSides }]" 
          @click="toggleBothSides"
        >
          <template #icon><swap-outlined /></template>
          {{ render.showBothSides ? 'Single Side' : 'Both Sides' }}
        </a-button>
        <a-button 
          class="layer-settings" 
          @click="showLayerSettings = true"
        >
          <template #icon><setting-outlined /></template>
          Layer Settings
        </a-button>
      </div>
    </div>

    <!-- PCB Details Section (Always visible) -->
    <div class="content-container">
      <div class="pcb-details">
        <div class="detail-section">
          <div class="section-row">
            <label>Dimensions</label>
            <div class="dimensions-input">
              <a-input-number 
                v-model:value="pcbDimensions.width" 
                :min="0" 
                :disabled="!layers.length"
                placeholder="Length"
              /> x
              <a-input-number 
                v-model:value="pcbDimensions.height" 
                :min="0" 
                :disabled="!layers.length"
                placeholder="Width"
              />
              <a-select 
                v-model:value="dimensionUnit" 
                style="width: 70px"
                :disabled="!layers.length"
              >
                <a-select-option value="mm">mm</a-select-option>
                <a-select-option value="in">in</a-select-option>
              </a-select>
            </div>
          </div>

          <div class="section-row">
            <label>File Name</label>
            <span class="info-value">{{ gerber?.name || '-' }}</span>
          </div>

          <div class="section-row">
            <label>Total Layers</label>
            <span class="info-value">{{ layers.length || '-' }}</span>
          </div>

          <div class="section-row">
            <label>Copper Layers</label>
            <span class="info-value">{{ copperLayers.length || '-' }}</span>
          </div>

          <div class="section-row">
            <label>Mask Layers</label>
            <span class="info-value">{{ maskLayers.length || '-' }}</span>
          </div>

          <div class="section-row">
            <label>Number of Pads</label>
            <span class="info-value">{{ padCount || '-' }}</span>
          </div>
        </div>

        <div class="output-section">
          <h3>Output:</h3>
          <div class="section-row">
            <label>Format</label>
            <div class="format-options">
              <a-radio-group v-model:value="outputFormat" :disabled="!layers.length">
                <a-radio-button value="svg">SVG</a-radio-button>
                <a-radio-button value="png">PNG</a-radio-button>
              </a-radio-group>
            </div>
          </div>

          <div class="section-row">
            <label>Layering</label>
            <a-checkbox v-model:checked="outputLayered" :disabled="!layers.length">
              Output Layered files
            </a-checkbox>
          </div>

          <div class="section-row">
            <label>Relief</label>
            <a-checkbox v-model:checked="exportRelief" :disabled="!layers.length">
              Export relief texture
            </a-checkbox>
          </div>

          <a-button 
            type="primary" 
            class="output-button"
            :disabled="!layers.length"
          >
            Output File
          </a-button>
        </div>
      </div>
    </div>

    <!-- Layer Settings Modal -->
    <a-modal
      v-model:visible="showLayerSettings"
      title="Layer Settings"
      width="600px"
      @ok="applyLayerSettings"
    >
      <div class="layer-settings-content">
        <div v-for="layer in layers" :key="layer.filename" class="layer-item">
          <div class="layer-header">
            <a-checkbox v-model:checked="layer.visible">
              {{ layer.filename }}
            </a-checkbox>
            <span class="layer-type">{{ layer.type }}</span>
          </div>
          <div class="layer-controls" v-if="layer.visible">
            <div class="color-picker" v-if="layer.type === 'copper' || layer.type === 'soldermask'">
              <span>Color:</span>
              <a-select v-model:value="layer.color" style="width: 100px">
                <a-select-option value="green">Green</a-select-option>
                <a-select-option value="red">Red</a-select-option>
                <a-select-option value="blue">Blue</a-select-option>
                <a-select-option value="black">Black</a-select-option>
              </a-select>
            </div>
            <div class="opacity-slider">
              <span>Opacity:</span>
              <a-slider v-model:value="layer.opacity" :min="0" :max="100" :step="1" style="width: 150px" />
            </div>
          </div>
        </div>
      </div>
    </a-modal>
  </div>
</template>

<script lang="ts" setup>
import type { InputLayer } from 'pcb-stackup';
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick } from 'vue';
import { InfoCircleOutlined, UploadOutlined, EyeOutlined, SwapOutlined, SettingOutlined } from '@ant-design/icons-vue';

import XPanelContainer from '@/components/XPanelContainer.vue';
import XPanel from '@/components/XPanel.vue';
import GerberView from '@/components/GerberView.vue';
import Gerber3DView from '@/components/Gerber3DView.vue';

import LayersPanel from '@/panels/LayersPanel.vue';
import RenderPanel from '@/panels/RenderPanel.vue';
import OutputPanel from '@/panels/OutputPanel.vue';

import { loadLayers, renderStack, type RenderOptions } from '@/utils/gerber';

// Define extended layer interface with our additional properties
interface ExtendedLayer extends InputLayer {
  visible: boolean;
  opacity: number;
  color: string;
  svg?: string;
}

// Define render options interface
interface ExtendedRenderOptions {
  side: 'top' | 'bottom';
  sm: string;
  cf: string;
  sp: boolean;
  view3d: boolean;
  showBothSides: boolean;
  dimensions?: { width: number; height: number };
  layers?: Array<{
    filename: string;
    visible: boolean;
    opacity: number;
    color: string;
  }>;
}

const loading = ref(false);
const gerber = ref<File>();
const layers = ref<ExtendedLayer[]>([]);
const render = ref<ExtendedRenderOptions>({
  side: 'top',
  sm: 'green',
  cf: 'none',
  sp: false,
  view3d: false,
  showBothSides: false
});

// PCB Dimensions
const pcbDimensions = ref({ width: 100, height: 70 });

// Layer counts
const copperLayers = computed(() => layers.value.filter(layer => layer.type === 'copper'));
const maskLayers = computed(() => layers.value.filter(layer => layer.type === 'soldermask'));

const canvasTop = ref(0);
function handleResize(height: number): void {
  canvasTop.value = height;
}

// Watch for render option changes
watch(render, async (newValue) => {
  console.log('Render options changed in App:', newValue);
  if (layers.value.length > 0) {
    try {
      // Force re-render of the PCB view
      const stack = await renderStack(layers.value, newValue);
      console.log('Stack updated with new render options');
      
      // Update layers with SVG content
      layers.value = layers.value.map(layer => {
        const side = layer.side === 'top' ? stack.top : stack.bottom;
        if (side && layer.type === 'copper' || layer.type === 'soldermask') {
          return {
            ...layer,
            svg: side.svg
          };
        }
        return layer;
      });

      // Update PCB dimensions from the stackup
      if (stack.top) {
        const width = stack.top.width ? parseFloat(stack.top.width.toString()) || 150 : 150;
        const height = stack.top.height ? parseFloat(stack.top.height.toString()) || 100 : 100;
        render.value.dimensions = { width, height };
      }
    } catch (error) {
      console.error('Error updating render:', error);
    }
  }
}, { deep: true });

const showLayerSettings = ref(false);

// Add visible and opacity properties to layers when loading
function processLayer(layer: InputLayer): ExtendedLayer {
  return {
    ...layer,
    visible: true,
    opacity: 100,
    color: layer.type === 'soldermask' ? 'green' : 'copper'
  };
}

async function loadGerber({ file }: { file: File }): Promise<void> {
  try {
    loading.value = true;
    gerber.value = file;
    const loadedLayers = await loadLayers(file);
    layers.value = loadedLayers.map(processLayer);
    
    // Initial render
    const stack = await renderStack(layers.value, render.value);
    
    // Update layers with SVG content
    layers.value = layers.value.map(layer => {
      const side = layer.side === 'top' ? stack.top : stack.bottom;
      if (side && (layer.type === 'copper' || layer.type === 'soldermask')) {
        return {
          ...layer,
          svg: side.svg
        };
      }
      return layer;
    });

    // Set PCB dimensions
    if (stack.top) {
      const width = stack.top.width ? parseFloat(stack.top.width.toString()) || 150 : 150;
      const height = stack.top.height ? parseFloat(stack.top.height.toString()) || 100 : 100;
      pcbDimensions.value = { width, height };
    }
  } catch (error) {
    console.error('Error loading file:', error);
  } finally {
    loading.value = false;
  }
}

function calculatePCBDimensions(): void {
  const layerCount = layers.value.length;
  
  // Simple heuristic: more layers = larger board
  if (layerCount > 10) {
    pcbDimensions.value = { width: 150, height: 100 };
  } else if (layerCount > 5) {
    pcbDimensions.value = { width: 120, height: 80 };
  } else {
    pcbDimensions.value = { width: 100, height: 70 };
  }
}

function toggle3DView(): void {
  render.value.view3d = !render.value.view3d;
  // Force a redraw after mode switch
  nextTick(() => {
    handleCanvasResize();
  });
}

function toggleBothSides(): void {
  render.value.showBothSides = !render.value.showBothSides;
}

// Register components
const components = {
  'gerber-view': GerberView,
  'gerber-3d-view': Gerber3DView,
  'x-panel-container': XPanelContainer,
  'x-panel': XPanel,
  'layers-panel': LayersPanel,
  'render-panel': RenderPanel,
  'output-panel': OutputPanel
};

const dimensionUnit = ref('mm');
const outputFormat = ref('svg');
const outputLayered = ref(true);
const exportRelief = ref(false);
const padCount = computed(() => 35); // Replace with actual pad count calculation

// Add resize handling
const handleCanvasResize = () => {
  // Force canvas redraw when container size changes
  if (render.value) {
    const temp = { ...render.value };
    render.value = temp;
  }
};

// Setup resize observer
onMounted(() => {
  window.addEventListener('resize', handleCanvasResize);
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleCanvasResize);
});

function applyLayerSettings() {
  render.value = {
    ...render.value,
    layers: layers.value.map(layer => ({
      filename: layer.filename || '',
      visible: layer.visible,
      opacity: layer.opacity / 100,
      color: layer.color
    }))
  };
  showLayerSettings.value = false;
}

// Watch for layer setting changes
watch(() => layers.value, async (newLayers) => {
  if (newLayers.length > 0) {
    try {
      const stack = await renderStack(newLayers, render.value);
      layers.value = newLayers.map(layer => {
        const side = layer.side === 'top' ? stack.top : stack.bottom;
        if (side && (layer.type === 'copper' || layer.type === 'soldermask')) {
          return {
            ...layer,
            svg: side.svg
          };
        }
        return layer;
      });
    } catch (error) {
      console.error('Error updating layers:', error);
    }
  }
}, { deep: true });
</script>

<style lang="scss">
.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: #f0f2f5;
  overflow: hidden;
}

.top-section {
  height: 300px;
  background: #fff;
  border-bottom: 1px solid #e8e8e8;
  padding: 20px;
  overflow: hidden;
}

.upload-area {
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  
  .upload-box {
    width: 100%;
    max-width: 600px;
    height: 100%;
    background: #fff;
    border: 2px dashed #d9d9d9;
    border-radius: 8px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    text-align: center;
    
    h3 {
      font-size: 18px;
      color: #262626;
      margin-bottom: 8px;
    }
    
    p {
      margin: 4px 0;
      color: #8c8c8c;
      
      &.file-format {
        color: #bfbfbf;
      }
    }
  }
}

.preview-container {
  position: relative;
  height: 100%;
  width: 100%;
}

.canvas-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.gerber-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.preview-controls {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  gap: 8px;
  z-index: 10;
}

.content-container {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
  min-height: 0;
}

.pcb-details {
  max-width: 800px;
  margin: 0 auto;
  background: #fff;
  border-radius: 8px;
  padding: 24px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);

  .detail-section {
    margin-bottom: 32px;
  }

  .section-row {
    display: flex;
    align-items: center;
    margin-bottom: 16px;
    
    label {
      width: 140px;
      color: #262626;
      font-weight: 500;
    }

    .info-value {
      padding: 4px 12px;
      background: #f5f5f5;
      border-radius: 4px;
      color: #595959;
      min-width: 60px;
      text-align: center;
    }
  }

  .dimensions-input {
    display: flex;
    align-items: center;
    gap: 8px;
    
    .ant-input-number {
      width: 100px;
    }
  }

  .output-section {
    border-top: 1px solid #f0f0f0;
    padding-top: 24px;

    h3 {
      margin-bottom: 16px;
      color: #262626;
    }
  }

  .output-button {
    margin-top: 24px;
    width: 100%;
    height: 40px;
  }
}

.view3d-button, .both-sides-button {
  &.active {
    background: #1890ff;
    border-color: #1890ff;
    color: white;
  }
}

.layer-settings {
  background: white;
  border: 1px solid #d9d9d9;
}

.layer-settings-content {
  max-height: 400px;
  overflow-y: auto;
  padding: 0 16px;

  .layer-item {
    border-bottom: 1px solid #f0f0f0;
    padding: 12px 0;

    &:last-child {
      border-bottom: none;
    }

    .layer-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 8px;

      .layer-type {
        color: #8c8c8c;
        font-size: 12px;
        padding: 2px 8px;
        background: #f5f5f5;
        border-radius: 4px;
      }
    }

    .layer-controls {
      display: flex;
      gap: 16px;
      margin-top: 8px;
      padding-left: 24px;

      .color-picker, .opacity-slider {
        display: flex;
        align-items: center;
        gap: 8px;
      }
    }
  }
}

.layer-controls-container {
  background: #fff;
  border-bottom: 1px solid #e8e8e8;
  padding: 12px 20px;
}

.layer-controls-wrapper {
  display: flex;
  justify-content: center;
  gap: 12px;
  max-width: 800px;
  margin: 0 auto;
}
</style>
