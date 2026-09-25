<template>
  <div class="app-shell">
    <header class="topbar">
      <div class="brand">
        <div class="brand-mark">豆</div>
        <div>
          <h1>Webfem 拼豆图纸生成器</h1>
          <p>{{ cols }} × {{ rows }} 格 · 共 {{ totalCells.toLocaleString() }} 颗 · {{ palette.length }} 种 MARD 颜色</p>
        </div>
      </div>
      <div class="top-actions">
        <label class="button secondary upload-btn">
          <span>＋</span> 导入图片
          <input ref="fileInput" type="file" accept="image/*" @change="handleFile" />
        </label>
        <button class="button secondary" @click="exportPng">⇩ 导出 PNG</button>
        <button class="button primary" @click="resetPattern">↻ 重置</button>
      </div>
    </header>

    <section class="toolbar-card">
      <div class="toolbar-main">
        <div class="field-group">
          <label>拼豆盘宽度</label>
          <div class="number-control">
            <button @click="setCols(cols - 1)">−</button>
            <input v-model.number="colsInput" type="number" min="8" max="100" @change="setCols(colsInput)" />
            <button @click="setCols(cols + 1)">＋</button>
          </div>
        </div>
        <div class="field-group">
          <label>拼豆盘高度</label>
          <div class="number-control">
            <button @click="setRows(rows - 1)">−</button>
            <input v-model.number="rowsInput" type="number" min="8" max="100" @change="setRows(rowsInput)" />
            <button @click="setRows(rows + 1)">＋</button>
          </div>
        </div>
        <div class="preset-group">
          <label>快速尺寸</label>
          <div class="preset-list">
            <button v-for="preset in presets" :key="preset.label" :class="['preset', { active: cols === preset.w && rows === preset.h }]" @click="applyPreset(preset)">
              {{ preset.label }}
            </button>
          </div>
        </div>
        <div class="stat-block">
          <span>当前尺寸</span>
          <strong>{{ cols }} × {{ rows }}</strong>
        </div>
      </div>
      <div class="resize-hint">拖动画布右下角 ↘ 可以直接调整拼豆盘大小</div>
    </section>

    <main class="workspace">
      <aside class="settings-panel">
        <div class="panel-title">
          <div>
            <h2>图纸参数</h2>
            <p>上传照片后自动匹配 MARD 221 色。</p>
          </div>
        </div>

        <div class="source-card" :class="{ empty: !sourceImage }">
          <img v-if="sourceImage" :src="sourceImage" alt="原图" />
          <div v-else class="empty-source">
            <div class="empty-icon">▧</div>
            <strong>还没有照片</strong>
            <span>点击上方「导入图片」开始</span>
          </div>
        </div>

        <div class="option-row">
          <label class="check"><input v-model="showCoordinates" type="checkbox" /> <span>显示坐标</span></label>
          <label class="check"><input v-model="showCodes" type="checkbox" /> <span>显示格内色号</span></label>
        </div>

        <div class="range-group">
          <div class="range-head"><label>画布大小</label><strong>{{ zoom }}%</strong></div>
          <input v-model.number="zoom" type="range" min="50" max="180" />
        </div>

        <div class="tips">
          <div class="tip-icon">i</div>
          <div>
            <strong>使用提示</strong>
            <p>导入照片后会自动压缩到当前拼豆盘尺寸，并从 221 种 MARD 颜色中寻找最接近的颜色。</p>
          </div>
        </div>
      </aside>

      <section class="canvas-panel">
        <div class="tabs">
          <button v-for="tab in tabs" :key="tab.id" :class="['tab', { active: activeTab === tab.id }]" @click="activeTab = tab.id">
            <span class="tab-icon">{{ tab.icon }}</span>{{ tab.label }}
          </button>
        </div>

        <div class="canvas-toolbar">
          <div>
            <strong>{{ activeTabLabel }}</strong>
            <span>{{ activeTab === 'edit' ? '点击任意格子，再用右侧调色盘修改颜色' : activeTab === 'iron' ? '去除格线与色号，只保留颜色区块' : '显示拼豆位置、坐标与 MARD 色号' }}</span>
          </div>
          <div class="zoom-controls">
            <button @click="zoom = Math.max(50, zoom - 10)">−</button>
            <span>{{ zoom }}%</span>
            <button @click="zoom = Math.min(180, zoom + 10)">＋</button>
            <button class="fit" @click="zoom = 100">适配画布</button>
          </div>
        </div>

        <div ref="boardViewport" class="board-viewport">
          <div class="board-wrap" :style="{ width: boardWidthPx + 'px', height: boardHeightPx + 'px', '--cols': cols }">
            <div v-if="showCoordinates && activeTab !== 'iron'" class="col-labels">
              <span v-for="c in cols" :key="c">{{ c }}</span>
            </div>
            <div v-if="showCoordinates && activeTab !== 'iron'" class="row-labels">
              <span v-for="r in rows" :key="r">{{ r }}</span>
            </div>
            <canvas ref="boardCanvas" :width="canvasPixelWidth" :height="canvasPixelHeight" @pointerdown="handleCanvasPointer"></canvas>
            <div class="resize-handle" title="拖动调整尺寸" @pointerdown.stop="startResize">↘</div>
          </div>
          <div v-if="!sourceImage" class="canvas-empty">
            <div class="upload-drop-icon">↑</div>
            <h3>导入一张照片</h3>
            <p>系统会自动把照片转换成一格一格的拼豆图纸</p>
            <button class="button primary" @click="fileInput?.click()">选择图片</button>
          </div>
        </div>
      </section>

      <aside v-if="activeTab === 'edit'" class="palette-panel">
        <div class="palette-header">
          <div>
            <h2>调色盘</h2>
            <p>仅使用 MARD 221 色</p>
          </div>
          <div class="selected-color" :style="{ background: selectedColor.hex }" :title="selectedColor.id"></div>
        </div>
        <div class="palette-search">
          <span>⌕</span>
          <input v-model="paletteQuery" placeholder="搜索色号，例如 A1" />
        </div>
        <div class="palette-grid">
          <button v-for="color in filteredPalette" :key="color.id" :class="['swatch', { selected: color.id === selectedColor.id }]" :style="{ background: color.hex }" :title="`${color.id} · ${color.hex}`" @click="selectColor(color)">
            <span>{{ color.id }}</span>
          </button>
        </div>
      </aside>
      <aside v-else class="info-panel">
        <div class="info-card">
          <div class="mini-preview" :style="{ backgroundImage: sourceImage ? `url(${sourceImage})` : 'none' }"></div>
          <strong>{{ sourceImage ? '照片已转换' : '等待导入照片' }}</strong>
          <p>{{ sourceImage ? '切换到「修改图片」可以逐格重新上色。' : '支持 JPG、PNG、WEBP 等常见图片格式。' }}</p>
        </div>
        <div class="count-card">
          <span>已使用颜色</span>
          <strong>{{ usedColorCount }}</strong>
          <small>/ {{ palette.length }} MARD 色</small>
        </div>
      </aside>
    </main>
  </div>
