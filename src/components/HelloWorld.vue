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

// Crear el mundo 3D
const world = worlds.create(
  OBC.SimpleScene,
  OBC.SimpleCamera,
  OBC.SimpleRenderer
);

// Funciones de inicialización y manejo de eventos
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

  // Variables para la manipulación del modelo
  let isDragging = false;
  const rotationSpeed = new THREE.Vector2(0.003, 0.007);
  let startEvent = null;
  let rotationPoint = new THREE.Vector3(); // Punto de rotación

  // Funciones de control
  function onPointerDown(event) {
    isDragging = true;
    startEvent = event;

    // Obtener la posición del punto en el cubo
    const rect = container.value.getBoundingClientRect();
    const mouse = new THREE.Vector2(
      ((event.clientX - rect.left) / rect.width) * 3 - 1,
      -((event.clientY - rect.top) / rect.height) * 3 + 1
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

    rotateCamera(delta);
    startEvent = event; // Actualizar el evento de inicio
  }

  function rotateCamera(delta) {
    const angleX = delta.x * rotationSpeed.x;
    const angleY = delta.y * rotationSpeed.y;

    // Calcular la rotación de la cámara alrededor del punto de intersección
    const axis = new THREE.Vector3(0, 1, 0); // Eje de rotación

    world.camera.three.position.sub(rotationPoint); // Mover la cámara al origen del sistema de coordenadas
    world.camera.three.position.applyAxisAngle(axis, angleX); // Rotar alrededor del eje Y
    world.camera.three.position.applyAxisAngle(axis, angleY); // Rotar alrededor del eje X
    world.camera.three.position.add(rotationPoint); // Devolver la cámara a la posición original
    world.camera.three.lookAt(rotationPoint); // Mantener la cámara mirando hacia el punto de intersección
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
