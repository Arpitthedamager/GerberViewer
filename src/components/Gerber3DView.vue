<template>
  <div :ref="(ref) => containerRef = (ref as HTMLElement)" :class="$style.container">
    <div v-if="!loaded" :class="$style.loading">
      <a-spin tip="Loading 3D View..." size="large">
        <div :class="$style.loadingContent"></div>
      </a-spin>
    </div>
    <div v-if="loaded" :class="$style.controls">
      <a-button-group>
        <a-tooltip title="Reset View">
          <a-button @click="resetCamera">
            <template #icon><reload-outlined /></template>
          </a-button>
        </a-tooltip>
        <a-tooltip title="Top View">
          <a-button @click="setTopView">
            <template #icon><arrow-up-outlined /></template>
          </a-button>
        </a-tooltip>
        <a-tooltip title="Bottom View">
          <a-button @click="setBottomView">
            <template #icon><arrow-down-outlined /></template>
          </a-button>
        </a-tooltip>
      </a-button-group>
    </div>
    <div v-if="loaded" :class="$style.layerControls">
      <a-card :bordered="false" :class="$style.layerCard">
        <template #title>
          <div :class="$style.cardTitle">
            <span>Layer Settings</span>
            <a-space>
              <a-tooltip title="Show All Layers">
                <a-button type="text" @click="showAllLayers" size="small">
                  <template #icon><eye-outlined /></template>
                </a-button>
              </a-tooltip>
              <a-tooltip title="Hide All Layers">
                <a-button type="text" @click="hideAllLayers" size="small">
                  <template #icon><eye-invisible-outlined /></template>
                </a-button>
              </a-tooltip>
            </a-space>
          </div>
        </template>
        
        <a-tabs v-model:activeKey="activeTab">
          <a-tab-pane key="all" tab="All Layers">
            <a-space direction="vertical" style="width: 100%">
              <div v-for="layer in props.layers" :key="layer.filename" :class="$style.layerItem">
                <div :class="$style.layerInfo">
                  <a-checkbox 
                    v-model:checked="layerVisibility[layer.filename]"
                    :disabled="layer.type === 'outline'">
                    {{ layer.filename }}
                  </a-checkbox>
                  <a-tag :color="getLayerColor(layer.type)">{{ layer.type }}</a-tag>
                </div>
                <a-tooltip :title="getLayerDescription(layer.type)">
                  <info-circle-outlined :class="$style.infoIcon" />
                </a-tooltip>
              </div>
            </a-space>
          </a-tab-pane>
          
          <a-tab-pane key="copper" tab="Copper">
            <a-space direction="vertical" style="width: 100%">
              <div v-for="layer in copperLayers" :key="layer.filename" :class="$style.layerItem">
                <div :class="$style.layerInfo">
                  <a-checkbox 
                    v-model:checked="layerVisibility[layer.filename]"
                    :disabled="layer.type === 'outline'">
                    {{ layer.filename }}
                  </a-checkbox>
                  <a-tag color="gold">{{ layer.type }}</a-tag>
                </div>
                <a-tooltip title="Copper layer for circuit traces">
                  <info-circle-outlined :class="$style.infoIcon" />
                </a-tooltip>
              </div>
            </a-space>
          </a-tab-pane>
          
          <a-tab-pane key="mask" tab="Mask">
            <a-space direction="vertical" style="width: 100%">
              <div v-for="layer in maskLayers" :key="layer.filename" :class="$style.layerItem">
                <div :class="$style.layerInfo">
                  <a-checkbox 
                    v-model:checked="layerVisibility[layer.filename]"
                    :disabled="layer.type === 'outline'">
                    {{ layer.filename }}
                  </a-checkbox>
                  <a-tag color="green">{{ layer.type }}</a-tag>
                </div>
                <a-tooltip title="Solder mask layer">
                  <info-circle-outlined :class="$style.infoIcon" />
                </a-tooltip>
              </div>
            </a-space>
          </a-tab-pane>
          
          <a-tab-pane key="other" tab="Other">
            <a-space direction="vertical" style="width: 100%">
              <div v-for="layer in otherLayers" :key="layer.filename" :class="$style.layerItem">
                <div :class="$style.layerInfo">
                  <a-checkbox 
                    v-model:checked="layerVisibility[layer.filename]"
                    :disabled="layer.type === 'outline'">
                    {{ layer.filename }}
                  </a-checkbox>
                  <a-tag :color="getLayerColor(layer.type)">{{ layer.type }}</a-tag>
                </div>
                <a-tooltip :title="getLayerDescription(layer.type)">
                  <info-circle-outlined :class="$style.infoIcon" />
                </a-tooltip>
              </div>
            </a-space>
          </a-tab-pane>
        </a-tabs>
      </a-card>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { defineProps, onMounted, onBeforeUnmount, ref, watch, computed } from 'vue';
