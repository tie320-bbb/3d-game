
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>3D First Person World</title>

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
    width: 10px;
    height: 10px;
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
    <div>시점: 1인칭</div>
    <div>이동: WASD</div>
    <div>점프: SPACE</div>
    <div>카메라: 마우스 드래그</div>
    <div id="position">위치: 0, 0</div>
</div>

<div id="crosshair"></div>

<div id="help">
    WASD 이동 | SPACE 점프 | 마우스 드래그로 시점 회전
</div>

<canvas id="game"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

// 화면 크기
function resize() {
    canvas.width = innerWidth;
    canvas.height = innerHeight;
}

window.addEventListener("resize", resize);
resize();


// ===========================
// 플레이어 설정
// ===========================

const player = {
    x: 0,
    z: 0,
    y: 0,

    vy: 0,

    speed: 0.13,

    grounded: true
};


// ===========================
// 카메라 설정
// ===========================

const camera = {
    angle: 0,

    // 처음에 살짝 아래쪽을 바라봄
    pitch: 0.16,

    // 1인칭 눈높이
    eyeHeight: 1.65
};


// ===========================
// 키보드 입력
// ===========================

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


// ===========================
// 마우스 카메라 회전
// ===========================

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

    const dx = e.clientX - lastX;
    const dy = e.clientY - lastY;

    // 좌우 회전
    camera.angle += dx * 0.005;

    // 상하 회전
    camera.pitch += dy * 0.003;

    camera.pitch = Math.max(
        -0.55,
        Math.min(0.85, camera.pitch)
    );

    lastX = e.clientX;
    lastY = e.clientY;
});


// ===========================
// 3D 오브젝트 생성
// ===========================

const objects = [];

const colors = [
    "#e85d75",
    "#55aaff",
    "#f5c542",
    "#9b72cf"
];

for (let i = 0; i < 35; i++) {

    let x, z;

    // 시작 위치와 너무 가까운 곳에는 생성하지 않음
    do {
        x = (Math.random() - 0.5) * 40;
        z = (Math.random() - 0.5) * 40;
    } while (Math.hypot(x, z) < 5);

    objects.push({
        x: x,
        z: z,

        size: 0.7 + Math.random() * 1.3,

        color: colors[i % colors.length]
    });
}


// ===========================
// 3D 좌표 투영
// ===========================

function project(x, y, z) {

    // 카메라 위치를 기준으로 계산
    const dx = x - player.x;

    const dy = y - (player.y + camera.eyeHeight);

    const dz = z - player.z;

    const sinA = Math.sin(camera.angle);
    const cosA = Math.cos(camera.angle);

    const sinP = Math.sin(camera.pitch);
    const cosP = Math.cos(camera.pitch);

    // 카메라의 좌우 방향
    const right =
        dx * cosA +
        dz * sinA;

    // 카메라의 위아래 방향
    const up =
        dx * sinA * sinP +
        dy * cosP -
        dz * cosA * sinP;

    // 카메라 앞뒤 거리
    const depth =
        dx * sinA * cosP -
        dy * sinP -
        dz * cosA * cosP;

    // 카메라 뒤쪽은 표시하지 않음
    if (depth <= 0.1) return null;

    // 원근감
    const focal = Math.min(
        canvas.width,
        canvas.height
    ) * 0.9;

    const scale = focal / depth;

    return {
        x: canvas.width / 2 + right * scale,

        y: canvas.height / 2 - up * scale,

        depth: depth,

        scale: scale
    };
}


// ===========================
// 3D 선 그리기
// ===========================

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


// ===========================
// 회색 3D 바닥 그리기
// ===========================

function drawFloor() {

    const sinA = Math.sin(camera.angle);
    const cosA = Math.cos(camera.angle);

    // 카메라가 바라보는 앞쪽 방향
    const forwardX = sinA;
    const forwardZ = -cosA;

    // 카메라의 오른쪽 방향
    const rightX = cosA;
    const rightZ = sinA;

    // 카메라 기준 거리와 좌우 위치를
    // 실제 월드 좌표로 변환
    function floorPoint(distance, side) {

        return [
            player.x + forwardX * distance + rightX * side,
            0,
            player.z + forwardZ * distance + rightZ * side
        ];
    }

    // 넓은 바닥의 네 꼭짓점
    const corners = [
        floorPoint(1.5, -60),
        floorPoint(180, -220),
        floorPoint(180, 220),
        floorPoint(1.5, 60)
    ];

    const points = corners.map(p =>
        project(p[0], p[1], p[2])
    );

    if (points.some(p => !p)) return;

    // 바닥 면
    ctx.beginPath();

    ctx.moveTo(points[0].x, points[0].y);

    for (let i = 1; i < points.length; i++) {
        ctx.lineTo(points[i].x, points[i].y);
    }

    ctx.closePath();

    ctx.fillStyle = "#85898e";

    ctx.fill();

    // 바닥 테두리
    ctx.strokeStyle = "#777c82";

    ctx.lineWidth = 2;

    ctx.stroke();
}


