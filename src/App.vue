<template>
  <div class="app-shell">
    <header class="topbar">
      <div class="brand">
        <div class="brand-mark"><img src="/images/logo.png" alt="拼豆豆 Logo" /></div>
        <div>
          <h1>拼豆豆 拼豆图纸生成器</h1>
          <p>{{ cols }} × {{ rows }} 格 · {{ placedBeadCount.toLocaleString() }} 颗已上色 · {{ palette.length }} 种 MARD 颜色</p>
        </div>
      </div>
      <div class="top-actions">
        <label class="button secondary upload-btn">
          <span>＋</span> 导入图片
          <input ref="fileInput" type="file" accept="image/*" @change="handleFile" />
        </label>
        <button class="button secondary export-png-btn" @click="exportPng">⇩ 导出 PNG</button>
        <button class="button secondary export-pdf-btn" @click="exportPdf">▣ 导出 PDF</button>
        <button class="button primary" @click="resetPattern">↻ 重置</button>
      </div>
    </header>

    <section class="toolbar-card">
      <div class="toolbar-main">
        <div class="field-group">
          <label>拼豆盘宽度</label>
          <div class="number-control">
            <button @click="setCols(cols - 1)">−</button>
            <input v-model.number="colsInput" type="number" min="8" max="200" @change="setCols(colsInput)" />
            <button @click="setCols(cols + 1)">＋</button>
          </div>
        </div>
        <div class="field-group">
          <label>拼豆盘高度</label>
          <div class="number-control">
            <button @click="setRows(rows - 1)">−</button>
            <input v-model.number="rowsInput" type="number" min="8" max="200" @change="setRows(rowsInput)" />
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
            <p>直接在空白底板上编辑拼豆图案。</p>
          </div>
        </div>

        <div class="source-section">
          <div class="section-heading">
            <div>
              <span class="section-kicker">参考图片</span>
              <strong>{{ sourceImage ? '当前导入图片' : '暂无参考图片' }}</strong>
            </div>
            <label class="mini-upload">
              <span>＋ 导入</span>
              <input type="file" accept="image/*" @change="handleFile" />
            </label>
          </div>
          <div class="source-card">
            <img v-if="sourceImage" :src="sourceImage" alt="参考图片" />
            <div v-else class="empty-source">
              <div class="empty-icon">▧</div>
              <strong>还没有导入图片</strong>
              <span>可以直接在中间空白底板上创作</span>
            </div>
          </div>
        </div>

        <div class="board-summary-card">
          <div class="summary-row"><span>底板颜色</span><strong>{{ selectedBaseColorLabel }}</strong></div>
          <div class="summary-row"><span>已使用拼豆</span><strong>{{ placedBeadCount }} 颗</strong></div>
          <div class="summary-row"><span>使用颜色</span><strong>{{ usedColorCount }} 种</strong></div>
        </div>

        <div class="option-row">
          <label class="check"><input v-model="showCoordinates" type="checkbox" /> <span>显示坐标</span></label>
          <label class="check"><input v-model="showCodes" type="checkbox" /> <span>显示格内色号</span></label>
        </div>

        <div class="usage-section">
          <div class="usage-title">颜色使用数量</div>
          <div v-if="usedColors.length" class="usage-list">
            <div v-for="item in usedColors" :key="item.id" class="usage-item">
              <span class="usage-color" :style="{ background: item.hex }"></span>
              <span class="usage-id">{{ item.id }}</span>
              <strong>{{ item.count }} 颗</strong>
            </div>
          </div>
          <div v-else class="usage-empty">还没有使用任何拼豆颜色</div>
        </div>

        <div class="range-group">
          <div class="range-head"><label>画布大小</label><strong>{{ zoom }}%</strong></div>
          <input v-model.number="zoom" type="range" min="50" max="180" />
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
            <span>{{ activeTab === 'edit' ? '点击格子放置拼豆，右侧可以切换底板颜色或 MARD 调色盘' : activeTab === 'iron' ? '去除格线与色号，只保留颜色区块' : '显示拼豆位置、坐标与 MARD 色号' }}</span>
          </div>
          <div class="zoom-controls">
            <button @click="zoom = Math.max(50, zoom - 10)">−</button>
            <span>{{ zoom }}%</span>
            <button @click="zoom = Math.min(180, zoom + 10)">＋</button>
            <button class="fit" @click="zoom = 100">适配画布</button>
            <span class="toolbar-divider"></span>
            <button class="history-button" :disabled="!canUndo" title="撤销 (Ctrl/Cmd + Z)" @click="undo">↶</button>
            <button class="history-button" :disabled="!canRedo" title="重做 (Ctrl/Cmd + Shift + Z)" @click="redo">↷</button>
          </div>
        </div>

        <div
          ref="bottomHorizontalScroll"
          class="canvas-horizontal-scroll"
          aria-label="画布水平滚动条"
          @scroll="syncBoardHorizontalScroll"
        >
          <div
            class="canvas-horizontal-scroll-spacer"
            :style="{ width: horizontalScrollContentWidth + 'px' }"
          ></div>
        </div>

        <div ref="boardViewport" class="board-viewport" @scroll="syncBottomHorizontalScroll">
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

        </div>

      </section>

      <aside v-if="activeTab === 'edit'" class="palette-panel">
        <div class="palette-tabs">
          <button :class="{ active: paletteMode === 'board' }" @click="paletteMode = 'board'">底板颜色</button>
          <button :class="{ active: paletteMode === 'beads' }" @click="paletteMode = 'beads'">调色盘</button>
        </div>

        <template v-if="paletteMode === 'board'">
          <div class="palette-header">
            <div>
              <h2>底板颜色</h2>
              <p>选择拼豆底板颜色</p>
            </div>
            <div class="selected-color board-color-preview" :class="{ 'transparent-swatch': baseColor === 'transparent' }" :style="baseColorStyle" :title="selectedBaseColorLabel"></div>
          </div>
          <div class="palette-search">
            <span>⌕</span>
            <input v-model="boardColorQuery" placeholder="搜索底板颜色，例如 A1" />
          </div>
          <div class="palette-grid board-color-grid">
            <button
              v-for="color in filteredBoardColors"
              :key="color.id"
              :class="[
                'swatch',
                'board-color-option',
                {
                  selected: baseColor === color.id,
                  'transparent-swatch': color.id === 'transparent'
                }
              ]"
              :style="color.id === 'transparent' ? {} : { background: color.hex }"
              :title="color.label"
              @click="selectBaseColor(color.id)"
            >
              <span>{{ color.label }}</span>
            </button>
          </div>
        </template>

        <template v-else>
          <div class="palette-header">
            <div>
              <h2>调色盘</h2>
              <p>MARD 221 色 + 透明</p>
            </div>
            <div class="selected-color" :class="{ 'transparent-swatch': selectedColor.id === 'transparent' }" :style="selectedColor.id === 'transparent' ? {} : { background: selectedColor.hex }" :title="selectedColor.id"></div>
          </div>
          <div class="palette-search">
            <span>⌕</span>
            <input v-model="paletteQuery" placeholder="搜索色号，例如 A1" />
          </div>
          <div class="palette-grid">
            <button v-for="color in filteredPalette" :key="color.id" :class="['swatch', { selected: color.id === selectedColor.id }]" :style="{ background: color.hex }" :title="`${color.id} · ${color.hex}`" @click="selectColor(color)">
              <span>{{ color.id === 'transparent' ? '透明' : color.id }}</span>
            </button>
          </div>
        </template>
      </aside>
      <aside v-else class="info-panel">
        <div class="info-card">
          <div class="mini-preview">
            <canvas ref="miniPreviewCanvas" class="mini-preview-canvas"></canvas>
          </div>
          <strong>当前图纸</strong>
          <p>{{ cols }} × {{ rows }} 格 · 底板：{{ selectedBaseColorLabel }}</p>
        </div>
        <div class="count-card">
          <span>已使用拼豆</span>
          <strong>{{ placedBeadCount }}</strong>
          <small>颗</small>
        </div>
        <div class="usage-section info-usage">
          <div class="usage-title">颜色使用数量</div>
          <div v-if="usedColors.length" class="usage-list">
            <div v-for="item in usedColors" :key="item.id" class="usage-item">
              <span class="usage-color" :style="{ background: item.hex }"></span>
              <span class="usage-id">{{ item.id }}</span>
              <strong>{{ item.count }} 颗</strong>
            </div>
          </div>
          <div v-else class="usage-empty">还没有使用任何拼豆颜色</div>
        </div>
      </aside>
    </main>
  </div>
