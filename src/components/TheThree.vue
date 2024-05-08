<script setup lang="ts">
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import GUI from 'lil-gui';

const threeCanvas = ref<HTMLDivElement>();
let camera: THREE.PerspectiveCamera, scene: THREE.Scene, renderer: THREE.WebGLRenderer;

const cameraControls = {
  x: 5,
  y: 5,
  z: 5,
};

const gui = new GUI();
const cameraFolder = gui.addFolder('Camera');
cameraFolder.add(cameraControls, 'x', -10, 10).onChange((value: number) => {
  camera.position.setX(value);
  camera.lookAt(scene.position);
  renderer.render(scene, camera);
});
cameraFolder.add(cameraControls, 'y', -10, 10).onChange((value: number) => {
  camera.position.setY(value);
  camera.lookAt(scene.position);
  renderer.render(scene, camera);
});
cameraFolder.add(cameraControls, 'z', -10, 10).onChange((value: number) => {
  camera.position.setZ(value);
  camera.lookAt(scene.position);
  renderer.render(scene, camera);
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
  const cube = new THREE.Mesh(geometry, material);
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

  renderer.render(scene, camera);

  // animate();

  function animate() {
    requestAnimationFrame(animate);

    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;

    renderer.render(scene, camera);
  }
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
