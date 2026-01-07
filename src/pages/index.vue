<script setup lang="ts">
import { useAppStore } from '@/stores/appState'
import { onMounted, ref, watch, nextTick, onUnmounted, computed } from 'vue'
import { useMouse } from '@vueuse/core'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import PoemMiniature from '@/components/PoemMiniature.vue'
import type { Poem } from '@/core/domain/entities/Poem'

const appStore = useAppStore()
const { x, y } = useMouse({ type: 'client' })
const haikuStage = ref<HTMLElement | null>(null)
const introStarted = ref(false)
const isTransporting = ref(false)

// Transport Logic
const handlePoemSelect = (poem: Poem) => {
  isTransporting.value = true
  appStore.setCurrentPoem(poem)
  setTimeout(() => { isTransporting.value = false }, 1500)
}

const scrollToCollection = () => {
  if (!process.client) return
  window.scrollTo({ top: 20100, behavior: 'instant' as any })
}

if (process.client) {
  gsap.registerPlugin(ScrollTrigger)
}

const showScrollPrompt = computed(() => appStore.isReady && appStore.introStep === 0 && introStarted.value)
const showGallerySection = computed(() => appStore.introStep >= 8)

const wrapChars = (text: string) => {
  return text.split('').map(char => {
    if (char === ' ') return `<span class="char space">&nbsp;</span>`
    return `<span class="char">${char}</span>`
  }).join('')
}

onMounted(async () => {
  if (!process.client) return
  await nextTick()

  watch(() => appStore.isReady, (ready) => {
    if (ready && !introStarted.value) {
      introStarted.value = true
      setTimeout(() => setupPinnedTimeline(), 200)
    }
  }, { immediate: true })
})

