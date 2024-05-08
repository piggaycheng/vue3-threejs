<script setup lang="ts">
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import GUI from 'lil-gui';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const threeCanvas = ref<HTMLDivElement>();
let camera: THREE.PerspectiveCamera,
  scene: THREE.Scene,
  renderer: THREE.WebGLRenderer,
  orbitControls: OrbitControls,
  cube: THREE.Mesh;

const cubeControls = {
  x: 0,
  y: 0,
  z: 0,
  xSpeed: 0.1,
  zSpeed: 0.1,
};

const cameraControls = {
  lock: true
};

const gui = new GUI();
const cameraFolder = gui.addFolder('Camera');
cameraFolder.add(cameraControls, 'lock').onChange((value: boolean) => {
  cameraControls.lock = value;
});
const cubeFolder = gui.addFolder('Cube');
cubeFolder.add(cubeControls, 'x', -10, 10).onChange((value: number) => {
  cube.position.setX(value);
  orbitControls.update();
  render();
});
cubeFolder.add(cubeControls, 'y', -10, 10).onChange((value: number) => {
  cube.position.setY(value);
  orbitControls.update();
  render();
});
cubeFolder.add(cubeControls, 'z', -10, 10).onChange((value: number) => {
  cube.position.setZ(value);
  orbitControls.update();
  render();
});
cubeFolder.add(cubeControls, 'xSpeed', -1, 1).onChange((value: number) => {
  cubeControls.xSpeed = value;
});
cubeFolder.add(cubeControls, 'zSpeed', -1, 1).onChange((value: number) => {
  cubeControls.zSpeed = value;
});

document.addEventListener('keydown', (event) => {
  let deltaX = 0;
  let deltaZ = 0;
  switch (event.key) {
    case 'w':
      deltaZ = -1 * cubeControls.zSpeed;
      break;
    case 's':
      deltaZ = cubeControls.zSpeed;
      break;
    case 'a':
      deltaX = -1 * cubeControls.xSpeed;
      break;
    case 'd':
      deltaX = cubeControls.xSpeed;
      break;
  }

  cube.position.setX(cube.position.x + deltaX);
  cube.position.setZ(cube.position.z + deltaZ);

  if (!cameraControls.lock) {
    camera.position.setX(camera.position.x + deltaX);
    camera.position.setZ(camera.position.z + deltaZ);
  }
});

onMounted(() => {
  initThree();
});

function initThree() {
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, threeCanvas.value!.clientWidth / threeCanvas.value!.clientHeight, 0.1, 1000);
  camera.position.set(5, 5, 5)
  camera.lookAt(scene.position);

  const axesHelper = new THREE.AxesHelper(5);
  scene.add(axesHelper);

  renderer = new THREE.WebGLRenderer();
  renderer.setSize(threeCanvas.value!.clientWidth, threeCanvas.value!.clientHeight);
  threeCanvas.value!.appendChild(renderer.domElement);

  const geometry = new THREE.BoxGeometry(1, 1, 1);
  const material = new THREE.MeshPhongMaterial({ color: 0xff0000 });
  cube = new THREE.Mesh(geometry, material);
  scene.add(cube);

  const ambientLight = new THREE.AmbientLight(0x404040);
  scene.add(ambientLight);

  const spotLight = new THREE.SpotLight(0xffffff, 5);
  spotLight.position.set(3, 0, 0);
  spotLight.target = cube;
  scene.add(spotLight);

  const size = 10;
  const divisions = 10;
  const gridHelper = new THREE.GridHelper(size, divisions);
  scene.add(gridHelper);

  orbitControls = new OrbitControls(camera, renderer.domElement);
  orbitControls.target = cube.position;
  orbitControls.listenToKeyEvents(window);

  renderer.render(scene, camera);

  animate();

  function animate() {
    requestAnimationFrame(animate);

    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;

    orbitControls.update();

    render();
  }
}

function render() {
  renderer.render(scene, camera);
}
</script>

<template>
  <div ref="threeCanvas" class="three-container">
  </div>
</template>

<style scoped>
.three-container {
  width: 100vw;
  height: 100vh;
}
</style>
