<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref } from "vue"
import { useRouter } from "vue-router"

import { querySpheroidization } from "../api"
import type { SpheroidizationRecord } from "../api"

const router = useRouter()

// const videoFeed1 = `${videoBaseUrl}/video_feed1`
// const videoFeed2 = `${videoBaseUrl}/video_feed2`
// const videoFeed1 = `http://10.0.0.19:5080/video_feed1`
// const videoFeed2 = `http://10.0.0.19:5080/video_feed2`
const videoFeed1 = `/api/video_feed1`
const videoFeed2 = `/api/video_feed2`


const records = ref<SpheroidizationRecord[]>([])
const loading = ref(true)
const errorMessage = ref("")
const videoErrors = ref([false, false])
const popupRecord = ref<SpheroidizationRecord | null>(null)

const knownRecordIds = new Set<number>()
const popupQueue: SpheroidizationRecord[] = []

let initialRecordsLoaded = false
let isPolling = false
let isActive = true
let pollTimer: ReturnType<typeof setInterval> | null = null
let popupTimer: ReturnType<typeof setTimeout> | null = null

async function loadRecords(silent = false) {
  if (!isActive) {
    return
  }

  if (silent && isPolling) {
    return
  }

  if (silent) {
    isPolling = true
  } else {
    loading.value = true
    errorMessage.value = ""
  }

  try {
    const result = await querySpheroidization({ page: 1, page_size: 5 })

    if (result.code !== 200) {
      if (!silent) {
        errorMessage.value = result.message || "加载失败"
      }
      return
    }

    const items = result.data?.items ?? []
    records.value = items

    if (!initialRecordsLoaded) {
      items.forEach((record) => knownRecordIds.add(record.id))
      initialRecordsLoaded = true
    } else if (silent) {
      const newRecords = items
        .filter((record) => !knownRecordIds.has(record.id))
        .sort((a, b) => a.id - b.id)

      if (newRecords.length > 0) {
        newRecords.forEach((record) => knownRecordIds.add(record.id))
        enqueueResultPopups(newRecords)
      }
    }
  } finally {
    if (silent) {
      isPolling = false
    } else {
      loading.value = false
    }
  }

  await nextTick()
  fitColumnsToContainer()
}

function playResultSound() {
  try {
    const context = new AudioContext()
    const oscillator = context.createOscillator()
    const gain = context.createGain()

    oscillator.type = "sine"
    oscillator.frequency.value = 880
    gain.gain.setValueAtTime(0.0001, context.currentTime)
    gain.gain.exponentialRampToValueAtTime(
      0.18,
      context.currentTime + 0.02,
    )
    gain.gain.exponentialRampToValueAtTime(
      0.0001,
      context.currentTime + 0.35,
    )

    oscillator.connect(gain)
    gain.connect(context.destination)
    oscillator.start()
    oscillator.stop(context.currentTime + 0.4)

    oscillator.addEventListener("ended", () => {
      context.close().catch(() => undefined)
    })

    if (context.state === "suspended") {
      context.resume().catch(() => undefined)
    }
  } catch {
    return
  }
}

function showNextResultPopup() {
  const nextRecord = popupQueue.shift()

  if (!nextRecord) {
    popupRecord.value = null
    return
  }

  popupRecord.value = nextRecord
  playResultSound()

  if (popupTimer) {
    clearTimeout(popupTimer)
  }

  popupTimer = setTimeout(() => {
    popupTimer = null
    showNextResultPopup()
  }, 10000)
}

function enqueueResultPopups(newRecords: SpheroidizationRecord[]) {
  popupQueue.push(...newRecords)

  if (!popupRecord.value) {
    showNextResultPopup()
  }
}

function closeResultPopup() {
  if (popupTimer) {
    clearTimeout(popupTimer)
    popupTimer = null
  }

  showNextResultPopup()
}

const tableRows = computed(() => {
  return [...records.value]
    .sort((a, b) => {
      return (
        new Date(b.detection_time).getTime() -
        new Date(a.detection_time).getTime()
      )
    })
    .slice(0, 5)
})

const latestRecord = computed(() => {
  return tableRows.value[0] ?? null
})

const statusInfo = computed(() => {
  const record = latestRecord.value

  return {
    recordStatus: record
      ? record.processed
        ? "已处理"
        : "待处理"
      : "--",
    spheroidizationTime: record
      ? `${record.standard_spheroidization_time} / ${record.actual_spheroidization_time ?? "--"}`
      : "--",
    detectionTime: record
      ? record.detection_time
      : "--",
    machineResult: record
      ? record.spheroidization_abnormal
        ? "异常"
        : "正常"
      : "--",
    processedBy: record
      ? record.processed_by ?? "--"
      : "--",
  }
})

