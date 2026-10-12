<!--
  Why rotating registers in y/z but warp shuffles in x: loads must be coalesced. One 8×8 slice
  of a cube, stored row by row (x fastest); a sector = 32 bytes = 4 values = half a row.
  Drawn: 8 threads of a warp.
    step 0: the slice and its storage order
    step 1: memory delivers whole 32-byte sectors: 4 values in a row
    step 2: one value needed -> the whole sector is loaded, 3 values for nothing
    step 3: walk in y (z alike): threads side by side in x, each step the warp reads one row
            -> 2 sectors, every value used (coalesced)
    step 4: walk in x: threads stacked in y, each step the warp reads one column
            -> 8 sectors, a quarter of each used (not coalesced)
    step 5: conclusion: y, z -> rotating registers; x -> warp shuffle
-->
<script setup>
import { computed } from 'vue'

const props = defineProps({ step: { type: Number, default: 5 } })

const C = 26, N = 8
const LX = 120, RX = 570, Y0 = 62
const STEP = 3                                  // the row / column read during the walk
const ONE = { x: 5, y: 1 }                      // the single value of step 2
const idx = [0, 1, 2, 3, 4, 5, 6, 7]

// left slice: what each cell shows at the current step
const leftCell = (x, y) => {
  if (props.step === 2) {
    if (x === ONE.x && y === ONE.y) return 'read'
    if (y === ONE.y && Math.floor(x / 4) === Math.floor(ONE.x / 4)) return 'waste'
  }
  if (props.step >= 3 && y === STEP) return 'read'
  return ''
}
const leftText = computed(() => [
  null,
  { title: 'memory delivers 32-byte sectors', sub: '= 4 values in a row (8 bytes each)', cls: 'plain' },
  { title: 'need 1 value → load 4', sub: '3 of the 4 values loaded for nothing ✗', cls: 'bad' },
][props.step] ?? null)
</script>

