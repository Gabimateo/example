<template>
  <div ref="container"></div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import * as THREE from "three";
import * as BUI from "@thatopen/ui";
import * as OBC from "@thatopen/components";
import Stats from "stats.js";

const container = ref(null);
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);

const world = worlds.create(
  OBC.SimpleScene,
  OBC.SimpleCamera,
  OBC.SimpleRenderer
);

onMounted(() => {
  world.scene = new OBC.SimpleScene(components);
  world.renderer = new OBC.SimpleRenderer(components, container.value);
  world.camera = new OBC.SimpleCamera(components);

  components.init();
  world.scene.three.background = null;

  const material = new THREE.MeshLambertMaterial({ color: "#6528D7" });
  const geometry = new THREE.BoxGeometry();
  const cube = new THREE.Mesh(geometry, material);
  world.scene.three.add(cube);

  world.scene.setup();

  if (world.camera.controls) {
    world.camera.controls.setLookAt(3, 3, 3, 0, 0, 0);
  } else {
    console.error('Camera controls are not initialized');
  }

  const stats = new Stats();
  stats.showPanel(2);
  document.body.appendChild(stats.dom);
  stats.dom.style.left = "0px";
  stats.dom.style.zIndex = "unset";
  world.renderer.onBeforeUpdate.add(() => stats.begin());
  world.renderer.onAfterUpdate.add(() => stats.end());

  BUI.Manager.init();

  let isDragging = false;
  const rotationSpeed = new THREE.Vector2(0.003, 0.007);
  let startEvent = null;
  let rotationPoint = new THREE.Vector3(); 
  let initialCubeQuaternion = new THREE.Quaternion(); 
  let initialCubePosition = new THREE.Vector3();

  function onPointerDown(event) {
    isDragging = true;
    startEvent = event;

    const rect = container.value.getBoundingClientRect();
    const mouse = new THREE.Vector2(
      ((event.clientX - rect.left) / rect.width) * 2 - 1,
      -((event.clientY - rect.top) / rect.height) * 2 + 1
    );
    const raycaster = new THREE.Raycaster();
    raycaster.setFromCamera(mouse, world.camera.three);

    const intersects = raycaster.intersectObject(cube);
    if (intersects.length > 0) {
      rotationPoint.copy(intersects[0].point); 
      initialCubeQuaternion.copy(cube.quaternion);
      initialCubePosition.copy(cube.position);
    }
  }

  function onPointerUp() {
    isDragging = false;
  }

  function onPointerMove(event) {
    if (!isDragging) return;

    const delta = new THREE.Vector2(
      event.clientX - startEvent.clientX,
      event.clientY - startEvent.clientY
    );

    rotateModel(delta);
    startEvent = event; 
  }

  function rotateModel(delta) {
    const angleX = delta.x * rotationSpeed.x;
    const angleY = delta.y * rotationSpeed.y;

    const quaternionX = new THREE.Quaternion();
    const quaternionY = new THREE.Quaternion();
    quaternionX.setFromAxisAngle(new THREE.Vector3(0, 1, 0), angleX);
    quaternionY.setFromAxisAngle(new THREE.Vector3(1, 0, 0), angleY);

    const combinedQuaternion = new THREE.Quaternion();
    combinedQuaternion.multiplyQuaternions(quaternionY, quaternionX);
    combinedQuaternion.multiply(initialCubeQuaternion);

    cube.quaternion.copy(combinedQuaternion);

    const pivotMatrix = new THREE.Matrix4();
    pivotMatrix.makeTranslation(-rotationPoint.x, -rotationPoint.y, -rotationPoint.z);
    pivotMatrix.multiply(new THREE.Matrix4().makeRotationFromQuaternion(combinedQuaternion));
    pivotMatrix.multiply(new THREE.Matrix4().makeTranslation(rotationPoint.x, rotationPoint.y, rotationPoint.z));

    const newCubePosition = new THREE.Vector3();
    newCubePosition.copy(initialCubePosition).applyMatrix4(pivotMatrix);
    cube.position.copy(newCubePosition);
  }

  container.value.addEventListener('pointerdown', onPointerDown);
  container.value.addEventListener('pointerup', onPointerUp);
  container.value.addEventListener('pointermove', onPointerMove);

  onUnmounted(() => {
    container.value.removeEventListener('pointerdown', onPointerDown);
    container.value.removeEventListener('pointerup', onPointerUp);
    container.value.removeEventListener('pointermove', onPointerMove);
  });
});
</script>

<style scoped>
div {
  width: 100vw;
  height: 100vh;
}
</style>
