<!--
  Shared-memory kernel (Laplace16th_yz_shared) for one column of a block, simplified:
  positions along y or z: -8..-1 cube below, 0..7 own cube, 8..15 cube above; thread t computes point t.
    step 0: memory, empty shared memory, 8 threads
    step 1: the block loads all 24 values into shared memory (each value once)
    step 2: barrier (__syncthreads): nobody reads before everything is loaded
    step 3: example thread 1 takes its 17 values (-7 … 9) and computes point 1
    step 4: example thread 6 takes its 17 values (-2 … 14) and computes point 6
-->
<script setup>
import { computed } from 'vue'

const props = defineProps({ step: { type: Number, default: 4 } })

const CELL = 30, BLOCK_GAP = 34, MEM_X0 = 102, MEM_Y = 30
const SH_X0 = 136, SH_Y = 128
const BAR_Y = 234
const BOX_W = 84, BOX_H = 58, GAP = 18, BOX_X0 = 51, BOX_Y = 262

const positions = Array.from({ length: 24 }, (_, p) => p - 8)          // -8 .. 15
const block = x => (x < 0 ? 0 : x < 8 ? 1 : 2)
const blockX = b => MEM_X0 + b * (8 * CELL + BLOCK_GAP)
const memX = x => blockX(block(x)) + (((x % 8) + 8) % 8) * CELL
const shX = x => SH_X0 + (x + 8) * CELL
const boxX = t => BOX_X0 + t * (BOX_W + GAP)
const cube = x => ['below', 'own', 'above'][block(x)]

// the two example threads, each with its own colour and bracket level below the tile
const EXAMPLES = [
  { t: 1, cls: 'exa', step: 3, dy: 17, lblX: x => x - 6, anchor: 'end' },
  { t: 6, cls: 'exb', step: 4, dy: 30, lblX: x => x + 6, anchor: 'start' },
]
const shown = computed(() => EXAMPLES.filter(e => props.step >= e.step))
const example = t => EXAMPLES.find(e => e.t === t && props.step >= e.step)

const chipPos = x => (props.step >= 1 ? { x: shX(x) + 3, y: SH_Y + 3 } : { x: memX(x) + 3, y: MEM_Y + 3 })
const done = t => EXAMPLES.some(e => e.t === t && props.step >= e.step)
</script>

<template>
  <div class="sm">
    <svg width="900" height="390" viewBox="0 0 900 390">
      <defs>
        <marker id="sm-arr-exa" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L8,4 L0,8 Z" fill="#e08a1e" />
        </marker>
        <marker id="sm-arr-exb" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L8,4 L0,8 Z" fill="#023d6b" />
        </marker>
      </defs>

      <!-- global memory: three cubes, stored apart -->
      <text :x="MEM_X0 - 10" :y="MEM_Y + CELL / 2" class="side" text-anchor="end" dominant-baseline="middle">memory</text>
      <g v-for="(n, b) in ['neighbouring cube below', 'own cube', 'neighbouring cube above']" :key="n">
        <text :x="blockX(b) + 4 * CELL" :y="MEM_Y - 9" class="cubename" text-anchor="middle">{{ n }}</text>
      </g>
      <text :x="blockX(1) - BLOCK_GAP / 2" :y="MEM_Y + CELL / 2" class="dots" text-anchor="middle" dominant-baseline="middle">…</text>
      <text :x="blockX(2) - BLOCK_GAP / 2" :y="MEM_Y + CELL / 2" class="dots" text-anchor="middle" dominant-baseline="middle">…</text>
      <g v-for="x in positions" :key="'m' + x">
        <rect :x="memX(x)" :y="MEM_Y" :width="CELL" :height="CELL"
              :class="['mem', cube(x)]" />
        <text :x="memX(x) + CELL / 2" :y="MEM_Y + CELL / 2 + 1" class="memtext" text-anchor="middle" dominant-baseline="middle">{{ x }}</text>
      </g>

      <!-- shared memory of the block: one contiguous tile -->
      <rect :x="SH_X0 - 10" :y="SH_Y - 26" :width="24 * CELL + 20" :height="CELL + 34" rx="8" class="shbox" />
      <text :x="SH_X0 - 2" :y="SH_Y - 9" class="shname">shared memory of the block (visible to all its threads)</text>
      <rect v-for="x in positions" :key="'s' + x" :x="shX(x)" :y="SH_Y" :width="CELL" :height="CELL"
            class="slot" />

      <!-- the values -->
      <g v-for="x in positions" :key="'c' + x" class="chip"
         :style="{ transform: `translate(${chipPos(x).x}px, ${chipPos(x).y}px)`, transitionDelay: step === 1 ? `${(x + 8) * 20}ms` : '0ms' }">
        <rect width="24" height="24" :class="['chiprect', cube(x)]" />
        <text x="12" y="13" class="chiptext" text-anchor="middle" dominant-baseline="middle">{{ x }}</text>
      </g>

      <!-- barrier -->
      <g class="reveal" :class="{ shown: step >= 2 }">
        <line x1="172" :y1="BAR_Y" x2="860" :y2="BAR_Y" class="barrier" />
        <text x="164" :y="BAR_Y - 4" class="barlbl" text-anchor="end">barrier</text>
        <text x="164" :y="BAR_Y + 12" class="barsub" text-anchor="end">wait until all are loaded</text>
      </g>

      <!-- example threads take their 17 values from shared memory -->
      <g v-for="e in shown" :key="'rd' + e.t" :class="['pop', e.cls]">
        <path :d="`M${shX(e.t - 8) + 3},${SH_Y + CELL + e.dy - 5} L${shX(e.t - 8) + 3},${SH_Y + CELL + e.dy} L${shX(e.t + 8) + CELL - 3},${SH_Y + CELL + e.dy} L${shX(e.t + 8) + CELL - 3},${SH_Y + CELL + e.dy - 5}`"
              class="bracket" />
        <path :d="`M${shX(e.t) + CELL / 2},${SH_Y + CELL + e.dy} C${shX(e.t) + CELL / 2},${BOX_Y - 30} ${boxX(e.t) + BOX_W / 2},${SH_Y + CELL + 50} ${boxX(e.t) + BOX_W / 2},${BOX_Y - 4}`"
              class="readarrow" :marker-end="`url(#sm-arr-${e.cls})`" />
        <text :x="e.lblX(e.anchor === 'end' ? shX(e.t - 8) : shX(e.t + 8) + CELL)" :y="SH_Y + CELL + e.dy + 5"
              class="readlbl" :text-anchor="e.anchor">17 values</text>
      </g>

      <!-- the threads of this column -->
      <g v-for="t in 8" :key="'t' + t">
        <rect :x="boxX(t - 1)" :y="BOX_Y" :width="BOX_W" :height="BOX_H" rx="6"
              :class="['box', example(t - 1)?.cls]" />
        <text :x="boxX(t - 1) + BOX_W / 2" :y="BOX_Y + 18" class="tname" text-anchor="middle">thread {{ t - 1 }}</text>
        <rect :x="boxX(t - 1) + 14" :y="BOX_Y + 28" :width="BOX_W - 28" height="22" rx="3"
              :class="['res', { done: done(t - 1) }]" />
        <text :x="boxX(t - 1) + BOX_W / 2" :y="BOX_Y + 40" class="restext" :class="{ done: done(t - 1) }"
              text-anchor="middle" dominant-baseline="middle">point {{ t - 1 }}</text>
      </g>

    </svg>
  </div>
