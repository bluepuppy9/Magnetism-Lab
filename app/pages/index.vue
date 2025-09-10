<template>
  <div id="app">
    <div id="ui">
      <button @click="addMagnet">Add Magnet</button>
      <label
        ><input type="checkbox" v-model="showLines" /> Show Field Lines</label
      >
    </div>
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from "vue";

const canvas = ref(null);
const showLines = ref(true);
const magnets = [];
let ctx;
let dragging = null;

class Magnet {
  constructor(x, y, angle = 0, length = 120, strength = 10) {
    this.x = x;
    this.y = y;
    this.angle = angle;
    this.length = length;
    this.h = 20;
    this.strength = strength;
  }
  getPoles() {
    const dx = Math.cos(this.angle),
      dy = Math.sin(this.angle);
    const half = this.length / 2;
    return {
      north: {
        x: this.x + dx * half,
        y: this.y + dy * half,
        q: +this.strength,
      },
      south: {
        x: this.x - dx * half,
        y: this.y - dy * half,
        q: -this.strength,
      },
    };
  }
  draw() {
    ctx.save();
    ctx.translate(this.x, this.y);
    ctx.rotate(this.angle);
    ctx.fillStyle = "#337ab7";
    ctx.fillRect(-this.length / 2, -this.h / 2, this.length / 2, this.h);
    ctx.fillStyle = "#d9534f";
    ctx.fillRect(0, -this.h / 2, this.length / 2, this.h);
    ctx.fillStyle = "#000";
    ctx.font = "12px sans-serif";
    ctx.textAlign = "center";
    ctx.fillText("S", -this.length / 4, -this.h / 2 - 4);
    ctx.fillText("N", this.length / 4, -this.h / 2 - 4);
    ctx.restore();
  }
}

function addMagnet() {
  magnets.push(
    new Magnet(
      window.innerWidth / 2 + Math.random() * 200 - 100,
      window.innerHeight / 2 + Math.random() * 200 - 100,
      Math.random() * Math.PI * 2
    )
  );
  draw();
}

function fieldAt(x, y) {
  let fx = 0,
    fy = 0;
  for (const m of magnets) {
    const { north, south } = m.getPoles();
    for (const pole of [north, south]) {
      let rx = x - pole.x,
        ry = y - pole.y;
      let r2 = rx * rx + ry * ry;
      let r = Math.sqrt(r2);
      if (r > 1) {
        fx += (pole.q * rx) / (r2 * r);
        fy += (pole.q * ry) / (r2 * r);
      }
    }
  }
  return { x: fx, y: fy };
}

function drawLines() {
  ctx.strokeStyle = "rgba(0,0,0,0.4)";
  ctx.lineWidth = 1;
  for (const m of magnets) {
    const { north, south } = m.getPoles();
    for (const pole of [north, south]) {
      for (let a = 0; a < Math.PI * 2; a += Math.PI / 6) {
        let x = pole.x + Math.cos(a) * 5,
          y = pole.y + Math.sin(a) * 5;
        ctx.beginPath();
        ctx.moveTo(x, y);
        for (let i = 0; i < 200; i++) {
          const f = fieldAt(x, y);
          const mag = Math.hypot(f.x, f.y);
          if (mag < 1e-3) break;
          x += (f.x / mag) * 5;
          y += (f.y / mag) * 5;
          ctx.lineTo(x, y);
          if (x < 0 || x > window.innerWidth || y < 0 || y > window.innerHeight)
            break;
        }
        ctx.stroke();
      }
    }
  }
}

function draw() {
  ctx.clearRect(0, 0, window.innerWidth, window.innerHeight);
  if (showLines.value) drawLines();
  for (const m of magnets) m.draw();
}

onMounted(() => {
  ctx = canvas.value.getContext("2d");
  canvas.value.width = window.innerWidth;
  canvas.value.height = window.innerHeight;

  canvas.value.onmousedown = (e) => {
    for (const m of magnets) {
      if (Math.hypot(e.offsetX - m.x, e.offsetY - m.y) < 80) {
        dragging = { m, dx: e.offsetX - m.x, dy: e.offsetY - m.y };
        break;
      }
    }
  };
  canvas.value.onmousemove = (e) => {
    if (dragging) {
      dragging.m.x = e.offsetX - dragging.dx;
      dragging.m.y = e.offsetY - dragging.dy;
      draw();
    }
  };
  canvas.value.onmouseup = () => {
    dragging = null;
  };

  addMagnet();
  addMagnet();
  draw();
});

watch(showLines, draw);
</script>

<style>
body,
html,
#app {
  margin: 0;
  padding: 0;
  height: 100%;
  overflow: hidden;
  font-family: sans-serif;
}
#ui {
  position: absolute;
  top: 10px;
  left: 10px;
  background: #fff;
  padding: 10px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 10;
}
canvas {
  width: 100vw;
  height: 100vh;
  display: block;
  background: linear-gradient(#f8fbff, #e4f0ff);
}
</style>
