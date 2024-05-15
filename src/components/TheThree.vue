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
import { CSS3DRenderer, CSS3DObject } from 'three/addons/renderers/CSS3DRenderer.js';
import * as SkeletonUtils from 'three/addons/utils/SkeletonUtils.js';

interface ModelData {
  mixer: THREE.AnimationMixer,
  model: THREE.Object3D,
  rotateTween?: TWEEN.Tween<THREE.Euler>,
  moveTween?: TWEEN.Tween<THREE.Vector3>,
  isWalking: boolean,
}

const threeCanvas = ref<HTMLDivElement>();
const card = ref<HTMLDivElement>();
let camera: THREE.PerspectiveCamera,
  scene: THREE.Scene,
  renderer: THREE.WebGLRenderer,
  orbitControls: OrbitControls,
  cube: THREE.Mesh,
  stats: Stats;
let model: THREE.Object3D,
  modelAnimations: THREE.AnimationClip[],
  mixer: THREE.AnimationMixer,
  idleAction: THREE.AnimationAction,
  walkAction: THREE.AnimationAction,
  runAction: THREE.AnimationAction,
  actionsMapping: Map<string, THREE.AnimationAction>,
  clock: THREE.Clock = new THREE.Clock(),
  isRunInCircles = false,
  circlePoints: THREE.Vector3[];
let raycaster: THREE.Raycaster = new THREE.Raycaster(),
  enableRaycasting = true,
  pointer = new THREE.Vector2(),
  planeMesh: THREE.Mesh,
  // arrow: THREE.ArrowHelper,
  clickedPosition: THREE.Vector3,
  moveTween: TWEEN.Tween<THREE.Vector3>,
  rotateTween: TWEEN.Tween<THREE.Euler>;
let modelDatas: ModelData[] = [];

let scene2: THREE.Scene,
  css3DRenderer: CSS3DRenderer;

const cubeControls = {
  x: 0,
  y: 0,
  z: 0,
  visible: false,
};

const cameraControls = {
  lock: false
};

const characterControls = {
  action: 'Idle',
  idleWeight: 1,
  walkWeight: 0,
  runWeight: 0,
  speed: 0.1,
  rotationSpeed: 0.1,
  runInCircles: runInCircles,
  addModel: addModel,
}

