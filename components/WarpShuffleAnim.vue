<!--
  Step-by-step warp-shuffle animation for one x-row (Laplace16th_x_warpshuffle), for people
  who have never seen shuffles. Positions are x-coordinates: -8..-1 left cube, 0..7 own cube,
  8..15 right cube; each cube's row is contiguous in memory, the cubes are not.
    step 0: memory blocks, 8 threads; thread i computes point i
    step 1: thread 3 loads position 3 of the left, own and right cube (L, C, R)
    step 2: all threads load: thread i loads position i of each cube
    step 3: thread 3's 17-value window, 3 of 17 already there
    step 4 … 10: shuffle distance d = 1 … 7 (one click each): thread 3 gets x-d and x+d
  Who holds value x: thread x mod 8, register L (x < 0), C (0 <= x < 8) or R (x >= 8).
-->
<script setup>
import { computed } from 'vue'

const props = defineProps({ step: { type: Number, default: 10 } })

const FOCUS = 3
const CELL = 30, BLOCK_GAP = 34, MEM_X0 = 102, MEM_Y = 30
const BOX_W = 84, BOX_H = 74, GAP = 18, BOX_X0 = 51, BOX_Y = 190
const SLOT = 24, SLOT_Y = BOX_Y + 36
const TRACK_Y = 336, TRACK_W = 26

const positions = Array.from({ length: 24 }, (_, p) => p - 8)          // -8 .. 15
const block = x => (x < 0 ? 0 : x < 8 ? 1 : 2)
const blockX = b => MEM_X0 + b * (8 * CELL + BLOCK_GAP)
const memX = x => blockX(block(x)) + (((x % 8) + 8) % 8) * CELL
const boxX = t => BOX_X0 + t * (BOX_W + GAP)
const slotX = (t, s) => boxX(t) + 6 + s * SLOT                          // s: 0 = L, 1 = C, 2 = R
const cube = x => ['left', 'own', 'right'][block(x)]
const holder = x => ({ t: ((x % 8) + 8) % 8, s: block(x) })

// chips: each memory value goes to exactly one thread's register
const chips = positions.map(x => ({ x, ...holder(x) }))
const loaded = c => props.step >= 2 || (props.step === 1 && c.t === FOCUS)
const chipPos = c => (loaded(c)
  ? { x: slotX(c.t, c.s), y: SLOT_Y }
  : { x: memX(c.x) + (CELL - SLOT) / 2, y: MEM_Y + (CELL - SLOT) / 2 })

// shuffle distance shown: 0 = none yet; click 4 -> d = 1, …, click 10 -> d = 7
const d = computed(() => Math.min(7, Math.max(0, props.step - 3)))

const needed = x => x >= FOCUS - 8 && x <= FOCUS + 8
const have = computed(() => {
  if (props.step < 3) return []
  const got = [FOCUS - 8, FOCUS, FOCUS + 8]
  for (let k = 1; k <= d.value; k++) got.push(FOCUS - k, FOCUS + k)
  return got
})
const track = Array.from({ length: 17 }, (_, k) => FOCUS - 8 + k)
const trackX0 = 330

// the two arrows into thread 3 for the current distance
function arrow(x, fromLeft) {
  const h = holder(x)
  const sx = slotX(h.t, h.s) + SLOT / 2
  const tx = boxX(FOCUS) + BOX_W / 2 + (fromLeft ? -10 : 10)
  const below = !fromLeft
  const y = below ? BOX_Y + BOX_H + 4 : BOX_Y - 4
  const lift = 22 + Math.abs(h.t - FOCUS) * 7
  const dy = below ? lift : -lift
  return {
    d: `M${sx},${y} C${sx},${y + dy} ${tx},${y + dy} ${tx},${y}`,
    lx: (sx + tx) / 2, ly: y + dy * 0.75 + (below ? 16 : -6), val: x, from: h,
  }
}
const arrows = computed(() => (d.value > 0
  ? [{ ...arrow(FOCUS - d.value, true), cls: 'left' }, { ...arrow(FOCUS + d.value, false), cls: 'right' }]
  : []))
const sending = computed(() => arrows.value.map(a => ({ ...a.from, cls: a.cls })))
const sendCls = (t, s) => sending.value.find(f => f.t === t && (s === undefined || f.s === s))?.cls

// the 3 values thread 3 loads: position 3 of each cube
const focusLoads = [FOCUS - 8, FOCUS, FOCUS + 8]
</script>

