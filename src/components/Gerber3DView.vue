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

interface ExtendedInputLayer extends InputLayer {
  svg?: string;
}

const props = defineProps<{
  layers: ExtendedInputLayer[];
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

// Function to create texture from layer image
async function createTextureFromLayer(layer: ExtendedInputLayer) {
  if (!layer.svg) return null;
  
  // Create a temporary div to render SVG
  const div = document.createElement('div');
  div.innerHTML = layer.svg;
  const svg = div.querySelector('svg');
  if (!svg) return null;

  // Set SVG dimensions
  svg.setAttribute('width', '1024');
  svg.setAttribute('height', '1024');

  // Convert SVG to data URL
  const svgData = new XMLSerializer().serializeToString(svg);
  const svgBlob = new Blob([svgData], { type: 'image/svg+xml' });
  const url = URL.createObjectURL(svgBlob);

  // Create image and load texture
  return new Promise<THREE.Texture | null>((resolve) => {
    const img = new Image();
    img.onload = () => {
      const texture = new THREE.Texture(img);
      texture.needsUpdate = true;
      resolve(texture);
    };
    img.src = url;
  });
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
  // Initialize Three.js scene with better environment
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x1a1a1a);
  scene.fog = new THREE.Fog(0x1a1a1a, 150, 400);

  // Camera setup
  camera = new THREE.PerspectiveCamera(
    45,
    container.value ? container.value.width / container.value.height : 1,
    0.1,
    1000
  );
  camera.position.set(0, 0, 200);
  camera.lookAt(0, 0, 0);

  // Renderer setup
  renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true
  });
  renderer.setSize(container.value?.width || 800, container.value?.height || 600);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  containerRef.value?.appendChild(renderer.domElement);

  // Controls setup
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 50;
  controls.maxDistance = 500;
  controls.enablePan = true;

  // Lighting setup
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
  scene.add(ambientLight);

  const keyLight = new THREE.DirectionalLight(0xffffff, 1.5);
  keyLight.position.set(100, 100, 100);
  keyLight.castShadow = true;
  keyLight.shadow.mapSize.width = 2048;
  keyLight.shadow.mapSize.height = 2048;
  scene.add(keyLight);

  const fillLight = new THREE.DirectionalLight(0xccffff, 0.6);
  fillLight.position.set(-100, -100, -100);
  scene.add(fillLight);

  const rimLight = new THREE.DirectionalLight(0xffffcc, 0.5);
  rimLight.position.set(0, 0, -100);
  scene.add(rimLight);

  // Initialize PCB group
  pcbGroup = new THREE.Group();
  scene.add(pcbGroup);

  // Initialize component group
  componentGroup = new THREE.Group();
  scene.add(componentGroup);

  // Add floor and grid
  const floorGeometry = new THREE.PlaneGeometry(1000, 1000);
  const floorMaterial = new THREE.MeshStandardMaterial({
    color: 0x333333,
    side: THREE.DoubleSide
  });
  const floor = new THREE.Mesh(floorGeometry, floorMaterial);
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -50;
  floor.receiveShadow = true;
  scene.add(floor);

  const gridHelper = new THREE.GridHelper(1000, 100, 0x444444, 0x222222);
  gridHelper.position.y = -49.9;
  scene.add(gridHelper);

  loaded.value = true;
  updatePCB();
}

