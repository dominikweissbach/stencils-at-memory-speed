<!--
  Rotating registers (Laplace16th_multipass along y or z), step by step.
  One thread walks one column: positions -8..-1 cube below, 0..7 own cube, 8..15 cube above.
  It holds 17 values in registers r-8 .. r+8 (r_m8 .. r_p8 in the code).
    step 0: column, empty registers, empty results
    step 1: start: load -8..7 into r-8 .. r+7 (16 loads)
    step 2: point 0: load 8 into r+8, compute point 0 from r-8 .. r+8
    step 3: shift: every value moves one register down (r+8 free again, -8 dropped)
    step 4 … 10: points 1 … 7: shift + 1 load + compute (one click each)
  State: after the load for point p, register r_k holds value p + k.
-->
<script setup>
import { computed } from 'vue'

const props = defineProps({ step: { type: Number, default: 10 } })

const CELL = 30, BLOCK_GAP = 34, MEM_X0 = 102, MEM_Y = 30
const REG_W = 40, REG_X0 = 110, REG_Y = 190, REG_H = 30
const OUT_W = 34, OUT_Y = 318
const OUT_X0 = (900 - 8 * OUT_W) / 2 + 60

const positions = Array.from({ length: 24 }, (_, i) => i - 8)          // -8 .. 15
const block = x => (x < 0 ? 0 : x < 8 ? 1 : 2)
const blockX = b => MEM_X0 + b * (8 * CELL + BLOCK_GAP)
const memX = x => blockX(block(x)) + (((x % 8) + 8) % 8) * CELL
const regX = k => REG_X0 + (k + 8) * REG_W                             // k = -8 .. 8
const cube = x => ['below', 'own', 'above'][block(x)]

// state from the click: base = p such that r_k holds base + k; maxLoaded = highest value loaded
const state = computed(() => {
  const s = props.step
  if (s <= 0) return { base: 0, maxLoaded: -9, point: -1, mode: 'empty' }
  if (s === 1) return { base: 0, maxLoaded: 7, point: -1, mode: 'start' }
  if (s === 2) return { base: 0, maxLoaded: 8, point: 0, mode: 'compute' }
  if (s === 3) return { base: 1, maxLoaded: 8, point: 0, mode: 'shift' }
  const p = Math.min(7, s - 3)
  return { base: p, maxLoaded: p + 8, point: p, mode: 'compute' }
})
const loads = computed(() => Math.max(0, state.value.maxLoaded + 9))   // values -8 .. maxLoaded
const done = computed(() => state.value.point + 1)

// every value is a chip: in memory (not loaded yet), in a register, or dropped (left of r-8)
const chips = positions.map(x => x)
function chip(x) {
  const { base, maxLoaded } = state.value
  const k = x - base
  if (x > maxLoaded) return { x: memX(x) + 3, y: MEM_Y + 3, op: 0 }
  if (k < -8) return { x: regX(-8) - 46, y: REG_Y + 3, op: 0 }
  return { x: regX(k) + (REG_W - 24) / 2, y: REG_Y + 3, op: 1 }
}
const isCentre = x => state.value.mode !== 'empty' && state.value.mode !== 'start' && x === state.value.base
  && state.value.mode === 'compute'
const newLoad = computed(() => (state.value.mode === 'compute' ? state.value.maxLoaded : null))

const REG_NAMES = Array.from({ length: 17 }, (_, i) => i - 8)
const regLabel = k => (k === 0 ? 'r0' : k < 0 ? `r−${-k}` : `r+${k}`)

</script>