</template>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { MARD_PALETTE } from './palette'

const palette = MARD_PALETTE.map(c => ({ ...c, hex: rgbToHex(c.rgb) }))
const transparentColor = { id: 'transparent', label: '透明', hex: null, rgb: null }
const paletteWithTransparent = [transparentColor, ...palette]
const paletteMap = new Map(palette.map(c => [c.id, c]))

const presets = [
  { label: '29 × 29', w: 29, h: 29 },
  { label: '32 × 32', w: 32, h: 32 },
  { label: '52 × 36', w: 52, h: 36 },
  { label: '52 × 52', w: 52, h: 52 },
  { label: '64 × 64', w: 64, h: 64 },
  { label: '80 × 80', w: 80, h: 80 },
  { label: '100 × 100', w: 100, h: 100 },
  { label: '128 × 128', w: 128, h: 128 },
  { label: '200 × 200', w: 200, h: 200 }
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
const activeTab = ref('edit')
const paletteMode = ref('beads')
const baseColor = ref('transparent')
const sourceImage = ref('')
const showCoordinates = ref(true)
const showCodes = ref(false)
const zoom = ref(108)
const paletteQuery = ref('')
const boardColorQuery = ref('')
const selectedColor = ref(palette[0])
const cells = ref([])
const boardColors = [
  { id: 'transparent', label: '透明', hex: null, rgb: null },
  ...palette.map(color => ({ ...color, label: color.id }))
]
const boardColorMap = new Map(boardColors.map(color => [color.id, color]))
const BOARD_CELL_PREFIX = 'board:'
const fileInput = ref(null)
const boardCanvas = ref(null)
const miniPreviewCanvas = ref(null)
const boardViewport = ref(null)
const bottomHorizontalScroll = ref(null)
const isResizing = ref(false)
const undoStack = ref([])
const redoStack = ref([])
const isRestoringHistory = ref(false)
let resizeStart = null
let imageElement = null

const MAX_HISTORY = 60

const cellSize = computed(() => Math.max(10, 13 * (zoom.value / 100)))
const canvasPixelWidth = computed(() => Math.round(cols.value * cellSize.value))
const canvasPixelHeight = computed(() => Math.round(rows.value * cellSize.value))
const boardWidthPx = computed(() => canvasPixelWidth.value + 28)
const boardHeightPx = computed(() => canvasPixelHeight.value + 28)

// Match the viewport's horizontal scroll range, including its padding and board margin.
const horizontalScrollContentWidth = computed(() => boardWidthPx.value + 56)
const totalCells = computed(() => cols.value * rows.value)
const activeTabLabel = computed(() => tabs.find(t => t.id === activeTab.value)?.label)
const filteredPalette = computed(() => {
  const q = paletteQuery.value.trim().toLowerCase()
  if (!q) return paletteWithTransparent
  if ('透明'.includes(q) || 'transparent'.includes(q)) return [transparentColor]
  return palette.filter(c => c.id.toLowerCase().includes(q) || c.hex.toLowerCase().includes(q))
})
const filteredBoardColors = computed(() => {
  const q = boardColorQuery.value.trim().toLowerCase()
  if (!q) return boardColors
  if ('透明'.includes(q) || 'transparent'.includes(q)) return [transparentColor]
  return boardColors.filter(c =>
    c.id.toLowerCase().includes(q) ||
    c.label.toLowerCase().includes(q) ||
    (c.hex && c.hex.toLowerCase().includes(q))
  )
})
const usedBeadCells = computed(() => cells.value.filter(id => id && !isBoardCell(id)))
const usedColorCount = computed(() => new Set(usedBeadCells.value).size)
const placedBeadCount = computed(() => usedBeadCells.value.length)
const usedColors = computed(() => {
  const counts = new Map()
  for (const id of usedBeadCells.value) {
    counts.set(id, (counts.get(id) || 0) + 1)
  }
  return [...counts.entries()]
    .map(([id, count]) => {
      const color = paletteMap.get(id)
      return { id, count, hex: color?.hex || '#fff' }
    })
    .sort((a, b) => b.count - a.count || a.id.localeCompare(b.id))
})
const selectedBaseColor = computed(() => boardColors.find(c => c.id === baseColor.value) || boardColors[0])
const selectedBaseColorLabel = computed(() => selectedBaseColor.value.label)
const baseColorStyle = computed(() => {
  if (baseColor.value === 'transparent') {
    return { background: 'transparent', backgroundImage: 'linear-gradient(45deg,#eee 25%,transparent 25%),linear-gradient(-45deg,#eee 25%,transparent 25%),linear-gradient(45deg,transparent 75%,#eee 75%),linear-gradient(-45deg,transparent 75%,#eee 75%)', backgroundSize: '8px 8px', backgroundPosition: '0 0,0 4px,4px -4px,-4px 0' }
  }
  return { background: selectedBaseColor.value.hex }
})
const basePreviewStyle = computed(() => baseColorStyle.value)

function rgbToHex(rgb) {
  return '#' + rgb.map(v => Number(v).toString(16).padStart(2, '0')).join('')
}

function isBoardCell(value) {
  return typeof value === 'string' && value.startsWith(BOARD_CELL_PREFIX)
}

function getBoardCellId(value) {
  return isBoardCell(value) ? value.slice(BOARD_CELL_PREFIX.length) : null
}

function getCellColor(value) {
  if (!value) return null

  // A board-painted cell is intentionally stored as a board-cell marker,
  // not as the selected MARD color. Its actual color always follows the
  // current baseColor, so changing the base color updates every board cell.
  if (isBoardCell(value)) {
    return selectedBaseColor.value
  }

  return paletteMap.get(value) || null
}

function getCellCode(value) {
  const color = getCellColor(value)
  return color?.id || null
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

const canUndo = computed(() => undoStack.value.length > 0)
const canRedo = computed(() => redoStack.value.length > 0)

function cloneCells() {
  return cells.value.slice()
}

function pushHistory() {
  if (isRestoringHistory.value) return

  undoStack.value.push({
    cells: cloneCells(),
    cols: cols.value,
    rows: rows.value
  })

  if (undoStack.value.length > MAX_HISTORY) {
    undoStack.value.shift()
  }

  redoStack.value = []
}

function restoreSnapshot(snapshot) {
  isRestoringHistory.value = true
  cols.value = snapshot.cols
  rows.value = snapshot.rows
  colsInput.value = snapshot.cols
  rowsInput.value = snapshot.rows
  cells.value = snapshot.cells.slice()
  isRestoringHistory.value = false
  nextTick(drawBoard)
}

function undo() {
  if (!canUndo.value) return

  redoStack.value.push({
    cells: cloneCells(),
    cols: cols.value,
    rows: rows.value
  })

  restoreSnapshot(undoStack.value.pop())
}

function redo() {
  if (!canRedo.value) return

  undoStack.value.push({
    cells: cloneCells(),
    cols: cols.value,
    rows: rows.value
  })

  restoreSnapshot(redoStack.value.pop())
}

function createBlankPattern() {
  cells.value = Array.from({ length: totalCells.value }, () => null)
  drawBoard()
}

/**
 * Resize the grid while keeping the current bead pattern.
 *
 * `cells` is the actual editable pattern, so resizing must work from the
 * current cells instead of converting `imageElement` again. This means
 * colors placed by the user after importing an image are preserved.
 *
 * If the new grid is larger, existing cells stay at the same coordinates
 * and newly added cells are empty. If the new grid is smaller, only cells
 * outside the new boundary are cropped.
 */
function resizeGridPreservingCells(newCols, newRows) {
  const oldCols = cols.value
  const oldRows = rows.value
  const oldCells = cells.value.slice()

  const nextCells = Array.from({ length: newCols * newRows }, () => null)
  const copyCols = Math.min(oldCols, newCols)
  const copyRows = Math.min(oldRows, newRows)

  for (let row = 0; row < copyRows; row++) {
    for (let col = 0; col < copyCols; col++) {
      nextCells[row * newCols + col] = oldCells[row * oldCols + col] ?? null
    }
  }

  cols.value = newCols
  rows.value = newRows
  colsInput.value = newCols
  rowsInput.value = newRows
  cells.value = nextCells

  nextTick(drawBoard)
}

function setCols(value, recordHistory = true) {
  const next = Math.max(8, Math.min(200, Number(value) || 52))
  if (next === cols.value) return

  if (recordHistory) pushHistory()
  resizeGridPreservingCells(next, rows.value)
}

function setRows(value, recordHistory = true) {
  const next = Math.max(8, Math.min(200, Number(value) || 36))
  if (next === rows.value) return

  if (recordHistory) pushHistory()
  resizeGridPreservingCells(cols.value, next)
}

function applyPreset(preset) {
  const nextCols = Math.max(8, Math.min(200, Number(preset.w) || cols.value))
  const nextRows = Math.max(8, Math.min(200, Number(preset.h) || rows.value))

  if (nextCols === cols.value && nextRows === rows.value) return

  pushHistory()
  resizeGridPreservingCells(nextCols, nextRows)
}

// Kept for compatibility with existing callers. Resizing itself no longer
// calls this function, because regenerating from the source image would
// overwrite manual edits.
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

function drawMiniPreview() {
  const canvas = miniPreviewCanvas.value
  if (!canvas) return

  const width = Math.max(1, canvas.clientWidth)
  const height = Math.max(1, canvas.clientHeight)
  const dpr = window.devicePixelRatio || 1

  canvas.width = Math.max(1, Math.round(width * dpr))
  canvas.height = Math.max(1, Math.round(height * dpr))
  canvas.style.width = width + 'px'
  canvas.style.height = height + 'px'

  const ctx = canvas.getContext('2d')
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, width, height)

  const cellWidth = width / Math.max(1, cols.value)
  const cellHeight = height / Math.max(1, rows.value)

  for (let row = 0; row < rows.value; row++) {
    for (let col = 0; col < cols.value; col++) {
      const id = cells.value[row * cols.value + col]
      const color = getCellColor(id)

      if (color?.hex) {
        ctx.fillStyle = color.hex
        ctx.fillRect(col * cellWidth, row * cellHeight, cellWidth + 0.5, cellHeight + 0.5)
      } else if (baseColor.value !== 'transparent') {
        ctx.fillStyle = selectedBaseColor.value.hex
        ctx.fillRect(col * cellWidth, row * cellHeight, cellWidth + 0.5, cellHeight + 0.5)
      }
    }
  }
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
      const id = cells.value[row * cols.value + col]
      const color = getCellColor(id)
      const isBoardCellValue = isBoardCell(id)
      const x = col * size
      const y = row * size
      if (color?.hex) {
        ctx.fillStyle = color.hex
        ctx.fillRect(x, y, size, size)
      } else if (!isBoardCellValue && baseColor.value !== 'transparent') {
        ctx.fillStyle = selectedBaseColor.value.hex
        ctx.fillRect(x, y, size, size)
      }
      if (activeTab.value !== 'iron') {
        // User-painted cells keep their grid border.
        // Base cells only show grid lines while the base itself is transparent.
        if (color || isBoardCellValue || baseColor.value === 'transparent') {
          ctx.strokeStyle = 'rgba(60, 55, 58, .25)'
          ctx.lineWidth = 0.65
          ctx.strokeRect(x + .25, y + .25, size - .5, size - .5)
        }
        if (showCodes.value && size >= 10 && color) {
          ctx.fillStyle = getContrastText(color.rgb)
          ctx.font = `600 ${Math.max(5, Math.min(8, size * .42))}px Arial`
          ctx.textAlign = 'center'
          ctx.textBaseline = 'middle'
          ctx.fillText(getCellCode(id), x + size / 2, y + size / 2)
        }
      }
    }
  }

  drawMiniPreview()
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
  const index = row * cols.value + col
  const nextValue = paletteMode.value === 'board'
    ? BOARD_CELL_PREFIX
    : selectedColor.value.id === 'transparent'
      ? null
      : selectedColor.value.id

  if (cells.value[index] === nextValue) return

  pushHistory()
  cells.value[index] = nextValue
  drawBoard()
}