<template>
  <div class="ws">
    <svg width="900" height="380" viewBox="0 0 900 380">
      <defs>
        <marker id="ws-arr-b" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L8,4 L0,8 Z" fill="#023d6b" />
        </marker>
        <marker id="ws-arr-o" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L8,4 L0,8 Z" fill="#e08a1e" />
        </marker>
        <marker id="ws-arr-g" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto" markerUnits="userSpaceOnUse">
          <path d="M0,0 L7,3.5 L0,7 Z" fill="#666" />
        </marker>
      </defs>

      <!-- memory: three separate blocks, each row of 8 contiguous -->
      <text :x="MEM_X0 - 10" :y="MEM_Y + CELL / 2" class="side" text-anchor="end" dominant-baseline="middle">memory</text>
      <g v-for="(n, b) in ['left neighbouring cube', 'own cube', 'right neighbouring cube']" :key="n">
        <text :x="blockX(b) + 4 * CELL" :y="MEM_Y - 9" class="cubename" text-anchor="middle">{{ n }}</text>
        <line :x1="blockX(b) + 6" :y1="MEM_Y + CELL + 22" :x2="blockX(b) + 8 * CELL - 6" :y2="MEM_Y + CELL + 22"
              class="contig" marker-end="url(#ws-arr-g)" />
        <text v-if="b === 1" :x="blockX(b) + 4 * CELL" :y="MEM_Y + CELL + 38" class="contiglbl" text-anchor="middle">8 consecutive addresses</text>
      </g>
      <text :x="blockX(1) - BLOCK_GAP / 2" :y="MEM_Y + CELL / 2" class="dots" text-anchor="middle" dominant-baseline="middle">…</text>
      <text :x="blockX(2) - BLOCK_GAP / 2" :y="MEM_Y + CELL / 2" class="dots" text-anchor="middle" dominant-baseline="middle">…</text>
      <g v-for="x in positions" :key="'m' + x">
        <rect :x="memX(x)" :y="MEM_Y" :width="CELL" :height="CELL"
              :class="['mem', cube(x), { window: step >= 3 && needed(x), pick: step === 1 && focusLoads.includes(x) }]" />
        <text :x="memX(x) + CELL / 2" :y="MEM_Y + CELL + 11" class="pos"
              :class="{ pickpos: step === 1 && focusLoads.includes(x) }" text-anchor="middle">{{ ((x % 8) + 8) % 8 }}</text>
      </g>

      <text :x="MEM_X0 - 10" :y="MEM_Y + CELL + 11" class="poslbl" text-anchor="end">position</text>

      <!-- step 1: thread 3 loads the same position from all three cubes -->
      <g class="reveal" :class="{ shown: step === 1 }">
        <line v-for="(x, s) in focusLoads" :key="'l' + x" :x1="memX(x) + CELL / 2" :y1="MEM_Y + CELL + 14"
              :x2="slotX(FOCUS, s) + SLOT / 2" :y2="SLOT_Y - 2" class="loadline" />
      </g>

      <!-- step 0: thread i computes point i -->
      <g class="reveal" :class="{ shown: step === 0 }">
        <line v-for="t in 8" :key="'a' + t" :x1="memX(t - 1) + CELL / 2" :y1="MEM_Y + CELL + 50"
              :x2="boxX(t - 1) + BOX_W / 2" :y2="BOX_Y - 4" class="assign" />
      </g>

      <!-- threads with registers L, C, R -->
      <g v-for="t in 8" :key="'t' + t">
        <rect :x="boxX(t - 1)" :y="BOX_Y" :width="BOX_W" :height="BOX_H" rx="6"
              :class="['box', { focus: step >= 1 && t - 1 === FOCUS }, sendCls(t - 1) && 'send-' + sendCls(t - 1)]" />
        <text :x="boxX(t - 1) + BOX_W / 2" :y="BOX_Y + 20" class="tname" text-anchor="middle">thread {{ t - 1 }}</text>
        <g v-for="(r, s) in ['L', 'C', 'R']" :key="r">
          <rect :x="slotX(t - 1, s)" :y="SLOT_Y" :width="SLOT" :height="SLOT"
                class="slot" />
          <text :x="slotX(t - 1, s) + SLOT / 2" :y="SLOT_Y + SLOT + 12" text-anchor="middle"
                :class="['reg', sendCls(t - 1, s) && 'reg-' + sendCls(t - 1, s)]">{{ r }}</text>
        </g>
      </g>

      <!-- the values: start in memory, drop into the registers -->
      <g v-for="(c, n) in chips" :key="'c' + c.x" class="chip"
         :style="{ transform: `translate(${chipPos(c).x}px, ${chipPos(c).y}px)`, transitionDelay: `${n * 25}ms` }">
        <rect :width="SLOT" :height="SLOT" :class="['chiprect', cube(c.x), { faded: step >= 3 && !needed(c.x) }]" />
        <text :x="SLOT / 2" :y="SLOT / 2 + 1" class="chiptext" text-anchor="middle" dominant-baseline="middle">{{ c.x }}</text>
      </g>

      <!-- frame around the register each sender hands over -->
      <rect v-for="f in sending" :key="'f' + f.t + f.s" :x="slotX(f.t, f.s) - 2" :y="SLOT_Y - 2"
            :width="SLOT + 4" :height="SLOT + 4" :class="['sendframe', f.cls]" />

      <!-- shuffle: the two values thread 3 receives at distance d -->
      <g v-for="a in arrows" :key="a.cls + a.val" class="arr">
        <path :d="a.d" :class="['arrow', a.cls]" :marker-end="a.cls === 'left' ? 'url(#ws-arr-b)' : 'url(#ws-arr-o)'" />
        <text :x="a.lx" :y="a.ly" :class="['val', a.cls]" text-anchor="middle">{{ a.val }}</text>
      </g>
      <text v-if="d > 0" x="860" :y="BOX_Y - 30" class="dlabel" text-anchor="end">d = {{ d }}</text>

      <!-- what thread 3 has collected -->
      <g class="reveal" :class="{ shown: step >= 3 }">
        <text :x="trackX0 - 12" :y="TRACK_Y + TRACK_W / 2" class="side" text-anchor="end" dominant-baseline="middle">thread 3 has</text>
        <g v-for="(x, k) in track" :key="'k' + x">
          <rect :x="trackX0 + k * TRACK_W" :y="TRACK_Y" :width="TRACK_W" :height="TRACK_W"
                :class="['trk', { got: have.includes(x), own: [FOCUS - 8, FOCUS, FOCUS + 8].includes(x) }]" />
          <text :x="trackX0 + k * TRACK_W + TRACK_W / 2" :y="TRACK_Y + TRACK_W / 2 + 1" class="trktext"
                :class="{ got: have.includes(x), white: [FOCUS - 8, FOCUS, FOCUS + 8].includes(x) && have.includes(x) }"
                text-anchor="middle" dominant-baseline="middle">{{ x }}</text>
        </g>
        <text :x="trackX0 + 17 * TRACK_W + 12" :y="TRACK_Y + TRACK_W / 2" class="count" dominant-baseline="middle">{{ have.length }} / 17</text>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.ws { display: flex; flex-direction: column; align-items: center; }
