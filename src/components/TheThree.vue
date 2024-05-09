<script setup lang="ts">
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import GUI from 'lil-gui';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const threeCanvas = ref<HTMLDivElement>();
let camera: THREE.PerspectiveCamera,
  scene: THREE.Scene,
  renderer: THREE.WebGLRenderer,
  orbitControls: OrbitControls,
  cube: THREE.Mesh;
let model: THREE.Object3D,
  mixer: THREE.AnimationMixer,
  idleAction: THREE.AnimationAction,
  walkAction: THREE.AnimationAction,
  runAction: THREE.AnimationAction,
  tPoseAction: THREE.AnimationAction,
  actions: THREE.AnimationAction[],
  clock: THREE.Clock = new THREE.Clock()

const cubeControls = {
  x: 0,
  y: 0,
  z: 0,
  xSpeed: 0.1,
  zSpeed: 0.1,
  visible: false,
};

const cameraControls = {
  lock: true
};

const characterControls = {
  action: 'idle'
}

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
cubeFolder.add(cubeControls, 'visible').onChange((value: boolean) => {
  cube.visible = value;
});
const characterFolder = gui.addFolder('Character');
characterFolder.add(characterControls, 'action', ['Idle', 'Walk', 'Run', 'TPose']).onChange((value: string) => {
  characterControls.action = value;
  switch (value) {
    case 'Idle':
      idleAction.play();
      walkAction.stop();
      runAction.stop();
      tPoseAction.stop();
      break;
    case 'Walk':
      idleAction.stop();
      walkAction.play();
      runAction.stop();
      tPoseAction.stop();
      break;
    case 'Run':
      idleAction.stop();
      walkAction.stop();
      runAction.play();
      tPoseAction.stop();
      break;
    case 'TPose':
      idleAction.stop();
      walkAction.stop();
      runAction.stop();
      tPoseAction.play();
      break;
  }
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
  loadModel();
});

function initThree() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xa0a0a0);
  scene.fog = new THREE.Fog(0xa0a0a0, 10, 50);
  camera = new THREE.PerspectiveCamera(75, threeCanvas.value!.clientWidth / threeCanvas.value!.clientHeight, 0.1, 1000);
  camera.position.set(5, 5, 5)
  camera.lookAt(scene.position);

  const axesHelper = new THREE.AxesHelper(5);
  scene.add(axesHelper);

  renderer = new THREE.WebGLRenderer();
  renderer.setSize(threeCanvas.value!.clientWidth, threeCanvas.value!.clientHeight);
  renderer.shadowMap.enabled = true;
  threeCanvas.value!.appendChild(renderer.domElement);

  const geometry = new THREE.BoxGeometry(1, 1, 1);
  const material = new THREE.MeshPhongMaterial({ color: 0xff0000 });
  cube = new THREE.Mesh(geometry, material);
  scene.add(cube);
  cube.visible = cubeControls.visible;

  // const ambientLight = new THREE.AmbientLight(0x404040);
  // scene.add(ambientLight);
  const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 3);
  hemiLight.position.set(0, 20, 0);
  scene.add(hemiLight);

  // const spotLight = new THREE.SpotLight(0xffffff, 5);
  // spotLight.position.set(3, 0, 0);
  // spotLight.target = cube;
  // scene.add(spotLight);
  const dirLight = new THREE.DirectionalLight(0xffffff, 3);
  dirLight.position.set(- 3, 10, - 10);
  dirLight.castShadow = true;
  dirLight.shadow.camera.top = 2;
  dirLight.shadow.camera.bottom = - 2;
  dirLight.shadow.camera.left = - 2;
  dirLight.shadow.camera.right = 2;
  dirLight.shadow.camera.near = 0.1;
  dirLight.shadow.camera.far = 40;
  scene.add(dirLight);

  // const size = 10;
  // const divisions = 10;
  // const gridHelper = new THREE.GridHelper(size, divisions);
  // scene.add(gridHelper);

  orbitControls = new OrbitControls(camera, renderer.domElement);
  orbitControls.target = cube.position;
  orbitControls.listenToKeyEvents(window);

  const mesh = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), new THREE.MeshPhongMaterial({ color: 0xcbcbcb, depthWrite: false }));
  mesh.rotation.x = - Math.PI / 2;
  mesh.receiveShadow = true;
  scene.add(mesh);

  renderer.render(scene, camera);

  animate();

  function animate() {
    requestAnimationFrame(animate);

    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;

    orbitControls.update();

    const mixerUpdateDelta = clock.getDelta();
    if (mixer) mixer.update(mixerUpdateDelta);

    render();
  }
}

function render() {
  renderer.render(scene, camera);
}

function loadModel() {
  const loader = new GLTFLoader();
  loader.load('3dModels/Soldier.glb', (gltf) => {
    model = gltf.scene;
    scene.add(model);

    mixer = new THREE.AnimationMixer(model);
    idleAction = mixer.clipAction(gltf.animations[0]);
    walkAction = mixer.clipAction(gltf.animations[3]);
    runAction = mixer.clipAction(gltf.animations[1]);
    tPoseAction = mixer.clipAction(gltf.animations[2]);

    model.traverse(function (object) {
      if ((object as THREE.Mesh).isMesh) object.castShadow = true;
    });
    console.log(gltf);
  });
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