</template>

<style scoped>
.sm { display: flex; flex-direction: column; align-items: center; }
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.side { font-size: 15px; fill: #444; }
.cubename { font-size: 14px; fill: #444; }
.dots { font-size: 18px; fill: #888; }
/* memory keeps its values after the loads (loads copy): light fill per cube */
.mem { stroke: #bbb; stroke-width: 1; transition: fill 0.4s ease; }
.mem.below, .mem.above { fill: #e1e8ed; }
.mem.own { fill: #c9d6e2; }
.memtext { font-size: 11px; fill: #888; }
.shbox { fill: #f4f7fa; stroke: var(--fzj-blue); stroke-width: 1.5; }
.shname { font-size: 14px; font-weight: 700; fill: var(--fzj-blue); }
.slot { fill: #fff; stroke: #999; stroke-width: 1; stroke-dasharray: 3 2; transition: fill 0.4s ease; }
.chip { transition: transform 0.7s cubic-bezier(.4, 0, .2, 1); }
.chiprect { stroke: #fff; stroke-width: 1; }
.chiprect.below, .chiprect.above { fill: #7494ae; }
.chiprect.own { fill: var(--fzj-blue); }
.chiptext { font-size: 11.5px; font-weight: 700; fill: #fff; pointer-events: none; }
.barrier { stroke: #444; stroke-width: 2.5; stroke-dasharray: 10 6; }
.barlbl { font-size: 15px; font-weight: 700; fill: #444; }
.barsub { font-size: 11.5px; fill: #444; }
.bracket { fill: none; stroke-width: 2.5; }
.readarrow { fill: none; stroke-width: 2.5; }
.readlbl { font-size: 13px; font-weight: 700; }
.exa .bracket, .exa .readarrow { stroke: #e08a1e; }
.exa .readlbl { fill: #e08a1e; }
.exb .bracket, .exb .readarrow { stroke: #023d6b; }
.exb .readlbl { fill: #023d6b; }
.box { fill: #f4f7fa; stroke: var(--fzj-blue); stroke-width: 1.5; transition: fill 0.4s ease, stroke-width 0.4s ease; }
.box.exa { fill: #fff3e3; stroke: #e08a1e; stroke-width: 3; }
.box.exb { fill: #e6eef5; stroke: #023d6b; stroke-width: 3; }
.tname { font-size: 15px; font-weight: 700; fill: var(--fzj-blue); }
.res { fill: #fff; stroke: #bbb; stroke-width: 1; transition: fill 0.4s ease; }
.res.done { fill: var(--fzj-blue); stroke: var(--fzj-blue); }
.restext { font-size: 12px; fill: #999; }
.restext.done { fill: #fff; font-weight: 700; }
.pop { animation: fadein 0.45s ease 0.25s both; }
@keyframes fadein { from { opacity: 0; } to { opacity: 1; } }
.reveal { opacity: 0; transition: opacity 0.4s ease; }
.reveal.shown { opacity: 1; }
</style>