function setupPinnedTimeline() {
  const stage = haikuStage.value
  if (!stage) return

  ScrollTrigger.getAll().forEach(st => st.kill())
  
  const lines = stage.querySelectorAll('.poem-line')
  gsap.set(lines, { autoAlpha: 0, y: 30, display: 'none', visibility: 'hidden' })

    const mainTimeline = gsap.timeline({
        scrollTrigger: {
          trigger: stage,
          start: 'top top',
          end: '+=20000', // Increased scroll height for better control
          pin: true,
          scrub: 1,
    snap: {
              // Snapping to the middle of the "static" plateaus for each line
              snapTo: [0, 0.085, 0.185, 0.285, 0.385, 0.485, 0.605, 0.745, 0.885, 0.955, 1],
              duration: { min: 0.2, max: 0.4 },
            delay: 0,
            ease: 'power1.inOut'
          },
          onUpdate: (self) => {
              const p = self.progress
              if (p < 0.05) appStore.setIntroStep(0)
              else if (p < 0.15) appStore.setIntroStep(1)
              else if (p < 0.25) appStore.setIntroStep(2)
              else if (p < 0.40) appStore.setIntroStep(3) // Tail step aligned with line-3
              else if (p < 0.50) appStore.setIntroStep(4)
              else if (p < 0.60) appStore.setIntroStep(5)
              else if (p < 0.75) appStore.setIntroStep(6) 
              else if (p < 0.85) appStore.setIntroStep(7)
              else if (p < 0.92) {
                appStore.setIntroStep(7)
                appStore.setPlaygroundMode(false)
              }
              else {
                appStore.setIntroStep(8)
                appStore.setPlaygroundMode(true)
              }
          }
        }
    })

    gsap.fromTo('.scroll-prompt', { autoAlpha: 0 }, { autoAlpha: 1, duration: 2, delay: 1.5 })

      const animateIn = (selector: string, startTime: number) => {
        const el = stage.querySelector(selector)
        if (!el) return
        const chars = el.querySelectorAll('.char')
        if (chars.length === 0) return
        
        mainTimeline
          .set(el, { display: 'flex', visibility: 'visible' }, startTime)
          .to(el, { autoAlpha: 1, duration: 0.005 }, startTime)
          .fromTo(chars, {
            opacity: 0,
            filter: 'blur(30px)',
            scale: 1.4,
            y: 40
          }, {
            opacity: 1,
            filter: 'blur(0px)',
            scale: 1,
            y: 0,
            duration: 0.03,
            stagger: 0.0015,
            ease: 'expo.out'
          }, startTime + 0.001)
          // THE SCROLL HALT / BUMP: A static plateau where the line remains fully visible
          // This creates the "bump" sensation as the scroll progress "waits"
          .to(el, {
            scale: 1.05,
            duration: 0.015,
            ease: 'power2.out'
          }, startTime + 0.035)
          .to(el, {
            scale: 1,
            duration: 0.015,
            ease: 'power2.in'
          }, startTime + 0.05)
          // Hold the state until animateOut starts (usually startTime + 0.08)
      }

      const animateOut = (selector: string, startTime: number) => {
        const el = stage.querySelector(selector)
        if (!el) return
        const chars = el.querySelectorAll('.char')
        
        if (chars.length > 0) {
          mainTimeline.to(chars, {
            opacity: 0, 
            filter: 'blur(20px)',
            y: -40,
            scale: 0.9,
            duration: 0.02,
            stagger: 0.001,
            ease: 'power2.in'
          }, startTime)
        }
        
        mainTimeline
          .to(el, { autoAlpha: 0, duration: 0.01 }, startTime + 0.01)
          .set(el, { display: 'none', visibility: 'hidden' }, startTime + 0.03)
      }


      // REFINED SEQUENCE - Intentional halts and spacing
      animateIn('.line-1', 0.05)
      animateOut('.line-1', 0.13)

      animateIn('.line-2', 0.15)
      animateOut('.line-2', 0.23)

      animateIn('.line-3', 0.25)
      animateOut('.line-3', 0.33)

      animateIn('.stanza-a', 0.35)
      animateOut('.stanza-a', 0.43)

      animateIn('.stanza-b', 0.45)
      animateOut('.stanza-b', 0.53)

      const stanzaC = stage.querySelector('.stanza-c')
      if (stanzaC) {
        mainTimeline.set(stanzaC, { display: 'flex', visibility: 'visible' }, 0.55)
        mainTimeline.to(stanzaC, { autoAlpha: 1, duration: 0.01 }, 0.55)
        mainTimeline.fromTo(stanzaC.querySelectorAll('.char, .char-tremble'), {
          opacity: 0,
          filter: 'blur(20px)',
          y: 30
        }, {
          opacity: 1,
          filter: 'blur(0px)',
          y: 0,
          duration: 0.03,
          stagger: 0.001,
          ease: 'power3.out'
        }, 0.551)
        
        // Scroll Bump for stanza-c
        mainTimeline.to(stanzaC, { scale: 1.05, duration: 0.015 }, 0.58)
        mainTimeline.to(stanzaC, { scale: 1, duration: 0.015 }, 0.595)

        const trembleChars = stanzaC.querySelectorAll('.char-tremble')
        trembleChars.forEach((char, i) => {
          mainTimeline.to(char, {
            x: 'random(-4, 4)',
            y: 'random(-4, 4)',
            duration: 0.005,
            repeat: 8,
            yoyo: true,
            ease: 'none'
          }, 0.60 + (i * 0.001))
        })
        animateOut('.stanza-c', 0.65)
      }

      // Step 7: FINALE
      const finale = stage.querySelector('.line-finale')
      if (finale) {
        mainTimeline.set(finale, { display: 'flex', visibility: 'visible' }, 0.70)
        mainTimeline.to(finale, { autoAlpha: 1, duration: 0.01 }, 0.70)
        
        mainTimeline.fromTo(finale.querySelectorAll('.finale-2 .char'), {
          opacity: 0,
          filter: 'blur(20px)',
          y: 30,
        }, {
          opacity: 1,
          filter: 'blur(0px)',
          y: 0,
          duration: 0.03,
          stagger: 0.0015,
          ease: 'power2.out'
        }, 0.701)

        // Finale Bump
        mainTimeline.to(finale, { scale: 1.08, duration: 0.02, ease: 'expo.out' }, 0.74)
        mainTimeline.to(finale, { scale: 1, duration: 0.02, ease: 'expo.in' }, 0.76)

        mainTimeline.to(finale.querySelectorAll('.finale-2 .char'), {
          opacity: 0,
          filter: 'blur(15px)',
          y: -30,
          duration: 0.02,
          stagger: 0.001,
          ease: 'power2.in'
        }, 0.78)
        
        mainTimeline.set(finale, { display: 'none', visibility: 'hidden' }, 0.81)
      }


      // Step 8: MOMENTUM (ON WHITE) - WITH SCATTER
      const momentum = stage.querySelector('.line-momentum')
      if (momentum) {
        mainTimeline.set(momentum, { display: 'flex', visibility: 'visible' }, 0.85)
        mainTimeline.to(momentum, { autoAlpha: 1, duration: 0.01 }, 0.85)
        
        const allChars = momentum.querySelectorAll('.char')
        const transienceChars = momentum.querySelectorAll('.transience-scatter .char')
        const otherChars = Array.from(allChars).filter(c => !c.closest('.transience-scatter'))

        mainTimeline.fromTo(allChars, {
          opacity: 0,
          filter: 'blur(20px)',
          y: 30,
        }, {
          opacity: 1,
          filter: 'blur(0px)',
          y: 0,
          duration: 0.04,
          stagger: 0.0015,
          ease: 'power2.out'
        }, 0.86)

        // Momentum Bump before scatter
        mainTimeline.to(momentum, { y: -10, scale: 1.03, duration: 0.02 }, 0.89)
        mainTimeline.to(momentum, { y: 0, scale: 1, duration: 0.02 }, 0.91)

        // Scatter transience into the air
        mainTimeline.to(transienceChars, {
          x: () => (Math.random() - 0.5) * 800,
          y: () => -600 - Math.random() * 600,
          rotation: () => Math.random() * 720,
          filter: 'blur(25px)',
          opacity: 0,
          scale: 0.3,
          duration: 0.06,
          stagger: 0.003,
          ease: 'power2.out'
        }, 0.94)

        mainTimeline.to(otherChars, {
          opacity: 0,
          filter: 'blur(25px)',
          y: -100,
          stagger: 0.001,
          duration: 0.04,
          ease: 'power2.in'
        }, 0.94)
        
        mainTimeline.to(momentum, { autoAlpha: 0, duration: 0.005 }, 0.995)
      }





}

