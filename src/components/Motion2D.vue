<style scoped>
#ui {
  position: absolute;
  top: 10px;
  left: 10px;
  background: rgba(0, 0, 0, 0.35);
  padding: 10px 12px;
  border-radius: 8px;
  backdrop-filter: blur(4px);
}

.row {
  margin: 4px 0;
}

label {
  display: inline-block;
  width: 120px;
}

input {
  width: 100px;
}

button {
  margin-left: 6px;
}

canvas {
  display: block;
  width: 100vw;
  height: 100vh;
}
</style>

<template>
  <div id="ui">
    <div class="row"><label>Speed v0 (m/s)</label><input ref="v0Ref" type="number" :value="800" step="1"/></div>
    <div class="row"><label>Angle (deg)</label><input ref="angRef" type="number" :value="5" step="0.1"/></div>
    <div class="row"><label>Mass (kg)</label><input ref="mRef" type="number" :value="0.0095" step="0.0001"/></div>
    <div class="row"><label>drag coefficient (dimensionless) Cd</label><input ref="cdRef" type="number" :value="0.295"
                                                                              step="0.01"/></div>
    <div class="row"><label>Diameter (m)</label><input ref="diaRef" type="number" :value="0.00782" step="0.0001"/></div>
    <div class="row"><label>Air ρ (kg/m³)</label><input ref="rhoRef" type="number" :value="1.225" step="0.01"/></div>
    <div class="row">
      <button @click="onLaunch">Launch</button>
      <button @click="onReset">Reset View</button>
    </div>
    <div class="row">{{ stats }}</div>
  </div>
  <canvas ref="canvasRef"></canvas>
</template>

<script setup lang="ts">
import {onMounted, onBeforeUnmount, ref} from 'vue'

type State = { t: number; x: number; y: number; vx: number; vy: number }

const canvasRef = ref<HTMLCanvasElement | null>(null)
const v0Ref = ref<HTMLInputElement | null>(null)
const angRef = ref<HTMLInputElement | null>(null)
const mRef = ref<HTMLInputElement | null>(null)
const cdRef = ref<HTMLInputElement | null>(null)
const diaRef = ref<HTMLInputElement | null>(null)
const rhoRef = ref<HTMLInputElement | null>(null)

const stats = ref('t=0.000 s, x=0.0 m, y=0.0 m')

// World/view parameters
const zoom = ref(0.6)           // pixels per meter
let panX = 80                   // pixels
let panY = 0                    // initialized after mount to bottom padding

// Simulation state
let path: { x: number; y: number }[] = []
let s: State | null = null
let accelFn: ((vx: number, vy: number) => { ax: number; ay: number }) | null = null
let running = false

// Animation loop state
let rafId = 0
let lastTime: number | null = null
const simDt = 0.0005            // physics step (s) for bullets
const maxFrameCatchup = 0.05     // anti-spiral cap (s)

// Utils
function degToRad(d: number) {
  return (d * Math.PI) / 180
}

function setupCanvas(canvas: HTMLCanvasElement): CanvasRenderingContext2D {
  const dpr = window.devicePixelRatio || 1
  const rect = canvas.getBoundingClientRect()
  canvas.width = Math.max(1, Math.floor(rect.width * dpr))
  canvas.height = Math.max(1, Math.floor(rect.height * dpr))
  const ctx = canvas.getContext('2d')!
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  return ctx
}

function rk2Step(
    st: State,
    dt: number,
    accel: (vx: number, vy: number) => { ax: number; ay: number }
): State {
  const {ax, ay} = accel(st.vx, st.vy)
  const vxMid = st.vx + 0.5 * dt * ax
  const vyMid = st.vy + 0.5 * dt * ay
  const {ax: axMid, ay: ayMid} = accel(vxMid, vyMid)

  const vx = st.vx + dt * axMid
  const vy = st.vy + dt * ayMid
  const x = st.x + dt * (st.vx + vx) * 0.5
  const y = st.y + dt * (st.vy + vy) * 0.5
  const t = st.t + dt
  return {t, x, y, vx, vy}
}

type DragInputs = { m: number; Cd: number; A: number; rho: number; g: number }

function makeAccel({m, Cd, A, rho, g}: DragInputs) {
  return (vx: number, vy: number) => {
    const v = Math.hypot(vx, vy) + 1e-15
    const factor = 0.5 * rho * Cd * A / m   // 1/m
    const ax = -factor * v * vx
    const ay = -g - factor * v * vy
    return {ax, ay}
  }
}

// Drawing helpers
function toScreen(x: number, y: number) {
  return {
    sx: x * zoom.value + panX,
    sy: panY - y * zoom.value,
  }
}

function drawAxes(ctx: CanvasRenderingContext2D, canvas: HTMLCanvasElement) {
  const w = canvas.clientWidth
  const h = canvas.clientHeight
  ctx.clearRect(0, 0, w, h)

  // background
  ctx.fillStyle = '#0b1020'
  ctx.fillRect(0, 0, w, h)

  // grid
  ctx.strokeStyle = '#1a2747'
  ctx.lineWidth = 1
  const spacing = 50
  for (let x = (panX % spacing + spacing) % spacing; x < w; x += spacing) {
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, h);
    ctx.stroke()
  }
  for (let y = (panY % spacing + spacing) % spacing; y < h; y += spacing) {
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(w, y);
    ctx.stroke()
  }

  // ground y=0
  ctx.strokeStyle = '#4aa3ff'
  ctx.lineWidth = 2
  const {sy} = toScreen(0, 0)
  ctx.beginPath();
  ctx.moveTo(0, sy);
  ctx.lineTo(w, sy);
  ctx.stroke()
}

