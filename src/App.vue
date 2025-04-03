<template>
  <x-panel-container @resize="handleResize">
    <template #extra>
      <a-row :gutter="[8]">
        <a-col>
          <a-upload
            :custom-request="loadGerber"
            :show-upload-list="false"
            accept=".zip,.gbr,.GTL,.GBL,.GTO,.GBO,.GTS,.GBS,.GTP,.GBP,.DRL,.TXT,.GKO">
            <a-button type="primary" :loading="loading" class="upload-button">
              <template #icon><upload-outlined /></template>
              Open Gerber File
            </a-button>
          </a-upload>
        </a-col>
      </a-row>
    </template>
    
    <!-- PCB Info Panel -->
    <div v-if="layers.length > 0" class="pcb-info">
      <div class="info-header">
        <div class="header-left">
          <span class="info-title">PCB Details</span>
          <a-tag color="blue" class="status-tag">Loaded</a-tag>
        </div>
        <a-tooltip title="PCB information">
          <info-circle-outlined class="info-icon" />
        </a-tooltip>
      </div>
      <div class="info-content">
        <div class="info-section">
          <div class="section-title">
            <span>Board Specifications</span>
            <a-tooltip title="Physical dimensions of the PCB">
              <info-circle-outlined class="section-icon" />
            </a-tooltip>
          </div>
          <div class="info-item highlight">
            <span class="info-label">File Name</span>
            <span class="info-value">{{ gerber?.name }}</span>
          </div>
          <div class="info-item highlight">
            <span class="info-label">Dimensions</span>
            <span class="info-value">{{ pcbDimensions.width }}mm × {{ pcbDimensions.height }}mm</span>
          </div>
        </div>
        
        <div class="info-section">
          <div class="section-title">
            <span>Layer Stack</span>
            <a-tooltip title="PCB layer configuration">
              <info-circle-outlined class="section-icon" />
            </a-tooltip>
          </div>
          <div class="info-item">
            <span class="info-label">Total Layers</span>
            <span class="info-value">{{ layers.length }}</span>
          </div>
          <div class="info-item">
            <span class="info-label">Copper Layers</span>
            <span class="info-value copper">{{ copperLayers.length }}</span>
          </div>
          <div class="info-item">
            <span class="info-label">Mask Layers</span>
            <span class="info-value mask">{{ maskLayers.length }}</span>
          </div>
        </div>
        
        <div class="info-section">
          <div class="section-title">
            <span>View Settings</span>
            <a-tooltip title="Current view configuration">
              <info-circle-outlined class="section-icon" />
            </a-tooltip>
          </div>
          <div class="info-item">
            <span class="info-label">View Mode</span>
            <span class="info-value">{{ render.view3d ? '3D' : '2D' }}</span>
          </div>
          <div class="info-item">
            <span class="info-label">Side</span>
            <span class="info-value">{{ render.side }}</span>
          </div>
          <div class="info-item">
            <span class="info-label">Mask Color</span>
            <span class="info-value" :style="{ color: render.sm }">{{ render.sm }}</span>
          </div>
        </div>
      </div>
    </div>
    
    <a-tab-pane key="options" tab="Options">
      <x-panel>
        <render-panel v-model:render="render" />
        <layers-panel v-model:layers="layers" />
      </x-panel>
    </a-tab-pane>
    <a-tab-pane key="output" tab="Output">
      <output-panel :gerber="gerber" :layers="layers" :render="render" />
    </a-tab-pane>
  </x-panel-container>
  <gerber-view :layers="layers" :render="render" :style="{ top: `${canvasTop}px` }" />
</template>

<script lang="ts" setup>
import type { InputLayer } from 'pcb-stackup';
import { ref, computed, watch } from 'vue';
import { InfoCircleOutlined, UploadOutlined } from '@ant-design/icons-vue';

import XPanelContainer from '@/components/XPanelContainer.vue';
import XPanel from '@/components/XPanel.vue';
import GerberView from '@/components/GerberView.vue';

import LayersPanel from '@/panels/LayersPanel.vue';
import RenderPanel from '@/panels/RenderPanel.vue';
import OutputPanel from '@/panels/OutputPanel.vue';

import { loadLayers, renderStack, type RenderOptions } from '@/utils/gerber';

const loading = ref(false);
const gerber = ref<File | null>(null);
const layers = ref<InputLayer[]>([]);
const render = ref<RenderOptions>({
  side: 'top',
  sm: 'green',
  cf: 'none',
  sp: false
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
    } catch (error) {
      console.error('Error updating render:', error);
    }
  }
}, { deep: true });

async function loadGerber({ file }: { file: File }): Promise<void> {
  try {
    loading.value = true;
    gerber.value = file;
    layers.value = await loadLayers(file);
    calculatePCBDimensions();
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
</script>

<style lang="scss">
html,
body {
  overscroll-behavior-x: none;
}

body,
.x-panel-container {
  background: #263238;
}

.view3d-button {
  border: 1px solid #1890ff;
  font-weight: 500;
}

.pcb-info {
  position: fixed;
  top: 60px;
  right: 20px;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  color: white;
  z-index: 1000;
  width: 280px;
  overflow: hidden;
  transition: all 0.3s ease;
  border: 1px solid rgba(255, 255, 255, 0.1);
  
  &:hover {
    box-shadow: 0 6px 24px rgba(0, 0, 0, 0.4);
    transform: translateY(-2px);
  }
  
  .info-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 16px;
    background: rgba(0, 0, 0, 0.3);
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  }
  
  .header-left {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  
  .info-title {
    font-size: 16px;
    font-weight: 600;
    letter-spacing: 0.5px;
  }
  
  .status-tag {
    font-size: 12px;
    padding: 0 6px;
    height: 20px;
    line-height: 18px;
    border-radius: 4px;
  }
  
  .info-icon, .section-icon {
    color: #1890ff;
    font-size: 16px;
    cursor: pointer;
    transition: transform 0.2s ease;
    
    &:hover {
      transform: scale(1.1);
    }
  }
  
  .info-content {
    padding: 12px 16px;
  }
  
  .info-section {
    margin-bottom: 16px;
    
    &:last-child {
      margin-bottom: 0;
    }
  }
  
  .section-title {
    display: flex;
    align-items: center;
    gap: 6px;
    margin-bottom: 8px;
    color: #1890ff;
    font-size: 14px;
    font-weight: 500;
  }
  
  .info-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 0;
    border-bottom: 1px solid rgba(255, 255, 255, 0.05);
    
    &:last-child {
      border-bottom: none;
    }
    
    &.highlight {
      background: rgba(24, 144, 255, 0.1);
      margin: 0 -16px;
      padding: 8px 16px;
      border-bottom: 1px solid rgba(24, 144, 255, 0.2);
    }
  }
  
  .info-label {
    color: #a0a0a0;
    font-size: 14px;
  }
  
  .info-value {
    font-weight: 500;
    color: #ffffff;
    background: rgba(24, 144, 255, 0.2);
    padding: 4px 8px;
    border-radius: 4px;
    min-width: 60px;
    text-align: center;
    
    &.copper {
      background: rgba(255, 193, 7, 0.2);
      color: #ffc107;
    }
    
    &.mask {
      background: rgba(76, 175, 80, 0.2);
      color: #4caf50;
    }
  }
}

.upload-buttons {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.upload-button {
  min-width: 120px;
}

.info-section h3 {
  margin: 10px 0;
  color: #1890ff;
}
</style>
