<!--
  Interactive finite-difference explorer: central difference for f'(x) with 2m+1 points
  (m = 1 … 8, i.e. 3 … 17 points), f(x) = sin(2x).
  Left: curve sampled at all grid points (ticks on the axis, grey dots), stencil points,
  exact tangent vs. finite-difference slope. With 3 points it also
  shows the basic idea: the line through the two neighbours, their positions and the spacing h.
  Right: error of the slope at the current spacing h, one bar per stencil size (log scale).
  `step` (e.g. $clicks) picks m from `steps`; one more click halves the spacing (H_START -> H_FINE). The buttons and the h slider work on top of that.
-->
<script setup>
import { computed, ref, watch } from 'vue'

const props = defineProps({
  step: { type: Number, default: 0 },
  steps: { type: Array, default: () => [1, 2, 4, 8] },   // m per click: 3, 5, 9, 17 points
})

const M_MAX = 8
const H_START = 0.8  // coarse: the 3-point slope is clearly off (38 %)
const H_FINE = 0.4   // last click: halve the spacing, high-order bars drop by orders of magnitude
const mFor = s => props.steps[Math.min(s, props.steps.length - 1)]
const refined = s => s >= props.steps.length
const m = ref(mFor(props.step))
const h = ref(refined(props.step) ? H_FINE : H_START)
watch(() => props.step, s => { m.value = mFor(s); h.value = refined(s) ? H_FINE : H_START })

const f = x => Math.sin(2 * x)
const df = x => 2 * Math.cos(2 * x)
const X0 = 0.3

// weights w_1..w_m with f'(x) ~ sum_k w_k (f(x+kh) - f(x-kh)) / h:
// sum_k 2 w_k k^j = (j == 1) for odd j = 1, 3, ..., 2m-1
function weights(mm) {
  const A = [], b = []
  for (let r = 0; r < mm; r++) {
    const j = 2 * r + 1
    A.push(Array.from({ length: mm }, (_, k) => 2 * (k + 1) ** j))
    b.push(r === 0 ? 1 : 0)
  }
  for (let c = 0; c < mm; c++) {           // Gaussian elimination with partial pivoting
    let p = c
    for (let r = c + 1; r < mm; r++) if (Math.abs(A[r][c]) > Math.abs(A[p][c])) p = r
    ;[A[c], A[p]] = [A[p], A[c]]; [b[c], b[p]] = [b[p], b[c]]
    for (let r = c + 1; r < mm; r++) {
      const q = A[r][c] / A[c][c]
      for (let k = c; k < mm; k++) A[r][k] -= q * A[c][k]
      b[r] -= q * b[c]
    }
  }
  const w = Array(mm).fill(0)
  for (let r = mm - 1; r >= 0; r--) {
    let s = b[r]
    for (let k = r + 1; k < mm; k++) s -= A[r][k] * w[k]
    w[r] = s / A[r][r]
  }
  return w
}
const W = Array.from({ length: M_MAX }, (_, i) => weights(i + 1))
const fd = (mm, hh) => W[mm - 1].reduce((s, w, k) => s + w * (f(X0 + (k + 1) * hh) - f(X0 - (k + 1) * hh)), 0) / hh
const err = (mm, hh) => Math.max(Math.abs(fd(mm, hh) - df(X0)), 1e-16)

// ---- left plot: curve and slopes ----
const L = { w: 420, h: 220, x0: -2.6, x1: 3.4, y0: -1.4, y1: 1.4 }
const lx = x => (x - L.x0) / (L.x1 - L.x0) * L.w
const ly = y => L.h - (y - L.y0) / (L.y1 - L.y0) * L.h
const curve = Array.from({ length: 241 }, (_, i) => {
  const x = L.x0 + i / 240 * (L.x1 - L.x0)
  return `${i ? 'L' : 'M'}${lx(x).toFixed(1)},${ly(f(x)).toFixed(1)}`
}).join('')
const points = computed(() => Array.from({ length: 2 * m.value + 1 }, (_, i) => X0 + (i - m.value) * h.value)
  .filter(x => x >= L.x0 && x <= L.x1 && x !== X0))  // points outside the plot are not drawn
const line = slope => {
  const a = X0 - 0.5, b = X0 + 0.5
  return { x1: lx(a), y1: ly(f(X0) + slope * (a - X0)), x2: lx(b), y2: ly(f(X0) + slope * (b - X0)) }
}
// all grid points x_i + k h in the plot (keyed by k so they glide when h changes)
const grid = computed(() => {
  const k0 = Math.ceil((L.x0 - X0) / h.value), k1 = Math.floor((L.x1 - X0) / h.value)
  return Array.from({ length: k1 - k0 + 1 }, (_, i) => ({ k: k0 + i, x: X0 + (k0 + i) * h.value }))
})
const exact = line(df(X0))
const approx = computed(() => line(fd(m.value, h.value)))