// ===========================
// 3D 박스 그리기
// ===========================

function drawCube(x, y, z, size, color) {

    const s = size / 2;

    const vertices = [

        [-s, 0, -s],
        [s, 0, -s],
        [s, size, -s],
        [-s, size, -s],

        [-s, 0, s],
        [s, 0, s],
        [s, size, s],
        [-s, size, s]

    ];

    const faces = [

        [0, 1, 2, 3],
        [4, 5, 6, 7],

        [0, 1, 5, 4],
        [3, 2, 6, 7],

        [0, 3, 7, 4],
        [1, 2, 6, 5]

    ];

    const points = vertices.map(v =>
        project(
            x + v[0],
            y + v[1],
            z + v[2]
        )
    );

    // 면 그리기
    for (const face of faces) {

        if (face.some(i => !points[i])) {
            continue;
        }

        ctx.beginPath();

        ctx.moveTo(
            points[face[0]].x,
            points[face[0]].y
        );

        for (let i = 1; i < face.length; i++) {

            ctx.lineTo(
                points[face[i]].x,
                points[face[i]].y
            );

        }

        ctx.closePath();

        ctx.fillStyle = color;

        ctx.globalAlpha = 0.9;

        ctx.fill();

        ctx.globalAlpha = 1;

        ctx.strokeStyle = "rgba(255,255,255,0.3)";

        ctx.lineWidth = 1;

        ctx.stroke();
    }
}


// ===========================
// 배경 그리기
// ===========================

function drawBackground() {

    const gradient = ctx.createLinearGradient(
        0,
        0,
        0,
        canvas.height
    );

    gradient.addColorStop(0, "#101d38");

    gradient.addColorStop(1, "#314b63");

    ctx.fillStyle = gradient;

    ctx.fillRect(
        0,
        0,
        canvas.width,
        canvas.height
    );
}


// ===========================
// 플레이어 움직임
// ===========================

function update() {

    let forward = 0;

    let sideways = 0;

    // WASD 이동
    if (keys["KeyW"]) forward += 1;

    if (keys["KeyS"]) forward -= 1;

    if (keys["KeyA"]) sideways -= 1;

    if (keys["KeyD"]) sideways += 1;

    // 대각선 속도 보정
    const length = Math.hypot(
        forward,
        sideways
    );

    if (length > 0) {

        forward /= length;

        sideways /= length;

        // 카메라 방향에 따른 이동
        const dx =
            sideways * Math.cos(camera.angle) +
            forward * Math.sin(camera.angle);

        const dz =
            -forward * Math.cos(camera.angle) +
            sideways * Math.sin(camera.angle);

        player.x += dx * player.speed;

        player.z += dz * player.speed;
    }

    // 점프 및 중력
    player.y += player.vy;

    player.vy -= 0.012;

    // 바닥에 착지
    if (player.y <= 0) {

        player.y = 0;

        player.vy = 0;

        player.grounded = true;
    }

    // 위치 표시
    document.getElementById("position").textContent =
        `위치: ${player.x.toFixed(1)}, ${player.z.toFixed(1)}`;
}


// ===========================
// 메인 렌더링
// ===========================

function render() {

    update();

    // 배경
    drawBackground();

    // 회색 바닥
    drawFloor();

    // 먼 오브젝트부터 그리기
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

    // 장애물
    for (const obj of sorted) {

        drawCube(
            obj.x,
            0,
            obj.z,
            obj.size,
            obj.color
        );
    }

    // 플레이어 몸은 그리지 않음.
    // 카메라가 플레이어의 눈높이에 위치하므로
    // 기본적으로 1인칭 시점으로 움직임.

    requestAnimationFrame(render);
}


// 게임 시작
render();

</script>

</body>
</html>
