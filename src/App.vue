<script lang="ts" setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'
import * as THREE from 'three'
import { IconCameraRotate, IconDownload } from '@tabler/icons-vue'

const DEFAULT_ROTATION_Y = -0.75

const video = ref<HTMLVideoElement>()
const canvasContainer = ref<HTMLDivElement>()
const taken = ref(false)
const facingMode = ref<'user' | 'environment'>('user')
const mirrored = ref(true)
let stream: MediaStream | undefined

async function startCamera() {
  stopCamera()
  stream = await navigator.mediaDevices.getUserMedia({
    video: { facingMode: facingMode.value, width: { ideal: 1920 }, height: { ideal: 1920 } },
    audio: false,
  })
  // Desktop webcams usually report no facing mode, and those should be mirrored too
  mirrored.value = stream.getVideoTracks()[0].getSettings().facingMode !== 'environment'
  video.value!.srcObject = stream
}

function stopCamera() {
  stream?.getTracks().forEach(track => track.stop())
  stream = undefined
}

function switchCamera() {
  facingMode.value = facingMode.value === 'user' ? 'environment' : 'user'
  startCamera()
}

const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100)
camera.position.set(0, 0, 2.5)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
renderer.setClearColor(0x000000, 0)
renderer.setPixelRatio(window.devicePixelRatio)

const neutralMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 })
const cube = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), Array(6).fill(neutralMaterial))
cube.rotation.set(0.6, DEFAULT_ROTATION_Y, 0)
scene.add(cube)

const resizeObserver = new ResizeObserver(([entry]) => {
  renderer.setSize(entry.contentRect.width, entry.contentRect.height)
})

onMounted(() => {
  canvasContainer.value!.appendChild(renderer.domElement)
  resizeObserver.observe(canvasContainer.value!)
  startCamera()
})

onUnmounted(() => {
  stopCamera()
  resizeObserver.disconnect()
  renderer.setAnimationLoop(null)
})

const saveAngle = ref(-45)
let snapped = false
let dragging = false
let lastPointerX = 0
let lastFrameTime = 0

function selectAngle(angle: number) {
  saveAngle.value = angle
  snapped = true
}

function onPointerDown(event: PointerEvent) {
  dragging = true
  snapped = false
  lastPointerX = event.clientX
  ;(event.currentTarget as HTMLElement).setPointerCapture(event.pointerId)
}

function onPointerMove(event: PointerEvent) {
  if (!dragging) return
  cube.rotation.y += (event.clientX - lastPointerX) * 0.01
  lastPointerX = event.clientX
}

function onPointerUp() {
  dragging = false
}

watch(taken, () => {
  if (taken.value) {
    stopCamera()
    cube.rotation.y = DEFAULT_ROTATION_Y
    snapped = false
    lastFrameTime = performance.now()
    renderer.setAnimationLoop(time => {
      const elapsed = time - lastFrameTime
      lastFrameTime = time
      if (snapped) {
        // Ease toward the chosen angle the short way around
        const offset = THREE.MathUtils.degToRad(saveAngle.value) - cube.rotation.y
        cube.rotation.y += Math.atan2(Math.sin(offset), Math.cos(offset)) * Math.min(1, elapsed * 0.01)
      } else if (!dragging) {
        cube.rotation.y += elapsed * 0.00015
      }
      renderer.render(scene, camera)
    })
  } else {
    renderer.setAnimationLoop(null)
    startCamera()
  }
})

function faceTexture(source: CanvasImageSource, sx: number, sy: number, sw: number, sh: number) {
  const canvas = document.createElement('canvas')
  canvas.width = 1024
  canvas.height = 1024
  canvas.getContext('2d')!.drawImage(source, sx, sy, sw, sh, 0, 0, 1024, 1024)
  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  return new THREE.MeshBasicMaterial({ map: texture })
}

