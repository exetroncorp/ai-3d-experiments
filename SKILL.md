---
name: threejs-game-dev
description: Three.js game development - scene setup, asset loading from local zips/folders, vehicle physics, city generation, camera systems, collision, HUD. Use when creating 3D games, loading model assets, building interactive experiences, or setting up game loops with Three.js.
---

# Three.js Game Development Skill

Comprehensive skill for building 3D games with Three.js using local 3D model assets (GLTF, FBX, OBJ from extracted zip archives).

## Quick Start — Vite + Three.js Game

```bash
npm create vite@latest my-game -- --template vanilla
cd my-game
npm install three
npm run dev
```

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

// Lighting
scene.add(new THREE.AmbientLight(0xffffff, 0.6));
const sun = new THREE.DirectionalLight(0xffffff, 1.2);
sun.position.set(50, 50, 50);
sun.castShadow = true;
scene.add(sun);

// Load model from extracted zip
const loader = new GLTFLoader();
loader.load('/models/city/gltf/building_A.gltf', (gltf) => {
  const model = gltf.scene;
  model.traverse(c => { if (c.isMesh) { c.castShadow = true; c.receiveShadow = true; } });
  scene.add(model);
});

const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  renderer.render(scene, camera);
}
animate();
```

## Game Loop Architecture

```javascript
class Game {
  constructor() {
    this.clock = new THREE.Clock();
    this.scene = new THREE.Scene();
    this.fixedTimeStep = 1 / 60;
    this.accumulator = 0;
    this.state = 'loading'; // loading | playing | paused | gameover
  }

  update(delta) {
    // Fixed timestep for physics
    this.accumulator += delta;
    while (this.accumulator >= this.fixedTimeStep) {
      this.fixedUpdate(this.fixedTimeStep);
      this.accumulator -= this.fixedTimeStep;
    }
  }

  fixedUpdate(dt) {
    // Physics, collision, AI here
  }

  render() {
    this.renderer.render(this.scene, this.camera);
  }

  loop() {
    requestAnimationFrame(() => this.loop());
    const delta = this.clock.getDelta();
    if (this.state === 'playing') this.update(delta);
    this.render();
  }
}
```

## Asset Loading from Extracted Zips

Place extracted zip contents in `public/models/`. Vite serves `public/` as static root.

```
public/
  models/
    city/     ← KayKit City Builder (GLTF files)
    cars/     ← Low Poly Cars (OBJ + textures)
```

### GLTF Loading (preferred for web)
```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const gltfLoader = new GLTFLoader();

function loadGLTF(url) {
  return new Promise((resolve, reject) => {
    gltfLoader.load(url, resolve, undefined, reject);
  });
}

const gltf = await loadGLTF('/models/city/gltf/building_A.gltf');
scene.add(gltf.scene);
```

### FBX Loading
```javascript
import { FBXLoader } from 'three/addons/loaders/FBXLoader.js';

const fbxLoader = new FBXLoader();
fbxLoader.load('/models/city/fbx/building_A.fbx', (object) => {
  object.scale.setScalar(0.01); // FBX often has large scale
  object.traverse(c => { if (c.isMesh) { c.castShadow = true; c.receiveShadow = true; } });
  scene.add(object);
});
```

### OBJ + MTL Loading
```javascript
import { OBJLoader } from 'three/addons/loaders/OBJLoader.js';
import { MTLLoader } from 'three/addons/loaders/MTLLoader.js';

const mtlLoader = new MTLLoader();
mtlLoader.setPath('/models/cars/');
mtlLoader.load('model.mtl', (materials) => {
  materials.preload();
  const objLoader = new OBJLoader();
  objLoader.setMaterials(materials);
  objLoader.setPath('/models/cars/');
  objLoader.load('model.obj', (object) => { scene.add(object); });
});
```

### Loading Manager with Progress
```javascript
const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => {
  const pct = (loaded / total * 100).toFixed(0);
  document.getElementById('loading-bar').style.width = pct + '%';
};
manager.onLoad = () => { document.getElementById('loading-screen').style.display = 'none'; };

