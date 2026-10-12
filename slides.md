---
theme: default
layout: title
title: Stencils at Memory Speed
subtitle: Accelerating AngstromCube's Poisson solver on GPUs
author: Dominik Weißbach
institute: Jülich Supercomputing Centre
aspectRatio: 16/9
canvasWidth: 980
colorSchema: light
transition: fade
fonts:
  provider: none
  sans: TeX Gyre Heros
drawings:
  persist: false
mdc: true
exportFilename: stencils-at-memory-speed
download: https://dominikweissbach.github.io/stencils-at-memory-speed/stencils-at-memory-speed.pdf
---

---

# GPUs went from about 4× to about 10× the memory bandwidth of CPUs

<div class="center">
  <img src="/figures/memory_evolution.svg" style="width: 660px; margin: 0 auto;">
</div>

<div class="tiny center" style="margin-top: 10px;">
Peak bandwidth per server GPU / CPU socket. Data: K. Rupp (2007–2018, CC BY 4.0, modified), vendor specifications (2014–2025). See sources.
</div>

---

# Outline

<div class="roadmap">
  <div class="step"><div class="n">1</div><div class="t">The problem</div><ul class="k"><li>Poisson equation</li><li>finite differences</li></ul></div>
  <div class="arrow">→</div>
  <div class="step"><div class="n">2</div><div class="t">Why memory decides</div><ul class="k"><li>grid layout</li><li>flops per byte</li></ul></div>
  <div class="arrow">→</div>
  <div class="step"><div class="n">3</div><div class="t">Reusing data</div><ul class="k"><li>naive kernel</li><li>shared memory</li><li>warp shuffle</li><li>rotating registers</li><li>cube order</li></ul></div>
  <div class="arrow">→</div>
  <div class="step"><div class="n">4</div><div class="t">Results &amp; next steps</div><ul class="k"><li>kernel speed-ups</li><li>GPU vs. CPU</li><li>outlook</li></ul></div>
</div>