function updatePCB() {
  if (!pcbGroup || !props.layers.length) return;

  // Clear existing PCB
  pcbGroup.clear();
  componentGroup.clear();

  // Get PCB dimensions from actual board dimensions
  const pcbWidth = props.render.dimensions?.width || 150;
  const pcbHeight = props.render.dimensions?.height || 100;
  const pcbDepth = 1.6;

  // Create PCB base
  const pcbGeometry = new THREE.BoxGeometry(pcbWidth, pcbHeight, pcbDepth);
  const pcbMaterial = new THREE.MeshStandardMaterial({
    color: new THREE.Color(COLORS[props.render.sm][0]),
    roughness: 0.7,
    metalness: 0.2
  });
  const pcbMesh = new THREE.Mesh(pcbGeometry, pcbMaterial);
  pcbMesh.castShadow = true;
  pcbMesh.receiveShadow = true;
  pcbGroup.add(pcbMesh);

  // Get layers
  const topCopper = props.layers.find(layer => layer.type === 'copper' && layer.side === 'top');
  const bottomCopper = props.layers.find(layer => layer.type === 'copper' && layer.side === 'bottom');
  const topMask = props.layers.find(layer => layer.type === 'soldermask' && layer.side === 'top');
  const bottomMask = props.layers.find(layer => layer.type === 'soldermask' && layer.side === 'bottom');

  // Create materials with textures
  Promise.all([
    topCopper && createTextureFromLayer(topCopper),
    bottomCopper && createTextureFromLayer(bottomCopper),
    topMask && createTextureFromLayer(topMask),
    bottomMask && createTextureFromLayer(bottomMask)
  ]).then(([topCopperTexture, bottomCopperTexture, topMaskTexture, bottomMaskTexture]) => {
    // Create copper material with texture
    const createCopperMaterial = (texture: THREE.Texture | null) => new THREE.MeshStandardMaterial({
      color: new THREE.Color(FINISHES[props.render.cf] || '#cc9933'),
      roughness: 0.3,
      metalness: 0.8,
      map: texture || undefined,
      transparent: true,
      opacity: 0.9
    });

    // Create mask material with texture
    const createMaskMaterial = (texture: THREE.Texture | null) => new THREE.MeshStandardMaterial({
      color: new THREE.Color(COLORS[props.render.sm][0]),
      roughness: 0.8,
      metalness: 0.1,
      map: texture || undefined,
      transparent: true,
      opacity: 0.8
    });

    // Create top copper layer
    if (topCopper && topCopperTexture) {
      const topCopperGeometry = new THREE.PlaneGeometry(pcbWidth, pcbHeight);
      const topCopperMesh = new THREE.Mesh(topCopperGeometry, createCopperMaterial(topCopperTexture));
      topCopperMesh.position.z = pcbDepth/2 + 0.01;
      topCopperMesh.rotation.x = -Math.PI/2;
      topCopperMesh.castShadow = true;
      topCopperMesh.receiveShadow = true;
      pcbGroup.add(topCopperMesh);
    }

    // Create bottom copper layer
    if (bottomCopper && bottomCopperTexture) {
      const bottomCopperGeometry = new THREE.PlaneGeometry(pcbWidth, pcbHeight);
      const bottomCopperMesh = new THREE.Mesh(bottomCopperGeometry, createCopperMaterial(bottomCopperTexture));
      bottomCopperMesh.position.z = -pcbDepth/2 - 0.01;
      bottomCopperMesh.rotation.x = Math.PI/2;
      bottomCopperMesh.castShadow = true;
      bottomCopperMesh.receiveShadow = true;
      pcbGroup.add(bottomCopperMesh);
    }

    // Create top mask layer
    if (topMask && topMaskTexture) {
      const topMaskGeometry = new THREE.PlaneGeometry(pcbWidth, pcbHeight);
      const topMaskMesh = new THREE.Mesh(topMaskGeometry, createMaskMaterial(topMaskTexture));
      topMaskMesh.position.z = pcbDepth/2 + 0.02;
      topMaskMesh.rotation.x = -Math.PI/2;
      topMaskMesh.castShadow = true;
      topMaskMesh.receiveShadow = true;
      pcbGroup.add(topMaskMesh);
    }

    // Create bottom mask layer
    if (bottomMask && bottomMaskTexture) {
      const bottomMaskGeometry = new THREE.PlaneGeometry(pcbWidth, pcbHeight);
      const bottomMaskMesh = new THREE.Mesh(bottomMaskGeometry, createMaskMaterial(bottomMaskTexture));
      bottomMaskMesh.position.z = -pcbDepth/2 - 0.02;
      bottomMaskMesh.rotation.x = Math.PI/2;
      bottomMaskMesh.castShadow = true;
      bottomMaskMesh.receiveShadow = true;
      pcbGroup.add(bottomMaskMesh);
    }

    // Add vias and components after textures are loaded
    addVias(pcbWidth, pcbHeight, pcbDepth);
    addComponents(pcbWidth, pcbHeight, pcbDepth);

    // Center camera on PCB
    pcbGroup.position.set(-pcbWidth/2, -pcbHeight/2, 0);
    camera.position.set(0, 0, Math.max(pcbWidth, pcbHeight) * 1.5);
    camera.lookAt(0, 0, 0);
    controls.update();
  });
}

