<!--
  Why the order of the cubes matters for the z pass. A cut through the grid of cubes (x right,
  z up); the number in a cube = when its block runs. The L2 cache is drawn as "the last 4 cubes
  visited" (in 3D, at 512³ points: a few thousand cubes, 50 MB).
    step 0: the z pass: the block for a cube needs the cubes below and above
    step 1: default x-major order: numbers row by row
    step 2: L2 holds the last few cubes; the cube below was visited a whole row earlier -> gone
    step 3: z order: numbers column by column; the cube below was visited just before -> in L2
-->
<script setup>
import { computed } from 'vue'

const props = defineProps({ step: { type: Number, default: 3 } })

const NX = 6, NZ = 4, S = 58, GX = 70, GY = 40, L2 = 4
const C = { x: 2, z: 2 }                                    // the current cube
const cubes = []
for (let z = 0; z < NZ; z++) for (let x = 0; x < NX; x++) cubes.push({ x, z })
const px = c => GX + c.x * S
const py = c => GY + (NZ - 1 - c.z) * S                       // z up

const zOrder = computed(() => props.step >= 3)
const idx = c => (zOrder.value ? c.x * NZ + c.z : c.z * NX + c.x)
const cur = computed(() => idx(C))
const role = c => (c.x === C.x && c.z === C.z ? 'cur'
  : c.x === C.x && Math.abs(c.z - C.z) === 1 ? (c.z < C.z ? 'below' : 'above') : '')
const inL2 = c => props.step >= 2 && idx(c) < cur.value && idx(c) >= cur.value - L2

// visiting order as arrows: rows (x-major) or columns (z order)
const lanes = computed(() => (zOrder.value
  ? Array.from({ length: NX }, (_, x) => ({ x1: GX + x * S + S / 2, y1: GY + NZ * S + 4, x2: GX + x * S + S / 2, y2: GY - 14 }))
  : Array.from({ length: NZ }, (_, z) => ({ x1: GX - 4, y1: py({ z }) + S / 2, x2: GX + NX * S + 14, y2: py({ z }) + S / 2 }))))

const blocks = computed(() => {
  const s = props.step
  if (s === 0) return [{ head: 'z pass', body: ['the block for a cube needs', 'the cubes below and above'] }]
  const order = zOrder.value
    ? { head: 'z order', body: ['blocks run column by column'] }
    : { head: 'default: x-major order', body: ['blocks run row by row'] }
  if (s === 1) return [order]
  if (!zOrder.value) return [order, { head: 'cube below: a whole row ago', cls: 'bad', body: ['no longer in L2', '→ read from memory again'] }]
  return [order, { head: 'cube below: just before', cls: 'good', body: ['still in L2', 'cube above: next → stays in L2'] }]
})
</script>

<template>
  <div class="co">
    <svg width="900" height="390" viewBox="0 0 900 390">
      <defs>
        <marker id="co-arr" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L7,3.5 L0,7 Z" fill="#9aa9b6" />
        </marker>
        <marker id="co-ax" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L8,4 L0,8 Z" fill="#444" />
        </marker>
      </defs>

      <!-- axes -->
      <line :x1="GX - 30" :y1="GY + NZ * S + 18" :x2="GX + 20" :y2="GY + NZ * S + 18" class="axis" marker-end="url(#co-ax)" />
      <text :x="GX + 26" :y="GY + NZ * S + 23" class="axlbl">x</text>
      <line :x1="GX - 30" :y1="GY + NZ * S + 18" :x2="GX - 30" :y2="GY + NZ * S - 32" class="axis" marker-end="url(#co-ax)" />
      <text :x="GX - 35" :y="GY + NZ * S - 40" class="axlbl">z</text>

      <!-- visiting order: lines behind the cubes, arrowheads outside the grid -->
      <g class="reveal" :class="{ shown: step >= 1 }">
        <line v-for="(l, i) in lanes" :key="(zOrder ? 'c' : 'r') + i" v-bind="l" class="lane" marker-end="url(#co-arr)" />
      </g>

      <!-- cubes -->
      <g v-for="c in cubes" :key="`${c.x}-${c.z}`">
        <rect :x="px(c) + 2" :y="py(c) + 2" :width="S - 4" :height="S - 4" rx="4"
              :class="['cube', role(c), { l2: inL2(c) && !role(c) }]" />
      </g>

      <!-- visiting order -->
      <g class="reveal" :class="{ shown: step >= 1 }">
        <text v-for="c in cubes" :key="'n' + c.x + c.z + zOrder" :x="px(c) + S / 2" :y="py(c) + S / 2 + 1"
              :class="['num', role(c)]" text-anchor="middle" dominant-baseline="middle">{{ idx(c) }}</text>
      </g>

      <!-- in L2 or not: mark in the corner of the cube below -->
      <g v-if="step >= 2" :key="'mk' + zOrder" class="pop">
        <text :x="px({ x: C.x }) + S - 7" :y="py({ z: C.z - 1 }) + 17" :class="['mark', zOrder ? 'good' : 'bad']" text-anchor="end">
          {{ zOrder ? '✓' : '✗' }}
        </text>
      </g>

      <!-- legend -->
      <g :transform="`translate(${GX} ${GY + NZ * S + 44})`">
        <rect width="16" height="16" rx="3" class="cube cur" /><text x="22" y="12" class="leg">current cube</text>
        <rect x="128" width="16" height="16" rx="3" class="cube below" /><text x="150" y="12" class="leg">needed neighbours</text>
        <g class="reveal" :class="{ shown: step >= 2 }">
          <rect x="290" width="16" height="16" rx="3" class="cube l2" /><text x="312" y="12" class="leg">in L2 (last {{ L2 }} cubes)</text>
        </g>
      </g>

      <!-- explanation on the right -->
      <g v-for="(b, i) in blocks" :key="'t' + step + i" class="pop" :transform="`translate(500 ${60 + i * 120})`">
        <text y="0" :class="['head', b.cls]">{{ b.head }}</text>
        <text v-for="(t, j) in b.body" :key="j" :y="28 + j * 24" class="body">{{ t }}</text>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.co { display: flex; flex-direction: column; align-items: center; }
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.axis { stroke: #444; stroke-width: 1.5; }
.axlbl { font-size: 16px; font-style: italic; font-weight: 700; fill: #444; }
.cube { fill: #f2f4f6; stroke: #c9d6e2; stroke-width: 1.5; transition: fill 0.4s ease; }
.cube.l2 { fill: #c9d6e2; stroke: #7494ae; }
.cube.cur { fill: var(--fzj-blue); stroke: var(--fzj-blue); }
.cube.below, .cube.above { fill: #fde1bd; stroke: #e08a1e; stroke-width: 2.5; }
.lane { stroke: #9aa9b6; stroke-width: 1.5; stroke-dasharray: 4 3; }
.num { font-size: 17px; font-weight: 700; fill: #444; }
.num.cur { fill: #fff; }
.mark { font-size: 18px; font-weight: 700; }
.mark.bad { fill: #d42d12; }
.mark.good { fill: var(--fzj-blue); }
.leg { font-size: 13px; fill: #444; }
.head { font-size: 19px; font-weight: 700; fill: var(--fzj-blue); }
.head.bad { fill: #d42d12; }
.head.good { fill: var(--fzj-blue); }
.body { font-size: 17px; fill: #444; }
.pop { animation: fadein 0.45s ease 0.15s both; }
@keyframes fadein { from { opacity: 0; } to { opacity: 1; } }
.reveal { opacity: 0; transition: opacity 0.4s ease; }
.reveal.shown { opacity: 1; }
</style>
