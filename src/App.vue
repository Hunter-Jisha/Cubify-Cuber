<script lang="ts" setup>
import { ref, onMounted, watch } from 'vue'
import * as THREE from 'three'
import SimpleCamera from 'simple-vue-camera'

const taken = ref(false)

watch(taken, () => {
  if(taken.value) {
    renderer.setAnimationLoop(() => {
      renderer.setSize(canvas.value?.clientWidth ?? 512, canvas.value?.clientWidth ?? 512)
      renderer.render(scene, camera)
    })
  }
})

const actualCamera = ref < InstanceType < typeof SimpleCamera >> ()
const canvas = ref <HTMLCanvasElement> ()

const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100)
camera.position.set(0, 0, 2.5)

const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setSize(512, 512)

const geometry = new THREE.BoxGeometry(1, 1, 1)
const neutralMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 })
const cube = new THREE.Mesh(geometry, [
  neutralMaterial, neutralMaterial, neutralMaterial,
  neutralMaterial, neutralMaterial, neutralMaterial
])
cube.rotation.set(0.6, -0.75, 0)
scene.add(cube)

onMounted(() => {
  canvas.value!.appendChild(renderer.domElement)
})

function canvasFromCrop(
  source: CanvasImageSource,
  sx: number, sy: number, sw: number, sh: number,
  outW: number, outH: number
) {
  const c = document.createElement('canvas')
  c.width = outW
  c.height = outH
  const ctx = c.getContext('2d')!
  ctx.drawImage(source, sx, sy, sw, sh, 0, 0, outW, outH)
  return c
}

function textureFromCanvas(c: HTMLCanvasElement) {
  const tex = new THREE.CanvasTexture(c)
  tex.colorSpace = THREE.SRGBColorSpace
  tex.needsUpdate = true
  return tex
}

async function shoot() {
  const blob = await actualCamera.value?.snapshot()
  if (!blob) return

  const bitmap = await createImageBitmap(blob)

  const leftTexture = textureFromCanvas(canvasFromCrop(bitmap, 0, bitmap.height / 3, bitmap.width / 3, bitmap.height / 3 * 2, 512, 512))
  const rightTexture = textureFromCanvas(canvasFromCrop(bitmap, (bitmap.width / 3) * 2, bitmap.height / 3, bitmap.width / 3, bitmap.height / 3 * 2, 512, 512))
  const centerTexture = textureFromCanvas(canvasFromCrop(bitmap, bitmap.width / 3, bitmap.height / 3, bitmap.width / 3, bitmap.height / 3 * 2, 512, 512))
  const topTexture = textureFromCanvas(canvasFromCrop(bitmap, bitmap.width / 3, 0, bitmap.width / 3, bitmap.height / 3, 512, 512))

  const matLeft = new THREE.MeshBasicMaterial({ map: leftTexture })
  const matRight = new THREE.MeshBasicMaterial({ map: rightTexture })
  const matCenter = new THREE.MeshBasicMaterial({ map: centerTexture })
  const matTop = new THREE.MeshBasicMaterial({ map: topTexture })

  cube.material = [matRight, matLeft, matTop, neutralMaterial, matCenter, matTop]

  renderer.render(scene, camera)

  taken.value = true
}
</script>

<template>
  <div class="w-screen h-dvh">
    <div :class="taken ? 'hidden' : 'flex'" class="size-full bg-black relative">
      <SimpleCamera :resolution="{ width: 512, height: 512 }" ref="actualCamera" autoplay />
      <img class="absolute inset-1/2 -translate-1/2" alt="face" src="/face.webp" />
      <button @click="shoot"
        class="bg-white border-gray-200 border-8 size-24 absolute left-1/2 bottom-4 rounded-full -translate-x-1/2" />
    </div>

    <div :class="taken ? 'flex' : 'hidden'" class="size-full bg-black relative">
      <div class="size-full absolute inset-0 flex items-center justify-center" ref="canvas" />
      <button @click="taken = false" class="px-4 py-2 bg-white font-bold text-black text-lg absolute bottom-8 left-1/2 -translate-x-1/2 rounded-full">Retake</button>
    </div>
  </div>
</template>
