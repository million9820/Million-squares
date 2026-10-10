index.html 
<!DOCTYPE html>
<html lang="uk">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>Million Squares</title>
<style>
* { box-sizing: border-box; }
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f5f5f5;
  color: #222;
}
header {
  padding: 12px;
  text-align: center;
  background: #202020;
  color: white;
}
h2 { margin: 0 0 5px; font-size: 21px; }
#info { font-size: 13px; }
#controls {
  display: flex;
  justify-content: center;
  gap: 10px;
  padding: 10px;
}
button {
  border: 0;
  border-radius: 8px;
  padding: 10px 18px;
  font-size: 18px;
  background: #292929;
  color: white;
}
#map {
  display: block;
  width: 100%;
  height: 70vh;
  background: #eeeeee;
  touch-action: none;
}
#hint {
  text-align: center;
  padding: 10px;
  font-size: 13px;
}
</style>
</head>
<body>
<header>
  <h2>Million Squares</h2>
  <div id="info">1 000 000 квадратів · 1000 × 1000</div>
</header>

<div id="controls">
  <button id="minus">−</button>
  <button id="reset">На початок</button>
  <button id="plus">+</button>
</div>

<canvas id="map"></canvas>
<div id="hint">Рухай пальцем, щоб переміщати мапу. Натискай на квадрат, щоб вибрати його.</div>

<script>
const canvas = document.getElementById("map");
const ctx = canvas.getContext("2d");

const TOTAL = 1000;
let scale = 12;
let ox = 0;
let oy = 0;
let selected = null;
let pointers = new Map();
let lastX = 0;
let lastY = 0;
let pinchDistance = 0;
let pinchScale = 12;
let pinchCenter = null;

function resize() {
  const rect = canvas.getBoundingClientRect();
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.round(rect.width * dpr);
  canvas.height = Math.round(rect.height * dpr);
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  draw();
}

function draw() {
  const w = canvas.clientWidth;
  const h = canvas.clientHeight;

  ctx.clearRect(0, 0, w, h);
  ctx.fillStyle = "#eeeeee";
  ctx.fillRect(0, 0, w, h);

  const left = Math.max(0, Math.floor(-ox / scale));
  const top = Math.max(0, Math.floor(-oy / scale));
  const right = Math.min(TOTAL, Math.ceil((w - ox) / scale));
  const bottom = Math.min(TOTAL, Math.ceil((h - oy) / scale));

  if (right <= left || bottom <= top) return;

  ctx.fillStyle = "#dcdcdc";
  ctx.fillRect(
    ox + left * scale,
    oy + top * scale,
    (right - left) * scale,
    (bottom - top) * scale
  );

  if (scale >= 3) {
    ctx.beginPath();
    ctx.strokeStyle = "#a7a7a7";
    ctx.lineWidth = 0.6;

    for (let x = left; x <= right; x++) {
      const px = ox + x * scale;
      ctx.moveTo(px, oy + top * scale);
      ctx.lineTo(px, oy + bottom * scale);
    }

    for (let y = top; y <= bottom; y++) {
      const py = oy + y * scale;
      ctx.moveTo(ox + left * scale, py);
      ctx.lineTo(ox + right * scale, py);
    }
    ctx.stroke();
  }

  ctx.strokeStyle = "#555";
  ctx.lineWidth = 2;
  ctx.strokeRect(ox, oy, TOTAL * scale, TOTAL * scale);

  if (selected) {
    ctx.fillStyle = "#46bb68";
    ctx.fillRect(
      ox + selected.x * scale,
      oy + selected.y * scale,
      scale,
      scale
    );
    ctx.strokeStyle = "#087c2c";
    ctx.lineWidth = 2;
    ctx.strokeRect(
      ox + selected.x * scale,
      oy + selected.y * scale,
      scale,
      scale
    );
  }
}

function zoom(factor, cx = canvas.clientWidth / 2,
              cy = canvas.clientHeight / 2) {
  const old = scale;
  scale = Math.max(2, Math.min(50, scale * factor));
  ox = cx - (cx - ox) * scale / old;
  oy = cy - (cy - oy) * scale / old;
  draw();
}

document.getElementById("plus").onclick = () => zoom(1.4);
document.getElementById("minus").onclick = () => zoom(1 / 1.4);
document.getElementById("reset").onclick = () => {
  scale = 12;
  ox = 0;
  oy = 0;
  selected = null;
  draw();
};

canvas.addEventListener("pointerdown", e => {
  canvas.setPointerCapture(e.pointerId);
  pointers.set(e.pointerId, {x: e.clientX, y: e.clientY});

  if (pointers.size === 1) {
    lastX = e.clientX;
    lastY = e.clientY;
  } else if (pointers.size === 2) {
    const p = [...pointers.values()];
    pinchDistance = Math.hypot(p[0].x-p[1].x, p[0].y-p[1].y);
    pinchScale = scale;
    pinchCenter = {
      x: (p[0].x+p[1].x)/2 - canvas.getBoundingClientRect().left,
      y: (p[0].y+p[1].y)/2 - canvas.getBoundingClientRect().top
    };
  }
});

canvas.addEventListener("pointermove", e => {
  if (!pointers.has(e.pointerId)) return;
  pointers.set(e.pointerId, {x: e.clientX, y: e.clientY});

  if (pointers.size === 1) {
    ox += e.clientX - lastX;
    oy += e.clientY - lastY;
    lastX = e.clientX;
    lastY = e.clientY;
    draw();
  } else if (pointers.size === 2) {
    const p = [...pointers.values()];
    const distance = Math.hypot(p[0].x-p[1].x, p[0].y-p[1].y);
    if (pinchDistance > 0) {
      const old = scale;
      scale = Math.max(2, Math.min(50,
        pinchScale * distance / pinchDistance));
      ox = pinchCenter.x - (pinchCenter.x-ox) * scale / old;
      oy = pinchCenter.y - (pinchCenter.y-oy) * scale / old;
      draw();
    }
  }
});

function release(e) {
  pointers.delete(e.pointerId);
  if (pointers.size === 1) {
    const p = [...pointers.values()][0];
    lastX = p.x;
    lastY = p.y;
  }
  if (pointers.size < 2) pinchDistance = 0;
}

canvas.addEventListener("pointerup", e => {
  const rect = canvas.getBoundingClientRect();
  if (pointers.size === 1) {
    const x = Math.floor((e.clientX - rect.left - ox) / scale);
    const y = Math.floor((e.clientY - rect.top - oy) / scale);
    if (x >= 0 && x < TOTAL && y >= 0 && y < TOTAL) {
      selected = {x, y};
      document.getElementById("info").textContent =
        `Квадрат ${y * TOTAL + x + 1} · X: ${x+1}, Y: ${y+1}`;
      draw();
    }
  }
  release(e);
});

canvas.addEventListener("pointercancel", release);
window.addEventListener("resize", resize);
resize();
</script>
</body>
</html># Million-squares
