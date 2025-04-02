<template>
  <x-panel-container @resize="handleResize">
    <template #extra>
      <a-row :gutter="[8]">
        <a-col>
          <a-upload
            :custom-request="loadGerber"
            :show-upload-list="false"
            accept=".zip,application/zip">
            <a-button type="primary" :loading="loading">Open Gerber File</a-button>
          </a-upload>
        </a-col>
        <a-col>
          <a-button 
            :type="render.view3d ? 'primary' : 'default'" 
            @click="toggleView3D" 
            :disabled="!gerber || layers.length === 0"
            class="view3d-button">
            <template #icon><experiment-outlined /></template>
            3D View
          </a-button>
        </a-col>
      </a-row>
    </template>
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
  <gerber-view v-if="!render.view3d" :layers="layers" :render="render" :style="{ top: `${canvasTop}px` }" />
  <gerber-3d-view v-else :layers="layers" :render="render" :style="{ top: `${canvasTop}px` }" />
</template>

<script lang="ts" setup>
import type { InputLayer } from 'pcb-stackup';
import { ref } from 'vue';
import { ExperimentOutlined } from '@ant-design/icons-vue';

import XPanelContainer from '@/components/XPanelContainer.vue';
import XPanel from '@/components/XPanel.vue';
import GerberView from '@/components/GerberView.vue';
import Gerber3DView from '@/components/Gerber3DView.vue';

import LayersPanel from '@/panels/LayersPanel.vue';
import RenderPanel from '@/panels/RenderPanel.vue';
import OutputPanel from '@/panels/OutputPanel.vue';

import { loadLayers, type RenderOptions } from '@/utils/gerber';

const gerber = ref<File>();

const layers = ref<InputLayer[]>([]);

const render = ref<RenderOptions>({
  side: 'top',
  sm: 'blue',
  cf: 'gold',
  sp: false,
  view3d: false,
});

const canvasTop = ref(0);
function handleResize(height: number): void {
  canvasTop.value = height;
}

const loading = ref(false);
async function loadGerber({ file }: { file: File }): Promise<void> {
  loading.value = true;
  gerber.value = file;
  layers.value = await loadLayers(file);
  loading.value = false;
}

function toggleView3D(): void {
  render.value.view3d = !render.value.view3d;
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
</style>