function drawPath(ctx: CanvasRenderingContext2D) {
  if (path.length < 2) return
  ctx.strokeStyle = '#ffcf33'
  ctx.lineWidth = 2
  ctx.beginPath()
  const p0 = toScreen(path[0].x, path[0].y)
  ctx.moveTo(p0.sx, p0.sy)
  for (let i = 1; i < path.length; i++) {
    const p = toScreen(path[i].x, path[i].y)
    ctx.lineTo(p.sx, p.sy)
  }
  ctx.stroke()
}

function drawProjectile(ctx: CanvasRenderingContext2D) {
  if (!s) return
  const p = toScreen(s.x, s.y)
  ctx.fillStyle = '#ff6b6b'
  ctx.beginPath()
  ctx.arc(p.sx, p.sy, 4, 0, Math.PI * 2)
  ctx.fill()
}

// Interaction: pan/zoom
let dragging = false
let lastX = 0, lastY = 0

function addInputHandlers(canvas: HTMLCanvasElement) {
  canvas.addEventListener('mousedown', (e) => {
    dragging = true;
    lastX = e.clientX;
    lastY = e.clientY
  })
  window.addEventListener('mouseup', () => {
    dragging = false
  })
  window.addEventListener('mousemove', (e) => {
    if (!dragging) return
    const dx = e.clientX - lastX
    const dy = e.clientY - lastY
    panX += dx
    panY += dy
    lastX = e.clientX;
    lastY = e.clientY
  })
  canvas.addEventListener('wheel', (e) => {
    e.preventDefault()
    const scale = Math.exp(-e.deltaY * 0.001)
    const mouseX = e.clientX, mouseY = e.clientY
    const worldX = (mouseX - panX) / zoom.value
    const worldY = (panY - mouseY) / zoom.value
    zoom.value *= scale
    panX = mouseX - worldX * zoom.value
    panY = mouseY + worldY * zoom.value
  }, {passive: false})
}

function removeInputHandlers(canvas: HTMLCanvasElement) {
  // Rely on passive listeners scoped to this component instance; for brevity,
  // not individually removing anonymous handlers. In production, keep references
  // to listeners to properly remove them here.
}

// Launch and reset
function onReset() {
  zoom.value = 0.6
  const canvas = canvasRef.value!
  panX = 80
  panY = canvas.clientHeight - 80
}

function onLaunch() {
  const canvas = canvasRef.value!
  const v0 = Number(v0Ref.value?.value ?? 800)
  const ang = Number(angRef.value?.value ?? 5)
  const m = Number(mRef.value?.value ?? 0.0095)
  const Cd = Number(cdRef.value?.value ?? 0.295)
  const dia = Number(diaRef.value?.value ?? 0.00782)
  const rho = Number(rhoRef.value?.value ?? 1.225)
  const A = Math.PI * (dia / 2) ** 2
  const g = 9.81

  const theta = degToRad(ang)
  s = {t: 0, x: 0, y: 1.0, vx: v0 * Math.cos(theta), vy: v0 * Math.sin(theta)}
  accelFn = makeAccel({m, Cd, A, rho, g})
  path = [{x: s.x, y: s.y}]
  running = true
  stats.value = `t=${s.t.toFixed(3)} s, x=${s.x.toFixed(1)} m, y=${Math.max(0, s.y).toFixed(1)} m`
  // ensure canvas present and sized
  const ctx = setupCanvas(canvas)
  drawAxes(ctx, canvas)
}

// RAF loop
function tick(ts: number) {
  const canvas = canvasRef.value
  if (!canvas) {
    rafId = requestAnimationFrame(tick);
    return
  }
  const ctx = canvas.getContext('2d')!
  if (lastTime === null) lastTime = ts
  const dtMs = Math.min(1000 * maxFrameCatchup, ts - lastTime)
  lastTime = ts

  let acc = dtMs / 1000
  while (running && acc > 0) {
    const step = Math.min(simDt, acc)
    if (s && accelFn) {
      const prev = s
      s = rk2Step(s, step, accelFn)
      path.push({x: s.x, y: s.y})
      if (s.y < 0 && prev.y >= 0) {
        const alpha = prev.y / (prev.y - s.y)
        s = {
          t: prev.t + alpha * (s.t - prev.t),
          x: prev.x + alpha * (s.x - prev.x),
          y: 0, vx: s.vx, vy: s.vy
        }
        running = false
      }
      stats.value = `t=${s.t.toFixed(3)} s, x=${s.x.toFixed(1)} m, y=${Math.max(0, s.y).toFixed(1)} m`
    }
    acc -= step
  }

  drawAxes(ctx, canvas)
  drawPath(ctx)
  drawProjectile(ctx)

  rafId = requestAnimationFrame(tick)
}

// Resize
function onResize() {
  const canvas = canvasRef.value
  if (!canvas) return
  setupCanvas(canvas)
}

onMounted(() => {
  const canvas = canvasRef.value!
  // initial sizing and pan baseline
  setupCanvas(canvas)
  panY = canvas.clientHeight - 80
  addInputHandlers(canvas)
  window.addEventListener('resize', onResize)
  rafId = requestAnimationFrame(tick)
})

onBeforeUnmount(() => {
  const canvas = canvasRef.value
  if (canvas) removeInputHandlers(canvas)
  window.removeEventListener('resize', onResize)
  cancelAnimationFrame(rafId)
})
</script>

