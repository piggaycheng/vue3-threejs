<script setup lang="ts">
import { ref, onMounted } from 'vue';
import * as THREE from 'three';

const threeContainer = ref<HTMLDivElement>();

onMounted(() => {
  initThree();
});

function initThree() {
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, threeContainer.value!.clientWidth / threeContainer.value!.clientHeight, 0.1, 1000);
  camera.position.set(5, 5, 5)
  camera.lookAt(scene.position);

  const axesHelper = new THREE.AxesHelper(5);
  scene.add(axesHelper);

  const renderer = new THREE.WebGLRenderer();
  renderer.setSize(threeContainer.value!.clientWidth, threeContainer.value!.clientHeight);
  threeContainer.value!.appendChild(renderer.domElement);

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

  animate();

  function animate() {
    requestAnimationFrame(animate);

    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;

    renderer.render(scene, camera);
  }
}
</script>

<template>
  <div ref="threeContainer" class="three-container">
  </div>
</template>

<style scoped>
.three-container {
  width: 100vw;
  height: 100vh;
}
</style>