<template>
  <div class="dt">
    <svg width="900" height="390" viewBox="0 0 900 390">
      <defs>
        <marker id="dt-arr" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L7,3.5 L0,7 Z" fill="#023d6b" />
        </marker>
        <marker id="dt-arr-g" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L7,3.5 L0,7 Z" fill="#666" />
        </marker>
      </defs>

      <!-- left slice -->
      <text :x="LX + 4 * C" y="14" class="hdr" text-anchor="middle">one slice of a cube</text>
      <g v-for="y in idx" :key="'l' + y">
        <rect v-for="x in idx" :key="x" :x="LX + x * C" :y="Y0 + y * C" :width="C" :height="C"
              :class="['cell', leftCell(x, y)]" />
      </g>
      <!-- axes -->
      <line :x1="LX - 62" :y1="Y0 + 8" :x2="LX - 30" :y2="Y0 + 8" class="garrow" marker-end="url(#dt-arr-g)" />
      <text :x="LX - 26" :y="Y0 + 13" class="ax">x</text>
      <line :x1="LX - 62" :y1="Y0 + 8" :x2="LX - 62" :y2="Y0 + 40" class="garrow" marker-end="url(#dt-arr-g)" />
      <text :x="LX - 67" :y="Y0 + 56" class="ax">y</text>
      <!-- storage order -->
      <g class="reveal" :class="{ shown: step <= 2 }">
        <line :x1="LX" :y1="Y0 - 12" :x2="LX + N * C" :y2="Y0 - 12" class="garrow" marker-end="url(#dt-arr-g)" />
        <text :x="LX + 4 * C" :y="Y0 - 20" class="note" text-anchor="middle">stored row by row: <tspan font-style="italic">x</tspan> is contiguous</text>
      </g>
      <!-- all sectors of the slice -->
      <g class="reveal" :class="{ shown: step === 1 }">
        <g v-for="y in idx" :key="'s' + y">
          <rect v-for="h in [0, 1]" :key="h" :x="LX + h * 4 * C" :y="Y0 + y * C" :width="4 * C" :height="C" class="sector" />
        </g>
      </g>
      <!-- the sector of the single value -->
      <rect v-if="step === 2" :x="LX + Math.floor(ONE.x / 4) * 4 * C" :y="Y0 + ONE.y * C" :width="4 * C" :height="C" class="sector pop" />
      <!-- explanation under the slice (steps 1, 2) -->
      <g v-if="leftText" :key="'lt' + step" class="pop">
        <text :x="LX + 4 * C" :y="Y0 + N * C + 26" class="title" text-anchor="middle">{{ leftText.title }}</text>
        <text :x="LX + 4 * C" :y="Y0 + N * C + 48" :class="leftText.cls" text-anchor="middle">{{ leftText.sub }}</text>
      </g>
      <!-- threads side by side in x, walking down in y -->
      <g class="reveal" :class="{ shown: step >= 3 }">
        <g v-for="x in idx" :key="'tl' + x">
          <circle :cx="LX + x * C + C / 2" :cy="Y0 - 16" r="8" class="thread" />
          <line :x1="LX + x * C + C / 2" :y1="Y0 + 4" :x2="LX + x * C + C / 2" :y2="Y0 + N * C - 6" class="walk" marker-end="url(#dt-arr)" />
        </g>
        <rect v-for="h in [0, 1]" :key="'sl' + h" :x="LX + h * 4 * C" :y="Y0 + STEP * C" :width="4 * C" :height="C" class="sector" />
        <text :x="LX + 4 * C" :y="Y0 + N * C + 26" class="title" text-anchor="middle">walk in <tspan font-style="italic">y</tspan> (or <tspan font-style="italic">z</tspan>)</text>
        <text :x="LX + 4 * C" :y="Y0 + N * C + 48" class="good" text-anchor="middle">coalesced: one row = 2 sectors, all used ✓</text>
      </g>

      <!-- right: walk in x -->
      <g class="reveal" :class="{ shown: step >= 4 }">
        <g v-for="y in idx" :key="'r' + y">
          <rect v-for="x in idx" :key="x" :x="RX + x * C" :y="Y0 + y * C" :width="C" :height="C"
                :class="['cell', x === STEP ? 'read' : x < 4 ? 'waste' : '']" />
          <rect :x="RX" :y="Y0 + y * C" :width="4 * C" :height="C" class="sector" />
        </g>
        <g v-for="y in idx" :key="'tr' + y">
          <circle :cx="RX - 16" :cy="Y0 + y * C + C / 2" r="8" class="thread" />
          <line :x1="RX + 4" :y1="Y0 + y * C + C / 2" :x2="RX + N * C - 6" :y2="Y0 + y * C + C / 2" class="walk" marker-end="url(#dt-arr)" />
        </g>
        <text :x="RX + 4 * C" :y="Y0 + N * C + 26" class="title" text-anchor="middle">walk in <tspan font-style="italic">x</tspan></text>
        <text :x="RX + 4 * C" :y="Y0 + N * C + 48" class="bad" text-anchor="middle">not coalesced: one column = 8 sectors, ¼ used ✗</text>
      </g>

      <!-- legend: items appear when they are first used -->
      <g transform="translate(332 18)">
        <g class="reveal" :class="{ shown: step >= 3 }">
          <circle cx="7" cy="-4" r="7" class="thread" /><text x="20" y="0" class="leg">thread of a warp</text>
        </g>
        <g class="reveal" :class="{ shown: step >= 2 }">
          <rect x="136" y="-11" width="14" height="14" class="cell read" /><text x="156" y="0" class="leg">value needed</text>
          <rect x="256" y="-11" width="14" height="14" class="cell waste" /><text x="276" y="0" class="leg">loaded, not needed</text>
        </g>
        <g class="reveal" :class="{ shown: step >= 1 }">
          <rect x="410" y="-11" width="28" height="14" class="sector" /><text x="444" y="0" class="leg">32-byte sector</text>
        </g>
      </g>

      <!-- conclusion -->
      <g class="reveal" :class="{ shown: step >= 5 }">
        <rect x="40" y="338" width="390" height="40" rx="6" class="conc" />
        <text x="235" y="364" class="conctext" text-anchor="middle"><tspan font-style="italic">y</tspan>, <tspan font-style="italic">z</tspan> → rotating registers</text>
        <rect x="470" y="338" width="390" height="40" rx="6" class="conc" />
        <text x="665" y="364" class="conctext" text-anchor="middle"><tspan font-style="italic">x</tspan> → side by side in <tspan font-style="italic">x</tspan> + warp shuffle</text>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.dt { display: flex; flex-direction: column; align-items: center; }
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.hdr { font-size: 14px; fill: #666; }
.note { font-size: 14px; fill: #444; }
.ax { font-size: 15px; font-style: italic; font-weight: 700; fill: #444; }
.garrow { stroke: #666; stroke-width: 1.5; }
.cell { fill: #f2f4f6; stroke: #d0d5da; stroke-width: 0.8; transition: fill 0.4s ease; }
.cell.read { fill: var(--fzj-blue); }
.cell.waste { fill: #fde1bd; }
.sector { fill: none; stroke: #e08a1e; stroke-width: 2; }
.thread { fill: #fff; stroke: var(--fzj-blue); stroke-width: 2; }
.walk { stroke: var(--fzj-blue); stroke-width: 1.4; stroke-dasharray: 4 3; opacity: 0.6; }
.title { font-size: 18px; font-weight: 700; fill: var(--fzj-blue); }
.good { font-size: 15px; font-weight: 700; fill: var(--fzj-blue); }
.bad { font-size: 15px; font-weight: 700; fill: #d42d12; }
.plain { font-size: 15px; font-weight: 700; fill: #444; }
.pop { animation: fadein 0.4s ease both; }
@keyframes fadein { from { opacity: 0; } to { opacity: 1; } }
.leg { font-size: 12.5px; fill: #444; }
.conc { fill: #f4f7fa; stroke: var(--fzj-blue); stroke-width: 1.5; }
.conctext { font-size: 18px; font-weight: 700; fill: var(--fzj-blue); }
.reveal { opacity: 0; transition: opacity 0.4s ease; }
.reveal.shown { opacity: 1; }
</style>