function selectColor(color) {
  selectedColor.value = color
  paletteMode.value = 'beads'
}

function selectBaseColor(colorId) {
  baseColor.value = colorId
  paletteMode.value = 'board'
  nextTick(drawBoard)
}

function resetPattern() {
  if (!window.confirm('确定要重置当前图纸吗？当前编辑内容将被清除。')) return

  imageElement = null
  sourceImage.value = ''
  if (fileInput.value) fileInput.value.value = ''
  createBlankPattern()
}

function startResize(event) {
  isResizing.value = true
  resizeStart = { x: event.clientX, y: event.clientY, cols: cols.value, rows: rows.value }

  // One drag gesture is one history entry.
  pushHistory()

  window.addEventListener('pointermove', handleResize)
  window.addEventListener('pointerup', stopResize, { once: true })
}

function handleResize(event) {
  if (!isResizing.value || !resizeStart) return

  const deltaX = event.clientX - resizeStart.x
  const deltaY = event.clientY - resizeStart.y
  const nextCols = Math.max(8, Math.min(200, Math.round(resizeStart.cols + deltaX / 14)))
  const nextRows = Math.max(8, Math.min(200, Math.round(resizeStart.rows + deltaY / 14)))

  if (nextCols !== cols.value || nextRows !== rows.value) {
    resizeGridPreservingCells(nextCols, nextRows)
  }
}
function stopResize() {
  isResizing.value = false
  resizeStart = null
  window.removeEventListener('pointermove', handleResize)
}

