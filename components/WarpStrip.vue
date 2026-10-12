<!--
  The 24 values in x that 8 threads need (left cube | own cube | right cube), each cell
  labelled with the thread that loads it. Built up step by step:
    step 0: owners, step 1: the 3 loads of thread 3, step 2: the 17 values thread 3 needs.
  Coordinates in cells (1 cell = 0.3 cm in the Beamer version).
-->
<script setup>
defineProps({ step: { type: Number, default: 2 } })
const cells = Array.from({ length: 24 }, (_, p) => p)
const isLoad = p => p % 8 === 3          // thread 3 loads positions 3, 11, 19
const needed = p => p >= 3 && p <= 19    // its 17-point stencil
</script>

<template>
  <div class="warp-strip">
    <svg viewBox="-0.5 -1.6 25 6.6" width="459">
      <!-- cube names -->
      <text v-for="(n, i) in ['left cube', 'own cube', 'right cube']" :key="n"
            :x="4 + 8 * i" y="-0.6" class="label" text-anchor="middle">{{ n }}</text>
      <!-- cells: shading appears with the steps -->
      <rect v-for="p in cells" :key="'f' + p" :x="p" y="0" width="1" height="1.4"
            class="fill" :class="{ needed: step >= 2 && needed(p), load: step >= 1 && isLoad(p) }" />
      <rect v-for="p in cells" :key="'g' + p" :x="p" y="0" width="1" height="1.4" class="grid" />
      <text v-for="p in cells" :key="'t' + p" :x="p + 0.5" y="0.75" class="owner"
            :class="{ white: step >= 1 && isLoad(p) }" text-anchor="middle" dominant-baseline="middle">{{ p % 8 }}</text>
      <!-- cube boundaries -->
      <line x1="8" y1="-0.3" x2="8" y2="1.7" class="boundary" />
      <line x1="16" y1="-0.3" x2="16" y2="1.7" class="boundary" />
      <!-- reach of thread 3 -->
      <g class="reveal" :class="{ shown: step >= 2 }">
        <line x1="3" y1="2.45" x2="20" y2="2.45" class="reach" />
        <line x1="3" y1="2.1" x2="3" y2="2.8" class="reach" />
        <line x1="20" y1="2.1" x2="20" y2="2.8" class="reach" />
        <text x="11.5" y="3.75" class="label" text-anchor="middle">thread 3 needs 17 values</text>
      </g>
    </svg>
    <div class="legend">
      <span class="reveal" :class="{ shown: step >= 1 }"><i class="sw load" />3 loaded by thread 3</span>
      <span class="reveal" :class="{ shown: step >= 2 }"><i class="sw needed" />14 by shuffle</span>
    </div>
    <div class="note">number in a cell = thread that loads it</div>
  </div>
</template>

<style scoped>
.warp-strip { display: flex; flex-direction: column; align-items: center; }
svg { overflow: visible; }
.fill { fill: #fff; transition: fill 0.4s ease; }
.fill.needed { fill: #fde1bd; }            /* fzjorange 40 % */
.fill.load { fill: var(--fzj-blue); }
.grid { fill: none; stroke: #b3b3b3; stroke-width: 0.04; }
.owner { font-size: 0.85px; fill: #000; transition: fill 0.4s ease; }
.owner.white { fill: #fff; }
.label { font-size: 0.85px; fill: #000; }
.boundary { stroke: #000; stroke-width: 0.08; }
.reach { stroke: var(--fzj-orange); stroke-width: 0.13; }
.reveal { opacity: 0; transition: opacity 0.4s ease; }
.reveal.shown { opacity: 1; }
.legend { display: flex; gap: 22px; font-size: 15px; margin-top: 4px; }
.sw { display: inline-block; width: 13px; height: 13px; margin-right: 6px; vertical-align: -1px; }
.sw.load { background: var(--fzj-blue); }
.sw.needed { background: #fde1bd; }
.note { font-size: 15px; margin-top: 4px; }
</style>