<template>
  <div class="rr">
    <svg width="900" height="380" viewBox="0 0 900 380">
      <defs>
        <marker id="rr-arr" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L8,4 L0,8 Z" fill="#e08a1e" />
        </marker>
        <marker id="rr-arr-b" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L8,4 L0,8 Z" fill="#023d6b" />
        </marker>
      </defs>

      <!-- the column in memory -->
      <text :x="MEM_X0 - 10" :y="MEM_Y + CELL / 2" class="side" text-anchor="end" dominant-baseline="middle">column</text>
      <g v-for="(n, b) in ['neighbouring cube below', 'own cube', 'neighbouring cube above']" :key="n">
        <text :x="blockX(b) + 4 * CELL" :y="MEM_Y - 9" class="cubename" text-anchor="middle">{{ n }}</text>
      </g>
      <text :x="blockX(1) - BLOCK_GAP / 2" :y="MEM_Y + CELL / 2" class="dots" text-anchor="middle" dominant-baseline="middle">…</text>
      <text :x="blockX(2) - BLOCK_GAP / 2" :y="MEM_Y + CELL / 2" class="dots" text-anchor="middle" dominant-baseline="middle">…</text>
      <g v-for="x in positions" :key="'m' + x">
        <rect :x="memX(x)" :y="MEM_Y" :width="CELL" :height="CELL"
              :class="['mem', cube(x), { loaded: x <= state.maxLoaded, fresh: x === newLoad }]" />
        <text :x="memX(x) + CELL / 2" :y="MEM_Y + CELL / 2 + 1" class="memtext" text-anchor="middle" dominant-baseline="middle">{{ x }}</text>
      </g>

      <!-- the one new load of this point -->
      <g v-if="newLoad !== null" :key="'ld' + newLoad" class="pop">
        <path :d="`M${memX(newLoad) + CELL / 2},${MEM_Y + CELL + 2} C${memX(newLoad) + CELL / 2},${MEM_Y + CELL + 70} ${regX(8) + REG_W / 2},${REG_Y - 60} ${regX(8) + REG_W / 2},${REG_Y - 4}`"
              class="loadarrow" marker-end="url(#rr-arr)" />
        <text :x="memX(newLoad) - 4" :y="MEM_Y + CELL + 40" class="loadlbl" text-anchor="end">1 new load</text>
      </g>

      <!-- the thread and its 17 registers -->
      <rect :x="REG_X0 - 12" :y="REG_Y - 30" :width="17 * REG_W + 24" :height="REG_H + 64" rx="8" class="thread" />
      <text :x="REG_X0 - 2" :y="REG_Y - 12" class="tname">one thread: 17 registers</text>
      <g v-for="k in REG_NAMES" :key="'r' + k">
        <rect :x="regX(k) + 2" :y="REG_Y" :width="REG_W - 4" :height="REG_H" :class="['reg', { centre: k === 0 }]" />
        <text :x="regX(k) + REG_W / 2" :y="REG_Y + REG_H + 14" class="reglbl" :class="{ c: k === 0 }" text-anchor="middle">{{ regLabel(k) }}</text>
      </g>

      <!-- the values -->
      <g v-for="x in chips" :key="'c' + x" class="chip"
         :style="{ transform: `translate(${chip(x).x}px, ${chip(x).y}px)`, opacity: chip(x).op, transitionDelay: step === 1 ? `${(x + 8) * 30}ms` : '0ms' }">
        <rect width="24" height="24" :class="['chiprect', cube(x), { centre: isCentre(x) }]" />
        <text x="12" y="13" class="chiptext" text-anchor="middle" dominant-baseline="middle">{{ x }}</text>
      </g>

      <!-- compute: all 17 registers -> result of the current point -->
      <g v-if="state.mode === 'compute'" :key="'cp' + state.point" class="pop">
        <path :d="`M${regX(-8) + 4},${REG_Y + REG_H + 22} L${regX(-8) + 4},${REG_Y + REG_H + 28} L${regX(8) + REG_W - 4},${REG_Y + REG_H + 28} L${regX(8) + REG_W - 4},${REG_Y + REG_H + 22}`"
              class="bracket" />
        <path :d="`M${regX(0) + REG_W / 2},${REG_Y + REG_H + 28} C${regX(0) + REG_W / 2},${OUT_Y - 18} ${OUT_X0 + state.point * OUT_W + OUT_W / 2},${REG_Y + REG_H + 40} ${OUT_X0 + state.point * OUT_W + OUT_W / 2},${OUT_Y - 4}`"
              class="computearrow" marker-end="url(#rr-arr-b)" />
      </g>

      <!-- results of the thread's 8 points -->
      <text :x="OUT_X0 - 12" :y="OUT_Y + OUT_W / 2" class="side" text-anchor="end" dominant-baseline="middle">results</text>
      <g v-for="i in 8" :key="'o' + i">
        <rect :x="OUT_X0 + (i - 1) * OUT_W" :y="OUT_Y" :width="OUT_W - 2" :height="OUT_W - 2"
              :class="['out', { done: i - 1 <= state.point, current: i - 1 === state.point && state.mode === 'compute' }]" />
        <text :x="OUT_X0 + (i - 1) * OUT_W + OUT_W / 2 - 1" :y="OUT_Y + OUT_W / 2" class="outtext"
              :class="{ done: i - 1 <= state.point }" text-anchor="middle" dominant-baseline="middle">{{ i - 1 }}</text>
      </g>
      <text :x="OUT_X0 + 8 * OUT_W + 16" :y="OUT_Y + 10" class="counter">{{ loads }} loads</text>
      <text :x="OUT_X0 + 8 * OUT_W + 16" :y="OUT_Y + 30" class="counter">{{ Math.max(0, done) }} / 8 points</text>
    </svg>
  </div>