// 后加的几列在老记录上没有值，空值统一显示成 --
function display(value: string | number | null | undefined) {
  return value === null || value === undefined || value === ""
    ? "--"
    : value
}

function getAbnormalReason(record: SpheroidizationRecord) {
  const reasons: string[] = []

  if (record.spheroidization_abnormal) {
    reasons.push("球化异常")
  }

  return reasons.length > 0
    ? reasons.join("、")
    : "无"
}

function openStandardSettings() {
  router.push("/records/standard")
}

const tableScroll = ref<HTMLElement | null>(null)
const tableEl = ref<HTMLTableElement | null>(null)

// 表头分隔线可拖拽调整列宽（Excel 式），只作用于本页表格。
// 前 9 列是固定宽度；最后一列（报警原因）自动吃掉剩余宽度，
// 它的右边缘始终贴着容器右边，不会因为拖拽其它列而脱开。
const MIN_COLUMN_WIDTH = 40
const LAST_COLUMN_MIN = 72
const LAST_COLUMN_INDEX = 9

const columnWidths = ref<number[]>([
  46, 78, 140, 140, 88, 140, 140, 100, 64, 72,
])

// 表格容器宽度（换算成表格自身像素，抵消页面 125% 缩放）
const containerWidth = ref(0)
// 真实像素 ÷ 表格像素，只在首次测量时确定一次
let zoomFactor = 1

const fixedTotal = computed(() =>
  columnWidths.value
    .slice(0, LAST_COLUMN_INDEX)
    .reduce((sum, width) => sum + width, 0),
)

// 最后一列宽度 = 容器宽度 - 前面所有列的宽度，最小保留 LAST_COLUMN_MIN
const lastColumnWidth = computed(() => {
  const rest = containerWidth.value - fixedTotal.value
  return Math.max(LAST_COLUMN_MIN, Math.round(rest))
})

// 真正写到 colgroup 上的列宽
const effectiveWidths = computed(() => {
  const widths = columnWidths.value.slice(0, LAST_COLUMN_INDEX)
  widths.push(lastColumnWidth.value)
  return widths
})

const totalWidth = computed(() =>
  effectiveWidths.value.reduce((sum, width) => sum + width, 0),
)

let hasFittedColumns = false

function measureZoomFactor() {
  const table = tableEl.value
  const rendered = totalWidth.value

  if (!table || rendered <= 0) {
    return
  }

  const tableWidth = table.getBoundingClientRect().width

  if (tableWidth > 0) {
    zoomFactor = tableWidth / rendered
  }
}

function measureContainerWidth() {
  const scroll = tableScroll.value

  if (!scroll || zoomFactor <= 0) {
    return
  }

  const scrollWidth = scroll.getBoundingClientRect().width

  if (scrollWidth > 0) {
    containerWidth.value = scrollWidth / zoomFactor
  }
}

function handleWindowResize() {
  measureContainerWidth()
}

// 首次渲染时按容器宽度等比铺满（最后一列保留默认宽度）；
// 之后拖拽只改变被拖的那一列，其它列宽度不变。
function fitColumnsToContainer() {
  if (!hasFittedColumns) {
    measureZoomFactor()
  }

  measureContainerWidth()

  if (hasFittedColumns) {
    return
  }

  const defaultLastWidth = columnWidths.value[LAST_COLUMN_INDEX] ?? LAST_COLUMN_MIN
  const available = containerWidth.value - defaultLastWidth

  if (available > 0 && fixedTotal.value > 0) {
    const scale = available / fixedTotal.value

    if (scale > 1) {
      columnWidths.value = columnWidths.value.map((width, index) =>
        index === LAST_COLUMN_INDEX ? width : Math.round(width * scale),
      )
    }
  }

  hasFittedColumns = true
}

interface ResizeState {
  index: number
  startX: number
  startWidth: number
  factor: number
}

let resizeState: ResizeState | null = null

function startResize(index: number, event: MouseEvent) {
  const handle = event.currentTarget as HTMLElement | null
  const header = handle?.parentElement ?? null

  if (!header) {
    return
  }

  const startWidth = columnWidths.value[index] ?? 80
  const measured = header.getBoundingClientRect().width / startWidth
  const factor = measured > 0 ? measured : 1

  resizeState = {
    index,
    startX: event.clientX,
    startWidth,
    factor,
  }

  document.addEventListener("mousemove", onResizeMove)
  document.addEventListener("mouseup", stopResize)
  document.body.style.cursor = "col-resize"
  document.body.style.userSelect = "none"
}

