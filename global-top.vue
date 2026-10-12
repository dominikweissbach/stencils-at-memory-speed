<!-- Footer of every slide, as in the Beamer theme. The title page only shows the left text and the logo. -->
<script setup>
const today = new Date().toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' })
</script>

<template>
  <footer class="fzj-footer">
    <span class="member">Member of the Helmholtz Association</span>
    <template v-if="$nav.currentLayout !== 'title'">
      <span class="date">{{ $slidev.configs.date ?? today }}</span>
      <span class="page">Slide {{ $nav.currentPage - 1 }}</span>
    </template>
    <img class="logo" src="/fzj.svg" alt="Forschungszentrum Jülich">
  </footer>
</template>

<style scoped>
.fzj-footer {
  position: absolute; left: 0; right: 0; bottom: 0; height: 60px;
  font-family: "TeX Gyre Heros", Helvetica, Arial, sans-serif;
  font-size: 9.5px; color: var(--fzj-blue);
}
.member { position: absolute; left: 29px; bottom: 27px; }
.date { position: absolute; left: 392px; bottom: 27px; transform: translateX(-50%); }
.page { position: absolute; left: 612px; bottom: 27px; transform: translateX(-50%); }
.logo { position: absolute; right: 27px; bottom: 16px; height: 44px; }
</style>