// Keep the dedicated bottom scrollbar and the canvas viewport in sync.
function syncBoardHorizontalScroll() {
  if (!boardViewport.value || !bottomHorizontalScroll.value) return
  if (boardViewport.value.scrollLeft !== bottomHorizontalScroll.value.scrollLeft) {
    boardViewport.value.scrollLeft = bottomHorizontalScroll.value.scrollLeft
  }
}

function syncBottomHorizontalScroll() {
  if (!boardViewport.value || !bottomHorizontalScroll.value) return
  if (bottomHorizontalScroll.value.scrollLeft !== boardViewport.value.scrollLeft) {
    bottomHorizontalScroll.value.scrollLeft = boardViewport.value.scrollLeft
  }
}

function refreshHorizontalScroll() {
  nextTick(() => {
    if (!boardViewport.value || !bottomHorizontalScroll.value) return
    bottomHorizontalScroll.value.scrollLeft = boardViewport.value.scrollLeft
  })
}

function roundedRect(ctx, x, y, width, height, radius) {
  const r = Math.min(radius, width / 2, height / 2)
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.arcTo(x + width, y, x + width, y + height, r)
  ctx.arcTo(x + width, y + height, x, y + height, r)
  ctx.arcTo(x, y + height, x, y, r)
  ctx.arcTo(x, y, x + width, y, r)
  ctx.closePath()
}