function onResizeMove(event: MouseEvent) {
  if (!resizeState) {
    return
  }

  const delta = (event.clientX - resizeState.startX) / resizeState.factor
  const next = Math.max(
    MIN_COLUMN_WIDTH,
    Math.round(resizeState.startWidth + delta),
  )
  const list = [...columnWidths.value]
  list[resizeState.index] = next
  columnWidths.value = list
}

function stopResize() {
  resizeState = null
  document.removeEventListener("mousemove", onResizeMove)
  document.removeEventListener("mouseup", stopResize)
  document.body.style.cursor = ""
  document.body.style.userSelect = ""
}

onUnmounted(() => {
  isActive = false
  window.removeEventListener("resize", handleWindowResize)
  stopResize()

  if (pollTimer) {
    clearInterval(pollTimer)
    pollTimer = null
  }

  if (popupTimer) {
    clearTimeout(popupTimer)
    popupTimer = null
  }

  popupQueue.length = 0
})

onMounted(() => {
  window.addEventListener("resize", handleWindowResize)
  loadRecords().catch(() => undefined)

  pollTimer = setInterval(() => {
    loadRecords(true).catch(() => undefined)
  }, 3000)
})
</script>

<template>
  <div class="home-wrapper">
    <div class="page-title">
      浇注工位AI智能化系统
    </div>

    <div class="video-wrap">
      <div class="video-box">
        <img
          :src="videoFeed1"
          alt="摄像头1"
          @load="videoErrors[0] = false"
          @error="videoErrors[0] = true"
        />

        <div
          v-if="videoErrors[0]"
          class="video-placeholder"
        >
          等待视频流接入
        </div>

        <span class="cam-label">
          浇注工位 #01
        </span>

        <span
          class="live-dot"
          :class="{ offline: videoErrors[0] }"
        >
          {{ videoErrors[0] ? "未连接" : "视频流" }}
        </span>
      </div>

      <div class="video-box">
        <img
          :src="videoFeed2"
          alt="摄像头2"
          @load="videoErrors[1] = false"
          @error="videoErrors[1] = true"
        />

        <div
          v-if="videoErrors[1]"
          class="video-placeholder"
        >
          等待视频流接入
        </div>

        <span class="cam-label">
          浇注工位 #02
        </span>

        <span
          class="live-dot"
          :class="{ offline: videoErrors[1] }"
        >
          {{ videoErrors[1] ? "未连接" : "视频流" }}
        </span>
      </div>
    </div>

    <button
      class="btn-bar standard-switch"
      type="button"
      @click="openStandardSettings"
    >
      切换标准
    </button>

    <div
      v-if="loading"
      class="empty-state"
    >
      正在加载数据...
    </div>

    <div
      v-else-if="errorMessage"
      class="empty-state"
    >
      {{ errorMessage }}
    </div>

    <div
      v-else
      ref="tableScroll"
      class="table-scroll"
    >
      <table
        ref="tableEl"
        :style="{ width: totalWidth + 'px' }"
      >
        <colgroup>
          <col
            v-for="(width, index) in effectiveWidths"
            :key="index"
            :style="{ width: width + 'px' }"
          />
        </colgroup>

        <thead>
          <tr>
            <th>
              序号
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(0, $event)"
              />
            </th>
            <th>
              铁水重量
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(1, $event)"
              />
            </th>
            <th>
              进线开始时间
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(2, $event)"
              />
            </th>
            <th>
              进线结束时间
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(3, $event)"
              />
            </th>
            <th>
              进线持续时间
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(4, $event)"
              />
            </th>
            <th>
              球化开始时间
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(5, $event)"
              />
            </th>
            <th>
              球化结束时间
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(6, $event)"
              />
            </th>
            <th>
              球化持续时长
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(7, $event)"
              />
            </th>
            <th>
              球化结果
              <span
                class="col-resizer"
                @mousedown.prevent="startResize(8, $event)"
              />
            </th>
            <th>
              报警原因
            </th>
          </tr>
        </thead>

        <tbody>
          <tr
            v-for="row in tableRows"
            :key="row.id"
          >
            <td>{{ row.id }}</td>
            <td>{{ display(row.spheroidization_weight) }}</td>
            <td>{{ display(row.spheroidization_wire_entry_start_time) }}</td>
            <td>{{ display(row.spheroidization_wire_entry_end_time) }}</td>
            <td>{{ display(row.spheroidization_wire_entry_total_duration) }}</td>
            <td>{{ display(row.spheroidization_start_time) }}</td>
            <td>{{ display(row.spheroidization_end_time) }}</td>
            <td>{{ display(row.actual_spheroidization_time) }}</td>
            <td>
              <span
                :class="
                  row.spheroidization_abnormal
                    ? 'result-abnormal'
                    : 'result-normal'
                "
              >
                {{
                  row.spheroidization_abnormal
                    ? "异常"
                    : "正常"
                }}
              </span>
            </td>
            <td>{{ getAbnormalReason(row) }}</td>
          </tr>

          <tr v-if="tableRows.length === 0">
            <td
              class="empty-cell"
              colspan="10"
            >
              暂无检测记录
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="info-panel">
      <div class="info-column">
        <div>
          当前记录状态：
          <input
            :value="statusInfo.recordStatus"
            readonly
          />
        </div>

        <div>
          标准/实际球化时间：
          <input
            :value="statusInfo.spheroidizationTime"
            readonly
          />
        </div>

        <div>
          检测时间：
          <input
            :value="statusInfo.detectionTime"
            readonly
          />
        </div>
      </div>

      <div class="info-column">
        <div>
          检测结果：
          <input
            :value="statusInfo.machineResult"
            readonly
          />
        </div>

        <div>
          处理人：
          <input
            :value="statusInfo.processedBy"
            readonly
          />
        </div>
      </div>
    </div>

    <Teleport to="body">
      <Transition name="result-popup">
        <div
          v-if="popupRecord"
          class="result-overlay"
          role="alertdialog"
          aria-modal="true"
          aria-labelledby="result-popup-title"
        >
          <div
            :key="popupRecord.id"
            class="result-popup"
            :class="popupRecord.spheroidization_abnormal ? 'is-ng' : 'is-ok'"
          >
            <button
              class="result-popup-close"
              type="button"
              aria-label="关闭提示"
              @click="closeResultPopup"
            >
              ×
            </button>

            <div
              id="result-popup-title"
              class="result-popup-caption"
            >
              新球化检测记录
            </div>

            <div class="result-popup-value">
              {{ popupRecord.spheroidization_abnormal ? "NG" : "OK" }}
            </div>

            <div class="result-popup-status">
              {{ popupRecord.spheroidization_abnormal ? "球化异常" : "球化正常" }}
            </div>

            <div class="result-popup-detail">
              记录 #{{ popupRecord.id }}
            </div>

            <div class="result-popup-progress" />
          </div>
        </div>
      </Transition>
    </Teleport>
  </div>