const gui = new GUI();
const cameraFolder = gui.addFolder('Camera');
cameraFolder.add(cameraControls, 'lock').onChange((value: boolean) => {
  cameraControls.lock = value;
  if (value) model.add(camera);
  else scene.add(camera);
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
characterFolder.add(characterControls, 'runInCircles').name('Run in Circles');
characterFolder.add(characterControls, 'addModel').name('Add Model');

gui.domElement.addEventListener('mouseenter', () => {
  enableRaycasting = false;
});

gui.domElement.addEventListener('mouseleave', () => {
  enableRaycasting = true;
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
});

document.addEventListener('click', (event: MouseEvent) => {
  if (!enableRaycasting) return;
  if (isRunInCircles) isRunInCircles = false;

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
    rotateModel(clickedPosition, model, rotateTween);
    moveModel(clickedPosition, model, moveTween);
  }
});

onMounted(() => {
  initThree();
  loadModel();
  initLine();

  initCss3DRenderer();
  create3dCard(card.value!, 100, 100, new THREE.Vector3(0, 3, -5), new THREE.Euler(0, 0, 0));

  animate();
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

  // orbitControls = new OrbitControls(camera, renderer.domElement);  // OrbitControls for WebGLRenderer
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
}

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
  render2();

  if (isRunInCircles) {
    if (model.position.distanceTo(circlePoints[0]) < 0.1) {
      circlePoints.push(circlePoints.shift()!);
      rotateModel(circlePoints[0], model, rotateTween);
      moveModel(circlePoints[0], model, moveTween);
    }
  }

  for (const modelData of modelDatas) {
    modelData.mixer.update(mixerUpdateDelta);
    if (!modelData.isWalking) {
      randomWalk(modelData);
    }
  }
}

function render() {
  renderer.render(scene, camera);
}

function loadModel() {
  const loader = new GLTFLoader();
  loader.load('3dModels/Soldier.glb', (gltf) => {
    model = gltf.scene;
    modelAnimations = gltf.animations;
    scene.add(model);

    orbitControls.target.copy(model.position);

    mixer = new THREE.AnimationMixer(model);
    idleAction = mixer.clipAction(modelAnimations[0]);
    walkAction = mixer.clipAction(modelAnimations[3]);
    runAction = mixer.clipAction(modelAnimations[1]);

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

function moveModel(destination: THREE.Vector3, model: THREE.Object3D, moveTween?: TWEEN.Tween<THREE.Vector3>, modelData?: ModelData, duration = 1500) {
  if (moveTween) moveTween.stop();

  const newMoveTween = new TWEEN.Tween(model.position)
    .to(destination, duration)
    .start();

  if (modelData) modelData.moveTween = newMoveTween;
  else moveTween = newMoveTween;
}

function rotateModel(destination: THREE.Vector3, model: THREE.Object3D, rotateTween?: TWEEN.Tween<THREE.Euler>, modelData?: ModelData, duration = 500) {
  if (rotateTween) rotateTween.stop();

  const modelDirection = new THREE.Vector3();
  model.getWorldDirection(modelDirection);
  modelDirection.multiplyScalar(-1);
  const targetDirection = new THREE.Vector3().subVectors(destination, model.position).normalize();
  const angle = modelDirection.angleTo(targetDirection);
  const axis = new THREE.Vector3().crossVectors(modelDirection, targetDirection).normalize();
  let signedAngle = axis.y < 0 ? -angle : angle;
  // model.rotateOnWorldAxis(axis, angle);

  const newRotateTween = new TWEEN.Tween(model.rotation)
    .to({ y: model.rotation.y + signedAngle }, duration)
    .onUpdate((object) => {
      model.rotation.y = object.y;
    })
    .start();

  if (modelData) modelData.rotateTween = newRotateTween;
  else rotateTween = newRotateTween;
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

function runInCircles() {
  if (lastAction !== 'runAction') {
    executeCrossFade(actionsMapping.get(lastAction)!, runAction, 1)
    lastAction = 'runAction';
  };
  isRunInCircles = true;
  circlePoints = [
    new THREE.Vector3(5, 0, 5),
    new THREE.Vector3(5, 0, -5),
    new THREE.Vector3(-5, 0, -5),
    new THREE.Vector3(-5, 0, 5),
  ];
  rotateModel(circlePoints[0], model, rotateTween);
  moveModel(circlePoints[0], model, moveTween);
}

function addModel() {
  const clonedModel = SkeletonUtils.clone(model);
  const mixer = new THREE.AnimationMixer(clonedModel);
  const walkAction = mixer.clipAction(modelAnimations[3]);
  walkAction.play();
  scene.add(clonedModel);

  const modelData: ModelData = {
    mixer: mixer,
    model: clonedModel,
    isWalking: false,
  };
  modelDatas.push(modelData);
}

function randomWalk(modelData: ModelData) {
  const randomX = Math.random() * 10 - 5;
  const randomZ = Math.random() * 10 - 5;
  const destination = new THREE.Vector3(randomX, 0, randomZ);
  moveModel(destination, modelData.model, modelData.moveTween, modelData, 3000);
  rotateModel(destination, modelData.model, modelData.rotateTween, modelData);
  modelData.isWalking = true;
  if (modelData.moveTween) modelData.moveTween.onComplete(() => {
    modelData.isWalking = false;
  });
}

function initCss3DRenderer() {
  scene2 = new THREE.Scene();
  css3DRenderer = new CSS3DRenderer();
  css3DRenderer.setSize(threeCanvas.value!.clientWidth, threeCanvas.value!.clientHeight);
  css3DRenderer.domElement.style.position = 'absolute';
  css3DRenderer.domElement.style.top = '0px';
  threeCanvas.value!.appendChild(css3DRenderer.domElement);

  orbitControls = new OrbitControls(camera, css3DRenderer.domElement);
}

function create3dCard(dom: HTMLElement, width: number, height: number, position: THREE.Vector3, rotation: THREE.Euler) {
  const card = dom;
  card.style.width = `${width}px`;
  card.style.height = `${height}px`;
  card.style.backgroundColor = '#42d4f5';
  card.style.opacity = '0.8';
  card.style.borderRadius = '10px';
  const css3DObject = new CSS3DObject(card);
  css3DObject.position.copy(position);
  css3DObject.rotation.copy(rotation);
  css3DObject.scale.set(0.01, 0.01, 0.01);
  scene2.add(css3DObject);
}

function render2() {
  css3DRenderer.render(scene2, camera);
}

</script>

<template>
  <div ref="threeCanvas" class="three-container">
    <div ref="card">你好~</div>
  </div>
</template>

<style scoped>
.three-container {
  width: 100vw;
  height: 100vh;
}
</style>
