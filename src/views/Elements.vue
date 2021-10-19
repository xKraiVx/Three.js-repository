/* eslint-disable */
<template>
  <div ref="wrapper" class="elements">
    Elements
    <canvas ref="canvas"></canvas>
  </div>
</template>
<script>
import * as THREE from "three";

export default {
  data: () => ({
    canvas: null,
    scene: null,
    camera: null,
    renderer: null,
    controls: null,
    sizes: {
      width: null,
      height: null,
    },
    meshes: {
      box: null,
    },
  }),
  mounted() {
    // Canvas
    const canvas = this.$refs.canvas;

    // Sizes
    const sizes = {
      width: innerWidth,
      height: innerHeight,
    };

    // Scene
    const scene = new THREE.Scene();

    // Camera
    const camera = new THREE.PerspectiveCamera(75, sizes.width / sizes.height);
    camera.position.z = 3;
    scene.add(camera);

    //Meshes
    const cube = new THREE.Mesh(
      new THREE.BoxGeometry(1, 1, 1),
      new THREE.MeshBasicMaterial({ color: 0xff0000 })
    );
    scene.add(cube);

    // Renderer
    const renderer = new THREE.WebGLRenderer({
      canvas: canvas,
    });
    renderer.setSize(sizes.width, sizes.height);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    /* window.addEventListener("resize", () => {
      resizeUpdate();
    }); */
    /* const clock = new THREE.Clock(); */

    const tick = () => {
      /* const elapsedTime = clock.getElapsedTime(); */

      // Update controls

      // Render
      renderer.render(scene, camera);

      // Call tick again on the next frame
      window.requestAnimationFrame(tick);
    };

    tick();
  },
  beforeUnmount() {
    window.removeEventListener("resize", () => {
      this.resizeUpdate();
    });
  },
  methods: {
    resizeUpdate() {
      // Update sizes
      this.sizes.width = window.innerWidth;
      this.sizes.height = window.innerHeight;

      // Update camera
      this.camera.aspect = this.sizes.width / this.sizes.height;
      this.camera.updateProjectionMatrix();

      // Update renderer
      this.renderer.setSize(this.sizes.width, this.sizes.height);
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    },
    boxInit() {
      this.meshes.box = new THREE.Mesh(
        new THREE.BoxBufferGeometry(1, 1, 1),
        new THREE.MeshBasicMaterial({ color: "red" })
      );
      this.scene.add(this.meshes.box);
    },
  },
};
</script>
<style lang="sass">
@import "../assets/scss/pages/elements.scss"
</style>