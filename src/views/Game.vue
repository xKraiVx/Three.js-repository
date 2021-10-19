/* eslint-disable */
<template>
  <div ref="wrapper" class="game">
    <canvas ref="canvas"></canvas>
  </div>
</template>
<script>
import * as THREE from "three";
import * as CANNON from "cannon-es";

export default {
  data: () => ({
    scene: null,
    world: null,
    wrapperSizes: {
      width: 0,
      height: 0,
    },
    defaultMaterial: null,
    camera: null,
    cameraPosition: {
      x: -2,
      y: 7,
      z: 10,
    },
    renderer: null,
    quadcopter: {
      mesh: null,
      body: null,
      parametrs: {
        size: {
          width: 1,
          height: 0.5,
          depth: 1,
        },
        metalness: 0.4,
        roughness: 0.4,
      },
    },
    floor: {
      mesh: null,
      body: null,
      size: {
        width: 3,
        leng: 100,
      },
    },
    objectsToUpdate: [],
    clock: 0,
    oldElapsedTime: 0,
  }),
  mounted() {
    /**
     * Base
     */

    //Canvas
    const canvas = this.$refs.canvas;
    //Sizes
    this.wrapperSizes.width = this.$refs.wrapper.clientWidth;
    this.wrapperSizes.height = this.$refs.wrapper.clientHeight;
    //Scene
    const scene = new THREE.Scene();
    this.scene = scene;

    /**
     * Camera
     */

    this.createCamera();

    /**
     * Textures
     */

    /**
     * Light
     */
    this.createLight();

    // World
    this.createWorld();

    //Floor
    this.createFloor();

    //Quadcopter
    this.createQuadrocopter();

    //save to objects to update

    /* Renderer */
    this.renderer = new THREE.WebGLRenderer({
      canvas: canvas,
    });

    this.renderer.setSize(this.wrapperSizes.width, this.wrapperSizes.height);
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    window.addEventListener("resize", this.resizeUpdate);

    //Keypress event
    document.addEventListener("keypress", this.handleKeypress);

    this.clock = new THREE.Clock();

    const tick = () => {
      const elapsedTime = this.clock.getElapsedTime();
      const deltaTime = elapsedTime - this.oldElapsedTime;
      this.oldElapsedTime = elapsedTime;

      //Update physics world

      this.world.step(1 / 60, deltaTime, 3);
      for (const object of this.objectsToUpdate) {
        object.mesh.position.copy(object.body.position);
        object.mesh.quaternion.copy(object.body.quaternion);
      }

      //Moving of quadcopter
      this.quadcopter.body.applyLocalForce(
        new CANNON.Vec3(2, 0, 0),
        new CANNON.Vec3(0, 0, 0)
      );
      this.camera.position.x = this.quadcopter.body.position.x;

      // Render
      this.renderer.render(scene, this.camera);

      // Call tick again on the next frame
      window.requestAnimationFrame(tick);
    };
    tick();
  },
  beforeUnmount() {
    window.removeEventListener("resize", this.resizeUpdate);
    document.removeEventListener("keypress", this.handleKeypress);
  },
  methods: {
    handleKeypress(e) {
      if (e.code === "Space") {
        this.quadcopter.body.applyLocalImpulse(
          new CANNON.Vec3(0, 5, 0),
          new CANNON.Vec3(0, 0, 0)
        );
      }
    },
    createCamera() {
      const width = this.wrapperSizes.width,
        height = this.wrapperSizes.height;

      this.camera = new THREE.PerspectiveCamera(75, width / height);
      this.camera.position.set(
        this.cameraPosition.x,
        this.cameraPosition.y,
        this.cameraPosition.z
      );
      this.scene.add(this.camera);
    },
    createLight() {
      //Ambient light
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
      this.scene.add(ambientLight);

      //Directional light
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.3);
      directionalLight.castShadow = true;
      directionalLight.shadow.mapSize.set(1024, 1024);
      directionalLight.shadow.camera.far = 15;
      directionalLight.shadow.camera.left = -7;
      directionalLight.shadow.camera.top = 7;
      directionalLight.shadow.camera.right = 7;
      directionalLight.shadow.camera.bottom = -7;
      directionalLight.position.set(-1, 3, 5);
      this.scene.add(directionalLight);
    },
    createWorld() {
      this.world = new CANNON.World();
      this.world.gravity.set(0, -9.82, 0);
      this.world.broadphase = new CANNON.SAPBroadphase(this.world);
      this.world.allowSleep = true;

      //Material
      this.defaultMaterial = new CANNON.Material("default");
      const defaultContactMaterial = new CANNON.ContactMaterial(
        this.defaultMaterial,
        this.defaultMaterial,
        {
          friction: 0.7,
          restitution: 0.3,
        }
      );

      this.world.addContactMaterial(defaultContactMaterial);
      this.world.defaultContactMaterial = defaultContactMaterial;
    },
    createFloor() {
      //Mesh
      const floorGeometry = new THREE.PlaneBufferGeometry(
        this.floor.size.leng,
        this.floor.size.width
      );
      const floorMaterial = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.quadcopter.parametrs.metalness,
        roughness: this.quadcopter.parametrs.roughness,
      });
      this.floor.mesh = new THREE.Mesh(floorGeometry, floorMaterial);
      this.floor.mesh.rotation.x = -Math.PI / 2;
      this.scene.add(this.floor.mesh);

      //Body
      const floorShape = new CANNON.Plane();
      this.floor.body = new CANNON.Body();
      this.floor.body.mass = 0;
      this.floor.body.addShape(floorShape);
      this.floor.body.quaternion.setFromAxisAngle(
        new CANNON.Vec3(-1, 0, 0),
        Math.PI * 0.5
      );
      this.world.addBody(this.floor.body);
    },
    createQuadrocopter() {
      //Mesh
      const geometry = new THREE.BoxBufferGeometry(
        this.quadcopter.parametrs.size.width,
        this.quadcopter.parametrs.size.height,
        this.quadcopter.parametrs.size.depth
      );
      const material = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.quadcopter.parametrs.metalness,
        roughness: this.quadcopter.parametrs.roughness,
      });

      this.quadcopter.mesh = new THREE.Mesh(geometry, material);
      this.quadcopter.mesh.position.y = 5;
      this.scene.add(this.quadcopter.mesh);

      //Body
      const shapeQuadcopter = new CANNON.Box(
        new CANNON.Vec3(
          this.quadcopter.parametrs.size.width,
          this.quadcopter.parametrs.size.height,
          this.quadcopter.parametrs.size.depth
        )
      );
      this.quadcopter.body = new CANNON.Body({
        mass: 1,
        position: new CANNON.Vec3(0, 3, 0),
        shape: shapeQuadcopter,
        material: this.defaultMaterial,
      });
      this.quadcopter.body.position.copy(this.quadcopter.mesh.position);
      this.world.addBody(this.quadcopter.body);

      //Add to objects to update
      const quadcopterMesh = this.quadcopter.mesh;
      const quadcopterBody = this.quadcopter.body;
      this.objectsToUpdate.push({
        mesh: quadcopterMesh,
        body: quadcopterBody,
      });
    },
    resizeUpdate() {
      if (!this.camera || !this.renderer) {
        return;
      }
      // Update sizes
      this.wrapperSizes.width = this.$refs.wrapper.clientWidth;
      this.wrapperSizes.height = this.$refs.wrapper.clientHeight;

      const width = this.wrapperSizes.width,
        height = this.wrapperSizes.height;

      // Update camera
      this.camera.aspect = width / height;
      this.camera.updateProjectionMatrix();

      // Update renderer
      this.renderer.setSize(width, height);
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    },
  },
};
</script>
<style lang="sass">
@import "../assets/scss/pages/game.scss"
</style>