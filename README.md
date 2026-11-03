
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>3D 자유 이동 게임</title>

<style>
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    overflow: hidden;
    background: #101827;
    font-family: Arial, sans-serif;
}

canvas {
    display: block;
}

#ui {
    position: fixed;
    top: 20px;
    left: 20px;
    color: white;
    z-index: 10;
    background: rgba(0,0,0,0.55);
    padding: 15px 20px;
    border-radius: 12px;
    line-height: 1.8;
    pointer-events: none;
}

#help {
    position: fixed;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    color: white;
    background: rgba(0,0,0,0.55);
    padding: 12px 20px;
    border-radius: 10px;
    font-size: 13px;
    white-space: nowrap;
    pointer-events: none;
}

#crosshair {
    position: fixed;
    left: 50%;
    top: 50%;
    width: 8px;
    height: 8px;
    border: 2px solid white;
    border-radius: 50%;
    transform: translate(-50%, -50%);
    pointer-events: none;
}
</style>
</head>

<body>

<div id="ui">
    <h3>🎮 3D WORLD</h3>
    <div>이동: WASD</div>
    <div>점프: SPACE</div>
    <div>카메라: 마우스 드래그</div>
    <div id="position">위치: 0, 0</div>
</div>

<div id="crosshair"></div>

<div id="help">
    WASD 이동 | SPACE 점프 | 마우스 드래그 카메라 회전 | 휠 확대/축소
</div>

<canvas id="game"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

function resize() {
    canvas.width = innerWidth;
    canvas.height = innerHeight;
}
window.addEventListener("resize", resize);
resize();

// 플레이어
const player = {
    x: 0,
    z: 0,
    y: 0,
    vy: 0,
    speed: 0.13,
    size: 0.6,
    grounded: true
};

// 카메라
let camera = {
    angle: 0,
    pitch: 0.48,
    distance: 11
};

// 키 입력
const keys = {};

window.addEventListener("keydown", e => {
    keys[e.code] = true;

    if (e.code === "Space") {
        e.preventDefault();

        if (player.grounded) {
            player.vy = 0.23;
            player.grounded = false;
        }
    }
});

window.addEventListener("keyup", e => {
    keys[e.code] = false;
});

// 마우스 카메라
let dragging = false;
let lastX = 0;
let lastY = 0;

canvas.addEventListener("mousedown", e => {
    dragging = true;
    lastX = e.clientX;
    lastY = e.clientY;
});

window.addEventListener("mouseup", () => {
    dragging = false;
});

window.addEventListener("mousemove", e => {
    if (!dragging) return;

    camera.angle += (e.clientX - lastX) * 0.005;
    camera.pitch += (e.clientY - lastY) * 0.003;

    camera.pitch = Math.max(0.15, Math.min(1.2, camera.pitch));

    lastX = e.clientX;
    lastY = e.clientY;
});

canvas.addEventListener("wheel", e => {
    camera.distance += e.deltaY * 0.01;
    camera.distance = Math.max(4, Math.min(25, camera.distance));
});

// 3D 오브젝트
const objects = [];

// 장애물 생성
for (let i = 0; i < 35; i++) {
    objects.push({
        x: (Math.random() - 0.5) * 35,
        z: (Math.random() - 0.5) * 35,
        size: 0.7 + Math.random() * 1.3,
        color: ["#e85d75", "#55aaff", "#f5c542", "#9b72cf"][i % 4]
    });
}

// 투영 계산
function project(x, y, z) {
    let dx = x - player.x;
    let dz = z - player.z;

    const cos = Math.cos(camera.angle);
    const sin = Math.sin(camera.angle);

    let rx = dx * cos - dz * sin;
    let rz = dx * sin + dz * cos;

    let cy = Math.cos(camera.pitch);
    let sy = Math.sin(camera.pitch);

    let ry = y * cy - rz * sy;
    let depth = y * sy + rz * cy + camera.distance;

    if (depth < 0.1) return null;

    const scale = 650 / depth;

    return {
        x: canvas.width / 2 + rx * scale,
        y: canvas.height / 2 - ry * scale,
        scale,
        depth
    };
}

