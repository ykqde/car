<template>
  <div class="home">
    <div class="canvas-container" ref="canvasDom"></div>
    
    <!-- 模型介绍面板 -->
    <div class="info-panel" :class="{ collapsed: isCollapsed }">
      <div class="info-header">
        <h1>BMW 概念跑车</h1>
        <p class="subtitle">BMW Concept Car</p>
      </div>
      
      <div class="info-content" v-show="!isCollapsed">
        <div class="info-section">
          <h2>车型概述</h2>
          <p>这是一款BMW概念跑车，融合了未来主义设计语言与卓越的空气动力学性能。流畅的车身线条和大胆的前脸造型展现了BMW品牌对未来出行方式的愿景。</p>
        </div>
        
        <div class="info-section">
          <h2>设计特点</h2>
          <ul>
            <li>流线型车身设计，风阻系数优化</li>
            <li>标志性双肾型进气格栅</li>
            <li>全景天窗玻璃座舱</li>
            <li>21英寸运动轮毂</li>
          </ul>
        </div>
        
        <div class="info-section">
          <h2>动力参数</h2>
          <div class="specs">
            <div class="spec-item">
              <span class="spec-value">750</span>
              <span class="spec-label">马力</span>
            </div>
            <div class="spec-item">
              <span class="spec-value">3.2</span>
              <span class="spec-label">零百加速(秒)</span>
            </div>
            <div class="spec-item">
              <span class="spec-value">305</span>
              <span class="spec-label">最高时速(km/h)</span>
            </div>
          </div>
        </div>
        
        <div class="info-section">
          <h2>交互说明</h2>
          <p>拖拽鼠标旋转视角 | 滚轮缩放 | 右键拖拽平移</p>
        </div>
      </div>
      
      <button class="toggle-btn" @click="isCollapsed = !isCollapsed">
        {{ isCollapsed ? '展开详情' : '收起' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import * as THREE from "three";
import { onMounted, ref } from "vue";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";

const canvasDom = ref(null);
const isCollapsed = ref(false);

let controls;
let mixer = null;
let clock = new THREE.Clock();

// 创建场景
const scene = new THREE.Scene();
// 创建相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.set(-10, 10, 10);

// 创建渲染器
const renderer = new THREE.WebGLRenderer({
  antialias: true,
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.4;

const render = () => {
  renderer.render(scene, camera);
  controls && controls.update();
  
  if (mixer) {
    mixer.update(clock.getDelta());
  }
  
  requestAnimationFrame(render);
};

let wheels = [];
let carBody, frontCar, hoodCar, glassCar;

// 创建材质
const bodyMaterial = new THREE.MeshPhysicalMaterial({
  color: 0x00ff00,
  metalness: 1,
  roughness: 0.5,
  clearcoat: 1,
  clearcoatRoughness: 0,
});

onMounted(() => {
  canvasDom.value.appendChild(renderer.domElement);
  renderer.setClearColor("#000");
  
  // 深色渐变背景
  scene.background = new THREE.Color("#1a1a2e");
  
  render();

  // 添加网格地面
  const gridHelper = new THREE.GridHelper(50, 50, 0x444444, 0x222222);
  gridHelper.material.opacity = 0.3;
  gridHelper.material.transparent = true;
  scene.add(gridHelper);

  // 添加圆形地面
  const groundGeometry = new THREE.CircleGeometry(15, 64);
  const groundMaterial = new THREE.MeshStandardMaterial({
    color: 0x222233,
    roughness: 0.8,
    metalness: 0.2,
  });
  const ground = new THREE.Mesh(groundGeometry, groundMaterial);
  ground.rotation.x = -Math.PI / 2;
  ground.position.y = -0.01;
  ground.receiveShadow = true;
  scene.add(ground);

  // 添加控制器
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 5;
  controls.maxDistance = 50;
  controls.maxPolarAngle = Math.PI / 2;
  controls.update();

  // ==================== 工作室展示风格灯光 ====================

  // 1. 环境光 - 大幅提升，消除死黑区域
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.5);
  scene.add(ambientLight);

  // 2. 半球光 - 天空/地面双向漫反射，填充轮拱底部
  const hemiLight = new THREE.HemisphereLight(0xffffff, 0xcccccc, 1.0);
  hemiLight.position.set(0, 50, 0);
  scene.add(hemiLight);

  // 3. 主光源（Key Light）- 左前方45度
  const mainLight = new THREE.DirectionalLight(0xffffff, 2.0);
  mainLight.position.set(-8, 15, 12);
  mainLight.castShadow = true;
  mainLight.shadow.mapSize.width = 2048;
  mainLight.shadow.mapSize.height = 2048;
  mainLight.shadow.camera.near = 0.5;
  mainLight.shadow.camera.far = 50;
  mainLight.shadow.camera.left = -20;
  mainLight.shadow.camera.right = 20;
  mainLight.shadow.camera.top = 20;
  mainLight.shadow.camera.bottom = -20;
  mainLight.shadow.bias = -0.0001;
  scene.add(mainLight);

  // 4. 右侧补光 - 填充主光照不到的右侧
  const fillRight = new THREE.DirectionalLight(0xfff5ee, 1.5);
  fillRight.position.set(15, 6, 8);
  scene.add(fillRight);

  // 5. 左后补光 - 填充后侧阴影
  const fillLeft = new THREE.DirectionalLight(0xfff0e8, 1.2);
  fillLeft.position.set(-12, 5, -10);
  scene.add(fillLeft);

  // 6. 轮廓光（Rim Light）- 右后上方，勾勒车顶线条
  const rimLight = new THREE.DirectionalLight(0xe8f0ff, 1.5);
  rimLight.position.set(10, 14, -15);
  scene.add(rimLight);

  // 7. 低位前补光 - 照亮前脸/进气格栅细节
  const frontLow = new THREE.DirectionalLight(0xffffff, 1.2);
  frontLow.position.set(0, 2, 20);
  scene.add(frontLow);

  // 8. 低位侧补光 - 照亮轮拱/底裙细节，消除黑色区域
  const sideLow1 = new THREE.DirectionalLight(0xffffff, 1.0);
  sideLow1.position.set(20, 1, 0);
  scene.add(sideLow1);

  const sideLow2 = new THREE.DirectionalLight(0xffffff, 1.0);
  sideLow2.position.set(-20, 1, 0);
  scene.add(sideLow2);

  // 9. 地面反弹光 - 模拟地面对底盘的漫反射
  const groundBounce = new THREE.DirectionalLight(0xfff8f0, 0.8);
  groundBounce.position.set(0, -2, 0);
  scene.add(groundBounce);

  // 11. 座舱内部点光源 - 从车顶内侧向下照亮座椅/仪表盘
  const cockpitTop = new THREE.PointLight(0xffffff, 3.0, 8);
  cockpitTop.position.set(0, 4, 0);
  scene.add(cockpitTop);

  // 12. 座舱前方补光 - 照亮仪表盘和前舱区域
  const cockpitFront = new THREE.PointLight(0xfff8f0, 2.0, 6);
  cockpitFront.position.set(0, 3, 2);
  scene.add(cockpitFront);

  // 13. 座舱后方补光 - 照亮座椅背部
  const cockpitBack = new THREE.PointLight(0xffffff, 2.0, 6);
  cockpitBack.position.set(0, 3, -2);
  scene.add(cockpitBack);

  // 10. 顶部柔光灯
  const topSpot = new THREE.SpotLight(0xffffff, 1.5);
  topSpot.position.set(0, 22, 5);
  topSpot.angle = Math.PI / 3;
  topSpot.penumbra = 0.9;
  topSpot.decay = 1.2;
  topSpot.distance = 60;
  topSpot.castShadow = true;
  topSpot.shadow.mapSize.width = 2048;
  topSpot.shadow.mapSize.height = 2048;
  scene.add(topSpot);

  // 添加gltf汽车模型
  const loader = new GLTFLoader();
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath("/draco/gltf/");
  loader.setDRACOLoader(dracoLoader);
  loader.load("/model/scene.gltf", (gltf) => {
    const bmw = gltf.scene;
    
    // 调整模型大小和位置
    bmw.scale.set(1, 1, 1);
    bmw.position.set(0, 0, 0);
    
    bmw.traverse((child) => {
      if (child.isMesh) {
        child.castShadow = true;
        child.receiveShadow = true;
        
        // 判断是否是轮廓
        if (child.name.includes("Object_13")) {
          wheels.push(child);
        }
        // 判断是否是车身
        if (child.name.includes("Object_16")) {
          carBody = child;
          carBody.material = bodyMaterial;
        }
        // 判断是否是前脸
        if (child.name.includes("Object_19")) {
          frontCar = child;
        }
        // 判断是否是引擎盖
        if (child.name.includes("Object_25")) {
          hoodCar = child;
        }
        // 判断是否是挡风玻璃
        if (child.name.includes("Object_41")) {
          glassCar = child;
        }
      }
    });
    
    scene.add(bmw);
    
    // 如果有动画
    if (gltf.animations && gltf.animations.length > 0) {
      mixer = new THREE.AnimationMixer(bmw);
      gltf.animations.forEach((clip) => {
        mixer.clipAction(clip).play();
      });
    }
  });

  // 窗口大小变化处理
  window.addEventListener("resize", () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
});
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.home {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  position: relative;
}

.canvas-container {
  width: 100%;
  height: 100%;
}

/* 模型介绍面板 */
.info-panel {
  position: absolute;
  top: 20px;
  left: 20px;
  width: 380px;
  max-height: calc(100vh - 40px);
  background: linear-gradient(135deg, rgba(20, 25, 40, 0.95) 0%, rgba(30, 35, 55, 0.95) 100%);
  backdrop-filter: blur(10px);
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  overflow: hidden;
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
}

.info-panel.collapsed {
  width: 180px;
}

.info-panel.collapsed .info-header h1 {
  font-size: 1.1rem;
}

.info-panel.collapsed .info-header .subtitle {
  display: none;
}

.info-header {
  padding: 24px 24px 16px;
  background: linear-gradient(135deg, rgba(255, 107, 107, 0.2) 0%, rgba(255, 142, 84, 0.2) 100%);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.info-header h1 {
  font-size: 1.6rem;
  font-weight: 700;
  color: #ffffff;
  margin-bottom: 6px;
  letter-spacing: 1px;
}

.info-header .subtitle {
  font-size: 0.9rem;
  color: rgba(255, 255, 255, 0.6);
  font-style: italic;
}

.info-content {
  padding: 20px 24px;
  overflow-y: auto;
  max-height: calc(100vh - 250px);
}

.info-section {
  margin-bottom: 20px;
}

.info-section:last-child {
  margin-bottom: 0;
}

.info-section h2 {
  font-size: 1rem;
  font-weight: 600;
  color: #ff7b54;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.info-section h2::before {
  content: '';
  width: 4px;
  height: 16px;
  background: linear-gradient(180deg, #ff7b54, #ffab76);
  border-radius: 2px;
}

.info-section p {
  font-size: 0.88rem;
  line-height: 1.7;
  color: rgba(255, 255, 255, 0.8);
}

.info-section ul {
  list-style: none;
  padding: 0;
}

.info-section ul li {
  font-size: 0.88rem;
  color: rgba(255, 255, 255, 0.8);
  padding: 6px 0;
  padding-left: 20px;
  position: relative;
}

.info-section ul li::before {
  content: '▸';
  position: absolute;
  left: 0;
  color: #ffab76;
}

/* 规格参数 */
.specs {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.spec-item {
  background: rgba(255, 107, 107, 0.1);
  border: 1px solid rgba(255, 107, 107, 0.3);
  border-radius: 10px;
  padding: 14px 10px;
  text-align: center;
  transition: all 0.3s ease;
}

.spec-item:hover {
  background: rgba(255, 107, 107, 0.2);
  transform: translateY(-2px);
}

.spec-value {
  display: block;
  font-size: 1.5rem;
  font-weight: 700;
  color: #ffab76;
  line-height: 1;
}

.spec-label {
  display: block;
  font-size: 0.75rem;
  color: rgba(255, 255, 255, 0.6);
  margin-top: 6px;
}

/* 折叠按钮 */
.toggle-btn {
  width: 100%;
  padding: 14px;
  background: rgba(255, 107, 107, 0.15);
  border: none;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.8);
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.3s ease;
}

.toggle-btn:hover {
  background: rgba(255, 107, 107, 0.25);
  color: #ffffff;
}

/* 滚动条样式 */
.info-content::-webkit-scrollbar {
  width: 4px;
}

.info-content::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
}

.info-content::-webkit-scrollbar-thumb {
  background: rgba(255, 107, 107, 0.5);
  border-radius: 2px;
}

/* 响应式 */
@media (max-width: 768px) {
  .info-panel {
    width: calc(100vw - 40px);
    max-width: 380px;
    top: 10px;
    left: 10px;
  }
  
  .info-header h1 {
    font-size: 1.3rem;
  }
  
  .specs {
    grid-template-columns: repeat(3, 1fr);
    gap: 8px;
  }
  
  .spec-value {
    font-size: 1.2rem;
  }
}
</style>