import type { InputLayer } from 'pcb-stackup';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import SpriteText from 'three-spritetext';
import type { RenderOptions } from '@/utils/gerber';
import { useClientSize } from '@/composables/useClientSize';
import { COLORS, FINISHES } from '@/utils/gerber';
import { 
  ReloadOutlined, 
  ArrowUpOutlined, 
  ArrowDownOutlined,
  EyeOutlined,
  EyeInvisibleOutlined,
  InfoCircleOutlined
} from '@ant-design/icons-vue';

const props = defineProps<{
  layers: InputLayer[];
  render: RenderOptions;
}>();

const containerRef = ref<HTMLElement>();
const container = useClientSize(containerRef);
const loaded = ref(false);

// Three.js variables
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let pcbMesh: THREE.Mesh;
let pcbGroup: THREE.Group;
let componentGroup: THREE.Group;

// Texture loaders
const textureLoader = new THREE.TextureLoader();

const layerVisibility = ref<Record<string, boolean>>({});

// Initialize layer visibility
watch(() => props.layers, (newLayers) => {
  newLayers.forEach(layer => {
    layerVisibility.value[layer.filename] = layer.type !== 'outline';
  });
}, { immediate: true });

function getLayerColor(type: string | null): string {
  switch (type) {
    case 'copper': return 'gold';
    case 'soldermask': return 'green';
    case 'silkscreen': return 'white';
    case 'solderpaste': return 'blue';
    case 'drill': return 'red';
    case 'outline': return 'purple';
    default: return 'default';
  }
}

// Create texture function
function createPCBTexture(color: string, size = 512) {
  const canvas = document.createElement('canvas');
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext('2d');
  if (!ctx) return null;
  
  // Fill with base color
  ctx.fillStyle = color;
  ctx.fillRect(0, 0, size, size);
  
  // Add some PCB pattern
  ctx.fillStyle = 'rgba(0,0,0,0.05)';
  
  // Grid pattern
  const gridSize = 20;
  for (let i = 0; i < size; i += gridSize) {
    ctx.fillRect(0, i, size, 1);
    ctx.fillRect(i, 0, 1, size);
  }
  
  // Create some random circuit traces
  ctx.strokeStyle = 'rgba(255,255,255,0.1)';
  ctx.lineWidth = 1;
  
  for (let i = 0; i < 30; i++) {
    ctx.beginPath();
    const x = Math.random() * size;
    const y = Math.random() * size;
    ctx.moveTo(x, y);
    
    for (let j = 0; j < 5; j++) {
      const newX = x + (Math.random() - 0.5) * 100;
      const newY = y + (Math.random() - 0.5) * 100;
      ctx.lineTo(newX, newY);
    }
    
    ctx.stroke();
  }
  
  return new THREE.CanvasTexture(canvas);
}

onMounted(() => {
  setTimeout(() => {
    initScene();
    animate();
  }, 300);
});

watch([props, container], () => {
  if (container.value && renderer) {
    renderer.setSize(container.value.width, container.value.height);
    camera.aspect = container.value.width / container.value.height;
    camera.updateProjectionMatrix();
    updatePCB();
  }
});

onBeforeUnmount(() => {
  if (renderer) {
    renderer.dispose();
    scene.clear();
  }
});