svg { overflow: visible; font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif; }
.side { font-size: 15px; fill: #444; }
.cubename { font-size: 14px; fill: #444; }
.pos { font-size: 10.5px; fill: #666; }
.pos.pickpos { fill: #e08a1e; font-weight: 700; font-size: 12px; }
.poslbl { font-size: 11px; fill: #888; }
.mem.left.pick, .mem.own.pick, .mem.right.pick { fill: #fab45a; }
.loadline { stroke: #e08a1e; stroke-width: 2; stroke-dasharray: 5 3; }
.dots { font-size: 18px; fill: #888; }
.contig { stroke: #666; stroke-width: 1.3; }
.contiglbl { font-size: 12px; fill: #555; }
/* memory keeps its values after the loads (loads copy): light fill per cube */
.mem { stroke: #bbb; stroke-width: 1; transition: fill 0.4s ease; }
.mem.left, .mem.right { fill: #e1e8ed; }
.mem.own { fill: #c9d6e2; }
.mem.window { fill: #fde1bd; }
.assign { stroke: #888; stroke-width: 1.3; stroke-dasharray: 3 3; }
.box { fill: #f4f7fa; stroke: var(--fzj-blue); stroke-width: 1.5; transition: fill 0.4s ease, stroke-width 0.4s ease; }
.box.focus { fill: #fff3e3; stroke: #e08a1e; stroke-width: 3; }
.tname { font-size: 15px; font-weight: 700; fill: var(--fzj-blue); }
.slot { fill: #fff; stroke: #999; stroke-width: 1; stroke-dasharray: 3 2; }
.box.send-left { stroke: var(--fzj-blue); stroke-width: 3; fill: #e6eef5; }
.box.send-right { stroke: #e08a1e; stroke-width: 3; fill: #fff3e3; }
.sendframe { fill: none; stroke-width: 3.5; }
.sendframe.left { stroke: var(--fzj-blue); }
.sendframe.right { stroke: #e08a1e; }
.reg-left { fill: var(--fzj-blue); font-weight: 700; font-size: 13px; }
.reg-right { fill: #e08a1e; font-weight: 700; font-size: 13px; }
.reg { font-size: 11px; fill: #666; }
.chip { transition: transform 0.7s cubic-bezier(.4, 0, .2, 1); }
.chiprect { stroke: #fff; stroke-width: 1; transition: opacity 0.4s ease; }
.chiprect.left, .chiprect.right { fill: #7494ae; }
.chiprect.own { fill: var(--fzj-blue); }
.chiprect.faded { opacity: 0.35; }
.chiptext { font-size: 11.5px; font-weight: 700; fill: #fff; pointer-events: none; }
.arr { animation: fadein 0.35s ease; }
@keyframes fadein { from { opacity: 0; } to { opacity: 1; } }
.arrow { fill: none; stroke-width: 2.5; }
.arrow.left { stroke: var(--fzj-blue); }
.arrow.right { stroke: #e08a1e; }
.val { font-size: 14px; font-weight: 700; }
.val.left { fill: var(--fzj-blue); }
.val.right { fill: #e08a1e; }
.dlabel { font-size: 22px; font-weight: 700; fill: #444; }
.trk { fill: #fff; stroke: #ccc; stroke-width: 1; transition: fill 0.35s ease; }
.trk.got { fill: #fab45a; }
.trk.got.own { fill: var(--fzj-blue); }
.trktext { font-size: 11px; fill: #999; }
.trktext.got { fill: #000; font-weight: 700; }
.trktext.white { fill: #fff; }
.count { font-size: 16px; font-weight: 700; fill: #e08a1e; }
.reveal { opacity: 0; transition: opacity 0.4s ease; }
.reveal.shown { opacity: 1; }
</style>