function shoot() {
  const { videoWidth, videoHeight } = video.value!
  const size = Math.min(videoWidth, videoHeight)

  // Grab the centered square the user sees, flipped if the preview is mirrored
  const snapshot = document.createElement('canvas')
  snapshot.width = size
  snapshot.height = size
  const ctx = snapshot.getContext('2d')!
  if (mirrored.value) {
    ctx.translate(size, 0)
    ctx.scale(-1, 1)
  }
  ctx.drawImage(video.value!, (videoWidth - size) / 2, (videoHeight - size) / 2, size, size, 0, 0, size, size)

  const third = size / 3
  const left = faceTexture(snapshot, 0, third, third, third * 2)
  const right = faceTexture(snapshot, third * 2, third, third, third * 2)
  const center = faceTexture(snapshot, third, third, third, third * 2)
  const top = faceTexture(snapshot, third, 0, third, third)

  cube.material = [right, left, top, neutralMaterial, center, top]
  taken.value = true
}

function download() {
  const size = renderer.getSize(new THREE.Vector2())
  const pixelRatio = renderer.getPixelRatio()
  const rotationY = cube.rotation.y

  renderer.setPixelRatio(1)
  renderer.setSize(2048, 2048, false)
  cube.rotation.y = THREE.MathUtils.degToRad(saveAngle.value)
  renderer.render(scene, camera)
  const url = renderer.domElement.toDataURL('image/png')

  renderer.setPixelRatio(pixelRatio)
  renderer.setSize(size.x, size.y)
  cube.rotation.y = rotationY

  // Phones ignore download links, so use the share sheet; decoding synchronously keeps the tap that Safari requires
  const bytes = Uint8Array.from(atob(url.split(',')[1]), char => char.charCodeAt(0))
  const file = new File([bytes], 'cube.png', { type: 'image/png' })
  if (matchMedia('(pointer: coarse)').matches && navigator.canShare?.({ files: [file] })) {
    navigator.share({ files: [file] })
    return
  }

  const link = document.createElement('a')
  link.href = url
  link.download = 'cube.png'
  link.click()
}
</script>

<template>
  <main class="w-screen h-dvh bg-black flex flex-col items-center justify-center overflow-hidden select-none">
    <div class="relative size-[min(100vw,calc(100dvh-8rem))]">
      <div v-show="!taken" class="absolute inset-0">
        <video ref="video" class="size-full object-cover" :class="{ '-scale-x-100': mirrored }" autoplay muted
          playsinline />
        <img class="absolute inset-0 size-full pointer-events-none" alt="" src="/face.svg" />
      </div>
      <div v-show="taken" ref="canvasContainer" class="absolute inset-0 touch-none cursor-grab active:cursor-grabbing"
        @pointerdown="onPointerDown" @pointermove="onPointerMove" @pointerup="onPointerUp"
        @pointercancel="onPointerUp" />
    </div>

    <div class="relative w-full max-w-md h-32 shrink-0 flex items-center justify-center">
      <template v-if="!taken">
        <button @click="shoot" aria-label="Take picture"
          class="bg-white border-gray-300 border-8 size-20 rounded-full active:scale-95 transition-transform" />
        <button @click="switchCamera" aria-label="Switch camera"
          class="bg-white p-3 rounded-full text-black absolute right-6 size-14 active:scale-95 transition-transform">
          <IconCameraRotate class="size-full" />
        </button>
      </template>
      <div v-else class="flex items-center gap-3">
        <button @click="taken = false"
          class="h-12 px-5 rounded-full font-semibold text-white bg-white/10 ring-1 ring-white/15 hover:bg-white/15 active:scale-95 transition">
          Retake
        </button>
        <div class="flex items-center h-12 p-1 rounded-full bg-white/10 ring-1 ring-white/15 backdrop-blur">
          <button @click="download"
            class="h-full flex items-center gap-1.5 pl-3 pr-4 rounded-full bg-white text-black font-semibold active:scale-95 transition">
            <IconDownload class="size-5" stroke-width="2.25" />
            Save
          </button>
          <button v-for="angle in [45, 0, -45]" :key="angle" @click="selectAngle(angle)" :aria-pressed="saveAngle === angle"
            class="h-full w-12 rounded-full text-sm font-medium tabular-nums transition-colors"
            :class="saveAngle === angle ? 'bg-white/20 text-white' : 'text-white/50 hover:text-white/80'">
            {{ angle < 0 ? `−${-angle}` : angle }}°
          </button>
        </div>
      </div>
    </div>
  </main>
</template>