// 3-point intro: secant through the two neighbours, their labels and the spacing h
const intro = computed(() => m.value === 1)
const secant = computed(() => {
  const a = X0 - h.value, b = X0 + h.value, ext = 0.1
  const k = (f(b) - f(a)) / (b - a)
  return { x1: lx(a - ext), y1: ly(f(a) - k * ext), x2: lx(b + ext), y2: ly(f(b) + k * ext) }
})
const neighbours = computed(() => [
  { x: X0 - h.value, t: 'xᵢ₋₁' }, { x: X0, t: 'xᵢ' }, { x: X0 + h.value, t: 'xᵢ₊₁' },
])

// ---- right plot: error per stencil size at the current h ----
const B = { w: 380, h: 220, e0: 1e-10, e1: 10 }
const slot = B.w / M_MAX
const by = e => B.h - Math.log(Math.min(Math.max(e, B.e0), B.e1) / B.e0) / Math.log(B.e1 / B.e0) * B.h
const eTicks = [1e-9, 1e-6, 1e-3, 1]
const bars = computed(() => Array.from({ length: M_MAX }, (_, i) => {
  const e = err(i + 1, h.value)
  return { m: i + 1, n: 2 * i + 3, e, x: i * slot + slot * 0.18, y: by(e), shown: i + 1 <= m.value }
}))
const showBefore = computed(() => refined(props.step) && h.value !== H_START)

const fmt = e => {
  const [mant, ex] = e.toExponential(0).split('e')
  const n = Number(ex)
  if (n === 0) return mant
  const sup = String(Math.abs(n)).split('').map(c => '⁰¹²³⁴⁵⁶⁷⁸⁹'[c]).join('')
  return `${mant}·10${n < 0 ? '⁻' : ''}${sup}`
}
const tickLabel = e => (e === 1 ? '1' : fmt(e).replace('1·', ''))
</script>

<template>
  <div class="fd-explorer">
    <div class="plots">
      <!-- curve with stencil points and the two slopes -->
      <svg :width="L.w" :height="L.h + 26" :viewBox="`-10 -4 ${L.w + 20} ${L.h + 30}`">
        <defs>
          <marker id="fd-arr" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto-start-reverse" markerUnits="userSpaceOnUse">
            <path d="M0,0 L8,4 L0,8 Z" fill="#000" />
          </marker>
        </defs>
        <line x1="0" :y1="ly(0)" :x2="L.w" :y2="ly(0)" class="axis" />
        <path :d="curve" class="curve" />
        <!-- the grid: every point the function is known at, stencil or not -->
        <g v-for="g in grid" :key="'g' + g.k" class="gridpt">
          <line :x1="lx(g.x)" :y1="ly(0) - 5" :x2="lx(g.x)" :y2="ly(0) + 5" class="gtick" />
          <circle :cx="lx(g.x)" :cy="ly(f(g.x))" r="3.5" class="gdot" />
        </g>
        <line v-bind="exact" class="exact" />
        <line v-bind="approx" class="approx" />
        <!-- 3 points: the line through the two neighbours, their positions and the spacing -->
        <g class="reveal" :class="{ shown: intro }">
          <line v-bind="secant" class="secant" />
          <line v-for="n in neighbours" :key="n.t" :x1="lx(n.x)" :y1="ly(f(n.x))" :x2="lx(n.x)" :y2="L.h + 4" class="drop" />
          <line :x1="lx(X0 - h)" :y1="L.h - 6" :x2="lx(X0)" :y2="L.h - 6" class="harrow" marker-start="url(#fd-arr)" marker-end="url(#fd-arr)" />
          <text :x="lx(X0 - h / 2)" :y="L.h - 12" class="lbl" text-anchor="middle">h</text>
          <text v-for="n in neighbours.filter(n => n.x !== X0)" :key="'t' + n.t" :x="lx(n.x)" :y="L.h + 20" class="lbl" text-anchor="middle">{{ n.t }}</text>
        </g>
        <circle v-for="x in points" :key="x" :cx="lx(x)" :cy="ly(f(x))" r="5" class="pt" />
        <circle :cx="lx(X0)" :cy="ly(f(X0))" r="6" class="center" />
        <text :x="lx(X0)" :y="L.h + 20" class="lbl" text-anchor="middle">xᵢ</text>
        <g class="legend" :transform="`translate(${L.w - 135} -4)`">
          <line x1="0" y1="0" x2="26" y2="0" class="exact" />
          <text x="34" y="0" dominant-baseline="middle">exact slope</text>
          <line x1="0" y1="22" x2="26" y2="22" class="approx" />
          <text x="34" y="22" dominant-baseline="middle">finite difference</text>
          <circle cx="13" cy="44" r="3.5" class="gdot" />
          <text x="34" y="44" dominant-baseline="middle">grid point</text>
        </g>
      </svg>

      <!-- error of the slope for each stencil size, at the current h -->
      <svg :width="B.w + 60" :height="B.h + 42" :viewBox="`-55 -6 ${B.w + 64} ${B.h + 48}`">
        <g v-for="e in eTicks" :key="e">
          <line x1="-4" :y1="by(e)" :x2="B.w" :y2="by(e)" class="grid" />
          <text x="-8" :y="by(e)" class="tick" text-anchor="end" dominant-baseline="middle">{{ tickLabel(e) }}</text>
        </g>
        <!-- after halving h: label the change of spacing -->
        <g class="reveal" :class="{ shown: showBefore }">
          <text :x="B.w" y="-2" class="hlabel" text-anchor="end">h = {{ H_START }} → {{ h.toFixed(1) }}</text>
        </g>
        <g v-for="b in bars" :key="b.m" class="bar" :class="{ shown: b.shown, current: b.m === m }">
          <rect :x="b.x" :y="b.y" :width="slot * 0.64" :height="B.h - b.y" />
          <text :x="b.x + slot * 0.32" :y="b.y - 6" class="val" text-anchor="middle">{{ fmt(b.e) }}</text>
        </g>
        <line x1="0" :y1="B.h" :x2="B.w" :y2="B.h" class="axis" />
        <line x1="0" y1="0" x2="0" :y2="B.h" class="axis" />
        <text v-for="b in bars" :key="'n' + b.m" :x="b.x + slot * 0.32" :y="B.h + 18" class="tick"
              :class="{ dim: !b.shown }" text-anchor="middle">{{ b.n }}</text>
        <text :x="B.w / 2" :y="B.h + 38" class="axislbl" text-anchor="middle">points in the stencil</text>
        <text :transform="`translate(-46 ${B.h / 2}) rotate(-90)`" class="axislbl" text-anchor="middle">error of the slope</text>
      </svg>
    </div>

    <div class="controls">
      <button aria-label="fewer points" @click="m = Math.max(1, m - 1)">−</button>
      <span class="count"><b>{{ 2 * m + 1 }}</b> points</span>
      <button aria-label="more points" @click="m = Math.min(M_MAX, m + 1)">+</button>
      <label class="slider">spacing h <input v-model.number="h" type="range" min="0.2" max="1.2" step="0.01"> {{ h.toFixed(2) }}</label>
    </div>
  </div>
