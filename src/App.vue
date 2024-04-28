<template>
  <div class="home">
    <div class="canvas-container" ref="canvasDom"></div>
  </div>
</template>

<script setup>
import * as THREE from "three";
import { onMounted, reactive, ref } from "vue";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";

let controls;
let canvasDom = ref(null);

// 创建场景
const scene = new THREE.Scene();
// 创建相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.set(-10, 10,10);

// 创建渲染器
const renderer = new THREE.WebGLRenderer({
  // 抗锯齿
  antialias: true,
});
renderer.setSize(window.innerWidth, window.innerHeight);

const render = () => {
  renderer.render(scene, camera);
  controls && controls.update();
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
  // 把渲染器插入到dom中
  // console.log(canvasDom.value);
  canvasDom.value.appendChild(renderer.domElement);
  // 初始化渲染器，渲染背景
  renderer.setClearColor("#000");
  scene.background = new THREE.Color("#ccc");
  scene.environment = new THREE.Color("#ccc");
  render();

  // 添加网格地面
  const gridHelper = new THREE.GridHelper(30, 30);
  gridHelper.material.opacity = 0.2;
  gridHelper.material.transparent = true;
  scene.add(gridHelper);

  // 添加控制器
  controls = new OrbitControls(camera, renderer.domElement);
  controls.update();

  // 添加gltf汽车模型
  const loader = new GLTFLoader();
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath("/draco/gltf/");
  loader.setDRACOLoader(dracoLoader);
  loader.load("/model/scene.gltf", (gltf) => {
    const bmw = gltf.scene;
    bmw.traverse((child) => {
      if (child.isMesh) {
        child.castShadow = true;
        child.receiveShadow = true;
        console.log(child.name);
      }
      // 判断是否是轮廓
      if (child.isMesh && child.name.includes("Object_13")) {
        wheels.push(child);
      }
      // 判断是否是车身
      if (child.isMesh && child.name.includes("Object_16")) {
        carBody = child;
        carBody.material = bodyMaterial;
      }
      // 判断是否是前脸
      if (child.isMesh && child.name.includes("Object_19")) {
        frontCar = child;
      }
      // 判断是否是引擎盖
      if (child.isMesh && child.name.includes("Object_25")) {
        // child.material = hoodMaterial;
        hoodCar = child;
      }
      // 判断是否是挡风玻璃
      if (child.isMesh && child.name.includes("Object_41")) {
        // child.material = glassMaterial;
        glassCar = child;
      }
    });
    scene.add(bmw);
  });


  // 添加灯光
  // 此处为Z轴灯光效果
  const light1 = new THREE.DirectionalLight(0xffffff, 1);
  light1.position.set(10, 0, 10);
  scene.add(light1);
  const light2 = new THREE.DirectionalLight(0xffffff, 1);
  light2.position.set(10, 0, -10);
  scene.add(light2);
  const light3 = new THREE.DirectionalLight(0xffffff, 1);
  light3.position.set(10, 0, -10);
  scene.add(light3);
  const light4 = new THREE.DirectionalLight(0xffffff, 1);
  light4.position.set(-10, 0, 10);
  scene.add(light4);
  // 顶上的灯光
  const light5 = new THREE.DirectionalLight(0xffffff, 1);
  light5.position.set(0, 10, 0);
  scene.add(light5);
  const light6 = new THREE.DirectionalLight(0xffffff, 1);
  light6.position.set(10, -10, 0);
  scene.add(light6);
  const light7 = new THREE.DirectionalLight(0xffffff, 1);
  light7.position.set(0, 10, 10);
  scene.add(light7);
  const light8 = new THREE.DirectionalLight(0xffffff, 1);
  light8.position.set(0, 10, -10);
  scene.add(light8);
  const light9 = new THREE.DirectionalLight(0xffffff, 1);
  light9.position.set(-10, -10, 0);
  scene.add(light9);
});
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
}
</style>