const gltfLoader = new GLTFLoader(manager);
```

## Vehicle Physics (Arcade Style)

```javascript
class Car {
  constructor(mesh) {
    this.mesh = mesh;
    this.speed = 0;
    this.maxSpeed = 30;
    this.acceleration = 15;
    this.braking = 30;
    this.friction = 5;
    this.turnSpeed = 2.5;
    this.steerAngle = 0;
    this.velocity = new THREE.Vector3();
  }

  update(dt, input) {
    // Acceleration / braking
    if (input.forward) this.speed += this.acceleration * dt;
    else if (input.backward) this.speed -= this.braking * dt;
    else this.speed -= Math.sign(this.speed) * this.friction * dt;

    // Clamp speed
    this.speed = THREE.MathUtils.clamp(this.speed, -this.maxSpeed * 0.3, this.maxSpeed);
    if (Math.abs(this.speed) < 0.1) this.speed = 0;

    // Steering (only when moving)
    if (Math.abs(this.speed) > 0.5) {
      const turnFactor = this.speed > 0 ? 1 : -1;
      if (input.left) this.mesh.rotation.y += this.turnSpeed * dt * turnFactor;
      if (input.right) this.mesh.rotation.y -= this.turnSpeed * dt * turnFactor;
    }

    // Move forward in facing direction
    const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(this.mesh.quaternion);
    this.velocity.copy(forward.multiplyScalar(this.speed));
    this.mesh.position.add(this.velocity.clone().multiplyScalar(dt));
  }
}
```

## City Generation

```javascript
class City {
  constructor(scene) {
    this.scene = scene;
    this.tileSize = 2; // Size of one road/building tile
    this.buildings = [];
    this.roads = [];
  }

  async generate(loader) {
    // Load building models
    const buildingNames = ['building_A','building_B','building_C','building_D','building_E','building_F','building_G','building_H'];
    const roadNames = ['road_straight','road_corner','road_junction','road_tsplit','road_straight_crossing'];
    const propNames = ['bench','bush','streetlight','trafficlight_A','firehydrant','dumpster'];

    const models = {};
    for (const name of [...buildingNames, ...roadNames, ...propNames]) {
      const gltf = await loadGLTF(`/models/city/gltf/${name}.gltf`);
      models[name] = gltf.scene;
    }

    // Place buildings in grid
    const gridSize = 5;
    for (let x = -gridSize; x <= gridSize; x++) {
      for (let z = -gridSize; z <= gridSize; z++) {
        if (Math.abs(x) <= 1 || Math.abs(z) <= 1) {
          // Roads in center corridors
          this.placeRoad(models, x, z);
        } else {
          // Buildings on blocks
          this.placeBuilding(models, buildingNames, x, z);
        }
      }
    }
  }

  placeBuilding(models, names, x, z) {
    const name = names[Math.floor(Math.random() * names.length)];
    const building = models[name].clone();
    building.position.set(x * this.tileSize, 0, z * this.tileSize);
    building.rotation.y = (Math.floor(Math.random() * 4)) * Math.PI / 2;
    building.traverse(c => { if (c.isMesh) { c.castShadow = true; c.receiveShadow = true; } });
    this.scene.add(building);
    this.buildings.push(building);
  }

  placeRoad(models, x, z) {
    const road = models['road_straight'].clone();
    road.position.set(x * this.tileSize, 0, z * this.tileSize);
    if (Math.abs(x) <= 1) road.rotation.y = Math.PI / 2;
    road.traverse(c => { if (c.isMesh) { c.receiveShadow = true; } });
    this.scene.add(road);
    this.roads.push(road);
  }
}
```

## Third-Person Chase Camera

```javascript
class ChaseCamera {
  constructor(camera, target) {
    this.camera = camera;
    this.target = target;
    this.offset = new THREE.Vector3(0, 5, 10);
    this.lookOffset = new THREE.Vector3(0, 1, 0);
    this.smoothSpeed = 5;
    this.currentPosition = new THREE.Vector3();
    this.currentLookAt = new THREE.Vector3();
  }

  update(dt) {
    // Desired position behind and above target
    const desiredPos = new THREE.Vector3()
      .copy(this.offset)
      .applyQuaternion(this.target.quaternion)
      .add(this.target.position);

    // Smooth follow
    this.currentPosition.lerp(desiredPos, this.smoothSpeed * dt);
    this.camera.position.copy(this.currentPosition);

    // Look at target
    const lookTarget = this.target.position.clone().add(this.lookOffset);
    this.currentLookAt.lerp(lookTarget, this.smoothSpeed * dt);
    this.camera.lookAt(this.currentLookAt);
  }
}
```

## Collision Detection (AABB)

```javascript
class CollisionSystem {
  constructor() {
    this.obstacles = [];
  }