</template>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { MARD_PALETTE } from './palette'

const palette = MARD_PALETTE.map(c => ({ ...c, hex: rgbToHex(c.rgb) }))
const paletteMap = new Map(palette.map(c => [c.id, c]))

const presets = [
  { label: '29 × 29', w: 29, h: 29 },
  { label: '32 × 32', w: 32, h: 32 },
  { label: '52 × 36', w: 52, h: 36 },
  { label: '64 × 64', w: 64, h: 64 },
  { label: '80 × 80', w: 80, h: 80 }
]
const tabs = [
  { id: 'preview', label: '预览图片', icon: '▦' },
  { id: 'iron', label: '熨烫效果', icon: '◉' },
  { id: 'edit', label: '修改图片', icon: '✎' }
]

const cols = ref(52)
const rows = ref(36)
const colsInput = ref(52)
const rowsInput = ref(36)
const activeTab = ref('preview')
const sourceImage = ref('')
const showCoordinates = ref(true)
const showCodes = ref(true)
const zoom = ref(108)
const paletteQuery = ref('')
const selectedColor = ref(palette[0])
const cells = ref([])
const fileInput = ref(null)
const boardCanvas = ref(null)
const boardViewport = ref(null)
const isResizing = ref(false)
let resizeStart = null
let imageElement = null