onUnmounted(() => {
  ScrollTrigger.getAll().forEach(st => st.kill())
})
</script>

<template>
  <div class="page-home">
    <div v-if="appStore.introStep === 0 && introStarted" class="collection-nav" @click="scrollToCollection">
      <span class="collection-label">COLLECTION</span>
      <img :src="'/paw-logo.png'" alt="Paw Logo" class="nav-logo" />
    </div>

    <div v-show="showScrollPrompt" class="scroll-prompt">
      <span>Scroll to begin</span>
    </div>

    <div ref="haikuStage" class="haiku-stage" v-show="introStarted">
      <div class="haiku-container">
        <div class="poem-line line-1 center-left">
          <p v-html="wrapChars('spinning pulse of green')"></p>
        </div>
        <div class="poem-line line-2 center-right">
          <p v-html="wrapChars('an unfolding breath on black')"></p>
        </div>
        
        <div class="poem-line line-3 top-center">
          <p v-html="wrapChars(`a cat's tail plaits through moonlight`)"></p>
        </div>
        
        <div class="poem-line stanza-a dead-center">
          <p v-html="wrapChars('unworried as secrets')"></p>
        </div>
        <div class="poem-line stanza-b dead-center heavy">
          <p v-html="wrapChars('I hold questions like stones')"></p>
        </div>
        <div class="poem-line stanza-c dead-center">
          <p>
            <span v-html="wrapChars('they ')"></span>
            <span class="tremble-wrap">
              <span v-for="(char, i) in 'tremble'" :key="i" class="char char-tremble">{{ char }}</span>
            </span>
            <span v-html="wrapChars(' on my tongue')"></span>
          </p>
        </div>
        
                <div class="poem-line line-finale dead-center hero">
          <div class="finale-2 greeny-text" v-html="wrapChars('This is a quiet orbit of change')"></div>
        </div>
        
        <div class="poem-line line-momentum dead-center hero">
          <div class="finale-2 smoke-text">
            <span v-html="wrapChars('and my way of ')"></span>
            <span class="transience-scatter" v-html="wrapChars('transience')"></span>
          </div>
        </div>
      </div>
    </div>

    <section v-if="showGallerySection" class="gallery-section">
      <div class="gallery-content">
        <div class="gallery-header">
          <h2 class="gallery-title">The Collection</h2>
          <p class="gallery-subtitle">A GATHERING OF VERSE</p>
        </div>
        
        <div class="miniatures-wrapper">
          <div class="miniatures-scroll">
            <PoemMiniature 
              v-for="poem in appStore.poems" 
              :key="poem.id" 
              :poem="poem"
              @select="handlePoemSelect"
            />
          </div>
        </div>
      </div>
    </section>

    <div v-if="isTransporting" class="transport-overlay">
      <div class="transport-line"></div>
    </div>

    <div class="film-grain"></div>
  </div>
