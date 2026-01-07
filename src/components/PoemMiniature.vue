<script setup lang="ts">
import type { Poem } from '@/core/domain/entities/Poem'

defineProps<{
  poem: Poem
}>()

defineEmits<{
  (e: 'select', poem: Poem): void
}>()
</script>

<template>
  <div class="poem-miniature" @click="$emit('select', poem)">
    <div class="miniature-content">
      <div v-if="poem.symbol" class="poem-symbol">{{ poem.symbol }}</div>
      <h3 class="poem-title">{{ poem.title }}</h3>
      <div class="poem-preview">
        <p v-for="(line, i) in poem.content.slice(0, 1)" :key="i">{{ line }}</p>
      </div>
    </div>
    <div class="miniature-border"></div>
  </div>
</template>

<style lang="scss" scoped>
.poem-miniature {
  flex: 0 0 clamp(260px, 80vw, 300px);
  height: clamp(350px, 60vh, 400px);
  margin: 0 1rem;
  background: rgba(255, 255, 255, 0.03);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(0, 0, 0, 0.05);
  position: relative;
  cursor: pointer;
  transition: all 0.6s cubic-bezier(0.23, 1, 0.32, 1);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 1.5rem;
  text-align: center;

  @media (min-width: 768px) {
    margin: 0 2rem;
    padding: 2rem;
  }

  &:hover {
    background: rgba(255, 255, 255, 0.08);
    transform: translateY(-10px) scale(1.02);
    border-color: rgba(0, 0, 0, 0.1);
    
    .miniature-border {
      opacity: 1;
      transform: scale(1);
    }
    
    .poem-title {
      letter-spacing: 0.2em;
    }

    .poem-symbol {
      transform: scale(1.2) rotate(10deg);
      opacity: 1;
    }
  }

  .miniature-content {
    z-index: 2;
  }

  .poem-symbol {
    font-size: clamp(2rem, 8vw, 2.5rem);
    margin-bottom: 0.8rem;
    color: #1a1a1a;
    opacity: 0.5;
    transition: all 0.6s ease;
  }

  .poem-title {
    font-family: 'Cormorant Garamond', serif;
    font-size: clamp(1.3rem, 6vw, 1.6rem);
    font-weight: 300;
    margin-bottom: 1.2rem;
    color: #1a1a1a;
    letter-spacing: 0.1em;
    transition: all 0.6s ease;
  }

  .poem-preview {
    font-family: 'Cormorant Garamond', serif;
    font-size: 0.95rem;
    font-style: italic;
    color: #444;
    opacity: 0.7;
    line-height: 1.6;
    display: -webkit-box;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  .miniature-border {
    position: absolute;
    inset: 10px;
    border: 1px solid rgba(0, 0, 0, 0.1);
    opacity: 0;
    transform: scale(1.05);
    transition: all 0.6s ease;
    pointer-events: none;
  }
}
</style>
