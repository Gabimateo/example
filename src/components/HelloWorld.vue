<template>
  <div ref="container"></div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import * as THREE from "three";
import * as BUI from "@thatopen/ui";
import * as OBC from "@thatopen/components";
import Stats from "stats.js";

// Utilizamos ref para crear una referencia al contenedor
const container = ref(null);
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);

const world = worlds.create(
  OBC.SimpleScene,
  OBC.SimpleCamera,
  OBC.SimpleRenderer
);

onMounted(() => {
  // Configuración del mundo
  world.scene = new OBC.SimpleScene(components);
  world.renderer = new OBC.SimpleRenderer(components, container.value); // Accedemos al valor del ref
  world.camera = new OBC.SimpleCamera(components);

  components.init();
  world.scene.three.background = null;

  // Crear un cubo
  const material = new THREE.MeshLambertMaterial({ color: "#6528D7" });
  const geometry = new THREE.BoxGeometry();
  const cube = new THREE.Mesh(geometry, material);
  world.scene.three.add(cube);

  world.scene.setup();

  // Asegurarse de que la cámara tiene un world
  if (world.camera.controls) {
    world.camera.controls.setLookAt(3, 3, 3, 0, 0, 0);
  } else {
    console.error('Camera controls are not initialized');
  }

  // Configuración de Stats
  const stats = new Stats();
  stats.showPanel(2);
  document.body.appendChild(stats.dom);
  stats.dom.style.left = "0px";
  stats.dom.style.zIndex = "unset";
  world.renderer.onBeforeUpdate.add(() => stats.begin());
  world.renderer.onAfterUpdate.add(() => stats.end());

  // Inicializar BUI Manager
  BUI.Manager.init();

  // Variables para la manipulación del modelo
  let isDragging = false;
  const rotationSpeed = new THREE.Vector2(0.003, 0.007);
  let startEvent = null;
  let rotationPoint = new THREE.Vector3(0, 0, 0); // Inicialmente en el origen

  // Funciones de control
  function onPointerDown(event) {
    isDragging = true;
    startEvent = event;

    // Obtener la posición del punto en el cubo
    const rect = container.value.getBoundingClientRect();
    const mouse = new THREE.Vector2(
      ((event.clientX - rect.left) / rect.width) * 2 - 1,
      -((event.clientY - rect.top) / rect.height) * 2 + 1
    );
    const raycaster = new THREE.Raycaster();
    raycaster.setFromCamera(mouse, world.camera.three);

    const intersects = raycaster.intersectObject(cube);
    if (intersects.length > 0) {
      rotationPoint.copy(intersects[0].point); // Guardar el punto de intersección
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

    const sphericalDirection = directionToSpherical(
      new THREE.Vector3(),
      new THREE.Vector3(0, 1, 0)
    );

    translateModel(delta, sphericalDirection);
    startEvent = event; // Actualizar el evento de inicio
  }

  function directionToSpherical(direction, up) {
    const spherical = new THREE.Spherical();
    spherical.setFromVector3(direction);
    spherical.phi = Math.max(0.02, Math.min(spherical.phi, Math.PI - 0.02)); // Limitar el ángulo phi
    return spherical;
  }

  function translateModel(delta, sphericalDirection) {
    sphericalDirection.phi += delta.y * rotationSpeed.y;
    sphericalDirection.theta -= delta.x * rotationSpeed.x;

    const offset = new THREE.Vector3()
      .setFromSpherical(sphericalDirection)
      .multiplyScalar(rotationPoint.distanceTo(world.camera.three.position));
    cube.position.copy(rotationPoint).add(offset);
    cube.lookAt(rotationPoint);
  }

  // Eventos del mouse
  container.value.addEventListener('pointerdown', onPointerDown);
  container.value.addEventListener('pointerup', onPointerUp);
  container.value.addEventListener('pointermove', onPointerMove);

  // Cleanup
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