function initScene() {
  if (!containerRef.value || !container.value) return;
  
  // Initialize Three.js scene with better environment
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x1a1a1a);
  
  // Add fog for depth
  scene.fog = new THREE.Fog(0x1a1a1a, 150, 400);
  
  // Create camera
  camera = new THREE.PerspectiveCamera(
    45,
    container.value.width / container.value.height,
    0.1,
    1000
  );
  camera.position.set(0, -150, 120);
  camera.lookAt(0, 0, 0);
  
  // Create renderer with better quality
  renderer = new THREE.WebGLRenderer({ 
    antialias: true,
    alpha: true,
    powerPreference: 'high-performance'
  });
  renderer.setSize(container.value.width, container.value.height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.physicallyCorrectLights = true;
  renderer.outputEncoding = THREE.sRGBEncoding;
  containerRef.value.appendChild(renderer.domElement);
  
  // Add orbit controls with better feel
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.1;
  controls.rotateSpeed = 0.7;
  controls.enablePan = true;
  controls.minDistance = 50;
  controls.maxDistance = 300;
  
  // Add lights for better visibility
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
  scene.add(ambientLight);
  
  // Key light
  const keyLight = new THREE.DirectionalLight(0xffffff, 1.5);
  keyLight.position.set(50, 50, 70);
  keyLight.castShadow = true;
  keyLight.shadow.camera.near = 0.5;
  keyLight.shadow.camera.far = 500;
  keyLight.shadow.camera.left = -100;
  keyLight.shadow.camera.right = 100;
  keyLight.shadow.camera.top = 100;
  keyLight.shadow.camera.bottom = -100;
  keyLight.shadow.mapSize.width = 2048;
  keyLight.shadow.mapSize.height = 2048;
  scene.add(keyLight);
  
  // Fill light
  const fillLight = new THREE.DirectionalLight(0xccffff, 0.6);
  fillLight.position.set(-50, -50, 50);
  scene.add(fillLight);
  
  // Rim light
  const rimLight = new THREE.DirectionalLight(0xffffcc, 0.5);
  rimLight.position.set(10, 100, -50);
  scene.add(rimLight);
  
  // Create environment
  createEnvironment();
  
  // Create PCB group to hold all PCB elements
  pcbGroup = new THREE.Group();
  scene.add(pcbGroup);
  
  // Components group
  componentGroup = new THREE.Group();
  
  // Create initial PCB
  createPCB();
  
  // Add dimension labels
  addDimensionLabels();
  
  loaded.value = true;
}

function createEnvironment() {
  // Studio-like floor
  const floorGeometry = new THREE.PlaneGeometry(1000, 1000);
  const floorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x202020,
    metalness: 0.1,
    roughness: 0.8
  });
  const floor = new THREE.Mesh(floorGeometry, floorMaterial);
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -40;
  floor.receiveShadow = true;
  scene.add(floor);
  
  // Add a subtle grid
  const gridHelper = new THREE.GridHelper(1000, 100, 0x444444, 0x222222);
  gridHelper.position.y = -39.9;
  scene.add(gridHelper);
}

function createPCB() {
  // PCB dimensions
  const pcbWidth = 100;
  const pcbHeight = 70;
  const pcbDepth = 1.6;
  
  // Create textures
  const solderMaskColor = COLORS[props.render.sm][0].replace(/bf$/, '');
  const pcbTexture = createPCBTexture(solderMaskColor);
  
  // Create PCB with better materials
  const pcbGeometry = new THREE.BoxGeometry(pcbWidth, pcbHeight, pcbDepth);
  const pcbMaterial = new THREE.MeshStandardMaterial({ 
    color: new THREE.Color(solderMaskColor),
    roughness: 0.4,
    metalness: 0.2,
    map: pcbTexture
  });
  
  // Create multi-material PCB
  pcbMesh = new THREE.Mesh(pcbGeometry, pcbMaterial);
  pcbMesh.castShadow = true;
  pcbMesh.receiveShadow = true;
  pcbGroup.add(pcbMesh);
  
  // Add layer information
  addLayerInfo();
  
  // Add copper layers
  addCopperLayers(pcbWidth, pcbHeight);
  
  // Position the PCB
  pcbGroup.position.set(0, 0, 0);
  pcbGroup.rotation.x = Math.PI / 12;
}