  addObstacle(mesh) {
    const box = new THREE.Box3().setFromObject(mesh);
    this.obstacles.push({ mesh, box });
  }

  checkCollision(playerMesh, playerRadius = 1) {
    const playerPos = playerMesh.position;
    const playerBox = new THREE.Box3().setFromCenterAndSize(
      playerPos, new THREE.Vector3(playerRadius * 2, 2, playerRadius * 2)
    );

    for (const obs of this.obstacles) {
      if (playerBox.intersectsBox(obs.box)) {
        return { hit: true, obstacle: obs };
      }
    }
    return { hit: false };
  }

  resolveCollision(playerMesh, collision) {
    if (!collision.hit) return;
    const obsCenter = new THREE.Vector3();
    collision.obstacle.box.getCenter(obsCenter);
    const pushDir = playerMesh.position.clone().sub(obsCenter).normalize();
    playerMesh.position.add(pushDir.multiplyScalar(0.5));
  }
}
```

## HUD Overlay

```javascript
class HUD {
  constructor() {
    this.container = document.createElement('div');
    this.container.id = 'hud';
    this.container.innerHTML = `
      <div id="speed-display">0 km/h</div>
      <div id="controls-hint">WASD/Arrows to drive</div>
    `;
    document.body.appendChild(this.container);
  }

  updateSpeed(speed) {
    const kmh = Math.abs(Math.round(speed * 3.6));
    document.getElementById('speed-display').textContent = kmh + ' km/h';
  }
}
```

## Input Handler

```javascript
class InputHandler {
  constructor() {
    this.keys = {};
    window.addEventListener('keydown', e => this.keys[e.code] = true);
    window.addEventListener('keyup', e => this.keys[e.code] = false);
  }

  get forward() { return this.keys['KeyW'] || this.keys['ArrowUp']; }
  get backward() { return this.keys['KeyS'] || this.keys['ArrowDown']; }
  get left() { return this.keys['KeyA'] || this.keys['ArrowLeft']; }
  get right() { return this.keys['KeyD'] || this.keys['ArrowRight']; }
  get brake() { return this.keys['Space']; }
}
```

## Lighting Setup for Games

```javascript
// Ambient + Directional (sun) + Hemisphere
scene.add(new THREE.HemisphereLight(0x87CEEB, 0x362907, 0.5));

const sun = new THREE.DirectionalLight(0xffeedd, 1.5);
sun.position.set(50, 80, 50);
sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
sun.shadow.camera.left = -50;
sun.shadow.camera.right = 50;
sun.shadow.camera.top = 50;
sun.shadow.camera.bottom = -50;
sun.shadow.camera.near = 0.1;
sun.shadow.camera.far = 200;
sun.shadow.bias = -0.001;
scene.add(sun);

// Fog for atmosphere
scene.fog = new THREE.FogExp2(0xC8DCE8, 0.015);
scene.background = new THREE.Color(0x87CEEB);
```

## Responsive Canvas

```javascript
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

## Performance Tips for Games
1. **Use instancing** for repeated objects (trees, props)
2. **Pool objects** instead of create/destroy
3. **LOD** for distance-based mesh switching
4. **Frustum culling** is enabled by default
5. **Merge static geometries** for terrain/buildings
6. **Limit shadow casters** — only nearby objects
7. **Use `requestAnimationFrame`** — never `setInterval`
8. **Dispose unused resources** — geometry, materials, textures

```javascript
// InstancedMesh for many copies
const instancedMesh = new THREE.InstancedMesh(geometry, material, count);
const matrix = new THREE.Matrix4();
for (let i = 0; i < count; i++) {
  matrix.setPosition(x, y, z);
  instancedMesh.setMatrixAt(i, matrix);
}
instancedMesh.instanceMatrix.needsUpdate = true;
```

## See Also
- Three.js docs: https://threejs.org/docs/
- Three.js examples: https://threejs.org/examples/
- Import pattern: `import { X } from 'three/addons/...'`
