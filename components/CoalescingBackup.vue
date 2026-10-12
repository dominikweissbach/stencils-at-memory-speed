<!--
  Backup: what ONE load instruction of a warp touches when rotating registers walk along
  z, y or x (Laplace16th_multipass, offset_ortho). Four z-slices of a cube, each 8×8 values,
  stored row by row (x fastest). A sector = 32 bytes = 4 doubles = half a row.
    z (stride 64): threads (x, y = 0..3)       -> 4 full rows of one slice      -> 8 sectors
    y (stride 8):  threads (x, slice = 0..3)   -> 1 full row in each of 4 slices -> 8 sectors
    x (stride 1):  threads (row, slice = 0..3) -> 1 value per row (one column)  -> 32 sectors
-->
<script setup>
const C = 9, SLICE_GAP = 12, ROW_H = 96
const cases = [
  { dir: 'z', cells: (s, y, x) => s === 0 && y < 4, sectors: 8 },
  { dir: 'y', cells: (s, y, x) => y === 0, sectors: 8 },
  { dir: 'x', cells: (s, y, x) => x === 0, sectors: 32 },
]
const idx = [0, 1, 2, 3, 4, 5, 6, 7]
const sliceX = s => 110 + s * (8 * C + SLICE_GAP)
const touched = (c, s, y, h) => [0, 1, 2, 3].some(k => c.cells(s, y, h * 4 + k))
</script>

<template>
  <svg width="900" height="320" viewBox="0 0 900 320">
    <defs>
      <marker id="cb-arr" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto" markerUnits="userSpaceOnUse">
        <path d="M0,0 L7,3.5 L0,7 Z" fill="#444" />
      </marker>
    </defs>
    <!-- coordinate system: x right and y down within a slice, z across the slices -->
    <g class="axes">
      <line x1="22" y1="-14" x2="62" y2="-14" marker-end="url(#cb-arr)" />
      <text x="68" y="-10" class="axlbl">x</text>
      <line x1="22" y1="-14" x2="22" y2="24" marker-end="url(#cb-arr)" />
      <text x="17" y="38" class="axlbl" text-anchor="middle">y</text>
      <line :x1="sliceX(0)" y1="-10" :x2="sliceX(3) + 8 * C" y2="-10" marker-end="url(#cb-arr)" />
      <text :x="sliceX(3) + 8 * C + 8" y="-6" class="axlbl">z</text>
    </g>
    <text :x="sliceX(0) + 4 * C" y="14" class="hdr" text-anchor="middle">slice 0</text>
    <text :x="sliceX(1) + 4 * C" y="14" class="hdr" text-anchor="middle">slice 1</text>
    <text :x="sliceX(2) + 4 * C" y="14" class="hdr" text-anchor="middle">slice 2</text>
    <text :x="sliceX(3) + 4 * C" y="14" class="hdr" text-anchor="middle">slice 3</text>
    <g v-for="(c, r) in cases" :key="c.dir" :transform="`translate(0 ${26 + r * ROW_H})`">
      <text x="96" :y="4 * C" class="dir" :class="{ bad: c.dir === 'x' }" text-anchor="end" dominant-baseline="middle">walk in {{ c.dir }}</text>
      <g v-for="s in 4" :key="s">
        <g v-for="y in idx" :key="y">
          <rect v-for="x in idx" :key="x" :x="sliceX(s - 1) + x * C" :y="y * C" :width="C" :height="C"
                :class="['cell', { read: c.cells(s - 1, y, x), bad: c.dir === 'x' }]" />
          <rect v-for="h in [0, 1]" v-show="touched(c, s - 1, y, h)" :key="'s' + h"
                :x="sliceX(s - 1) + h * 4 * C" :y="y * C" :width="4 * C" :height="C" class="sector" />
        </g>
      </g>
      <text :x="sliceX(3) + 8 * C + 34" :y="2.5 * C" class="stat" :class="{ bad: c.dir === 'x' }">
        {{ c.sectors }} sectors · {{ c.sectors * 32 }} B
      </text>
      <text :x="sliceX(3) + 8 * C + 34" :y="5.5 * C" class="sub" :class="{ bad: c.dir === 'x' }">
        {{ c.dir === 'x' ? 'only 25 % of each sector used' : 'every byte used' }}
      </text>
    </g>
    <g transform="translate(110 312)">
      <rect width="9" height="9" class="cell read" /><text x="14" y="8" class="leg">value read by one load of the warp (32 threads)</text>
      <rect x="370" width="36" height="9" class="sector" /><text x="412" y="8" class="leg">32-byte sector fetched</text>
    </g>
  </svg>
</template>

<style scoped>
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.hdr { font-size: 12px; fill: #666; }
.axes line { stroke: #444; stroke-width: 1.5; }
.axlbl { font-size: 15px; font-style: italic; font-weight: 700; fill: #444; }
.dir { font-size: 17px; font-weight: 700; fill: var(--fzj-blue); }
.dir.bad { fill: #d42d12; }
.cell { fill: #f2f4f6; stroke: #d0d5da; stroke-width: 0.5; }
.cell.read { fill: var(--fzj-blue); }
.cell.read.bad { fill: #d42d12; }
.sector { fill: none; stroke: #e08a1e; stroke-width: 1.5; }
.stat { font-size: 18px; font-weight: 700; fill: var(--fzj-blue); }
.stat.bad { fill: #d42d12; }
.sub { font-size: 14px; fill: #444; }
.sub.bad { fill: #d42d12; }
.leg { font-size: 12px; fill: #444; }
</style>