function addLayerInfo() {
  const pcbWidth = 100;
  const pcbHeight = 70;
  
  // Add layer count
  const layerCount = new SpriteText(`Layers: ${props.layers.length}`, 2, 'white');
  layerCount.position.set(0, pcbHeight/2 + 10, 0);
  pcbGroup.add(layerCount);
  
  // Add dimensions
  const dimensions = new SpriteText(`${pcbWidth}mm × ${pcbHeight}mm`, 2, 'white');
  dimensions.position.set(0, pcbHeight/2 + 20, 0);
  pcbGroup.add(dimensions);
}

function addCopperLayers(pcbWidth: number, pcbHeight: number) {
  const copperColor = FINISHES[props.render.cf];
  const copperMaterial = new THREE.MeshStandardMaterial({ 
    color: new THREE.Color(copperColor),
    metalness: 0.9,
    roughness: 0.1
  });
  
  // Create top copper layer
  const topCopper = new THREE.Mesh(
    new THREE.PlaneGeometry(pcbWidth - 10, pcbHeight - 10),
    copperMaterial
  );
  topCopper.position.z = 0.8;
  topCopper.rotation.x = -Math.PI / 2;
  pcbGroup.add(topCopper);
  
  // Create bottom copper layer
  const bottomCopper = new THREE.Mesh(
    new THREE.PlaneGeometry(pcbWidth - 10, pcbHeight - 10),
    copperMaterial
  );
  bottomCopper.position.z = -0.8;
  bottomCopper.rotation.x = Math.PI / 2;
  pcbGroup.add(bottomCopper);
  
  // Add layer labels
  const topLabel = new SpriteText('Top Layer', 1.5, 'white');
  topLabel.position.set(0, 0, 1);
  pcbGroup.add(topLabel);
  
  const bottomLabel = new SpriteText('Bottom Layer', 1.5, 'white');
  bottomLabel.position.set(0, 0, -1);
  pcbGroup.add(bottomLabel);
}

function addDimensionLabels() {
  const pcbWidth = 100;
  const pcbHeight = 70;
  
  // Add width dimension
  const widthLabel = new SpriteText(`${pcbWidth}mm`, 2, 'white');
  widthLabel.position.set(0, -pcbHeight/2 - 10, 0);
  pcbGroup.add(widthLabel);
  
  // Add height dimension
  const heightLabel = new SpriteText(`${pcbHeight}mm`, 2, 'white');
  heightLabel.position.set(-pcbWidth/2 - 10, 0, 0);
  heightLabel.rotation.z = Math.PI / 2;
  pcbGroup.add(heightLabel);
  
  // Add component count
  const componentCount = new SpriteText(`Components: ${props.layers.filter(l => l.type === 'copper').length}`, 2, 'white');
  componentCount.position.set(0, pcbHeight/2 + 20, 0);
  pcbGroup.add(componentCount);
}

function updatePCB() {
  if (pcbMesh) {
    pcbGroup.clear();
    createPCB();
  }
}

function animate() {
  requestAnimationFrame(animate);
  
  if (controls) {
    controls.update();
  }
  
  if (renderer && scene && camera) {
    renderer.render(scene, camera);
  }
}

// Camera control functions
function resetCamera() {
  if (camera && controls) {
    camera.position.set(0, -150, 120);
    camera.lookAt(0, 0, 0);
    controls.update();
  }
}

function setTopView() {
  if (camera && controls) {
    camera.position.set(0, 0, 100);
    camera.lookAt(0, 0, 0);
    controls.update();
  }
}

function setBottomView() {
  if (camera && controls) {
    camera.position.set(0, 0, -100);
    camera.lookAt(0, 0, 0);
    controls.update();
  }
}

const activeTab = ref('all');

// Computed properties for layer filtering
const copperLayers = computed(() => props.layers.filter(l => l.type === 'copper'));
const maskLayers = computed(() => props.layers.filter(l => l.type === 'soldermask'));
const otherLayers = computed(() => props.layers.filter(l => !['copper', 'soldermask'].includes(l.type)));

