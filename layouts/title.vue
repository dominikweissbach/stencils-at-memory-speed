<!-- Title page as in the Beamer theme: image on top, blue band with title, subtitle and info line -->
<script setup>
import { computed } from 'vue'
import { useSlideContext } from '@slidev/client'
const { $frontmatter } = useSlideContext()
// runtime URL, not rewritten by the build: prefix the base path (GitHub Pages serves under /<repo>/)
const titleImage = computed(() => $frontmatter.image ?? `${import.meta.env.BASE_URL}title.png`)
const date = computed(() => $frontmatter.date
  ?? new Date().toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' }))
</script>

<template>
  <div class="slidev-layout title-page">
    <div class="image" :style="{ backgroundImage: `url(${titleImage})` }" />
    <div class="band">
      <div class="title">{{ $frontmatter.title }}</div>
      <div class="subtitle">{{ $frontmatter.subtitle }}</div>
      <div class="info">
        {{ date }}<span class="sep" />{{ $frontmatter.author }}<span class="sep" />{{ $frontmatter.institute }}
      </div>
    </div>
    <slot />
  </div>
</template>

<style scoped>
.title-page { position: relative; height: 100%; padding: 0; }
.image {
  position: absolute; left: 28px; right: 28px; top: 28px; height: 248px;
  background-size: cover; background-position: center;
}
.band {
  position: absolute; left: 0; right: 0; top: 276px; height: 188px;
  background: var(--fzj-blue); color: #fff;
  padding: 30px 28px 0 28px;
}
.title { font-size: 26px; font-weight: 700; text-transform: uppercase; line-height: 1.15; }
.subtitle { font-size: 26px; font-weight: 700; color: var(--fzj-lightblue); line-height: 1.15; margin-top: 2px; }
.info { font-size: 14px; margin-top: 14px; }
.sep { display: inline-block; width: 1px; height: 12px; background: #fff; margin: 0 9px; vertical-align: -1px; }
</style>
