/* eslint-disable */
<template>
  <div ref="wrapper" class="game">
    <transition name="fade">
      <popup v-if="popup.state" :content="popup.content" />
    </transition>
    <div class="distance">{{ distanceCounter }}</div>
    <canvas ref="canvas"></canvas>
  </div>
</template>
<script>
import * as THREE from "three";
import * as CANNON from "cannon-es";
import Popup from "../components/Popup.vue";
import dataJSON from "../data/data.json";

export default {
  data: () => ({
    data: dataJSON,
    popup: {
      state: false,
      content: null,
      timeout: 3000,
    },
    state: "beforeInit",
    scene: null,
    world: null,
    gravity: 0,
    velocity: 0,
    wrapperSizes: {
      width: 0,
      height: 0,
    },
    defaultMaterial: null,
    camera: null,
    cameraPosition: {
      x: -2,
      y: 6,
      z: 20,
    },
    renderer: null,
    quadcopter: {
      model: null,
      mesh: null,
      body: null,
      parametrs: {
        positionY: 5,
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
      parametrs: {
        size: {
          width: 10,
          leng: 1000,
          height: 10,
        },
      },
    },
    ceiling: {
      mesh: null,
      body: null,
    },
    windmillOffice: {
      mesh: null,
      raycaster: null,
      parametrs: {
        size: {
          width: 12,
          height: 10,
          depth: 10,
        },
        metalness: 0.4,
        roughness: 0.4,
      },
    },
    customersOffice: {
      mesh: null,
      raycaster: null,
      parametrs: {
        size: {
          width: 12,
          height: 10,
          depth: 10,
        },
        metalness: 0.4,
        roughness: 0.4,
      },
    },
    walls: {
      objects: [],
      parametrs: {
        size: {
          x: 1,
          y: 5,
          z: 10,
        },
        metalness: 0.4,
        roughness: 0.4,
      },
    },
    objectsToUpdate: [],
    clock: 0,
    oldElapsedTime: 0,
    distance: 0,
    objLoader: null,
  }),
  created() {
    this.data = dataJSON;
    //Popup
    this.popup.content = this.data.popups.start;
    this.popup.state = true;
  },
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

    this.createCamera();

    this.createLight();

    this.createWorld();

    this.createFloor(this.floor.parametrs.size);

    this.createCeiling(this.floor.parametrs.size);

    this.createQuadrocopter(
      this.quadcopter.parametrs.size,
      this.quadcopter.parametrs.positionY
    );

    this.createWindmillOffice();

    this.createCustomersOffice();

    this.createWall(
      this.walls.parametrs.size,
      new THREE.Vector3(20, this.walls.parametrs.size.y / 2, 0)
    );

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

      //Raycaster

      //start

      const intersectStart = this.windmillOffice.raycaster.intersectObject(
        this.quadcopter.mesh
      );

      if (intersectStart.length) {
        this.gravity = -9.82;
        this.world.gravity.set(0, this.gravity, 0);
        this.state = "start";
      }

      //finish
      const intersectFinish = this.customersOffice.raycaster.intersectObject(
        this.quadcopter.mesh
      );

      if (intersectFinish.length) {
        document.removeEventListener("keypress", this.handleKeypress);
        this.state = "finish";
        setTimeout(() => {
          document.addEventListener("keypress", this.handleKeypress);
          this.popup.content = this.data.popups.finish;
          this.popup.state = true;
        }, this.popup.timeout);
      }

      //Update physics world

      this.world.step(1 / 60, deltaTime, 3);
      for (const object of this.objectsToUpdate) {
        object.mesh.position.copy(object.body.position);
        object.mesh.quaternion.copy(object.body.quaternion);
      }

      //Moving of quadcopter
      this.quadcopter.body.applyLocalForce(
        new CANNON.Vec3(this.velocity, 0, 0),
        new CANNON.Vec3(0, 0, 0)
      );
      if (this.state !== "finish") {
        this.camera.position.x = this.quadcopter.body.position.x;
      }
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
  computed: {
    distanceCounter() {
      if (!this.camera) {
        return 0;
      }
      return this.camera.position.x.toFixed();
    },
  },
  methods: {
    handleKeypress(e) {
      if (e.code === "Space") {
        if (this.popup.state) {
          this.popup.state = false;
        }
        if (this.state === "beforeInit") {
          this.velocity = 1;
          this.state = "init";
        } else if (this.state === "start") {
          this.quadcopter.body.applyLocalImpulse(
            new CANNON.Vec3(0, 5, 0),
            new CANNON.Vec3(0, 0, 0)
          );
        }
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
      this.world.gravity.set(0, this.gravity, 0);
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
    createCeiling(size) {
      //Mesh
      const ceilingGeometry = new THREE.BoxBufferGeometry(
        size.leng + 100,
        size.width,
        size.height
      );
      const ceilingMaterial = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.quadcopter.parametrs.metalness,
        roughness: this.quadcopter.parametrs.roughness,
      });
      this.ceiling.mesh = new THREE.Mesh(ceilingGeometry, ceilingMaterial);
      this.ceiling.mesh.position.x = size.leng / 2;
      this.ceiling.mesh.position.y = size.height * 1.5;
      this.ceiling.mesh.rotation.x = -Math.PI / 2;
      this.scene.add(this.ceiling.mesh);

      //Body
      const ceilingShape = new CANNON.Plane();
      this.ceiling.body = new CANNON.Body();
      this.ceiling.body.mass = 0;
      this.ceiling.body.addShape(ceilingShape);
      this.ceiling.body.quaternion.setFromAxisAngle(
        new CANNON.Vec3(-1, 0, 0),
        -Math.PI * 0.5
      );
      this.ceiling.body.position.y = size.height;
      console.log(this.ceiling.body.position);
      this.world.addBody(this.ceiling.body);
    },
    createFloor(size) {
      //Mesh
      const floorGeometry = new THREE.BoxBufferGeometry(
        size.leng + 100,
        size.width,
        size.height
      );
      const floorMaterial = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.quadcopter.parametrs.metalness,
        roughness: this.quadcopter.parametrs.roughness,
      });
      this.floor.mesh = new THREE.Mesh(floorGeometry, floorMaterial);
      this.floor.mesh.position.x = size.leng / 2;
      this.floor.mesh.position.y = -size.height / 2;
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
    createQuadrocopter(size, positionY) {
      //Mesh
      const geometry = new THREE.BoxBufferGeometry(
        size.width,
        size.height,
        size.depth
      );
      const material = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.quadcopter.parametrs.metalness,
        roughness: this.quadcopter.parametrs.roughness,
      });

      this.quadcopter.mesh = new THREE.Mesh(geometry, material);
      this.quadcopter.mesh.position.y = positionY;
      this.scene.add(this.quadcopter.mesh);

      //Body
      const shapeQuadcopter = new CANNON.Box(
        new CANNON.Vec3(size.width / 2, size.height / 2, size.depth / 2)
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
    createWindmillOffice() {
      const width = this.windmillOffice.parametrs.size.width + 30,
        height = this.windmillOffice.parametrs.size.height,
        depth = this.windmillOffice.parametrs.size.depth;

      //Mesh
      const geometry = new THREE.BoxBufferGeometry(width, height, depth);
      const material = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.windmillOffice.parametrs.metalness,
        roughness: this.windmillOffice.parametrs.roughness,
      });

      this.windmillOffice.mesh = new THREE.Mesh(geometry, material);
      this.windmillOffice.mesh.position.y = height / 2;
      this.windmillOffice.mesh.position.x = -15;
      this.scene.add(this.windmillOffice.mesh);

      //Raycaster
      this.windmillOffice.raycaster = new THREE.Raycaster();
      const rayOrigin = new THREE.Vector3(
        this.windmillOffice.parametrs.size.width / 2 + 1,
        this.quadcopter.parametrs.positionY,
        0
      );
      const rayDiraction = new THREE.Vector3(1, 1, 0);
      rayDiraction.normalize();

      this.windmillOffice.raycaster.set(rayOrigin, rayDiraction);
    },
    createCustomersOffice() {
      const width = this.customersOffice.parametrs.size.width + 30,
        height = this.customersOffice.parametrs.size.height,
        depth = this.customersOffice.parametrs.size.depth,
        endOfWay = this.floor.parametrs.size.leng;

      //Mesh
      const geometry = new THREE.BoxBufferGeometry(width, height, depth);
      const material = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.customersOffice.parametrs.metalness,
        roughness: this.customersOffice.parametrs.roughness,
      });

      this.customersOffice.mesh = new THREE.Mesh(geometry, material);
      this.customersOffice.mesh.position.y = height / 2;
      this.customersOffice.mesh.position.x = endOfWay + 15;
      this.scene.add(this.customersOffice.mesh);

      //Raycaster
      this.customersOffice.raycaster = new THREE.Raycaster();
      const rayOrigin = new THREE.Vector3(
        endOfWay,
        this.quadcopter.parametrs.positionY,
        0
      );
      const rayDiraction = new THREE.Vector3(1, 1, 0);
      rayDiraction.normalize();

      this.customersOffice.raycaster.set(rayOrigin, rayDiraction);
    },
    createWall(size, position) {
      //Mesh
      const geometry = new THREE.BoxBufferGeometry(size.x, size.y, size.z);
      const material = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.walls.parametrs.metalness,
        roughness: this.walls.parametrs.roughness,
      });
      const mesh = new THREE.Mesh(geometry, material);
      mesh.position.copy(position);
      this.scene.add(mesh);

      //Body
      const shapeWalls = new CANNON.Box(
        new CANNON.Vec3(size.x / 2, size.y / 2, size.z / 2)
      );
      const body = new CANNON.Body({
        mass: 0,
        position: new CANNON.Vec3(0, 0, 0),
        shape: shapeWalls,
        material: this.defaultMaterial,
      });
      body.position.copy(position);
      this.world.addBody(body);
      this.walls.objects.push({ mesh, body });
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
  components: {
    Popup,
  },
};
</script>
<style lang="sass">
@import "../assets/scss/pages/game.scss"
</style>