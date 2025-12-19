# ASL-Flight-Simulator
ASL flight simulator
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Three.js 乒乓球（真实距离版）</title>
<style>
body { margin: 0; overflow: hidden; cursor: crosshair; }
canvas { display: block; }
</style>
</head>

<body>
<script type="module">
import * as THREE from 'https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.module.js';

let scene, camera, renderer;
let ballMesh, paddleMesh;

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

// ===== 尺寸参数 =====
const TABLE_WIDTH  = 5;
const TABLE_HEIGHT = 0.1;
const TABLE_DEPTH  = 12;

const PADDLE_SIZE  = 0.6;
const PADDLE_Y_POS = 0.18;

const BALL_RADIUS  = 0.15;

// ===== 球拍前后移动范围（真实感核心） =====
const PADDLE_Z_MIN = -5.5;
const PADDLE_Z_MAX =  4.8;

let targetX = 0;
let targetZ = 4;

let paddlePrevPos = new THREE.Vector3();
let lastTime = 0;

// ===== 物理 =====
const GRAVITY_Y = -9.8;
const BOUNCE_RESTITUTION = 1.08;
const HIT_FORCE_MULTIPLIER = 0.5;

// ===== 桌面平面（1:1 跟随基础） =====
const tablePlane = new THREE.Plane(
    new THREE.Vector3(0, 1, 0),
    -PADDLE_Y_POS
);

init();
animate(performance.now());

function init() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0xaaaaaa);

    camera = new THREE.PerspectiveCamera(
        70,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    );
    camera.position.set(0, 5.2, 10.5);
    camera.lookAt(0, 0.8, 0);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    document.body.appendChild(renderer.domElement);

    scene.add(new THREE.AmbientLight(0x404040, 5));

    const light = new THREE.DirectionalLight(0xffffff, 1.4);
    light.position.set(6, 12, 6);
    light.castShadow = true;
    scene.add(light);

    createTable();
    createBall();
    createPaddle();

    document.addEventListener('mousemove', onMouseMove);
    window.addEventListener('resize', onWindowResize);

    paddlePrevPos.copy(paddleMesh.position);
}

function createTable() {
    const table = new THREE.Mesh(
        new THREE.BoxGeometry(TABLE_WIDTH, TABLE_HEIGHT, TABLE_DEPTH),
        new THREE.MeshPhongMaterial({ color: 0x006400 })
    );
    table.position.y = -TABLE_HEIGHT / 2;
    table.receiveShadow = true;
    scene.add(table);

    const net = new THREE.Mesh(
        new THREE.BoxGeometry(TABLE_WIDTH, 0.5, 0.05),
        new THREE.MeshBasicMaterial({ color: 0xffffff, wireframe: true })
    );
    net.position.y = 0.25;
    scene.add(net);
}

function createBall() {
    ballMesh = new THREE.Mesh(
        new THREE.SphereGeometry(BALL_RADIUS, 32, 32),
        new THREE.MeshPhongMaterial({ color: 0xff4500 })
    );
    ballMesh.castShadow = true;
    ballMesh.position.set(0, 3, -4);
    ballMesh.userData.velocity = new THREE.Vector3(1, 0, 6);
    scene.add(ballMesh);
}

function createPaddle() {
    paddleMesh = new THREE.Mesh(
        new THREE.BoxGeometry(PADDLE_SIZE, 0.1, PADDLE_SIZE * 1.3),
        new THREE.MeshPhongMaterial({ color: 0x8b0000 })
    );
    paddleMesh.position.set(0, PADDLE_Y_POS, 3.5);
    paddleMesh.rotation.x = Math.PI / 2;
    paddleMesh.castShadow = true;
    scene.add(paddleMesh);
}

// ===== 鼠标 → 世界坐标（1:1） =====
function onMouseMove(e) {
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);

    const hit = new THREE.Vector3();
    if (raycaster.ray.intersectPlane(tablePlane, hit)) {
        targetX = THREE.MathUtils.clamp(
            hit.x,
            -TABLE_WIDTH / 2,
            TABLE_WIDTH / 2
        );
        targetZ = THREE.MathUtils.clamp(
            hit.z,
            PADDLE_Z_MIN,
            PADDLE_Z_MAX
        );
    }
}

function updatePhysics(dt) {
    const v = ballMesh.userData.velocity;
    const p = ballMesh.position;

    v.y += GRAVITY_Y * dt;
    const next = p.clone().addScaledVector(v, dt);

    if (next.y - BALL_RADIUS < 0) {
        next.y = BALL_RADIUS;
        v.y *= -0.9;
    }

    const pad = paddleMesh.position;
    const dx = Math.abs(next.x - pad.x);
    const dy = Math.abs(next.y - pad.y);
    const dz = Math.abs(next.z - pad.z);

    if (
        dx < PADDLE_SIZE / 2 + BALL_RADIUS &&
        dy < PADDLE_SIZE / 2 + BALL_RADIUS &&
        dz < 0.35 &&
        v.z > 0
    ) {
        const paddleVel = pad.clone()
            .sub(paddlePrevPos)
            .divideScalar(dt);

        v.z *= -BOUNCE_RESTITUTION;
        v.x += paddleVel.x * HIT_FORCE_MULTIPLIER;
        v.y += Math.abs(paddleVel.z) * 0.35;

        next.z = pad.z - 0.25;
    }

    p.copy(next);
}

function animate(t) {
    requestAnimationFrame(animate);

    if (!lastTime) lastTime = t;
    const dt = Math.min((t - lastTime) / 1000, 0.1);
    lastTime = t;

    paddlePrevPos.copy(paddleMesh.position);

    paddleMesh.position.x = targetX;
    paddleMesh.position.z = targetZ;

    let acc = dt;
    const step = 1 / 240;
    while (acc >= step) {
        updatePhysics(step);
        acc -= step;
    }

    if (
        ballMesh.position.y < -3 ||
        Math.abs(ballMesh.position.z) > 8
    ) {
        ballMesh.position.set(0, 3, -4);
        ballMesh.userData.velocity.set(
            THREE.MathUtils.randFloatSpread(2),
            0,
            6
        );
    }

    renderer.render(scene, camera);
}

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}
</script>
</body>
</html>