</template>

<style scoped>
/* 整页放大到 125%（等同浏览器缩放 125%），只作用于本页面 */
.home-wrapper {
  min-height: calc((100vh - 40px) / 1.25);
  padding: 10px 12px;
  zoom: 1.25;
  background: #f0f4f8;
}

.page-title {
  margin-bottom: 8px;
  color: #003399;
  font-size: 15px;
  font-weight: 700;
  letter-spacing: 2px;
  text-align: center;
}

.video-wrap {
  display: flex;
  gap: 4px;
  margin-bottom: 8px;
}

.video-box {
  position: relative;
  width: 50%;
  aspect-ratio: 16 / 9;
  overflow: hidden;
  background: #1a1a2e;
  border-radius: 4px;
}

.video-box img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: fill;
  background: #0a0a1a;
}

.video-placeholder {
  position: absolute;
  inset: 0;
  display: grid;
  color: #9fb3c8;
  font-size: 12px;
  background: #0a0a1a;
  place-items: center;
}

.cam-label {
  position: absolute;
  bottom: 8px;
  left: 12px;
  padding: 2px 9px;
  color: rgba(255, 255, 255, 0.85);
  font-size: 11px;
  background: rgba(0, 0, 0, 0.5);
  border-radius: 3px;
}

.live-dot {
  position: absolute;
  top: 8px;
  right: 12px;
  color: #00ff88;
  font-size: 11px;
}

.live-dot::before {
  color: #00ff88;
  content: "● ";
}

.live-dot.offline,
.live-dot.offline::before {
  color: #ff9f43;
}

.btn-bar {
  width: 100%;
  margin-bottom: 6px;
  padding: 4px;
  color: #003399;
  font-size: 11px;
  font-weight: 700;
  text-align: center;
  background: #eee;
  border: 1px solid #ddd;
  border-radius: 4px;
  user-select: none;
}

.btn-bar::before {
  font-size: 13px;
  content: "+ ";
}

.btn-bar:hover {
  background: #dde8f5;
  border-color: #4a8af4;
}

.standard-switch {
  width: 100%;
  margin-bottom: 8px;
  padding: 10px;
  font-size: 16px;
  white-space: nowrap;
}