<style>
.roadmap { display: flex; align-items: stretch; justify-content: center; gap: 10px; }
.roadmap .step { width: 190px; border: 2px solid var(--fzj-blue); border-radius: 10px; padding: 14px 12px 16px; background: #f4f7fa; }
.roadmap .n { width: 34px; height: 34px; border-radius: 50%; background: var(--fzj-blue); color: #fff; font-weight: 700; font-size: 19px; display: flex; align-items: center; justify-content: center; margin-bottom: 10px; }
.roadmap .t { font-size: 20px; font-weight: 700; color: var(--fzj-blue); line-height: 1.2; min-height: 48px; }
.roadmap .k { font-size: 16px; color: #444; line-height: 1.35; margin: 6px 0 0; }
.roadmap .k li { margin: 3px 0; padding-left: 0.95em; }
.roadmap .k li::before { top: 0.5em; width: 0.32em; height: 0.32em; }
.roadmap .arrow { align-self: center; font-size: 30px; color: var(--fzj-blue); }
</style>

---

# The Poisson equation turns sources and sinks into a smooth field

<div class="cols" style="grid-template-columns: 40fr 58fr;">
<div>

$$
-\left(\frac{\partial^2 T}{\partial x^2} + \frac{\partial^2 T}{\partial y^2} + \frac{\partial^2 T}{\partial z^2}\right) = q
$$

- $q$: heaters and coolers (**known**)
- $T$: temperature (**unknown**)

<div class="small" style="margin-top: 28px;">
In AngstromCube:<br>charge density → electrostatic potential
</div>

</div>
<div class="center">
  <img src="/figures/poisson_heat.png" style="height: 290px; margin: 0 auto;">
</div>
</div>

---
clicks: 4
---

# Finite differences approximate a derivative from neighbouring points — more points, higher accuracy

<FdExplorer :step="$clicks" />

---

# The grid is stored as 8×8×8 cubes, so only <i>x</i>-neighbours are adjacent in memory

<div class="cols" style="grid-template-columns: 58fr 40fr;">
<div class="center">
  <img src="/figures/cube.svg" style="width: 353px; margin: 0 auto;">
  <img src="/figures/memstrip.svg" style="width: 447px; margin: 14px auto 0;">
</div>
<div class="center" v-click>
  <img src="/figures/stencil.svg" style="width: 288px; margin: 0 auto;">
  <div class="figcap">17-point stencil reaches<br>into the neighbouring cubes</div>
</div>
</div>

---
clicks: 3
---

# The stencil does too little arithmetic per byte, so memory bandwidth limits its speed

<div class="cols" style="grid-template-columns: 38fr 60fr;">
<div>
  <div class="ratio">
    <div class="who">stencil, per grid point</div>
    <div class="eq"><span class="frac"><span>78 flops</span><span>16 bytes</span></span> ≈ <b class="ideal">4.9</b> <span class="unit">flops/byte</span></div>
    <div class="why">3 directions × (9 mult. + 16 add. + 1 scaling)<br>read 1 value + write 1 result, 8 bytes each</div>
  </div>
  <div class="ratio" v-click="1">
    <div class="who">GH200 (Hopper GPU)</div>
    <div class="eq"><span class="frac"><span>34 TFLOP/s</span><span>4 TB/s</span></span> ≈ <b class="gpu">8.5</b> <span class="unit">flops/byte</span></div>
    <div class="why">flops the GPU can do for every byte that arrives</div>
  </div>
</div>
<div class="center">
  <Roofline :step="Math.max(0, $clicks - 1)" />
</div>
</div>

<style>
.ratio { margin: 6px 0 22px; }
.ratio .who { font-size: 17px; color: #555; }
.ratio .eq { font-size: 22px; display: flex; align-items: center; gap: 10px; margin: 4px 0; }
.ratio .frac { display: inline-flex; flex-direction: column; align-items: center; line-height: 1.2; }
.ratio .frac span:first-child { border-bottom: 1.5px solid #000; padding: 0 4px 2px; }
.ratio .frac span:last-child { padding: 2px 4px 0; }
.ratio b.ideal { color: var(--fzj-blue); font-size: 26px; }
.ratio b.gpu { color: var(--fzj-branchred); font-size: 26px; }
.ratio .unit { font-size: 17px; }
.ratio .why { font-size: 14px; color: #555; line-height: 1.3; }
</style>

---

# A naive kernel loads every neighbour from memory, far from peak bandwidth

<div class="cols" style="grid-template-columns: 44fr 54fr;">
<div class="center">
  <img src="/figures/naive.svg" style="width: 363px; margin: 0 auto;">
  <div class="figcap" style="margin-top: 20px;">each value is loaded by 17 threads per direction</div>
</div>
<div class="center" v-click>
  <img src="/figures/bandwidth_stage1.svg" style="width: 499px; margin: 0 auto;">
</div>
</div>

---
clicks: 4
---

# The closer to the threads a value is kept, the cheaper it is to reuse

<div class="center">
  <MemoryPyramid :step="$clicks" />
</div>

<div class="tiny center" style="margin-top: 36px;">
[1] Luo et al., Benchmarking and Dissecting the Nvidia Hopper GPU Architecture, arXiv:2402.13499, Table V (H800, per clock), × 132 SMs × 1.98 GHz · [2] NVIDIA GH200 datasheet
</div>

---
clicks: 4
---

# Shared memory lets the threads of a block load each value once and share it

<SharedMemoryAnim :step="$clicks" />

---

# Shared memory gains little, because its size limits how many threads can run

<div class="cols" style="grid-template-columns: 44fr 54fr;">
<div>
  <table class="smtab">
    <thead><tr><th></th><th class="naive">naive</th><th class="sm">shared memory</th></tr></thead>
    <tbody>
      <tr><td><i>y</i></td><td class="naive">54 %</td><td class="sm">68 %</td></tr>
      <tr><td><i>z</i></td><td class="naive">41 %</td><td class="sm">42 %</td></tr>
    </tbody>
  </table>
  <div class="tiny" style="margin-top: 4px;">largest grids (≥ 512³ points), % of the copy-kernel bandwidth</div>
  <div class="occ" v-click>
    <div>13.8 KB shared memory per block</div>
    <div>→ only 15 blocks fit on an SM</div>
    <div class="big">→ 47 % occupancy</div>
  </div>
</div>
<div class="center">
  <img src="/figures/bandwidth_shared.svg" style="width: 499px; margin: 0 auto;">
</div>
</div>

<style>
.smtab { border-collapse: collapse; margin-top: 6px; }
.smtab th { font-size: 17px; font-weight: 400; color: #444; padding: 0 18px 2px 0; text-align: left; border: none; background: none; }
.smtab td { font-size: 40px; font-weight: 700; padding: 0 18px 0 0; border: none; background: none; }
.smtab td:first-child { font-size: 24px; color: #444; padding-right: 26px; }
.smtab tr { background: none !important; }
.smtab .naive { color: #8c8c8c; }
.smtab .sm { color: #4f86b8; }
.occ { margin-top: 26px; font-size: 19px; line-height: 1.45; border-left: 4px solid #4f86b8; padding-left: 12px; }
.occ .big { font-size: 26px; font-weight: 700; color: var(--fzj-blue); }
</style>

---
clicks: 5
---

# Walking along <i>y</i> or <i>z</i> keeps the loads coalesced, walking along <i>x</i> does not

<DirectionTricksAnim :step="$clicks" />

---
clicks: 10
---

# Warp shuffles let threads pass values to each other, so each thread loads 3 values instead of 17

<WarpShuffleAnim :step="$clicks" />

---

# In <i>x</i>, warp shuffles raise the bandwidth from 55 % to 87 % of memory speed

<div class="cols" style="grid-template-columns: 44fr 54fr;">
<div class="result">
  <div class="row"><span class="lbl">naive</span><span class="num naive">55 %</span></div>
  <div class="row"><span class="lbl">warp shuffle</span><span class="num ws">87 %</span></div>
  <div class="gain" v-click>≈ 1.6× faster</div>
  <div class="tiny" style="margin-top: 26px;">x direction, largest grids (≥ 512³ points), % of the copy-kernel bandwidth</div>
</div>
<div class="center">
  <img src="/figures/bandwidth_stage2.svg" style="width: 499px; margin: 0 auto;">
</div>
</div>

<style>
.result .row { display: flex; align-items: baseline; gap: 16px; margin: 4px 0; }
.result .lbl { width: 140px; font-size: 20px; color: #444; }
.result .num { font-size: 44px; font-weight: 700; }
.result .num.naive { color: #8c8c8c; }
.result .num.ws { color: var(--fzj-blue); }
.result .gain { font-size: 26px; font-weight: 700; margin-top: 18px; }
.result .why { font-size: 18px; color: #444; margin-top: 4px; }
</style>

---
clicks: 10
---

# Rotating registers let one thread reuse values as it walks along <i>y</i> or <i>z</i>

<RotatingRegistersAnim :step="$clicks" />

---

# Rotating registers reach 80&nbsp;% in <i>y</i>, but in <i>z</i> they break down on large grids

<div class="cols" style="grid-template-columns: 44fr 54fr;">
<div>
  <table class="rrtab">
    <thead><tr><th></th><th class="naive">naive</th><th class="sm">shared<br>memory</th><th class="rr">rotating<br>registers</th></tr></thead>
    <tbody>
      <tr><td><i>y</i></td><td class="naive">54 %</td><td class="sm">68 %</td><td class="rr">80 %</td></tr>
      <tr><td><i>z</i></td><td class="naive">41 %</td><td class="sm">42 %</td><td class="rr">65 %</td></tr>
    </tbody>
  </table>
  <div class="tiny" style="margin-top: 4px;">largest grids (≥ 512³ points), % of the copy-kernel bandwidth</div>
  <div class="gain" v-click>in <i>z</i>: 89 % up to ≈ 350³ points,<br>then down to 65 % — why?</div>
</div>
<div class="center">
  <img src="/figures/bandwidth_stage3.svg" style="width: 499px; margin: 0 auto;">
</div>
</div>

<style>
.rrtab { border-collapse: collapse; margin-top: 6px; }
.rrtab th { font-size: 16px; font-weight: 400; line-height: 1.15; color: #444; padding: 0 14px 4px 0; text-align: left; vertical-align: bottom; border: none; background: none; }
.rrtab td { font-size: 34px; font-weight: 700; padding: 0 14px 0 0; border: none; background: none; }
.rrtab td:first-child { font-size: 24px; color: #444; padding-right: 20px; }
.rrtab tr { background: none !important; }
.rrtab .naive { color: #8c8c8c; }
.rrtab .sm { color: #4f86b8; }
.rrtab .rr { color: #e08a1e; }
.gain { font-size: 26px; font-weight: 700; margin-top: 28px; }
</style>

---
clicks: 3
---

# The order in which the blocks visit the cubes decides whether a neighbouring cube is still in L2

<CubeOrderAnim :step="$clicks" />

---

# Visiting the cubes in <i>z</i> order lifts rotating registers in <i>z</i> from 65&nbsp;% to 92&nbsp;%

<div class="cols" style="grid-template-columns: 44fr 54fr;">
<div>
  <table class="ordtab">
    <thead><tr><th>rotating registers</th><th>x-major</th><th><i>z</i> order</th></tr></thead>
    <tbody>
      <tr><td><i>z</i></td><td class="xm">65 %</td><td class="rr">92 %</td></tr>
    </tbody>
  </table>
  <div class="tiny" style="margin-top: 4px;">largest grids (≥ 512³ points), % of the copy-kernel bandwidth</div>
  <div class="l2">
    <div class="l2h">L2 hit rate, <i>z</i> pass at 640³ points</div>
    <div class="l2row"><span class="xm">20 %</span><span class="arr">→</span><span class="rr">49 %</span></div>
    <div class="tiny">x-major → <i>z</i> order · Nsight Compute; 320³ x-major: 49 %</div>
  </div>
</div>
<div class="center">
  <img src="/figures/bandwidth_order.svg" style="width: 499px; margin: 0 auto;">
</div>
</div>

<style>
.ordtab { border-collapse: collapse; margin-top: 6px; }
.ordtab th { font-size: 17px; font-weight: 400; color: #444; padding: 0 18px 4px 0; text-align: left; border: none; background: none; }
.ordtab td { font-size: 40px; font-weight: 700; padding: 0 18px 0 0; border: none; background: none; }
.ordtab td:first-child { font-size: 24px; color: #444; padding-right: 26px; }
.ordtab tr { background: none !important; }
.ordtab .xm { color: #c9a57a; }
.ordtab .rr { color: #e08a1e; }
.l2 { margin-top: 26px; border-left: 4px solid #e08a1e; padding-left: 12px; }
.l2h { font-size: 18px; color: #444; }
.l2row { font-size: 34px; font-weight: 700; }
.l2row .xm { color: #c9a57a; }
.l2row .rr { color: #e08a1e; }
.l2row .arr { color: #444; margin: 0 12px; font-weight: 400; }
</style>

---

# Outlook

<div class="cols" style="grid-template-columns: 52fr 46fr;">
<div class="center">
  <img src="/figures/cpu_vs_gpu.svg" style="width: 470px; margin: 0 auto;">
</div>
<div class="small">

**Next steps**

- reduce the fixed cost per iteration (launches, synchronisations, small transfers): keep reductions on the GPU, fuse kernels, CUDA graphs → remove the small-grid CPU regime
- overlap MPI communication with computation
- until then: choose CPU or GPU by grid size

</div>
</div>

---

# Summary

<div class="recap">
  <div class="panel">
    <div class="ph">The stencil</div>
    <div class="pfig"><img src="/figures/stencil.svg" style="height: 150px;"></div>
    <div class="pt">17 points per direction, grid in 8×8×8 cubes → memory-bound</div>
  </div>
  <div class="panel">
    <div class="ph">Warp shuffle (<i>x</i>)</div>
    <div class="pfig"><div class="minibox"><div class="mini"><WarpShuffleAnim :step="10" /></div></div></div>
    <div class="pt">3 loads per point, rest passed between threads: <b>55 % → 87 %</b> of memory speed</div>
  </div>
  <div class="panel">
    <div class="ph">Rotating registers (<i>y</i>, <i>z</i>)</div>
    <div class="pfig"><div class="minibox"><div class="mini"><RotatingRegistersAnim :step="10" /></div></div></div>
    <div class="pt">1 new load per point: <b><i>y</i> 54 → 80 %</b>; in <i>z</i> with cubes in <i>z</i> order: <b>41 → 92 %</b></div>
  </div>
  <div class="panel">
    <div class="ph">GPU vs. CPU</div>
    <div class="pfig"><img src="/figures/cpu_vs_gpu.svg" style="height: 150px;"></div>
    <div class="pt">GPU ≈ 11× faster on large grids; CPU only wins on tiny grids (fixed per-iteration overhead)</div>
  </div>
</div>

<style>
.recap { display: grid; grid-template-columns: 1fr 1fr; grid-template-rows: 1fr 1fr; gap: 10px 14px; height: 420px; }
.recap .panel { border: 1.5px solid #c9d6e2; border-radius: 8px; padding: 6px 10px 6px; display: flex; flex-direction: column; overflow: hidden; }
.recap .ph { font-size: 16px; font-weight: 700; color: var(--fzj-blue); }
.recap .pfig { flex: 1; display: flex; align-items: center; justify-content: center; overflow: hidden; }
.recap .pt { font-size: 13px; color: #333; line-height: 1.25; }
/* scaled-down animations in their final state */
.recap .minibox { width: 306px; height: 136px; }  /* = 900 × 400 px scaled by 0.34 */
.recap .mini { width: 900px; height: 400px; transform: scale(0.34); transform-origin: top left; }
</style>

---

# Sources

<div class="small">

- K. Rupp, *CPU, GPU and MIC Hardware Characteristics over Time*,
  <https://github.com/karlrupp/cpu-gpu-mic-comparison>, licensed under CC BY 4.0
  (<https://creativecommons.org/licenses/by/4.0/>).
  Changes: AMD GPU data (consumer cards) and the Xeon Platinum 9282 entry removed;
  newer server CPUs and GPUs added from vendor specifications.
- Vendor specifications (NVIDIA, AMD, Intel product pages, datasheets and press releases);
  one source URL per value in `figures/data/memory_evolution/extra_server_parts.csv`.
- NVIDIA GH200 Grace Hopper Superchip datasheet.
- W. Luo et al., *Benchmarking and Dissecting the Nvidia Hopper GPU Architecture*, arXiv:2402.13499.

</div>

---

# Computing 8 points per thread is faster despite lower occupancy

<div class="cols" style="grid-template-columns: 64fr 34fr;">
<div class="center">
  <img src="/data/ThreadCoarsening.png" style="width: 100%;">
  <div class="legend-ncu">
    <span><i style="background:#2196dc"></i>1 point per thread</span>
    <span><i style="background:#50af4b"></i>8 points per thread</span>
  </div>
  <div class="tiny">Nsight Compute: executed instruction mix, <i>x</i> warp-shuffle kernel</div>
</div>
<div>

1 point per thread:

- +30 % runtime
- +40 % instructions
- higher occupancy

<div style="margin-top: 18px;">Same shuffles, but the setup work is repeated for every point.</div>

</div>
</div>

<style>
.legend-ncu { display: flex; justify-content: center; gap: 28px; font-size: 17px; margin: 8px 0 4px; }
.legend-ncu i { display: inline-block; width: 16px; height: 10px; margin-right: 6px; }
</style>

---

# Rotating registers in <i>x</i> would scatter every warp load over 4× as many memory sectors

<div class="center">
  <CoalescingBackup />
</div>
