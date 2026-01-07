<script setup lang="ts">
import { useRenderLoop } from '@tresjs/core'
import { shallowRef, watch, onMounted, onUnmounted } from 'vue'
import { CatmullRomCurve3, Vector3, Color, ShaderMaterial } from 'three'
import { useAppStore } from '@/stores/appState'
import fillVert from '@/assets/shaders/spiral/fill.vert'
import fillFrag from '@/assets/shaders/spiral/fill.frag'
import gsap from 'gsap'

const appStore = useAppStore()

const meshRef = shallowRef()
const groupRef = shallowRef()
const materialRef = shallowRef<ShaderMaterial>()
const isVisible = shallowRef(true)

const points: Vector3[] = []
for (let i = 0; i < 100; i++) {
  const t = i / 100
  const angle = t * Math.PI * 10
  const radius = 0.5 + t * 1.5
  points.push(new Vector3(
    Math.cos(angle) * radius,
    Math.sin(angle) * radius,
    t * 3
  ))
}
const curve = new CatmullRomCurve3(points)

const uniforms = {
  uProgress: { value: 0 },
  uOpacity: { value: 1.0 },
  uTime: { value: 0 },
  uGlow: { value: 0 },
  uColor: { value: new Color('#3a3a3a') },
  uEmissive: { value: new Color('#5a5a5a') }
}

const internalProgress = shallowRef(0)
let loadingTween: gsap.core.Tween | null = null

const { onLoop } = useRenderLoop()

const isMobile = computed(() => typeof window !== 'undefined' && window.innerWidth < 768)

onLoop(({ delta }) => {
  if (!isVisible.value) return
  const material = materialRef.value
  if (material?.uniforms) {
    if (material.uniforms.uProgress && typeof internalProgress.value === 'number') {
      material.uniforms.uProgress.value = internalProgress.value
    }
    if (material.uniforms.uOpacity && typeof uniforms.uOpacity.value === 'number') {
      material.uniforms.uOpacity.value = uniforms.uOpacity.value
    }
    if (material.uniforms.uGlow) {
      material.uniforms.uGlow.value = uniforms.uGlow.value
    }
    if (material.uniforms.uTime) {
      material.uniforms.uTime.value += delta
    }
  }
})

onMounted(() => {
  loadingTween = gsap.to(internalProgress, {
    value: 1,
    duration: 10,
    ease: 'power1.inOut',
    onUpdate: () => {
      appStore.setLoadingProgress(internalProgress.value)
    },
    onComplete: () => {
      appStore.setLoadingProgress(1)
      // Rapid faint glow at the end
      gsap.to(uniforms.uGlow, {
        value: 0.8,
        duration: 0.15,
        repeat: 1,
        yoyo: true,
        ease: 'sine.inOut'
      })
    }
  })
})

watch(() => appStore.loadingComplete, (complete) => {
  if (complete) {
    gsap.to(uniforms.uOpacity, {
      value: 0,
      duration: 0.8,
      ease: 'power2.inOut',
      onComplete: () => {
        isVisible.value = false
        appStore.finishLoading()
      }
    })
  }
})

onUnmounted(() => {
  if (loadingTween) loadingTween.kill()
})
</script>

<template>
  <TresGroup v-if="isVisible" ref="groupRef" :position="isMobile ? [0, 0, -2] : [3.5, 0, -2]">
    <TresMesh ref="meshRef">
      <TresTubeGeometry :args="[curve, 128, isMobile ? 0.04 : 0.06, 12, false]" />
      <TresShaderMaterial
        ref="materialRef"
        :vertex-shader="fillVert"
        :fragment-shader="fillFrag"
        :uniforms="uniforms"
        :transparent="true"
        :depth-write="false"
      />
    </TresMesh>
  </TresGroup>
</template>