.standard-switch::before {
  content: none;
}

.result-overlay {
  position: fixed;
  inset: 0;
  z-index: 2000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  background: rgba(0, 0, 0, 0.48);
}

.result-popup {
  position: relative;
  width: min(340px, 100%);
  overflow: hidden;
  padding: 28px 32px 30px;
  text-align: center;
  background: #fff;
  border: 3px solid #16803c;
  border-radius: 8px;
  box-shadow: 0 18px 48px rgba(0, 0, 0, 0.32);
}

.result-popup.is-ng {
  border-color: #c62828;
}

.result-popup-caption {
  color: #666;
  font-size: 15px;
  font-weight: 600;
}

.result-popup-value {
  margin: 12px 0 6px;
  color: #16803c;
  font-size: 82px;
  font-weight: 900;
  line-height: 1;
  letter-spacing: 4px;
}

.result-popup.is-ng .result-popup-value {
  color: #c62828;
}

.result-popup-status {
  color: #16803c;
  font-size: 19px;
  font-weight: 700;
  letter-spacing: 2px;
}

.result-popup.is-ng .result-popup-status {
  color: #c62828;
}

.result-popup-detail {
  margin-top: 8px;
  color: #777;
  font-size: 12px;
}

.result-popup-close {
  position: absolute;
  top: 4px;
  right: 6px;
  width: 32px;
  height: 32px;
  padding: 0;
  color: #666;
  font-size: 26px;
  line-height: 1;
  background: transparent;
  border: 0;
  border-radius: 50%;
}

.result-popup-close:hover {
  color: #1a2a3a;
  background: #f0f0f0;
}

.result-popup-progress {
  position: absolute;
  right: 0;
  bottom: 0;
  left: 0;
  height: 4px;
  background: #16803c;
  transform-origin: left center;
  animation: result-countdown 10s linear forwards;
}

.result-popup.is-ng .result-popup-progress {
  background: #c62828;
}

@keyframes result-countdown {
  from {
    transform: scaleX(1);
  }

  to {
    transform: scaleX(0);
  }
}

.result-popup-enter-active,
.result-popup-leave-active {
  transition: opacity 0.18s ease;
}

.result-popup-enter-active .result-popup,
.result-popup-leave-active .result-popup {
  transition: transform 0.18s ease, opacity 0.18s ease;
}

.result-popup-enter-from,
.result-popup-leave-to {
  opacity: 0;
}

.result-popup-enter-from .result-popup,
.result-popup-leave-to .result-popup {
  opacity: 0;
  transform: scale(0.9);
}

table {
  min-width: 0;
  font-size: 12px;
  border-collapse: collapse;
  table-layout: fixed;
  background: #fff;
}

.table-scroll {
  width: 100%;
  margin-bottom: 8px;
  overflow-x: auto;
}

th {
  position: relative;
  padding: 4px 6px;
  color: #1a2a3a;
  font-weight: 700;
  background: #ffdd00;
  border: 1px solid #333;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

td {
  padding: 4px 6px;
  color: #1a2a3a;
  text-align: center;
  background: #fff;
  border: 1px solid #333;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.empty-cell {
  color: #888;
}

.info-panel {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  padding: 6px 12px;
  background: #e6f2ff;
  border: 1px solid #b8d4e8;
  border-radius: 4px;
}

.info-column {
  display: grid;
  gap: 3px;
}

.info-column > div {
  font-size: 12px;
  line-height: 1.2;
  white-space: nowrap;
}

.info-panel input {
  width: 138px;
  padding: 1px 6px;
  color: #003399;
  font-size: 12px;
  font-weight: 500;
  background: #fff;
  border: 1px solid #b8d4e8;
  border-radius: 3px;
  outline: none;
}

@media (max-width: 768px) {
  .video-wrap {
    flex-direction: column;
  }

  .video-box {
    width: 100%;
  }

  .info-panel {
    flex-direction: column;
  }

}

/* ---------- 保留：球化结果颜色 + 表格文字观感（2026-10-10） ---------- */
table {
  font-weight: 600;
}

.result-normal {
  color: #16803c;
}

.result-abnormal {
  color: #c62828;
}

/* ---------- 表头可拖拽调整列宽（2026-10-11） ---------- */
.col-resizer {
  position: absolute;
  top: 0;
  right: 0;
  z-index: 3;
  width: 8px;
  height: 100%;
  cursor: col-resize;
  user-select: none;
}

.col-resizer:hover {
  background: rgba(0, 0, 0, 0.22);
}
</style>
