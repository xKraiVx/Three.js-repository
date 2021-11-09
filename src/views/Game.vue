<template>
  <div ref="wrapper" class="game">
    <navigation />
    <transition name="fade">
      <popup
        v-if="popup.state"
        :content="popup.content"
        :is-mobile="isTouchDevice"
      />
    </transition>
    <div class="distance">{{ distanceCounter }}</div>
    <div class="speed">{{ speed }}</div>
    <stats />
    <canvas ref="canvas"></canvas>
  </div>
</template>
<script>
import * as THREE from "three";
import { OBJLoader } from "three/examples/jsm/loaders/OBJLoader.js";
import { MTLLoader } from "three/examples/jsm/loaders/MTLLoader.js";
import * as CANNON from "cannon-es";
import Popup from "../components/Popup.vue";
import Stats from "../components/Stats.vue";
import Navigation from "../components/Navigation.vue";
import dataJSON from "../data/data.json";

export default {
  data: () => ({
    isTouchDevice: false,
    publicPath: process.env.BASE_URL,
    data: dataJSON,
    fog: {
      color: "#ffffff",
      distance: 35,
    },
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
      z: 15,
    },
    directionalLight: null,
    ambientLight: null,
    renderer: null,
    quadcopter: {
      model: {
        body: null,
        leftRotor: null,
        rightRotor: null,
      },
      mesh: null,
      body: null,
      parametrs: {
        positionY: 5,
        size: {
          width: 1,
          height: 0.5,
          depth: 4,
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
          leng: 300,
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
      building: {
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
      trapDoor: {
        parametrs: {
          size: {
            radiusTop: 3,
            radiusBottom: 3,
            height: 0.5,
            sectors: 64,
          },
          metalness: 0.4,
          roughness: 0.4,
        },
      },
    },
    customersOffice: {
      mesh: null,
      raycaster: null,
      building: {
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
      animatedPlain: {
        parametrs: {
          size: {
            height: 10,
            width: 10,
          },
          metalness: 0.4,
          roughness: 0.4,
        },
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
      optimization: {
        wallsDistance: 200,
        step: 20,
      },
    },
    objectsToUpdate: [],
    clock: 0,
    oldElapsedTime: 0,
    objLoader: null,
    speedCount: {
      maxSpeed: 10,
      speedInterval: null,
      oldTime: 0,
      oldDistance: 0,
    },
    speed: 0,
  }),
  created() {
    this.isTouchDevice = this.isMobile();
    this.oldTime = Date.now();

    this.data = dataJSON;
    //Popup
    this.popup.content = this.data.popups.start;
    this.popup.state = true;
    console.log(this.isTouchDevice);
  },
  mounted() {
    this.init();
  },
  computed: {
    distanceCounter() {
      if (!this.camera) {
        return 0;
      }

      return this.camera.position.x.toFixed();
    },
  },
  watch: {
    distanceCounter: function () {
      if (this.walls.objects.length === 0) {
        return;
      }

      const lastObjectX =
          this.walls.objects[this.walls.objects.length - 1].mesh.position.x,
        currentDistance = Number(this.distanceCounter);

      if (
        currentDistance + 30 > lastObjectX &&
        currentDistance < this.floor.parametrs.size.leng - 60
      ) {
        const positionX = Math.round(lastObjectX) + 15;

        const updateObjects = this.walls.objects.splice(0, 2);

        updateObjects.forEach((object) => {
          object.mesh.position.x = positionX;
          object.body.position.x = positionX;
          this.walls.objects.push(object);
        });
      }
    },
  },
  methods: {
    isMobile() {
      return "ontouchstart" in window;
    },
    init() {
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

      this.createFog();

      //save to objects to update

      /* Renderer */
      this.renderer = new THREE.WebGLRenderer({
        canvas: canvas,
      });
      this.renderer.setSize(this.wrapperSizes.width, this.wrapperSizes.height);
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      this.renderer.setClearColor(this.fog.color);

      this.createCustomersOffice();

      window.addEventListener("resize", this.resizeUpdate);

      //Keypress event
      document.addEventListener("keypress", this.handleKeypress);
      document.addEventListener("touchstart", this.hangleTap);

      this.clock = new THREE.Clock();

      this.createWallStack(
        this.walls.optimization.step,
        this.walls.optimization.wallsDistance
      );

      this.speedCounter();
      const tick = () => {
        const elapsedTime = this.clock.getElapsedTime();
        const deltaTime = elapsedTime - this.oldElapsedTime;
        this.oldElapsedTime = elapsedTime;

        //Raycaster

        //start
        this.start();
        //finish
        this.finish();

        //Moving of TrapDoor
        const trapDoor = this.windmillOffice.mesh.children[1];
        if (this.distanceCounter > 3 && trapDoor.rotation.z < 3.14) {
          trapDoor.rotation.z += (Math.PI * deltaTime) / 3;
          trapDoor.position.x += deltaTime / 0.5;
          trapDoor.position.z += deltaTime * 1.5;
        }

        //Update physics world

        this.world.step(1 / 60, deltaTime);
        if (this.quadcopter.model.body) {
          const body = this.quadcopter.body,
            rightRotor = this.quadcopter.model.rightRotor,
            leftRotor = this.quadcopter.model.leftRotor;
          this.quadcopter.model.body.position.copy(
            this.quadcopter.body.position
          );
          this.quadcopter.model.body.quaternion.copy(
            this.quadcopter.body.quaternion
          );

          if (this.state !== "fail") {
            rightRotor.position.x = body.position.x;
            rightRotor.position.y = body.position.y;
            rightRotor.rotation.y += Math.PI * deltaTime;

            leftRotor.position.x = body.position.x;
            leftRotor.position.y = body.position.y;
            leftRotor.rotation.y += Math.PI * deltaTime;
          } else {
            rightRotor.position.x = body.position.x;
            rightRotor.position.y = body.position.y;
            rightRotor.quaternion.copy(body.quaternion);

            leftRotor.position.x = body.position.x;
            leftRotor.position.y = body.position.y;
            leftRotor.quaternion.copy(body.quaternion);
          }
        }

        this.quadcopter.mesh.position.copy(this.quadcopter.body.position);
        this.quadcopter.mesh.quaternion.copy(this.quadcopter.body.quaternion);

        //Moving of quadcopter
        let currForse = this.velocity;
        if (this.speedCount.maxSpeed <= this.speed) {
          currForse = 0;
        }
        this.quadcopter.body.applyLocalForce(
          new CANNON.Vec3(currForse, 0, 0),
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
    handleKeypress(e) {
      if (e.code === "Space") {
        if (this.popup.state) {
          this.popup.state = false;
        }
        if (this.state === "finish" || this.state === "fail") {
          this.restart();
          return;
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
    hangleTap() {
      if (this.popup.state) {
        this.popup.state = false;
      }
      if (this.state === "finish" || this.state === "fail") {
        this.restart();
        return;
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
      this.ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
      this.scene.add(this.ambientLight);

      //Directional light
      this.directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
      this.directionalLight.position.set(-3, 10, 3);
      this.scene.add(this.directionalLight);
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
      //Model
      const OBJFile = `${this.publicPath}dron/dron_1/Drone_obj.obj`,
        MTLFile = `${this.publicPath}dron/dron_1/Drone_obj.mtl`,
        colorDroneTexture = `${this.publicPath}dron/dron_1/textures/Flying drone_col.jpg`,
        aoDroneTexture = `${this.publicPath}dron/dron_1/textures/Flying drone_Ao_2.jpg`,
        normalDroneTexture = `${this.publicPath}dron/dron_1/textures/Flying drone_Nor_2.jpg`;

      const textureLoader = new THREE.TextureLoader(),
        colorTexture = textureLoader.load(colorDroneTexture),
        aoTexture = textureLoader.load(aoDroneTexture),
        normalTexture = textureLoader.load(normalDroneTexture);

      new MTLLoader().load(MTLFile, (materials) => {
        materials.preload();
        new OBJLoader().setMaterials(materials).load(OBJFile, (object) => {
          object.rotation.y = Math.PI / 2;
          object.scale.set(4, 4, 4);

          object.traverse((child) => {
            if (child instanceof THREE.Mesh) {
              child.material.map = colorTexture;
              child.material.normalMap = normalTexture;
              child.material.aoMap = aoTexture;
            }
          });
          const dronBody = new THREE.Group();
          const dronRightRotor = new THREE.Group();
          const dronLeftRotor = new THREE.Group();

          const leftRotor = object.children.splice(2, 1)[0];
          leftRotor.scale.set(4, 4, 4);
          dronLeftRotor.add(leftRotor);
          this.quadcopter.model.leftRotor = dronLeftRotor;
          this.quadcopter.model.leftRotor.position.z = -1;
          this.quadcopter.model.leftRotor.children[0].position.x = -1;

          const rightRotor = object.children.splice(3, 1)[0];
          rightRotor.scale.set(4, 4, 4);
          dronRightRotor.add(rightRotor);
          this.quadcopter.model.rightRotor = dronRightRotor;

          this.quadcopter.model.rightRotor.position.z = 1;
          this.quadcopter.model.rightRotor.children[0].position.x = 1;

          dronBody.add(object);
          this.quadcopter.model.body = dronBody;

          this.scene.add(this.quadcopter.model.rightRotor);
          this.scene.add(this.quadcopter.model.leftRotor);
          this.scene.add(this.quadcopter.model.body);
        });
      });

      //Mesh
      const geometry = new THREE.BoxBufferGeometry(
        size.width,
        size.height,
        size.depth
      );
      /* const material = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.quadcopter.parametrs.metalness,
        roughness: this.quadcopter.parametrs.roughness,
      }); */
      const material = new THREE.MeshLambertMaterial({
        transparent: true,
      });
      material.opacity = 0;

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

      this.quadcopter.body.addEventListener(
        "collide",
        () => {
          if (this.state === "start") {
            this.fail();
          }
        },
        { once: true }
      );
    },
    createWindmillOffice() {
      const group = new THREE.Group(),
        textureLoader = new THREE.TextureLoader();

      const colorTrapDoorTexturePath = `${this.publicPath}textures/columns_textures/metal_plate_4k.blend/textures/metal_plate_diff_4k.jpg`,
        normalTrapDoorTexturePath = `${this.publicPath}textures/columns_textures/metal_plate_4k.blend/textures/metal_plate_nor_gl_4k.jpg`,
        roughnessTrapDoorTexturePath = `${this.publicPath}textures/columns_textures/metal_plate_4k.blend/textures/metal_plate_rough_4k.jpg`,
        metalnessTrapDoorTexturePath = `${this.publicPath}textures/columns_textures/metal_plate_4k.blend/textures/metal_plate_metal_4k.jpg`,
        aoTrapDoorTexturePath = `${this.publicPath}textures/columns_textures/metal_plate_4k.blend/textures/metal_plate_ao_4k.jpg`;

      //Building
      const buildingWidth =
          this.windmillOffice.building.parametrs.size.width + 30,
        buildingHeight = this.windmillOffice.building.parametrs.size.height,
        buildingDepth = this.windmillOffice.building.parametrs.size.depth;
      const buildingGeometry = new THREE.BoxBufferGeometry(
        buildingWidth,
        buildingHeight,
        buildingDepth
      );
      const buildingMaterial = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.windmillOffice.building.parametrs.metalness,
        roughness: this.windmillOffice.building.parametrs.roughness,
      });
      group.add(new THREE.Mesh(buildingGeometry, buildingMaterial));

      //TrapDoor
      const colorTrapDoorTexture = textureLoader.load(colorTrapDoorTexturePath),
        normalTrapDoorTexture = textureLoader.load(normalTrapDoorTexturePath),
        roughnessTrapDoor = textureLoader.load(roughnessTrapDoorTexturePath),
        metalnessTrapDoorTexture = textureLoader.load(
          metalnessTrapDoorTexturePath
        ),
        aoTrapDoorTexture = textureLoader.load(aoTrapDoorTexturePath);

      const trapDoorRadiusTop =
          this.windmillOffice.trapDoor.parametrs.size.radiusTop,
        trapDoorRadiusBottom =
          this.windmillOffice.trapDoor.parametrs.size.radiusBottom,
        trapDoorRadiusHeight =
          this.windmillOffice.trapDoor.parametrs.size.height,
        trapDoorSectors = this.windmillOffice.trapDoor.parametrs.size.sectors;
      const trapDoorGeometry = new THREE.CylinderGeometry(
        trapDoorRadiusTop,
        trapDoorRadiusBottom,
        trapDoorRadiusHeight,
        trapDoorSectors
      );
      const trapDoorMaterial = new THREE.MeshStandardMaterial({
        map: colorTrapDoorTexture,
        normalMap: normalTrapDoorTexture,
        roughnessMap: roughnessTrapDoor,
        metalnessMap: metalnessTrapDoorTexture,
        aoMap: aoTrapDoorTexture,
      });
      const trapDoorMesh = new THREE.Mesh(trapDoorGeometry, trapDoorMaterial);
      trapDoorMesh.position.x = buildingWidth / 2;
      trapDoorMesh.rotation.x = Math.PI / 2;
      trapDoorMesh.rotation.z = Math.PI / 2;
      group.add(trapDoorMesh);

      this.windmillOffice.mesh = group;
      this.windmillOffice.mesh.position.y = buildingHeight / 2;
      this.windmillOffice.mesh.position.x = -15;
      this.scene.add(this.windmillOffice.mesh);

      //Raycaster
      this.windmillOffice.raycaster = new THREE.Raycaster();
      const rayOrigin = new THREE.Vector3(
        this.windmillOffice.building.parametrs.size.width / 2 + 1,
        this.quadcopter.parametrs.positionY,
        0
      );
      const rayDiraction = new THREE.Vector3(1, 1, 0);
      rayDiraction.normalize();

      this.windmillOffice.raycaster.set(rayOrigin, rayDiraction);
    },
    createCustomersOffice() {
      const group = new THREE.Group();

      //Building
      const width = this.customersOffice.building.parametrs.size.width + 30,
        height = this.customersOffice.building.parametrs.size.height,
        depth = this.customersOffice.building.parametrs.size.depth,
        endOfWay = this.floor.parametrs.size.leng;
      const geometry = new THREE.BoxBufferGeometry(width, height, depth);
      const material = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.customersOffice.building.parametrs.metalness,
        roughness: this.customersOffice.building.parametrs.roughness,
      });
      group.add(new THREE.Mesh(geometry, material));

      //Animated plane
      const planeWidth =
          this.customersOffice.animatedPlain.parametrs.size.width,
        planeHeight = this.customersOffice.animatedPlain.parametrs.size.height;
      const planeGeometry = new THREE.PlaneBufferGeometry(
        planeWidth,
        planeHeight,
        32,
        32
      );
      const planeMaterial = new THREE.MeshStandardMaterial({
        color: "#777777",
        metalness: this.customersOffice.building.parametrs.metalness,
        roughness: this.customersOffice.building.parametrs.roughness,
      });
      const animatedPlain = new THREE.Mesh(planeGeometry, planeMaterial);
      animatedPlain.position.x = -width / 2 + 0.1;
      animatedPlain.rotation.y = -Math.PI / 2;
      group.add(animatedPlain);

      this.customersOffice.mesh = group;
      this.customersOffice.mesh.position.y = height / 2;
      this.customersOffice.mesh.position.x = endOfWay + 15;
      this.scene.add(this.customersOffice.mesh);

      //Animate
      /* const animate = () => {
        console.log("animate");
        if (
          this.distance + 30 < this.floor.parametrs.size.leng &&
          this.state !== "start"
        ) {
          return;
        }
        const now = Date.now() / 300;
        const animatedPlain = this.customersOffice.mesh.children[1];
        const animatedPlainPartialsCount =
          animatedPlain.geometry.attributes.position.count;

        for (let i = 0; i < animatedPlainPartialsCount; i++) {
          const x = animatedPlain.geometry.attributes.position.getX(i);
          const y = animatedPlain.geometry.attributes.position.getY(i);
          const xsin = Math.sin(x + now) / 2;
          const ycos = Math.cos(y + now) / 2;
          this.customersOffice.mesh.children[1].geometry.attributes.position.setZ(
            i,
            xsin + ycos
          );
        }
        this.customersOffice.mesh.children[1].geometry.attributes.position.needsUpdate = true;
        requestAnimationFrame(animate);
      };
      animate(); */

      //Raycaster
      this.customersOffice.building.raycaster = new THREE.Raycaster();
      const rayOrigin = new THREE.Vector3(endOfWay, 0, 0);
      const rayDiraction = new THREE.Vector3(1, 1, 0);
      rayDiraction.normalize();

      this.customersOffice.building.raycaster.set(rayOrigin, rayDiraction);
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
    createFog() {
      const fog = new THREE.Fog(this.fog.color, 1, this.fog.distance);
      this.scene.fog = fog;
    },
    removePhysicalObject(mesh, body) {
      this.world.removeBody(body);
      this.scene.remove(mesh);
    },
    createWallStack(start, end) {
      let pathLength = start;
      while (pathLength < end) {
        const step = Math.random() * 10 + 10,
          heightOfBottomWall = this.walls.parametrs.size.y * Math.random(),
          offset = 7,
          heightOfTopWall = 10 - (offset + heightOfBottomWall);

        this.createWall(
          this.walls.parametrs.size,
          new THREE.Vector3(pathLength, heightOfBottomWall / 2, 0)
        );

        this.createWall(
          this.walls.parametrs.size,
          new THREE.Vector3(pathLength, -heightOfTopWall / 2 + 10, 0)
        );
        pathLength += step;
      }
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
    speedCounter() {
      this.speedCount.speedInterval = setInterval(() => {
        const currDistance = this.camera.position.x.toFixed();
        this.speed = currDistance - this.speedCount.oldDistance;
        this.speedCount.oldDistance = currDistance;
      }, 1000);
    },
    start() {
      const intersectStart = this.windmillOffice.raycaster.intersectObject(
        this.quadcopter.mesh
      );

      if (intersectStart.length) {
        this.gravity = -9.82;
        this.world.gravity.set(0, this.gravity, 0);
        this.state = "start";
      }
    },
    finish() {
      const intersectFinish =
        this.customersOffice.building.raycaster.intersectObject(
          this.quadcopter.mesh
        );

      if (intersectFinish.length) {
        this.state = "finish";
        setTimeout(() => {
          this.popup.content = this.data.popups.finish;
          this.popup.state = true;
        }, this.popup.timeout);
      }
    },
    fail() {
      this.state = "fail";
      document.removeEventListener("touchstart", this.hangleTap);
      document.removeEventListener("keypress", this.handleKeypress);
      setTimeout(() => {
        document.addEventListener("touchstart", this.hangleTap);
        document.addEventListener("keypress", this.handleKeypress);
        this.popup.content = this.data.popups.fail;
        this.popup.state = true;
      }, this.popup.timeout);
    },
    restart() {
      this.$router.replace({ name: "Home", force: true });
    },
    destroy() {
      window.removeEventListener("resize", this.resizeUpdate);
      document.removeEventListener("keypress", this.handleKeypress);
      document.removeEventListener("touchstart", this.hangleTap);

      this.world.removeBody(this.floor.body);
      this.scene.remove(this.floor.mesh);
      this.world.removeBody(this.ceiling.body);
      this.scene.remove(this.ceiling.mesh);
      this.world.removeBody(this.quadcopter.body);
      this.scene.remove(this.quadcopter.mesh);
      for (const obj of this.walls.objects) {
        this.scene.remove(obj.mesh);
        this.world.removeBody(obj.body);
      }

      this.scene.remove(this.windmillOffice.mesh);
      this.scene.remove(this.customersOffice.mesh);
      this.scene.remove(this.camera);
      this.scene.remove(this.directionalLight);
      this.scene.remove(this.ambientLight);
      this.scene.remove(this.quadcopter.model.body);
      this.scene.remove(this.quadcopter.model.rightRotor);
      this.scene.remove(this.quadcopter.model.leftRotor);
    },
  },
  components: {
    Stats,
    Popup,
    Navigation,
  },
  beforeUnmount() {
    this.destroy();
  },
};
</script>
<style lang="sass">
@import "../assets/scss/default/colors.scss"
@import "../assets/scss/pages/game.scss"
</style>
