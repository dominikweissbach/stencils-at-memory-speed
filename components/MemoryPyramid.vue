<!--
  GPU memory hierarchy as a pyramid with the throughput of each level (sources on the slide),
  and where each technique keeps reused values. Techniques appear with the steps:
    step 1: naive, step 2: shared memory, step 3: registers (warp shuffle, rotating registers),
    step 4: L2 (cube ordering, the last part of the talk).
  Coordinates in cm, as in the Beamer TikZ version (y up, flipped below).
-->
<script setup>
defineProps({ step: { type: Number, default: 4 } })
const Y = y => 4.2 - y   // TikZ y (up) -> SVG y (down)
// half-widths at the bottom (b) and top (t) of each level: the sides keep one slope (0.6 per level)
const levels = [
  { y: 3, b: 1.6, t: 1.0, color: '#023d6b' },
  { y: 2, b: 2.2, t: 1.6, color: '#7494ae' },
  { y: 1, b: 2.8, t: 2.2, color: '#b3c5d3' },
  { y: 0, b: 3.4, t: 2.8, color: '#e1e8ed' },
]
const poly = l => `${-l.b},${Y(l.y)} ${l.b},${Y(l.y)} ${l.t},${Y(l.y + 1)} ${-l.t},${Y(l.y + 1)}`
// throughput of the whole GPU (GH200's H100: 132 SMs, 1.98 GHz); [n] = source on the slide
// (registers: no number, see the notes)
const latency = [
  { x: -2.4, y: 2.5, t: '≈ 33 TB/s', cite: '[1]' },
  { x: -3.0, y: 1.5, t: '≈ 9 TB/s', cite: '[1]' },
  { x: -3.6, y: 0.5, t: '4 TB/s', cite: '[2]' },
]
</script>

<template>
  <svg viewBox="-7.0 -0.1 16.4 4.5" width="935">
    <defs>
      <marker id="arr-blue" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto" markerUnits="strokeWidth">
        <path d="M0,0 L6,3 L0,6 Z" fill="#023d6b" />
      </marker>
      <marker id="arr-gray" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto" markerUnits="strokeWidth">
        <path d="M0,0 L6,3 L0,6 Z" fill="#888" />
      </marker>
      <marker id="arr-black" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto" markerUnits="strokeWidth">
        <path d="M0,0 L6,3 L0,6 Z" fill="#000" />
      </marker>
    </defs>

    <!-- pyramid -->
    <polygon v-for="l in levels" :key="l.y" :points="poly(l)" :fill="l.color" />
    <line x1="0" :y1="Y(2)" x2="0" :y2="Y(3)" stroke="#fff" stroke-width="0.05" />
    <text x="0" :y="Y(3.5)" class="lvl white">registers</text>
    <text x="-0.95" :y="Y(2.5)" class="lvl white">L1</text>
    <text x="0.95" :y="Y(2.5)" class="lvl white">shared</text>
    <text x="0" :y="Y(1.5)" class="lvl">L2 cache</text>
    <text x="0" :y="Y(0.5)" class="lvl">global memory</text>

    <!-- latencies -->
    <text v-for="l in latency" :key="l.y" :x="l.x" :y="Y(l.y)" class="lat">
      {{ l.t }}<tspan class="cite" dy="-0.12"> {{ l.cite }}</tspan>
    </text>

    <!-- faster, smaller -->
    <line x1="-6.6" :y1="Y(0.1)" x2="-6.6" :y2="Y(3.9)" stroke="#000" stroke-width="0.03" marker-end="url(#arr-black)" />
    <text :transform="`translate(-6.75 ${Y(2)}) rotate(-90)`" class="side">faster, smaller</text>

    <!-- techniques -->
    <g class="reveal" :class="{ shown: step >= 1 }">
      <line x1="3.9" :y1="Y(0.5)" x2="3.25" :y2="Y(0.5)" stroke="#888" stroke-width="0.035" marker-end="url(#arr-gray)" />
      <text x="4.0" :y="Y(0.5)" class="tech gray">naive: reload every time</text>
    </g>
    <g class="reveal" :class="{ shown: step >= 2 }">
      <line x1="3.9" :y1="Y(2.5)" x2="2.05" :y2="Y(2.5)" stroke="#888" stroke-width="0.035" marker-end="url(#arr-gray)" />
      <text x="4.0" :y="Y(2.5)" class="tech gray">shared memory</text>
    </g>
    <g class="reveal" :class="{ shown: step >= 3 }">
      <line x1="3.9" :y1="Y(3.5)" x2="1.45" :y2="Y(3.5)" stroke="#023d6b" stroke-width="0.035" marker-end="url(#arr-blue)" />
      <text x="4.0" :y="Y(3.5) - 0.2" class="tech blue">rotating registers (<tspan font-style="italic">y</tspan>, <tspan font-style="italic">z</tspan>)</text>
      <text x="4.0" :y="Y(3.5) + 0.25" class="tech blue">warp shuffle (<tspan font-style="italic">x</tspan>)</text>
    </g>
    <g class="reveal" :class="{ shown: step >= 4 }">
      <line x1="3.9" :y1="Y(1.5)" x2="2.65" :y2="Y(1.5)" stroke="#023d6b" stroke-width="0.035" marker-end="url(#arr-blue)" />
      <text x="4.0" :y="Y(1.5)" class="tech blue">cube ordering</text>
    </g>
  </svg>
</template>

<style scoped>
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.lvl { font-size: 0.35px; font-weight: 700; text-anchor: middle; dominant-baseline: middle; fill: #000; }
.lvl.white { fill: #fff; }
.lat { font-size: 0.316px; fill: #888; text-anchor: end; dominant-baseline: middle; }
.cite { font-size: 0.22px; }
.side { font-size: 0.316px; text-anchor: middle; }
.tech { font-size: 0.35px; dominant-baseline: middle; }
.tech.gray { fill: #888; }
.tech.blue { fill: #023d6b; font-weight: 700; }
.reveal { opacity: 0; transition: opacity 0.4s ease; }
.reveal.shown { opacity: 1; }
</style>
