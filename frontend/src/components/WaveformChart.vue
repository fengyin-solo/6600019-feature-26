<template>
  <div class="flex flex-col gap-2">
    <div v-for="comp in components" :key="comp.key" class="bg-gray-900 rounded-xl p-3">
      <h3 class="text-xs text-gray-400 mb-1">{{ comp.label }}</h3>
      <v-chart
        :ref="el => setChartRef(el, comp.key)"
        :option="chartOptions[comp.key]"
        class="h-40"
        autoresize
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { LineChart } from 'echarts/charts'
import { GridComponent, TooltipComponent, MarkLineComponent } from 'echarts/components'
import VChart from 'vue-echarts'
import { useSeismicStore } from '../store/seismic'
import { storeToRefs } from 'pinia'
import { computed, watch, onBeforeUnmount } from 'vue'
import type { EChartsOption, ECharts } from 'echarts'

use([CanvasRenderer, LineChart, GridComponent, TooltipComponent, MarkLineComponent])

const store = useSeismicStore()
const { hoverTime, waveform, picks } = storeToRefs(store)

const components = [
  { key: 'bhz' as const, label: 'BHZ (垂直分量)', color: '#22d3ee' },
  { key: 'bhn' as const, label: 'BHN (南北分量)', color: '#a78bfa' },
  { key: 'bhe' as const, label: 'BHE (东西分量)', color: '#34d399' },
]

const chartRefs = {} as Record<string, any>
const chartInstances = new Map<string, ECharts>()
let pollTimer: number | null = null

function setChartRef(el: any, key: string) {
  chartRefs[key] = el
}

function bindZREvents(chart: ECharts, key: string) {
  if (chartInstances.has(key)) return

  chartInstances.set(key, chart)

  const zr = chart.getZr()
  if (!zr) return

  zr.on('mousemove', (e: any) => {
    const point = chart.convertFromPixel(
      { xAxisIndex: 0, yAxisIndex: 0 },
      [e.offsetX, e.offsetY]
    )
    if (point && point[0] !== undefined) {
      const time = point[0]
      const wf = waveform.value
      if (wf && time >= 0 && time <= wf.time[wf.time.length - 1]) {
        store.hoverTime = time
      }
    }
  })

  zr.on('mouseout', () => {
    store.hoverTime = null
  })
}

function tryBindEvents() {
  for (const comp of components) {
    const ref = chartRefs[comp.key]
    if (!ref) continue

    const chart = ref.chart || ref.getEchartsInstance?.()
    if (chart) {
      bindZREvents(chart, comp.key)
    }
  }

  if (chartInstances.size < components.length) {
    pollTimer = window.setTimeout(tryBindEvents, 100)
  }
}

watch(waveform, (newVal) => {
  if (newVal) {
    if (pollTimer) {
      clearTimeout(pollTimer)
    }
    tryBindEvents()
  }
}, { immediate: true })

onBeforeUnmount(() => {
  if (pollTimer) {
    clearTimeout(pollTimer)
    pollTimer = null
  }
  for (const [, chart] of chartInstances) {
    const zr = chart.getZr()
    if (zr) {
      zr.off('mousemove')
      zr.off('mouseout')
    }
  }
  chartInstances.clear()
})

function buildChartOption(key: 'bhz' | 'bhn' | 'bhe', color: string): EChartsOption {
  const wf = waveform.value!
  const step = 5
  const time = wf.time.filter((_, i) => i % step === 0)
  const data = wf[key].filter((_, i) => i % step === 0)

  const markLines = picks.value.map(p => ({
    xAxis: p.time,
    label: { formatter: p.type, color: p.type === 'P' ? '#ef4444' : '#3b82f6', fontSize: 14, fontWeight: 'bold' as const },
    lineStyle: { color: p.type === 'P' ? '#ef4444' : '#3b82f6', width: 2, type: 'dashed' as const }
  }))

  if (hoverTime.value !== null) {
    markLines.push({
      xAxis: hoverTime.value,
      label: {
        formatter: `t=${hoverTime.value.toFixed(2)}s`,
        color: '#fbbf24',
        fontSize: 11,
        fontWeight: 'bold' as const,
        backgroundColor: 'rgba(0,0,0,0.6)',
        padding: [2, 6],
        borderRadius: 3
      },
      lineStyle: { color: '#fbbf24', width: 1.5, type: 'solid' as const },
      symbol: 'none'
    })
  }

  return {
    tooltip: { trigger: 'axis', formatter: (params: any) => `t=${params[0].value[0].toFixed(2)}s\namp=${params[0].value[1].toFixed(4)}` },
    grid: { left: 50, right: 20, top: 10, bottom: 25 },
    xAxis: { type: 'value', min: 0, max: time[time.length - 1], axisLabel: { color: '#666', formatter: '{value}s' } },
    yAxis: { type: 'value', axisLabel: { color: '#666' }, splitLine: { lineStyle: { color: '#1f2937' } } },
    series: [{
      type: 'line',
      showSymbol: false,
      lineStyle: { color, width: 1 },
      areaStyle: { color: { type: 'linear', x: 0, y: 0, x2: 0, y2: 1, colorStops: [{ offset: 0, color: color + '33' }, { offset: 1, color: 'transparent' }] } },
      data: time.map((t, i) => [t, data[i]]),
      markLine: { symbol: 'none', data: markLines }
    }]
  }
}

const chartOptions = computed(() => {
  if (!waveform.value) return {} as Record<string, EChartsOption>
  const options: Record<string, EChartsOption> = {}
  for (const comp of components) {
    options[comp.key] = buildChartOption(comp.key, comp.color)
  }
  return options
})
</script>
