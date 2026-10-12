<!-- Content slide: title top left, content vertically centred in the remaining space (like Beamer frames) -->
<template>
  <div class="slidev-layout fzj">
    <slot />
  </div>
</template>

<style scoped>
.fzj {
  height: 100%;
  padding: 18px 29px 64px 29px;
  display: flex;
  flex-direction: column;
}
.fzj :deep(> h1 + *) { margin-top: auto; }
.fzj :deep(> *:last-child) { margin-bottom: auto; }
</style>
