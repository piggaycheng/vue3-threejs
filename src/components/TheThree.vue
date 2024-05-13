<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import GUI from 'lil-gui';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import Stats from 'stats.js'
import * as TWEEN from '@tweenjs/tween.js'
import { Line2 } from 'three/addons/lines/Line2.js';
import { LineMaterial } from 'three/addons/lines/LineMaterial.js';
import { LineGeometry } from 'three/addons/lines/LineGeometry.js';

const threeCanvas = ref<HTMLDivElement>();
let camera: THREE.PerspectiveCamera,
  scene: THREE.Scene,
  renderer: THREE.WebGLRenderer,
  orbitControls: OrbitControls,
  cube: THREE.Mesh,
  stats: Stats;
let model: THREE.Object3D,
  mixer: THREE.AnimationMixer,
  idleAction: THREE.AnimationAction,
  walkAction: THREE.AnimationAction,
  runAction: THREE.AnimationAction,
  actionsMapping: Map<string, THREE.AnimationAction>,
  clock: THREE.Clock = new THREE.Clock();
let raycaster: THREE.Raycaster = new THREE.Raycaster(),
  pointer = new THREE.Vector2(),
  planeMesh: THREE.Mesh,
  // arrow: THREE.ArrowHelper,
  clickedPosition: THREE.Vector3,
  moveTween: TWEEN.Tween<THREE.Vector3>,
  rotateTween: TWEEN.Tween<THREE.Euler>;

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
  speed: 0.1,
  rotationSpeed: 0.1,
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
characterFolder.add(characterControls, 'action', { 'Idle': 'idleAction', 'Walk': 'walkAction', 'Run': 'runAction' }).listen().onChange((value: string) => {
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
characterFolder.add(characterControls, 'speed', 0, 1).onChange((value: number) => {
  characterControls.speed = value;
});
characterFolder.add(characterControls, 'rotationSpeed', 0, 1).onChange((value: number) => {
  characterControls.rotationSpeed = value;
});

document.addEventListener('keydown', (event) => {
  let deltaRadian = 0;
  let direction = new THREE.Vector3();
  model.getWorldDirection(direction);

  switch (event.key) {
    case 'w':
      direction = direction.multiplyScalar(-1 * characterControls.speed);
      model.position.add(direction);
      break;
    case 's':
      direction = direction.multiplyScalar(characterControls.speed);
      model.position.add(direction);
      break;
    case 'a':
      deltaRadian = characterControls.rotationSpeed;
      break;
    case 'd':
      deltaRadian = -1 * characterControls.rotationSpeed;
      break;
  }


  model.rotation.y += deltaRadian;

  if (!cameraControls.lock && (event.key === 'w' || event.key === 's')) {
    camera.position.add(direction);
  }
});

document.addEventListener('click', (event: MouseEvent) => {
  pointer.x = (event.clientX / threeCanvas.value!.clientWidth) * 2 - 1;
  pointer.y = - (event.clientY / threeCanvas.value!.clientHeight) * 2 + 1;

  raycaster.setFromCamera(pointer, camera);
  const intersects = raycaster.intersectObject(planeMesh, false)

  if (intersects.length > 0) {
    clickedPosition = intersects[0].point;
    if (lastAction !== 'runAction') {
      executeCrossFade(actionsMapping.get(lastAction)!, runAction, 1)
      lastAction = 'runAction';
    };
    rotateModel();
    moveModel();
  }
});

onMounted(() => {
  initThree();
  loadModel();
  initLine();
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
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
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
  const shadowSize = 20;
  dirLight.position.set(- 3, 10, - 10);
  dirLight.castShadow = true;
  dirLight.shadow.camera.top = shadowSize;
  dirLight.shadow.camera.bottom = -shadowSize;
  dirLight.shadow.camera.left = -shadowSize;
  dirLight.shadow.camera.right = shadowSize;
  dirLight.shadow.camera.near = 0.1;
  dirLight.shadow.camera.far = 40;
  scene.add(dirLight);

  const size = 10;
  const divisions = 10;
  const gridHelper = new THREE.GridHelper(size, divisions);
  scene.add(gridHelper);

  orbitControls = new OrbitControls(camera, renderer.domElement);
  // orbitControls.target = cube.position;

  planeMesh = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), new THREE.MeshPhongMaterial({ color: 0xcbcbcb, depthWrite: false }));
  planeMesh.rotation.x = - Math.PI / 2;
  planeMesh.receiveShadow = true;
  scene.add(planeMesh);

  // scene.add(new THREE.CameraHelper(dirLight.shadow.camera))

  stats = new Stats();
  stats.showPanel(0);
  threeCanvas.value!.appendChild(stats.dom);

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

    stats.update();
    TWEEN.update();

    // if (arrow) {
    //   arrow.position.copy(model.position);
    //   arrow.setDirection(model.getWorldDirection(new THREE.Vector3()));
    // }

    updateActionDropdown();

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

    // arrow = new THREE.ArrowHelper(model.getWorldDirection(new THREE.Vector3()), model.position, 1, 0xff0000);
    // scene.add(arrow);

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

function moveModel() {
  if (moveTween) moveTween.stop();

  moveTween = new TWEEN.Tween(model.position)
    .to(clickedPosition, 1500)
    .start();
}

function rotateModel() {
  if (rotateTween) rotateTween.stop();

  const modelDirection = new THREE.Vector3();
  model.getWorldDirection(modelDirection);
  modelDirection.multiplyScalar(-1);
  const targetDirection = new THREE.Vector3().subVectors(clickedPosition, model.position).normalize();
  const angle = modelDirection.angleTo(targetDirection);
  const axis = new THREE.Vector3().crossVectors(modelDirection, targetDirection).normalize();
  let signedAngle = axis.y < 0 ? -angle : angle;
  // model.rotateOnWorldAxis(axis, angle);

  rotateTween = new TWEEN.Tween(model.rotation)
    .to({ y: model.rotation.y + signedAngle }, 500)
    .onUpdate((object) => {
      model.rotation.y = object.y;
    })
    .start();
}

function initLine() {
  const lineMaterial = new LineMaterial({
    color: 0x00AA00,
    linewidth: 10,
    alphaToCoverage: false,
    resolution: new THREE.Vector2(threeCanvas.value!.clientWidth, threeCanvas.value!.clientHeight),
  });
  const geometry = new LineGeometry();
  const positions = [-5, 0, -5, -5, 0, 5, 5, 0, 5, 5, 0, -5, -5, 0, -5]
  geometry.setPositions(positions);
  var line = new Line2(geometry, lineMaterial);
  scene.add(line);
}

function updateActionDropdown() {
  if (characterControls.runWeight === 1) {
    characterControls.action = 'Run';
  } else if (characterControls.walkWeight === 1) {
    characterControls.action = 'Walk';
  } else {
    characterControls.action = 'Idle';
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