</template>

<style lang="scss" scoped>
.page-home {
  width: 100%;
  min-height: 100vh;
  position: relative;
  z-index: 10;
  pointer-events: auto;
}

.scroll-prompt {
  position: fixed;
  bottom: 8%;
  left: 50%;
  transform: translateX(-50%);
  z-index: 30;
  pointer-events: none;
  
    span {
      font-family: 'Cormorant Garamond', serif;
      font-size: 0.9rem;
      text-transform: uppercase;
      letter-spacing: 0.4em;
      color: #3d5a3a;
      opacity: 0.6;
    }
}


.haiku-stage {
  width: 100%;
  height: 100vh;
  position: relative;
  overflow: hidden;
  z-index: 20;
}

.haiku-container {
  position: relative;
  width: 100%;
  height: 100%;
}

  .poem-line {
  position: absolute;
  font-family: 'Cormorant Garamond', serif;
  font-weight: 300;
  font-size: clamp(1rem, 5vw, 1.6rem);
  color: #fafafa;
  line-height: 1.4;
  width: 90%;
  max-width: 60ch;
  will-change: transform, opacity, filter;
  opacity: 0;
  visibility: hidden;
  display: none;
  overflow-wrap: break-word;
  word-break: break-word;
  
  p { margin: 0; }
  
  :deep(.char) {
    display: inline-block;
    will-change: transform, filter, opacity;
    &.space { width: 0.25em; }
  }

  &.center-left { top: 40%; left: 50%; transform: translate(-50%, -50%); text-align: center; }
  &.center-right { top: 60%; left: 50%; transform: translate(-50%, -50%); text-align: center; }
  &.top-center { top: 30%; left: 50%; transform: translateX(-50%); text-align: center; }
  &.dead-center { top: 50%; left: 50%; transform: translate(-50%, -50%); text-align: center; }
  &.heavy { font-weight: 500; letter-spacing: -0.01em; }
  &.hero {
    font-size: clamp(1.2rem, 7vw, 2.2rem);
    width: 95%;
    max-width: 1200px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.8rem;
  }
}

@media (min-width: 768px) {
  .poem-line {
    width: auto;
    font-size: clamp(1.1rem, 2.6vw, 1.8rem);
    &.center-left { top: 50%; left: 10%; transform: translateY(-50%); text-align: left; }
    &.center-right { top: 50%; right: 10%; transform: translateY(-50%); text-align: right; }
    &.hero { font-size: clamp(1.4rem, 6vw, 2.4rem); width: 90%; }
  }
}


.faint {
  opacity: 0.6;
  font-size: 0.4em;
  letter-spacing: 0.5em;
  text-transform: uppercase;
  margin-bottom: 0.5rem;
  color: #3d5a3a;
}