function drawExportLegend(ctx, items, x, y, width) {
  const columns = width >= 900 ? 4 : 3
  const gap = 10
  const itemWidth = (width - gap * (columns - 1)) / columns
  const itemHeight = 42

  items.forEach((item, index) => {
    const column = index % columns
    const row = Math.floor(index / columns)
    const itemX = x + column * (itemWidth + gap)
    const itemY = y + row * (itemHeight + gap)

    roundedRect(ctx, itemX, itemY, itemWidth, itemHeight, 10)
    ctx.fillStyle = '#ffffff'
    ctx.fill()
    ctx.strokeStyle = '#eee6e2'
    ctx.lineWidth = 1
    ctx.stroke()

    roundedRect(ctx, itemX + 9, itemY + 9, 24, 24, 6)
    ctx.fillStyle = item.hex
    ctx.fill()
    ctx.strokeStyle = 'rgba(55,45,42,.18)'
    ctx.stroke()

    ctx.fillStyle = '#3d3431'
    ctx.font = '700 13px Arial'
    ctx.textAlign = 'left'
    ctx.textBaseline = 'middle'
    ctx.fillText(item.id, itemX + 42, itemY + 17)

    ctx.fillStyle = '#9a8e89'
    ctx.font = '500 11px Arial'
    ctx.fillText(`${item.count} 颗`, itemX + 42, itemY + 30)
  })

  return Math.ceil(items.length / columns) * (itemHeight + gap)
}

