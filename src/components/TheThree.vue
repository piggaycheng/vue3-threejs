<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import GUI from 'lil-gui';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { mod } from 'three/examples/jsm/nodes/Nodes.js';

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
  actionsMapping: Map<string, THREE.AnimationAction>,
  clock: THREE.Clock = new THREE.Clock()

const cubeControls = {
  x: 0,
  y: 0,
  z: 0,
  visible: false,
};

const cameraControls = {
  lock: true
};

const characterControls = {
  action: 'Idle',
  idleWeight: 1,
  walkWeight: 0,
  runWeight: 0,
  xSpeed: 0.1,
  zSpeed: 0.1,
}

const gui = new GUI();
const cameraFolder = gui.addFolder('Camera');
cameraFolder.add(cameraControls, 'lock').onChange((value: boolean) => {
  cameraControls.lock = value;
});
cameraFolder.close();
const cubeFolder = gui.addFolder('Cube');
cubeFolder.add(cubeControls, 'x', -10, 10).onChange((value: number) => {
  cube.position.setX(value);
});
cubeFolder.add(cubeControls, 'y', -10, 10).onChange((value: number) => {
  cube.position.setY(value);
});
cubeFolder.add(cubeControls, 'z', -10, 10).onChange((value: number) => {
  cube.position.setZ(value);
});
cubeFolder.add(cubeControls, 'visible').onChange((value: boolean) => {
  cube.visible = value;
});
cubeFolder.close();
const characterFolder = gui.addFolder('Character');
let lastAction = 'idleAction';
characterFolder.add(characterControls, 'action', { 'Idle': 'idleAction', 'Walk': 'walkAction', 'Run': 'runAction' }).onChange((value: string) => {
  executeCrossFade(actionsMapping.get(lastAction)!, actionsMapping.get(value)!, 1);

  characterControls.action = value;
  lastAction = value;
});
characterFolder.add(characterControls, 'idleWeight', 0, 1).listen().onChange((value: number) => {
  characterControls.idleWeight = value;
  setWeight(idleAction, value);
});
characterFolder.add(characterControls, 'walkWeight', 0, 1).listen().onChange((value: number) => {
  characterControls.walkWeight = value;
  setWeight(walkAction, value);
});
characterFolder.add(characterControls, 'runWeight', 0, 1).listen().onChange((value: number) => {
  characterControls.runWeight = value;
  setWeight(runAction, value);
});
characterFolder.add(characterControls, 'xSpeed', 0, 1).onChange((value: number) => {
  characterControls.xSpeed = value;
});
characterFolder.add(characterControls, 'zSpeed', 0, 1).onChange((value: number) => {
  characterControls.zSpeed = value;
});

document.addEventListener('keydown', (event) => {
  let deltaX = 0;
  let deltaZ = 0;
  switch (event.key) {
    case 'w':
      deltaZ = -1 * characterControls.zSpeed;
      break;
    case 's':
      deltaZ = characterControls.zSpeed;
      break;
    case 'a':
      deltaX = -1 * characterControls.xSpeed;
      break;
    case 'd':
      deltaX = characterControls.xSpeed;
      break;
  }

  model.position.setX(model.position.x + deltaX);
  model.position.setZ(model.position.z + deltaZ);

  if (!cameraControls.lock) {
    camera.position.setX(camera.position.x + deltaX);
    camera.position.setZ(camera.position.z + deltaZ);
  }
});

onMounted(() => {
  initThree();
  loadModel();
});

onUnmounted(() => {
  gui.destroy();
});

function initThree() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xa0a0a0);
  scene.fog = new THREE.Fog(0xa0a0a0, 10, 50);

  camera = new THREE.PerspectiveCamera(75, threeCanvas.value!.clientWidth / threeCanvas.value!.clientHeight, 0.1, 1000);
  camera.position.set(5, 5, 5)

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
  // orbitControls.target = cube.position;

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

    if (model) orbitControls.target.copy(model.position);
    orbitControls.update();

    const mixerUpdateDelta = clock.getDelta();
    if (mixer) mixer.update(mixerUpdateDelta);

    if (idleAction) characterControls.idleWeight = idleAction.getEffectiveWeight();
    if (walkAction) characterControls.walkWeight = walkAction.getEffectiveWeight();
    if (runAction) characterControls.runWeight = runAction.getEffectiveWeight();

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

    orbitControls.target.copy(model.position);

    mixer = new THREE.AnimationMixer(model);
    idleAction = mixer.clipAction(gltf.animations[0]);
    walkAction = mixer.clipAction(gltf.animations[3]);
    runAction = mixer.clipAction(gltf.animations[1]);

    actionsMapping = new Map([
      ['idleAction', idleAction],
      ['walkAction', walkAction],
      ['runAction', runAction]
    ]);
    actionsMapping.forEach((action) => {
      action.setEffectiveWeight(0);
      action.play();
    });
    idleAction.setEffectiveWeight(1);

    model.traverse(function (object) {
      if ((object as THREE.Mesh).isMesh) object.castShadow = true;
    });
    console.log(gltf);
  });
}

function executeCrossFade(startAction: THREE.AnimationAction, endAction: THREE.AnimationAction, duration: number) {
  // Not only the start action, but also the end action must get a weight of 1 before fading
  // (concerning the start action this is already guaranteed in this place)
  setWeight(endAction, 1);
  endAction.time = 0;

  // Crossfade with warping - you can also try without warping by setting the third parameter to false
  startAction.crossFadeTo(endAction, duration, true);
}

function setWeight(action: THREE.AnimationAction, weight: number) {
  action.enabled = true;
  action.setEffectiveTimeScale(1);
  action.setEffectiveWeight(weight);
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