const cellSize = computed(() => Math.max(10, 13 * (zoom.value / 100)))
const canvasPixelWidth = computed(() => Math.round(cols.value * cellSize.value))
const canvasPixelHeight = computed(() => Math.round(rows.value * cellSize.value))
const boardWidthPx = computed(() => canvasPixelWidth.value + 28)
const boardHeightPx = computed(() => canvasPixelHeight.value + 28)
const totalCells = computed(() => cols.value * rows.value)
const activeTabLabel = computed(() => tabs.find(t => t.id === activeTab.value)?.label)
const filteredPalette = computed(() => {
  const q = paletteQuery.value.trim().toLowerCase()
  return q ? palette.filter(c => c.id.toLowerCase().includes(q) || c.hex.toLowerCase().includes(q)) : palette
})
const usedColorCount = computed(() => new Set(cells.value.filter(Boolean)).size)

function rgbToHex(rgb) {
  return '#' + rgb.map(v => Number(v).toString(16).padStart(2, '0')).join('')
}
function hexToRgb(hex) {
  return [parseInt(hex.slice(1, 3), 16), parseInt(hex.slice(3, 5), 16), parseInt(hex.slice(5, 7), 16)]
}
function nearestColor(rgb) {
  let best = palette[0]
  let bestDistance = Infinity
  for (const color of palette) {
    const dr = rgb[0] - color.rgb[0]
    const dg = rgb[1] - color.rgb[1]
    const db = rgb[2] - color.rgb[2]
    const distance = dr * dr + dg * dg + db * db
    if (distance < bestDistance) {
      bestDistance = distance
      best = color
    }
  }
  return best.id
}

function createBlankPattern() {
  cells.value = Array.from({ length: totalCells.value }, () => palette[0].id)
  drawBoard()
}

function setCols(value) {
  const next = Math.max(8, Math.min(100, Number(value) || 52))
  if (next === cols.value) return
  cols.value = next
  colsInput.value = next
  regeneratePattern()
}
function setRows(value) {
  const next = Math.max(8, Math.min(100, Number(value) || 36))
  if (next === rows.value) return
  rows.value = next
  rowsInput.value = next
  regeneratePattern()
}
function applyPreset(preset) {
  cols.value = colsInput.value = preset.w
  rows.value = rowsInput.value = preset.h
  regeneratePattern()
}
function regeneratePattern() {
  if (imageElement) convertImageToPattern(imageElement)
  else createBlankPattern()
}

function handleFile(event) {
  const file = event.target.files?.[0]
  if (!file) return

  const reader = new FileReader()

  reader.onload = () => {
    const dataUrl = reader.result
    const img = new Image()

    img.onload = () => {
      imageElement = img
      sourceImage.value = dataUrl
      convertImageToPattern(img)
    }

    img.src = dataUrl
  }

  reader.readAsDataURL(file)
}

function convertImageToPattern(img) {
  const sourceCanvas = document.createElement('canvas')
  const context = sourceCanvas.getContext('2d', { willReadFrequently: true })
  const targetAspect = cols.value / rows.value
  const imageAspect = img.naturalWidth / img.naturalHeight
  let sx = 0, sy = 0, sw = img.naturalWidth, sh = img.naturalHeight
  if (imageAspect > targetAspect) {
    sw = img.naturalHeight * targetAspect
    sx = (img.naturalWidth - sw) / 2
  } else {
    sh = img.naturalWidth / targetAspect
    sy = (img.naturalHeight - sh) / 2
  }
  sourceCanvas.width = cols.value
  sourceCanvas.height = rows.value
  context.drawImage(img, sx, sy, sw, sh, 0, 0, cols.value, rows.value)
  const pixels = context.getImageData(0, 0, cols.value, rows.value).data
  const next = new Array(totalCells.value)
  for (let i = 0; i < totalCells.value; i++) {
    const p = i * 4
    const alpha = pixels[p + 3]
    const rgb = alpha === 0 ? [255, 255, 255] : [pixels[p], pixels[p + 1], pixels[p + 2]]
    next[i] = nearestColor(rgb)
  }
  cells.value = next
  nextTick(drawBoard)
}