function addVias(width: number, height: number, depth: number) {
  const viaMaterial = new THREE.MeshStandardMaterial({
    color: new THREE.Color(FINISHES[props.render.cf] || '#cc9933'),
    roughness: 0.2,
    metalness: 0.9
  });

  // Add some example vias
  for (let i = 0; i < 20; i++) {
    const viaGeometry = new THREE.CylinderGeometry(0.5, 0.5, depth, 16);
    const viaMesh = new THREE.Mesh(viaGeometry, viaMaterial);
    
    // Random position
    viaMesh.position.set(
      (Math.random() - 0.5) * (width - 20),
      (Math.random() - 0.5) * (height - 20),
      0
    );
    
    viaMesh.castShadow = true;
    viaMesh.receiveShadow = true;
    pcbGroup.add(viaMesh);
  }
}

function addComponents(width: number, height: number, depth: number) {
  // Component types and their properties
  const componentTypes = [
    { name: 'IC', width: 5, height: 1, depth: 5, color: 0x333333 },
    { name: 'Resistor', width: 2, height: 0.5, depth: 1, color: 0x666666 },
    { name: 'Capacitor', width: 2, height: 1, depth: 2, color: 0x999999 },
    { name: 'Connector', width: 3, height: 1.5, depth: 3, color: 0x444444 }
  ];

  // Add components
  for (let i = 0; i < 15; i++) {
    const type = componentTypes[Math.floor(Math.random() * componentTypes.length)];
    const componentMaterial = new THREE.MeshStandardMaterial({
      color: type.color,
      roughness: 0.5,
      metalness: 0.5
    });

    const component = new THREE.Mesh(
      new THREE.BoxGeometry(type.width, type.height, type.depth),
      componentMaterial
    );
    
    // Random position on top of PCB
    component.position.set(
      (Math.random() - 0.5) * (width - 20),
      (Math.random() - 0.5) * (height - 20),
      depth/2 + type.height/2
    );
    
    // Random rotation around Z axis
    component.rotation.z = Math.random() * Math.PI * 2;
    
    component.castShadow = true;
    component.receiveShadow = true;
    componentGroup.add(component);

    // Add component label
    const label = new SpriteText(type.name, 1, 'white');
    label.position.set(
      component.position.x,
      component.position.y,
      component.position.z + type.height/2 + 1
    );
    componentGroup.add(label);
  }
}

function animate() {
  requestAnimationFrame(animate);
  if (controls) controls.update();
  if (renderer && scene && camera) {
    renderer.render(scene, camera);
  }
}

// Camera control functions
function resetCamera() {
  if (camera && controls) {
    camera.position.set(0, 0, 200);
    camera.lookAt(0, 0, 0);
    controls.reset();
  }
}

function setTopView() {
  if (camera && controls) {
    camera.position.set(0, 200, 0);
    camera.lookAt(0, 0, 0);
    controls.reset();
  }
}

function setBottomView() {
  if (camera && controls) {
    camera.position.set(0, -200, 0);
    camera.lookAt(0, 0, 0);
    controls.reset();
  }
}

const activeTab = ref('all');

// Computed properties for layer filtering
const copperLayers = computed(() => props.layers.filter(layer => layer.type === 'copper'));
const maskLayers = computed(() => props.layers.filter(layer => layer.type === 'soldermask'));
const otherLayers = computed(() => props.layers.filter(layer => 
  layer.type !== 'copper' && layer.type !== 'soldermask' && layer.type !== 'outline'
));

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