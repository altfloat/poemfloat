<script setup lang="ts">
import { useRenderLoop } from '@tresjs/core'
import { ShaderMaterial, DoubleSide, Color, Vector2 } from 'three'
import { shallowRef, watch, ref, onMounted, onUnmounted, computed } from 'vue'
import { useAppStore } from '@/stores/appState'
import gsap from 'gsap'

const appStore = useAppStore()
const materialRef = shallowRef<ShaderMaterial>()

const props = defineProps<{
  mouse: { x: number, y: number }
}>()

const isMobile = computed(() => typeof window !== 'undefined' && window.innerWidth < 768)

// Interaction State
const isHolding = ref(false)
const holdProgress = ref(0)
const bursts = ref<any[]>([])

const startHold = () => {
  if (!appStore.isPlaygroundMode) return
  isHolding.value = true
  holdProgress.value = 0
  
  gsap.to(holdProgress, {
    value: 1,
    duration: 3,
    ease: 'none',
    onComplete: triggerBurst
  })
}

const stopHold = () => {
  isHolding.value = false
  gsap.killTweensOf(holdProgress)
  gsap.to(holdProgress, { value: 0, duration: 0.5 })
}

const triggerBurst = () => {
  if (!isHolding.value || !props.mouse) return
  
  const newBurst = {
    x: props.mouse.x,
    y: props.mouse.y,
    startTime: Date.now() / 1000,
    color1: new Color().setHSL(Math.random() * 0.2 + 0.35, 0.6, 0.3), // Richer Greens/Teals
    color2: new Color().setHSL(Math.random() * 0.1 + 0.05, 0.5, 0.1)  // Deep Inky Red/Purple
  }
  
  bursts.value.push(newBurst)
  if (bursts.value.length > 10) bursts.value.shift()
  
  // Feedback
  gsap.fromTo(holdProgress, { value: 1.5 }, { value: 0, duration: 0.8, ease: 'expo.out' })
}

// Shader Code
const vertexShader = `
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`

const fragmentShader = `
  varying vec2 vUv;
  uniform float uTime;
  uniform vec2 uMouse;
  uniform float uHoldProgress;
  uniform float uPlaygroundMode;
  uniform bool uIsMobile;
  
    struct Burst {
      vec2 pos;
      float startTime;
      vec3 color1;
      vec3 color2;
    };
    
    uniform Burst uBursts[10];
    uniform int uBurstCount;

    // Simple noise function
    float hash(vec2 p) {
      p = fract(p * vec2(123.34, 456.21));
      p += dot(p, p + 45.32);
      return fract(p.x * p.y);
    }

    float noise(vec2 p) {
      vec2 i = floor(p);
      vec2 f = fract(p);
      float a = hash(i);
      float b = hash(i + vec2(1.0, 0.0));
      float c = hash(i + vec2(0.0, 1.0));
      float d = hash(i + vec2(1.0, 1.0));
      vec2 u = f * f * (3.0 - 2.0 * f);
      return mix(a, b, u.x) + (c - a) * u.y * (1.0 - u.x) + (d - b) * u.x * u.y;
    }

    float fbm(vec2 p) {
      float v = 0.0;
      float a = 0.5;
      for (int i = 0; i < 3; i++) {
        v += a * noise(p);
        p *= 2.0;
        a *= 0.5;
      }
      return v;
    }

    void main() {
      vec3 finalColor = vec3(0.0);
      float finalAlpha = 0.0;
      
      float mobileScale = uIsMobile ? 0.7 : 1.0;
      
      // 1. Hold Build-up Effect (Intricate Gathering)
      float dMouse = distance(vUv, uMouse);
      if (uHoldProgress > 0.01) {
        // Intricate gathering with FBM
        float n = fbm(vUv * (uIsMobile ? 15.0 : 20.0) + uTime * 2.0);
        float coreRadius = 0.12 * uHoldProgress * mobileScale;
        float core = smoothstep(coreRadius, 0.0, dMouse);
        float pulse = sin(uTime * 20.0) * 0.02 * uHoldProgress;
        
        float outerRadius = 0.15 * uHoldProgress * mobileScale + pulse + n * 0.05;
        float innerRadius = 0.10 * uHoldProgress * mobileScale;
        
        float ring = smoothstep(outerRadius, outerRadius - 0.01, dMouse) * 
                     smoothstep(innerRadius, innerRadius + 0.01 + n * 0.05, dMouse);
        
        vec3 gatherCol = mix(vec3(0.05, 0.2, 0.15), vec3(0.3, 0.1, 0.2), n);
        finalColor += gatherCol * (core + ring * 2.0) * uHoldProgress;
        finalAlpha += (core + ring) * uHoldProgress;
      }

      // 2. Render Bursts
      for(int i = 0; i < 10; i++) {
        if (i >= uBurstCount) break;
        
        float age = uTime - uBursts[i].startTime;
        if (age > 8.0) continue;
        
        float radius = age * (uIsMobile ? 0.35 : 0.5);
        float dist = distance(vUv, uBursts[i].pos);
        
        // Organic bleeding edge with FBM
        float n = fbm(vUv * (uIsMobile ? 8.0 : 12.0) - age * 0.5) * 0.2;
        float inkEdge = smoothstep(radius + n, radius - 0.15 - n, dist);
        
        // Viscous bleeding
        float bleed = smoothstep(radius * 1.8 + n, radius, dist) * 0.4 * exp(-age * 0.4);
        bleed *= (0.8 + 0.4 * fbm(vUv * 25.0 + age));
        
          float mask = clamp(inkEdge + bleed, 0.0, 1.0);
          float fade = smoothstep(8.0, 5.0, age);
          
          // Intricate Fusion of colors
          float fusion = fbm(vUv * 6.0 + age * 0.3);
          vec3 burstCol = mix(uBursts[i].color1, uBursts[i].color2, fusion);
          
          // Darken edges for depth
          burstCol = mix(burstCol, vec3(0.01, 0.02, 0.01), dist / (radius + 0.2));
          
          finalColor = mix(finalColor, burstCol, mask * fade);
          finalAlpha = max(finalAlpha, mask * fade * 0.85);
        }

      if (finalAlpha < 0.01) discard;
      
      gl_FragColor = vec4(finalColor, finalAlpha * uPlaygroundMode);
    }
`