function drawBoard() {
  const canvas = boardCanvas.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  const size = cellSize.value
  const width = cols.value * size
  const height = rows.value * size
  const dpr = window.devicePixelRatio || 1
  canvas.width = Math.max(1, Math.round(width * dpr))
  canvas.height = Math.max(1, Math.round(height * dpr))
  canvas.style.width = width + 'px'
  canvas.style.height = height + 'px'
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, width, height)

  for (let row = 0; row < rows.value; row++) {
    for (let col = 0; col < cols.value; col++) {
      const id = cells.value[row * cols.value + col] || palette[0].id
      const color = paletteMap.get(id) || palette[0]
      const x = col * size
      const y = row * size
      ctx.fillStyle = color.hex
      ctx.fillRect(x, y, size, size)
      if (activeTab.value !== 'iron') {
        ctx.strokeStyle = 'rgba(60, 55, 58, .25)'
        ctx.lineWidth = 0.65
        ctx.strokeRect(x + .25, y + .25, size - .5, size - .5)
        if (showCodes.value && size >= 10) {
          ctx.fillStyle = getContrastText(color.rgb)
          ctx.font = `600 ${Math.max(5, Math.min(8, size * .42))}px Arial`
          ctx.textAlign = 'center'
          ctx.textBaseline = 'middle'
          ctx.fillText(id, x + size / 2, y + size / 2)
        }
      }
    }
  }
}

function getContrastText(rgb) {
  const luminance = (0.299 * rgb[0] + 0.587 * rgb[1] + 0.114 * rgb[2]) / 255
  return luminance > 0.63 ? '#241f21' : '#ffffff'
}

function handleCanvasPointer(event) {
  if (activeTab.value !== 'edit') return
  const canvas = boardCanvas.value
  const rect = canvas.getBoundingClientRect()
  const col = Math.floor((event.clientX - rect.left) / cellSize.value)
  const row = Math.floor((event.clientY - rect.top) / cellSize.value)
  if (col < 0 || row < 0 || col >= cols.value || row >= rows.value) return
  cells.value[row * cols.value + col] = selectedColor.value.id
  drawBoard()
}

function selectColor(color) {
  selectedColor.value = color
}

function resetPattern() {
  imageElement = null
  sourceImage.value = ''
  if (fileInput.value) fileInput.value.value = ''
  createBlankPattern()
}

function startResize(event) {
  isResizing.value = true
  resizeStart = { x: event.clientX, y: event.clientY, cols: cols.value, rows: rows.value }
  window.addEventListener('pointermove', handleResize)
  window.addEventListener('pointerup', stopResize, { once: true })
}
function handleResize(event) {
  if (!isResizing.value || !resizeStart) return
  const deltaX = event.clientX - resizeStart.x
  const deltaY = event.clientY - resizeStart.y
  const nextCols = Math.round(resizeStart.cols + deltaX / 14)
  const nextRows = Math.round(resizeStart.rows + deltaY / 14)
  if (Math.abs(nextCols - cols.value) >= 1) setCols(nextCols)
  if (Math.abs(nextRows - rows.value) >= 1) setRows(nextRows)
}
function stopResize() {
  isResizing.value = false
  resizeStart = null
  window.removeEventListener('pointermove', handleResize)
}

function exportPng() {
  const canvas = boardCanvas.value
  if (!canvas) return
  const link = document.createElement('a')
  link.download = `mard-bead-${cols.value}x${rows.value}-${activeTab.value}.png`
  link.href = canvas.toDataURL('image/png')
  link.click()
}

watch([activeTab, showCodes, zoom, showCoordinates, cols, rows], () => nextTick(drawBoard))

onMounted(() => {
  createBlankPattern()
  window.addEventListener('resize', drawBoard)
})
onBeforeUnmount(() => {
  window.removeEventListener('resize', drawBoard)
  window.removeEventListener('pointermove', handleResize)
})
</script>