.finale-3 {
  // Matched with finale-2
}

.smoke-text {
  color: #3d5a3a;
  font-weight: 400;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 0.35em;
  letter-spacing: 0.05em;
}

.greeny-text {
  color: #3d5a3a;
  font-weight: 400;
}

@media (max-width: 768px) {
  .collection-nav {
    top: 1.5rem;
    right: 1.5rem;
    gap: 0.8rem;
    .collection-label { display: none; }
  }
  
  .gallery-section .gallery-header {
    margin-bottom: 4rem;
    .gallery-title {
      font-size: clamp(2.2rem, 10vw, 3.5rem);
      letter-spacing: 0.2em;
    }
  }
}

.char-tremble { display: inline-block; }
.tremble-wrap { display: inline-block; font-style: italic; }

.gallery-section {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  background: transparent;
  padding: 15vh 0;
  
    .gallery-header {
      text-align: center;
      margin-bottom: 8rem;
      
      .gallery-title {
        font-family: 'Cormorant Garamond', serif;
        font-size: clamp(3.5rem, 8vw, 6rem);
        font-weight: 200;
        color: #3d5a3a;
        letter-spacing: 0.35em;
        line-height: 1.1;
        text-transform: uppercase;
        margin-bottom: 2rem;
      }
      
      .gallery-subtitle {
        font-family: 'Cormorant Garamond', serif;
        font-size: 0.85rem;
        color: #3d5a3a;
        letter-spacing: 0.8em;
        text-transform: uppercase;
        opacity: 0.6;
        font-weight: 300;
      }
    }

  .miniatures-wrapper {
    width: 100%;
    overflow-x: auto;
    padding: 4rem 0;
    -ms-overflow-style: none;
    scrollbar-width: none;
    &::-webkit-scrollbar { display: none; }
  }

  .miniatures-scroll {
    display: flex;
    padding: 0 15vw;
    width: max-content;
    gap: 5vw;
  }
}

.transport-overlay {
  position: fixed;
  inset: 0;
  background: #fdfaf6;
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
  .transport-line {
    width: 1px;
    height: 0;
    background: #3d5a3a;
    animation: transportGrow 1.5s cubic-bezier(0.77, 0, 0.175, 1) forwards;
  }
}

@keyframes transportGrow {
  0% { height: 0; opacity: 1; }
  50% { height: 100vh; opacity: 1; }
  100% { height: 100vh; opacity: 0; }
}

.film-grain {
  position: fixed;
  inset: 0;
  z-index: 100;
  pointer-events: none;
  opacity: 0.06;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
}

.collection-nav {
  position: fixed;
  top: 2rem;
  right: 2rem;
  display: flex;
  align-items: center;
  gap: 1.2rem;
  z-index: 100;
  cursor: pointer;
  padding: 0.6rem 1rem;
  border-radius: 8px;
  transition: all 0.6s cubic-bezier(0.23, 1, 0.32, 1);
  
  &:hover {
    background: rgba(61, 90, 58, 0.12);
    .collection-label { 
      letter-spacing: 0.65em;
      opacity: 1;
      color: #2d4a2a;
    }
    .nav-logo {
      filter: sepia(1) hue-rotate(70deg) saturate(2) drop-shadow(0 0 10px rgba(61, 90, 58, 0.8));
    }
  }

  .nav-logo {
    width: 28px;
    height: 28px;
    object-fit: contain;
    transition: all 0.6s cubic-bezier(0.23, 1, 0.32, 1);
    filter: sepia(1) hue-rotate(70deg) saturate(1.5);
  }

  .collection-label {
    font-family: 'Cormorant Garamond', serif;
    font-size: 0.8rem;
    font-weight: 400;
    letter-spacing: 0.5em;
    color: #3d5a3a;
    opacity: 0.8;
    text-transform: uppercase;
    transition: all 0.6s cubic-bezier(0.23, 1, 0.32, 1);
  }
}
</style>