const uniforms = {
  uTime: { value: 0 },
  uMouse: { value: new Vector2(0.5, 0.5) },
  uHoldProgress: { value: 0 },
  uPlaygroundMode: { value: 0 },
  uIsMobile: { value: isMobile.value },
  uBurstCount: { value: 0 },
  uBursts: { 
    value: Array(10).fill(0).map(() => ({
      pos: new Vector2(0, 0),
      startTime: 0,
      color1: new Color(0, 0, 0),
      color2: new Color(0, 0, 0)
    }))
  }
}

const { onLoop } = useRenderLoop()

onLoop(({ elapsed }) => {
  if (materialRef.value) {
    materialRef.value.uniforms.uTime.value = elapsed
    materialRef.value.uniforms.uMouse.value.set(props.mouse.x, props.mouse.y)
    materialRef.value.uniforms.uHoldProgress.value = holdProgress.value
    
    // Update bursts uniforms
    materialRef.value.uniforms.uBurstCount.value = bursts.value.length
    bursts.value.forEach((b, i) => {
      materialRef.value!.uniforms.uBursts.value[i].pos.set(b.x, b.y)
      materialRef.value!.uniforms.uBursts.value[i].startTime = b.startTime
      materialRef.value!.uniforms.uBursts.value[i].color1.copy(b.color1)
      materialRef.value!.uniforms.uBursts.value[i].color2.copy(b.color2)
    })
  }
})

watch(() => appStore.isPlaygroundMode, (isPlayground) => {
  if (materialRef.value) {
    gsap.to(materialRef.value.uniforms.uPlaygroundMode, {
      value: isPlayground ? 1.0 : 0.0,
      duration: 1.5,
      ease: 'expo.out'
    })
  }
})

onMounted(() => {
  window.addEventListener('mousedown', startHold)
  window.addEventListener('mouseup', stopHold)
  window.addEventListener('touchstart', startHold)
  window.addEventListener('touchend', stopHold)
})

onUnmounted(() => {
  window.removeEventListener('mousedown', startHold)
  window.removeEventListener('mouseup', stopHold)
  window.removeEventListener('touchstart', startHold)
  window.removeEventListener('touchend', stopHold)
  gsap.killTweensOf(holdProgress)
})
</script>

<template>
  <TresMesh :position="[0, 0, -4.5]">
    <TresPlaneGeometry :args="[40, 25]" />
    <TresShaderMaterial
      ref="materialRef"
      :vertex-shader="vertexShader"
      :fragment-shader="fragmentShader"
      :uniforms="uniforms"
      :transparent="true"
      :side="DoubleSide"
      :depth-write="false"
    />
  </TresMesh>
</template>