function buildExportCanvas() {
  const boardSize = Math.max(12, Math.min(28, 22 * (zoom.value / 100)))
  const boardWidth = cols.value * boardSize
  const boardHeight = rows.value * boardSize
  const pageWidth = Math.max(980, boardWidth + 120)
  const headerHeight = 132
  // Add a comfortable gap between the summary chips and the board.
  // This prevents the chips from visually touching/overlapping the board card.
  const boardTopGap = 38
  const boardCardPadding = 24
  const legendHeaderHeight = 58
  const legendItemWidth = pageWidth - 96
  const legendColumns = legendItemWidth >= 900 ? 4 : 3
  const legendItemHeight = 42
  const legendGap = 10
  const legendRows = Math.max(1, Math.ceil(usedColors.value.length / legendColumns))
  const legendHeight = legendHeaderHeight + 24 + legendRows * (legendItemHeight + legendGap) + 24
  const footerHeight = 46
  const height = headerHeight + boardTopGap + boardHeight + boardCardPadding * 2 + 24 + legendHeight + footerHeight
  const canvas = document.createElement('canvas')
  const dpr = 2
  canvas.width = Math.ceil(pageWidth * dpr)
  canvas.height = Math.ceil(height * dpr)
  const ctx = canvas.getContext('2d')
  ctx.scale(dpr, dpr)

  // Soft warm background.
  const bg = ctx.createLinearGradient(0, 0, pageWidth, height)
  bg.addColorStop(0, '#fffaf7')
  bg.addColorStop(1, '#f4f7f8')
  ctx.fillStyle = bg
  ctx.fillRect(0, 0, pageWidth, height)

  // Header.
  const margin = 48
  ctx.fillStyle = '#2f2927'
  ctx.textAlign = 'left'
  ctx.textBaseline = 'alphabetic'
  ctx.font = '800 30px Arial'
  ctx.fillText('拼豆图纸', margin, 46)
  ctx.fillStyle = '#9a8e89'
  ctx.font = '500 14px Arial'
  ctx.fillText('MARD Bead Pattern', margin, 70)

  const chips = [
    `${cols.value} × ${rows.value} 格`,
    `${placedBeadCount.value.toLocaleString()} 颗拼豆`,
    `${usedColorCount.value} 种颜色`
  ]
  let chipX = margin
  const chipY = 88
  chips.forEach((label, index) => {
    ctx.font = '700 11px Arial'
    const chipWidth = ctx.measureText(label).width + 24
    roundedRect(ctx, chipX, chipY, chipWidth, 25, 12)
    ctx.fillStyle = index === 0 ? '#fff0ea' : '#ffffff'
    ctx.fill()
    ctx.strokeStyle = '#eaded9'
    ctx.lineWidth = 1
    ctx.stroke()
    ctx.fillStyle = index === 0 ? '#c85e3a' : '#625650'
    ctx.textBaseline = 'middle'
    ctx.fillText(label, chipX + 12, chipY + 13)
    chipX += chipWidth + 8
  })

  const boardX = (pageWidth - boardWidth) / 2
  const boardY = headerHeight + boardTopGap
  const cardX = Math.max(20, boardX - boardCardPadding)
  const cardY = boardY - boardCardPadding
  const cardW = boardWidth + boardCardPadding * 2
  const cardH = boardHeight + boardCardPadding * 2

  // Board card.
  roundedRect(ctx, cardX, cardY, cardW, cardH, 18)
  ctx.fillStyle = '#ffffff'
  ctx.fill()
  ctx.strokeStyle = '#e8dfdb'
  ctx.lineWidth = 1
  ctx.stroke()

  // Board cells.
  for (let row = 0; row < rows.value; row++) {
    for (let col = 0; col < cols.value; col++) {
      const id = cells.value[row * cols.value + col]
      const color = getCellColor(id)
      const isBoardCellValue = isBoardCell(id)
      const x = boardX + col * boardSize
      const y = boardY + row * boardSize
      if (color?.hex) {
        ctx.fillStyle = color.hex
        ctx.fillRect(x, y, boardSize, boardSize)
      } else if (!isBoardCellValue && baseColor.value !== 'transparent') {
        ctx.fillStyle = selectedBaseColor.value.hex
        ctx.fillRect(x, y, boardSize, boardSize)
      } else {
        ctx.fillStyle = '#ffffff'
        ctx.fillRect(x, y, boardSize, boardSize)
      }

      // Transparent base cells and board-color-painted cells keep a visible grid.
      if (color || isBoardCellValue || baseColor.value === 'transparent') {
        ctx.strokeStyle = 'rgba(65,60,58,.24)'
        ctx.lineWidth = Math.max(.55, boardSize * .035)
        ctx.strokeRect(x + .2, y + .2, boardSize - .4, boardSize - .4)
      }

      if (showCodes.value && color && boardSize >= 12) {
        ctx.fillStyle = getContrastText(color.rgb)
        ctx.font = `700 ${Math.max(7, Math.min(12, boardSize * .42))}px Arial`
        ctx.textAlign = 'center'
        ctx.textBaseline = 'middle'
        ctx.fillText(getCellCode(id), x + boardSize / 2, y + boardSize / 2)
      }
    }
  }

  // Legend card.
  const legendY = cardY + cardH + 22
  const legendX = margin
  const legendW = pageWidth - margin * 2
  roundedRect(ctx, legendX, legendY, legendW, legendHeight, 18)
  ctx.fillStyle = '#ffffff'
  ctx.fill()
  ctx.strokeStyle = '#e8dfdb'
  ctx.stroke()

  ctx.fillStyle = '#3c3330'
  ctx.textAlign = 'left'
  ctx.textBaseline = 'alphabetic'
  ctx.font = '800 17px Arial'
  ctx.fillText('颜色使用数量', legendX + 18, legendY + 27)
  ctx.fillStyle = '#a09590'
  ctx.font = '500 11px Arial'
  ctx.fillText(`共使用 ${usedColorCount.value} 种颜色 · ${placedBeadCount.value.toLocaleString()} 颗拼豆`, legendX + 18, legendY + 45)

  if (usedColors.value.length) {
    drawExportLegend(ctx, usedColors.value, legendX + 18, legendY + legendHeaderHeight, legendW - 36)
  } else {
    ctx.fillStyle = '#aaa09b'
    ctx.font = '500 12px Arial'
    ctx.fillText('尚未放置拼豆', legendX + 18, legendY + 80)
  }

  // Footer.
  ctx.fillStyle = '#a79b96'
  ctx.font = '500 10px Arial'
  ctx.textAlign = 'center'
  ctx.fillText(`MARD 拼豆图纸 · ${cols.value} × ${rows.value} · 底板：${selectedBaseColorLabel.value}`, pageWidth / 2, height - 18)

  return canvas
}

