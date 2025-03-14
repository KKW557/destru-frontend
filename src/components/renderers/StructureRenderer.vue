<script setup lang="ts">
import { onMounted, onUnmounted, ref, watch } from 'vue'
import { type StructureProvider, StructureRenderer } from 'deepslate'
import { mat4 } from 'gl-matrix'
import { useResourceStore } from '@/stores/resource.store.ts'
import { storeToRefs } from 'pinia'

interface Options {
  grid: boolean
  invisibleBlocks: boolean
}

const props = defineProps<{
  structure: StructureProvider
  options: Options
}>()

const container = ref<HTMLElement>()
const canvas = ref<HTMLCanvasElement>()

const resizeObserver = new ResizeObserver(() => {
  if (!container.value || !canvas.value) return

  canvas.value.width = container.value.clientWidth
  canvas.value.height = container.value.clientHeight

  requestAnimationFrame(redraw)
})

onMounted(() => {
  if (container.value) {
    resizeObserver.observe(container.value)
  }
})

onUnmounted(() => {
  resizeObserver.disconnect()
  interactiveCanvas?.destroy()
})

class InteractiveCanvas {
  private xRot = 0.5
  private yRot = 0.5
  private dragPos: [number, number] | null = null
  private pinchStartDist: number | null = null
  private pinchStartViewDist: number | null = null

  private handleMouseDown = (evt: MouseEvent) => {
    if (evt.button === 0) {
      this.dragPos = [evt.clientX, evt.clientY]
    }
  }
  private handleMouseMove = (evt: MouseEvent) => {
    if (this.dragPos) {
      this.yRot += (evt.clientX - this.dragPos[0]) / 100
      this.xRot += (evt.clientY - this.dragPos[1]) / 100
      this.dragPos = [evt.clientX, evt.clientY]
      this.redraw()
    }
  }
  private handleMouseUp = () => {
    this.dragPos = null
  }
  private handleWheel = (evt: WheelEvent) => {
    evt.preventDefault()
    this.viewDist += evt.deltaY / 100
    this.redraw()
  }
  private handleTouchStart = (evt: TouchEvent) => {
    evt.preventDefault()
    if (evt.touches.length === 1) {
      const touch = evt.touches[0]
      this.dragPos = [touch.clientX, touch.clientY]
      this.pinchStartDist = null
    } else if (evt.touches.length === 2) {
      const t1 = evt.touches[0]
      const t2 = evt.touches[1]
      this.pinchStartDist = Math.hypot(t2.clientX - t1.clientX, t2.clientY - t1.clientY)
      this.pinchStartViewDist = this.viewDist
      this.dragPos = null
    }
  }
  private handleTouchMove = (evt: TouchEvent) => {
    evt.preventDefault()
    if (evt.touches.length === 1 && this.dragPos) {
      const touch = evt.touches[0]
      this.yRot += (touch.clientX - this.dragPos[0]) / 100
      this.xRot += (touch.clientY - this.dragPos[1]) / 100
      this.dragPos = [touch.clientX, touch.clientY]
      this.redraw()
    } else if (
      evt.touches.length === 2 &&
      this.pinchStartDist &&
      this.pinchStartViewDist !== null
    ) {
      const t1 = evt.touches[0]
      const t2 = evt.touches[1]
      const currentDist = Math.hypot(t2.clientX - t1.clientX, t2.clientY - t1.clientY)
      const scale = currentDist / this.pinchStartDist
      this.viewDist = this.pinchStartViewDist / scale
      this.redraw()
    }
  }
  private handleTouchEnd = (evt: TouchEvent) => {
    if (evt.touches.length === 0) {
      this.dragPos = null
      this.pinchStartDist = null
      this.pinchStartViewDist = null
    } else if (evt.touches.length === 1) {
      const touch = evt.touches[0]
      this.dragPos = [touch.clientX, touch.clientY]
      this.pinchStartDist = null
      this.pinchStartViewDist = null
    }
  }

  constructor(
    private readonly canvas: HTMLCanvasElement,
    private readonly onRender: (view: mat4) => void,
    private readonly size: [number, number, number],
    private readonly center: [number, number, number] = [size[0] / 2, size[1] / 2, size[2] / 2],
    private viewDist = Math.sqrt(size[0] ** 2 + size[1] ** 2 + size[2] ** 2),
    private maxViewDist = viewDist ** 2,
  ) {
    canvas.addEventListener('mousedown', this.handleMouseDown)
    canvas.ownerDocument.addEventListener('mousemove', this.handleMouseMove)
    canvas.ownerDocument.addEventListener('mouseup', this.handleMouseUp)
    canvas.addEventListener('wheel', this.handleWheel)
    canvas.addEventListener('touchstart', this.handleTouchStart)
    canvas.addEventListener('touchmove', this.handleTouchMove)
    canvas.addEventListener('touchend', this.handleTouchEnd)
  }

  public destroy() {
    this.canvas.removeEventListener('mousedown', this.handleMouseDown)
    this.canvas.ownerDocument.removeEventListener('mousemove', this.handleMouseMove)
    this.canvas.ownerDocument.removeEventListener('mouseup', this.handleMouseUp)
    this.canvas.removeEventListener('wheel', this.handleWheel)
    this.canvas.removeEventListener('touchstart', this.handleTouchStart)
    this.canvas.removeEventListener('touchmove', this.handleTouchMove)
    this.canvas.removeEventListener('touchend', this.handleTouchEnd)
  }

  public redraw() {
    requestAnimationFrame(() => this.renderImmediately())
  }

  private renderImmediately() {
    this.yRot = this.yRot % (Math.PI * 2)
    this.xRot = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, this.xRot))
    this.viewDist = Math.min(Math.max(this.viewDist, 1), this.maxViewDist)

    const view = mat4.create()
    mat4.translate(view, view, [0, 0, -this.viewDist])
    mat4.rotate(view, view, this.xRot, [1, 0, 0])
    mat4.rotate(view, view, this.yRot, [0, 1, 0])
    mat4.translate(view, view, [-this.center[0], -this.center[1], -this.center[2]])

    this.onRender(view)
  }
}

let renderer: StructureRenderer | null = null
let interactiveCanvas: InteractiveCanvas | null = null

const resourceStore = useResourceStore()
const { isReady, resources } = storeToRefs(resourceStore)

const redraw = () => {
  if (!isReady.value || !resources.value || !canvas.value) return

  const gl = canvas.value.getContext('webgl')
  if (!gl) return

  if (interactiveCanvas) {
    interactiveCanvas.destroy()
    interactiveCanvas = null
  }
  if (renderer) {
    renderer = null
  }

  const size = props.structure.getSize()
  renderer = new StructureRenderer(gl, props.structure, resources.value, {
    useInvisibleBlockBuffer: false,
  })

  interactiveCanvas = new InteractiveCanvas(
    canvas.value,
    (view) => {
      gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT)
      gl.viewport(0, 0, canvas.value!.width, canvas.value!.height)

      renderer!.drawStructure(view)
      if (props.options.grid) {
        renderer!.drawGrid(view)
      }
      if (props.options.invisibleBlocks) {
        renderer!.drawInvisibleBlocks(view)
      }
    },
    size,
  )
  interactiveCanvas.redraw()
}

watch([() => props.options.grid, () => props.options.invisibleBlocks], () => {
  interactiveCanvas?.redraw()
})
</script>

<template>
  <div ref="container" class="container">
    <canvas ref="canvas" />
  </div>
</template>

<style scoped>
.container {
  width: 100%;
  height: 100%;
}
</style>
