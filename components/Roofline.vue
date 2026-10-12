<!--
  Roofline sketch for the GH200 (Hopper GPU): reachable FP64 performance vs. flops per byte.
  Memory roof = 4 TB/s × (flops/byte), compute roof = 34 TFLOP/s, ridge at 8.5 flops/byte.
  Built up with `step`: 1 = roofs and the ideal stencil point, 2 = + the realistic 3-pass points.
-->
<script setup>
defineProps({ step: { type: Number, default: 2 } })

const BW = 4          // TB/s
const PEAK = 34       // TFLOP/s
const RIDGE = PEAK / BW
const W = 440, H = 270
const X0 = 0.1, X1 = 100, Y0 = 0.1, Y1 = 100   // flops/byte, TFLOP/s
const sx = ai => Math.log10(ai / X0) / Math.log10(X1 / X0) * W
const sy = p => H - Math.log10(p / Y0) / Math.log10(Y1 / Y0) * H
const roof = ai => Math.min(BW * ai, PEAK)

const xTicks = [0.1, 1, 10, 100]
const yTicks = [0.1, 1, 10, 100]
// labels above-left of a point are always above the rising roof; the last one goes below-right
const points = [
  { ai: 4.9, label: 'ideal', sub: '16 B / point', step: 1, cls: 'ideal', dx: -12, dy: -26, anchor: 'end' },
  { ai: 1.2, label: '3 passes', sub: '64 B / point', step: 2, cls: 'real', dx: -46, dy: -46, anchor: 'end', leader: true },
  { ai: 0.7, label: '+ halo reloads', sub: '112 B / point', step: 2, cls: 'halo', dx: 46, dy: 40, anchor: 'start', leader: true },
]
const ANGLE = -Math.atan((H / 3) / (W / 3)) * 180 / Math.PI  // slope of the memory roof on screen
</script>

<template>
  <svg :width="W + 70" :height="H + 50" :viewBox="`-60 -14 ${W + 74} ${H + 58}`"
       class="reveal" :class="{ shown: step >= 1 }">
    <!-- regions -->
    <g class="reveal" :class="{ shown: step >= 1 }">
      <rect x="0" y="0" :width="sx(RIDGE)" :height="H" class="mem-region" />
      <text :x="sx(RIDGE) / 2" y="18" class="region" text-anchor="middle">memory-bound</text>
      <text :x="(sx(RIDGE) + W) / 2" y="18" class="region" text-anchor="middle">compute-bound</text>
    </g>

    <!-- axes -->
    <g v-for="t in yTicks" :key="'y' + t">
      <line x1="-4" :y1="sy(t)" :x2="W" :y2="sy(t)" class="grid" />
      <text x="-8" :y="sy(t)" class="tick" text-anchor="end" dominant-baseline="middle">{{ t }}</text>
    </g>
    <g v-for="t in xTicks" :key="'x' + t">
      <line :x1="sx(t)" :y1="H" :x2="sx(t)" :y2="H + 4" class="axis" />
      <text :x="sx(t)" :y="H + 18" class="tick" text-anchor="middle">{{ t }}</text>
    </g>
    <line x1="0" :y1="H" :x2="W" :y2="H" class="axis" />
    <line x1="0" y1="0" x2="0" :y2="H" class="axis" />
    <text :x="W / 2" :y="H + 38" class="axislbl" text-anchor="middle">flops per byte moved</text>
    <text :transform="`translate(-46 ${H / 2}) rotate(-90)`" class="axislbl" text-anchor="middle">reachable TFLOP/s (FP64)</text>

    <!-- roofs and ridge -->
    <g class="reveal" :class="{ shown: step >= 1 }">
      <polyline :points="`${sx(X0)},${sy(roof(X0))} ${sx(RIDGE)},${sy(PEAK)} ${sx(X1)},${sy(PEAK)}`" class="roof" />
      <line :x1="sx(RIDGE)" :y1="sy(PEAK)" :x2="sx(RIDGE)" :y2="H" class="ridge" />
      <text :x="sx(RIDGE) + 5" :y="H - 8" class="ridgelbl">8.5</text>
      <text :x="sx(0.22)" :y="sy(roof(0.22)) + 18" class="rooflbl" text-anchor="middle"
            :transform="`rotate(${ANGLE} ${sx(0.22)} ${sy(roof(0.22)) + 18})`">memory: 4 TB/s</text>
      <text :x="sx(30)" :y="sy(PEAK) + 20" class="rooflbl" text-anchor="middle">compute: 34 TFLOP/s</text>
    </g>

    <!-- stencil points: on the memory roof -->
    <g v-for="p in points" :key="p.label" class="reveal" :class="{ shown: step >= p.step }">
      <line :x1="sx(p.ai)" :y1="sy(roof(p.ai))" :x2="sx(p.ai)" :y2="H" class="drop" />
      <line v-if="p.leader" :x1="sx(p.ai)" :y1="sy(roof(p.ai))" :x2="sx(p.ai) + p.dx * 0.85"
            :y2="sy(roof(p.ai)) + p.dy + (p.dy < 0 ? 10 : -14)" :class="['leader', p.cls]" />
      <circle :cx="sx(p.ai)" :cy="sy(roof(p.ai))" r="6.5" :class="['pt', p.cls]" />
      <text :x="sx(p.ai)" :y="H - 6" text-anchor="middle" :class="['aival', p.cls]">{{ p.ai }}</text>
      <text :x="sx(p.ai) + p.dx" :y="sy(roof(p.ai)) + p.dy" :text-anchor="p.anchor" :class="['ptlbl', p.cls]">{{ p.label }}</text>
      <text :x="sx(p.ai) + p.dx" :y="sy(roof(p.ai)) + p.dy + 15" :text-anchor="p.anchor" class="ptsub">{{ p.sub }}</text>
    </g>
  </svg>
</template>

<style scoped>
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.axis { stroke: #000; stroke-width: 1; }
.grid { stroke: #e6e6e6; stroke-width: 1; }
.tick { font-size: 13px; }
.axislbl { font-size: 14px; }
.mem-region { fill: #f2f5f8; }
.region { font-size: 13px; fill: #888; }
.roof { fill: none; stroke: var(--fzj-blue); stroke-width: 3; }
.ridge { stroke: var(--fzj-branchred); stroke-width: 1.5; stroke-dasharray: 5 4; }
.ridgelbl { font-size: 14px; font-weight: 700; fill: var(--fzj-branchred); }
.rooflbl { font-size: 13px; fill: var(--fzj-blue); }
.drop { stroke: #aaa; stroke-width: 1; stroke-dasharray: 2 3; }
.pt.ideal { fill: var(--fzj-blue); }
.pt.real { fill: var(--fzj-orange); }
.pt.halo { fill: var(--fzj-branchred); }
.leader { stroke-width: 1.3; }
.leader.real { stroke: #c77a1a; }
.leader.halo { stroke: var(--fzj-branchred); }
.aival { font-size: 12px; font-weight: 700; paint-order: stroke; stroke: #f2f5f8; stroke-width: 4px; }
.aival.ideal { fill: var(--fzj-blue); }
.aival.real { fill: #c77a1a; }
.aival.halo { fill: var(--fzj-branchred); }
.ptlbl { font-size: 14px; font-weight: 700; }
.ptlbl.ideal { fill: var(--fzj-blue); }
.ptlbl.real { fill: #c77a1a; }
.ptlbl.halo { fill: var(--fzj-branchred); }
.ptsub { font-size: 12px; fill: #555; }
.reveal { opacity: 0; transition: opacity 0.4s ease; }
.reveal.shown { opacity: 1; }
</style>