function exportPng() {
  if (!window.confirm('确定要导出当前图纸为 PNG 吗？')) return

  const exportCanvas = buildExportCanvas()
  const link = document.createElement('a')
  link.download = `mard-bead-${cols.value}x${rows.value}-pattern.png`
  link.href = exportCanvas.toDataURL('image/png')
  link.click()
}

function base64ToUint8Array(base64) {
  const binary = atob(base64)
  const bytes = new Uint8Array(binary.length)
  for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i)
  return bytes
}

function buildImagePdf(jpegDataUrl, imageWidth, imageHeight) {
  const jpegBytes = base64ToUint8Array(jpegDataUrl.split(',')[1])
  const portrait = imageHeight >= imageWidth
  const pageWidth = portrait ? 595.28 : 841.89
  const pageHeight = portrait ? 841.89 : 595.28
  const margin = 24
  const scale = Math.min((pageWidth - margin * 2) / imageWidth, (pageHeight - margin * 2) / imageHeight)
  const drawWidth = imageWidth * scale
  const drawHeight = imageHeight * scale
  const x = (pageWidth - drawWidth) / 2
  const y = (pageHeight - drawHeight) / 2
  const content = `q\n${drawWidth.toFixed(3)} 0 0 ${drawHeight.toFixed(3)} ${x.toFixed(3)} ${y.toFixed(3)} cm\n/Im0 Do\nQ\n`

  const objects = []
  objects.push('<< /Type /Catalog /Pages 2 0 R >>')
  objects.push('<< /Type /Pages /Kids [3 0 R] /Count 1 >>')
  objects.push(`<< /Type /Page /Parent 2 0 R /MediaBox [0 0 ${pageWidth.toFixed(2)} ${pageHeight.toFixed(2)}] /Resources << /XObject << /Im0 5 0 R >> >> /Contents 4 0 R >>`)
  objects.push(`<< /Length ${content.length} >>\nstream\n${content}endstream`)
  objects.push(`<< /Type /XObject /Subtype /Image /Width ${imageWidth} /Height ${imageHeight} /ColorSpace /DeviceRGB /BitsPerComponent 8 /Filter /DCTDecode /Length ${jpegBytes.length} >>\nstream\n`)

  const chunks = []
  const header = new TextEncoder().encode('%PDF-1.4\n%\xFF\xFF\xFF\xFF\n')
  chunks.push(header)
  const offsets = [0]
  let position = header.length

  for (let i = 0; i < objects.length; i++) {
    offsets.push(position)
    const objectHeader = new TextEncoder().encode(`${i + 1} 0 obj\n`)
    chunks.push(objectHeader)
    position += objectHeader.length

    if (i === 4) {
      const objectBody = new TextEncoder().encode(objects[i])
      chunks.push(objectBody)
      position += objectBody.length
      chunks.push(jpegBytes)
      position += jpegBytes.length
      const tail = new TextEncoder().encode('\nendstream\nendobj\n')
      chunks.push(tail)
      position += tail.length
    } else {
      const objectBody = new TextEncoder().encode(`${objects[i]}\nendobj\n`)
      chunks.push(objectBody)
      position += objectBody.length
    }
  }

  const xrefOffset = position
  let xref = `xref\n0 ${objects.length + 1}\n0000000000 65535 f \n`
  for (let i = 1; i <= objects.length; i++) {
    xref += `${String(offsets[i]).padStart(10, '0')} 00000 n \n`
  }
  xref += `trailer\n<< /Size ${objects.length + 1} /Root 1 0 R >>\nstartxref\n${xrefOffset}\n%%EOF\n`
  chunks.push(new TextEncoder().encode(xref))

  let total = 0
  for (const chunk of chunks) total += chunk.length
  const pdf = new Uint8Array(total)
  let cursor = 0
  for (const chunk of chunks) {
    pdf.set(chunk, cursor)
    cursor += chunk.length
  }
  return pdf
}