// 3D 선 그리기
function line3D(a, b, color, width = 1) {
    const p1 = project(a[0], a[1], a[2]);
    const p2 = project(b[0], b[1], b[2]);

    if (!p1 || !p2) return;

    ctx.beginPath();
    ctx.moveTo(p1.x, p1.y);
    ctx.lineTo(p2.x, p2.y);
    ctx.strokeStyle = color;
    ctx.lineWidth = width;
    ctx.stroke();
}

// 바닥 그리기
function drawFloor() {
    const range = 30;

    for (let i = -range; i <= range; i++) {
        line3D(
            [-range, 0, i],
            [range, 0, i],
            "#26374b"
        );

        line3D(
            [i, 0, -range],
            [i, 0, range],
            "#26374b"
        );
    }
}

// 박스 그리기
function drawCube(x, y, z, size, color) {
    const s = size / 2;

    const vertices = [
        [-s,0,-s], [s,0,-s],
        [s,size,-s], [-s,size,-s],
        [-s,0,s], [s,0,s],
        [s,size,s], [-s,size,s]
    ];

    const edges = [
        [0,1],[1,2],[2,3],[3,0],
        [4,5],[5,6],[6,7],[7,4],
        [0,4],[1,5],[2,6],[3,7]
    ];

    const faces = [
        [0,1,2,3],
        [4,5,6,7],
        [0,1,5,4],
        [3,2,6,7],
        [0,3,7,4],
        [1,2,6,5]
    ];

    const points = vertices.map(v =>
        project(x + v[0], y + v[1], z + v[2])
    );

    for (let i = 0; i < faces.length; i++) {
        const face = faces[i];

        if (face.some(index => !points[index])) continue;

        ctx.beginPath();
        ctx.moveTo(points[face[0]].x, points[face[0]].y);

        for (let j = 1; j < face.length; j++) {
            ctx.lineTo(
                points[face[j]].x,
                points[face[j]].y
            );
        }

        ctx.closePath();
        ctx.fillStyle = color;
        ctx.globalAlpha = 0.8;
        ctx.fill();
        ctx.globalAlpha = 1;

        ctx.strokeStyle = "rgba(255,255,255,0.25)";
        ctx.stroke();
    }
}

// 플레이어 그리기
function drawPlayer() {
    // 몸통
    drawCube(
        player.x,
        player.y + 0.7,
        player.z,
        0.8,
        "#36d9a0"
    );

    // 머리
    drawCube(
        player.x,
        player.y + 1.5,
        player.z,
        0.55,
        "#ffe0b2"
    );
}

// 배경
function drawBackground() {
    const gradient = ctx.createLinearGradient(
        0, 0, 0, canvas.height
    );

    gradient.addColorStop(0, "#101d38");
    gradient.addColorStop(1, "#314b63");

    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, canvas.width, canvas.height);
}

// 플레이어 움직임
function update() {
    let forward = 0;
    let sideways = 0;

    if (keys["KeyW"]) forward += 1;
    if (keys["KeyS"]) forward -= 1;
    if (keys["KeyA"]) sideways -= 1;
    if (keys["KeyD"]) sideways += 1;

    const length = Math.hypot(forward, sideways);

    if (length > 0) {
        forward /= length;
        sideways /= length;

        const dx =
            sideways * Math.cos(camera.angle) +
            forward * Math.sin(camera.angle);

        const dz =
            -forward * Math.cos(camera.angle) +
            sideways * Math.sin(camera.angle);

        player.x += dx * player.speed;
        player.z += dz * player.speed;
    }

    // 중력
    player.y += player.vy;
    player.vy -= 0.012;

    if (player.y <= 0) {
        player.y = 0;
        player.vy = 0;
        player.grounded = true;
    }

    document.getElementById("position").textContent =
        `위치: ${player.x.toFixed(1)}, ${player.z.toFixed(1)}`;
}

// 메인 렌더링
function render() {
    update();

    drawBackground();
    drawFloor();

    // 먼 오브젝트부터 렌더링
    const sorted = [...objects].sort((a, b) => {
        const da = Math.hypot(
            a.x - player.x,
            a.z - player.z
        );

        const db = Math.hypot(
            b.x - player.x,
            b.z - player.z
        );

        return db - da;
    });

    for (const obj of sorted) {
        drawCube(
            obj.x,
            0,
            obj.z,
            obj.size,
            obj.color
        );
    }

    drawPlayer();

    requestAnimationFrame(render);
}

render();
</script>

</body>
</html>
