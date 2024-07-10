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

  let isDragging = false;
  const rotationSpeed = new THREE.Vector2(0.003, 0.007);
  let startEvent = null;
  let rotationPoint = new THREE.Vector3(); 
  let intersectedPoint = new THREE.Vector3(); 
  let initialRotation = new THREE.Euler(); 

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
      initialRotation.copy(cube.rotation); 
      intersectedPoint.copy(intersects[0].point); 
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

    rotateCube(delta);
    startEvent = event; 
  }

  function rotateCube(delta) {
    const angleX = delta.x * rotationSpeed.x;
    const angleY = delta.y * rotationSpeed.y;

    const quaternionX = new THREE.Quaternion();
    const quaternionY = new THREE.Quaternion();

    quaternionX.setFromAxisAngle(new THREE.Vector3(0, 1, 0), angleX);
    quaternionY.setFromAxisAngle(new THREE.Vector3(1, 0, 0), angleY);

    cube.position.sub(rotationPoint); 
    cube.position.applyQuaternion(quaternionX);
    cube.position.applyQuaternion(quaternionY);
    cube.position.add(rotationPoint);

    cube.rotation.setFromRotationMatrix(new THREE.Matrix4().makeRotationFromQuaternion(quaternionX.multiply(quaternionY)));
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