function exportPdf() {
  if (!window.confirm('确定要导出当前图纸为 PDF 吗？')) return

  const exportCanvas = buildExportCanvas()
  const jpeg = exportCanvas.toDataURL('image/jpeg', 0.94)
  const pdfBytes = buildImagePdf(jpeg, exportCanvas.width, exportCanvas.height)
  const blob = new Blob([pdfBytes], { type: 'application/pdf' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.download = `mard-bead-${cols.value}x${rows.value}-pattern.pdf`
  link.href = url
  link.click()
  setTimeout(() => URL.revokeObjectURL(url), 1000)
}

watch(
  [activeTab, showCodes, zoom, showCoordinates, cols, rows, baseColor, boardWidthPx],
  () => {
    nextTick(() => {
      drawBoard()
      refreshHorizontalScroll()
    })
  }
)

function handleHistoryShortcut(event) {
  const isMod = event.ctrlKey || event.metaKey
  if (!isMod || event.altKey) return

  if (event.key.toLowerCase() === 'z') {
    event.preventDefault()
    if (event.shiftKey) redo()
    else undo()
  } else if (event.key.toLowerCase() === 'y') {
    event.preventDefault()
    redo()
  }
}

function handleWindowResize() {
  drawBoard()
  refreshHorizontalScroll()
}

onMounted(() => {
  createBlankPattern()
  nextTick(refreshHorizontalScroll)
  window.addEventListener('resize', handleWindowResize)
  window.addEventListener('keydown', handleHistoryShortcut)
})
onBeforeUnmount(() => {
  window.removeEventListener('resize', handleWindowResize)
  window.removeEventListener('pointermove', handleResize)
  window.removeEventListener('keydown', handleHistoryShortcut)
})
</script>