</template>

<style scoped>
.rr { display: flex; flex-direction: column; align-items: center; }
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.side { font-size: 15px; fill: #444; }
.cubename { font-size: 14px; fill: #444; }
.dots { font-size: 18px; fill: #888; }
.mem { stroke: #fff; stroke-width: 1; transition: fill 0.4s ease; }
.mem.below, .mem.above { fill: #7494ae; }
.mem.own { fill: var(--fzj-blue); }
.mem.loaded { fill: #d3d9df; }
.mem.fresh { fill: #fab45a; }
.memtext { font-size: 11.5px; font-weight: 700; fill: #fff; pointer-events: none; }
.thread { fill: #f4f7fa; stroke: var(--fzj-blue); stroke-width: 1.5; }
.tname { font-size: 14px; font-weight: 700; fill: var(--fzj-blue); }
.reg { fill: #fff; stroke: #999; stroke-width: 1; stroke-dasharray: 3 2; }
.reg.centre { stroke: var(--fzj-blue); stroke-width: 2; stroke-dasharray: none; }
.reglbl { font-size: 11px; fill: #666; }
.reglbl.c { fill: var(--fzj-blue); font-weight: 700; }
.chip { transition: transform 0.6s cubic-bezier(.4, 0, .2, 1), opacity 0.5s ease; }
.chiprect { stroke: #fff; stroke-width: 1; }
.chiprect.below, .chiprect.above { fill: #7494ae; }
.chiprect.own { fill: var(--fzj-blue); }
.chiprect.centre { fill: #e08a1e; }
.chiptext { font-size: 11.5px; font-weight: 700; fill: #fff; }
.pop { animation: fadein 0.45s ease 0.25s both; }
@keyframes fadein { from { opacity: 0; } to { opacity: 1; } }
.loadarrow { fill: none; stroke: #e08a1e; stroke-width: 2.5; stroke-dasharray: 6 4; }
.loadlbl { font-size: 13px; font-weight: 700; fill: #e08a1e; }
.bracket { fill: none; stroke: var(--fzj-blue); stroke-width: 2; }
.computearrow { fill: none; stroke: var(--fzj-blue); stroke-width: 2.5; }
.out { fill: #fff; stroke: #bbb; stroke-width: 1; transition: fill 0.4s ease; }
.out.done { fill: #c9d6e2; }
.out.current { fill: var(--fzj-blue); }
.outtext { font-size: 12px; fill: #999; }
.outtext.done { fill: #000; font-weight: 700; }
.out.current + .outtext { fill: #fff; }
.counter { font-size: 15px; font-weight: 700; fill: #444; }
</style>