function getLayerDescription(type: string | null): string {
  switch (type) {
    case 'copper': return 'Copper layer containing circuit traces and pads';
    case 'soldermask': return 'Protective layer that covers copper traces';
    case 'silkscreen': return 'Text and graphics printed on the PCB';
    case 'solderpaste': return 'Solder paste layer for SMD components';
    case 'drill': return 'Holes and vias in the PCB';
    case 'outline': return 'PCB outline and cutouts';
    default: return 'Additional layer';
  }
}

function showAllLayers() {
  Object.keys(layerVisibility.value).forEach(key => {
    layerVisibility.value[key] = true;
  });
}

function hideAllLayers() {
  Object.keys(layerVisibility.value).forEach(key => {
    layerVisibility.value[key] = false;
  });
}
</script>

<style lang="scss" module>
.container {
  width: 100%;
  height: 100%;
  position: relative;
  background: linear-gradient(135deg, #1a1a1a 0%, #2a2a2a 100%);
}

.loading {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(4px);
  z-index: 10;
}

.loadingContent {
  padding: 50px;
  border-radius: 12px;
  background: rgba(0, 0, 0, 0.6);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.controls {
  position: absolute;
  bottom: 24px;
  right: 24px;
  z-index: 5;
  background: rgba(0, 0, 0, 0.6);
  padding: 8px;
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
  backdrop-filter: blur(8px);
  
  :global {
    .ant-btn {
      background: rgba(255, 255, 255, 0.1);
      border: none;
      color: rgba(255, 255, 255, 0.8);
      transition: all 0.3s ease;
      
      &:hover {
        background: rgba(255, 255, 255, 0.2);
        color: #fff;
      }
      
      &:active {
        background: rgba(255, 255, 255, 0.3);
      }
    }
  }
}

.layerControls {
  position: absolute;
  top: 24px;
  right: 24px;
  z-index: 5;
  width: 320px;
}

.layerCard {
  background: rgba(0, 0, 0, 0.75);
  backdrop-filter: blur(12px);
  border-radius: 16px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.1);
  
  :global {
    .ant-card-head {
      border-bottom: 1px solid rgba(255, 255, 255, 0.1);
      padding: 16px 20px;
    }
    
    .ant-card-body {
      padding: 16px;
    }
    
    .ant-tabs-nav {
      margin-bottom: 12px;
      border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    }
    
    .ant-tabs-tab {
      color: rgba(255, 255, 255, 0.6);
      transition: all 0.3s ease;
      padding: 8px 16px;
      
      &:hover {
        color: #1890ff;
      }
      
      &.ant-tabs-tab-active {
        color: #1890ff;
        
        .ant-tabs-tab-btn {
          color: #1890ff;
        }
      }
    }
    
    .ant-tabs-ink-bar {
      background: #1890ff;
      height: 2px;
    }
    
    .ant-checkbox-wrapper {
      color: rgba(255, 255, 255, 0.85);
      transition: all 0.3s ease;
      
      &:hover {
        color: #fff;
      }
    }
    
    .ant-checkbox-checked .ant-checkbox-inner {
      background-color: #1890ff;
      border-color: #1890ff;
    }
    
    .ant-tag {
      border-radius: 4px;
      padding: 2px 8px;
      font-size: 12px;
      border: none;
      box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    }
  }
}

.cardTitle {
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: rgba(255, 255, 255, 0.9);
  font-weight: 500;
  font-size: 16px;
  
  :global {
    .ant-btn {
      color: rgba(255, 255, 255, 0.6);
      transition: all 0.3s ease;
      
      &:hover {
        color: #1890ff;
      }
    }
  }
}

.layerItem {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  border-radius: 8px;
  transition: all 0.3s ease;
  margin-bottom: 4px;
  
  &:hover {
    background: rgba(255, 255, 255, 0.05);
  }
}

.layerInfo {
  display: flex;
  align-items: center;
  gap: 12px;
}

.infoIcon {
  color: rgba(255, 255, 255, 0.4);
  cursor: help;
  transition: all 0.3s ease;
  font-size: 16px;
  
  &:hover {
    color: #1890ff;
  }
}

// Add smooth transitions for all interactive elements
:global {
  * {
    transition: all 0.3s ease;
  }
}
</style> 