</template>

<style scoped>
.fd-explorer { display: flex; flex-direction: column; align-items: center; gap: 4px; }
.plots { display: flex; gap: 22px; align-items: flex-end; }
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.axis { stroke: #000; stroke-width: 1; }
.grid { stroke: #e6e6e6; stroke-width: 1; }
.curve { fill: none; stroke: #000; stroke-width: 2; }
.exact { stroke: var(--fzj-blue); stroke-width: 3; }
.approx { stroke: var(--fzj-orange); stroke-width: 3; stroke-dasharray: 8 5; transition: all 0.35s ease; }
.pt { fill: var(--fzj-orange); transition: cx 0.3s ease, cy 0.3s ease; }
.center { fill: #000; }
.gtick { stroke: #000; stroke-width: 1.3; }
.gdot { fill: #9a9a9a; }
.gridpt line, .gridpt circle { transition: all 0.3s ease; }
.lbl { font-size: 16px; font-style: italic; }
.secant { stroke: var(--fzj-orange); stroke-width: 1.6; transition: all 0.3s ease; }
.drop { stroke: #999; stroke-width: 1; stroke-dasharray: 2 3; }
.harrow { stroke: #000; stroke-width: 1.3; }
.reveal { opacity: 0; transition: opacity 0.35s ease; }
.reveal.shown { opacity: 1; }
.legend text { font-size: 14px; }
.tick { font-size: 13px; transition: fill 0.3s ease; }
.tick.dim { fill: #bbb; }
.axislbl { font-size: 14px; }
.bar rect { fill: var(--fzj-blue); opacity: 0.55; transition: y 0.6s ease, height 0.6s ease, fill 0.3s ease; }
.bar.current rect { fill: var(--fzj-orange); opacity: 1; }
.bar .val { font-size: 11.5px; fill: #333; transition: y 0.6s ease; }
.hlabel { font-size: 14px; font-weight: 700; fill: var(--fzj-blue); }
.bar { opacity: 0; transition: opacity 0.35s ease; }
.bar.shown { opacity: 1; }
.controls { display: flex; align-items: center; gap: 14px; font-size: 17px; margin-top: 2px; }
.controls button {
  width: 28px; height: 28px; border: 1px solid var(--fzj-blue); color: var(--fzj-blue);
  background: #fff; font-size: 19px; line-height: 1; cursor: pointer;
}
.controls button:hover { background: var(--fzj-blue); color: #fff; }
.count { min-width: 80px; text-align: center; }
.slider { margin-left: 24px; }
.slider input { width: 140px; vertical-align: middle; accent-color: var(--fzj-blue); }
</style>
