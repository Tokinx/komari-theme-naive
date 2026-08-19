<script setup lang="ts">
import type { EChartsOption } from 'echarts/types/dist/shared'
import type { NodeData } from '@/stores/nodes'
import { computed, onMounted, ref } from 'vue'
import VChart from 'vue-echarts'
import { Empty } from '@/components/ui/empty'
import { Spinner } from '@/components/ui/spinner'
import { useAppStore } from '@/stores/app'
import { useNodesStore } from '@/stores/nodes'
import { ensureWorldMapRegistered } from '@/utils/echartsWorldMap'
import { getCoordByCode, getCountryCodeFromRegion } from '@/utils/geoHelper'
import { getRegionDisplayName } from '@/utils/regionHelper'
import '@/utils/echarts'

interface EarthMapPoint {
  code: string
  name: string
  coord: [number, number]
  online: number
  offline: number
  total: number
}

const props = defineProps<{
  nodes?: NodeData[]
}>()

const appStore = useAppStore()
const nodesStore = useNodesStore()
const displayNodes = computed(() => props.nodes ?? nodesStore.earthNodes)
const mapName = ref<string>()
const loading = ref(true)
const loadError = ref<string | null>(null)

const regionDisplayNames = typeof Intl.DisplayNames === 'function'
  ? new Intl.DisplayNames(['zh-Hans'], { type: 'region' })
  : null

function resolveCountryDisplayName(region: string, code: string): string {
  const regionName = getRegionDisplayName(region)
  if (regionName !== region)
    return regionName

  return regionDisplayNames?.of(code) ?? code
}

const points = computed<EarthMapPoint[]>(() => {
  const map = new Map<string, EarthMapPoint>()
  for (const node of displayNodes.value) {
    const code = getCountryCodeFromRegion(node.region)
    if (!code)
      continue
    const coord = getCoordByCode(code)
    if (!coord)
      continue
    const current = map.get(code)
    if (!current) {
      map.set(code, {
        code,
        name: resolveCountryDisplayName(node.region, code),
        coord,
        online: node.online ? 1 : 0,
        offline: node.online ? 0 : 1,
        total: 1,
      })
      continue
    }
    current.total += 1
    current.online += node.online ? 1 : 0
    current.offline += node.online ? 0 : 1
  }

  return Array.from(map.values()).sort((a, b) => b.online - a.online || b.total - a.total)
})

const totalServers = computed(() => displayNodes.value.length)
const onlineServers = computed(() => displayNodes.value.filter(node => node.online).length)
const offlineServers = computed(() => totalServers.value - onlineServers.value)

const chartThemeColors = computed(() => ({
  areaColor: appStore.isDark ? 'rgba(255, 255, 255, 0.08)' : 'rgba(15, 23, 42, 0.06)',
  borderColor: appStore.isDark ? 'rgba(255, 255, 255, 0.06)' : 'rgba(15, 23, 42, 0.06)',
  hoverBorderColor: appStore.isDark ? 'rgba(16, 185, 129, 0.9)' : 'rgba(5, 150, 105, 0.85)',
  activeAreaColor: appStore.isDark ? 'rgba(16, 185, 129, 0.52)' : 'rgba(16, 185, 129, 0.36)',
  offlineAreaColor: appStore.isDark ? 'rgba(234, 179, 8, 0.32)' : 'rgba(202, 138, 4, 0.22)',
  activeBorderColor: appStore.isDark ? 'rgba(16, 185, 129, 0.95)' : 'rgba(5, 150, 105, 0.92)',
  offlineBorderColor: appStore.isDark ? 'rgba(234, 179, 8, 0.8)' : 'rgba(202, 138, 4, 0.88)',
  dotEmerald: appStore.isDark ? 'rgba(16, 185, 129, 0.92)' : 'rgba(5, 150, 105, 0.9)',
  dotYellow: appStore.isDark ? 'rgba(234, 179, 8, 0.92)' : 'rgba(202, 138, 4, 0.9)',
  text: appStore.isDark ? 'rgba(255, 255, 255, 0.85)' : 'rgba(0, 0, 0, 0.85)',
  textSecondary: appStore.isDark ? 'rgba(255, 255, 255, 0.55)' : 'rgba(0, 0, 0, 0.55)',
  tooltipBg: appStore.isDark ? 'rgba(40, 40, 40, 0.95)' : 'rgba(255, 255, 255, 0.8)',
  tooltipShadow: appStore.isDark ? 'rgba(0, 0, 0, 0.4)' : 'rgba(0, 0, 0, 0.06)',
}))

const mapSeriesData = computed(() => points.value.map(point => ({
  name: point.code,
  value: point.total,
  itemStyle: {
    areaColor: point.online > 0
      ? chartThemeColors.value.activeAreaColor
      : chartThemeColors.value.offlineAreaColor,
    borderColor: point.online > 0
      ? chartThemeColors.value.activeBorderColor
      : chartThemeColors.value.offlineBorderColor,
    borderWidth: 0.5,
  },
  emphasis: {
    itemStyle: {
      areaColor: point.online > 0
        ? chartThemeColors.value.activeAreaColor
        : chartThemeColors.value.offlineAreaColor,
      borderColor: point.online > 0
        ? chartThemeColors.value.activeBorderColor
        : chartThemeColors.value.offlineBorderColor,
    },
  },
})))

const scatterData = computed(() => points.value.map(point => ({
  name: point.name,
  code: point.code,
  value: [point.coord[1], point.coord[0], point.total],
  online: point.online,
  offline: point.offline,
  symbolSize: point.total <= 1 ? 8 : 14,
  label: { show: point.total > 1 },
  itemStyle: {
    color: point.offline > 0
      ? chartThemeColors.value.dotYellow
      : chartThemeColors.value.dotEmerald,
  },
})))

const chartOption = computed<EChartsOption>(() => ({
  animationDurationUpdate: 300,
  animationEasingUpdate: 'cubicOut',
  tooltip: {
    trigger: 'item',
    confine: true,
    backgroundColor: chartThemeColors.value.tooltipBg,
    borderColor: 'transparent',
    borderWidth: 0,
    borderRadius: 6,
    textStyle: {
      color: chartThemeColors.value.text,
      fontSize: 12,
      lineHeight: 20,
    },
    extraCssText: `padding: 3px 6px;backdrop-filter: blur(5px);z-index:9;box-shadow:0 0 0 0.5px ${chartThemeColors.value.tooltipShadow}, 0 0 16px ${chartThemeColors.value.tooltipShadow}`,
    formatter: (params: unknown) => {
      const p = params as { name: string, data: { code: string, online: number, offline: number } }
      if (!p.data)
        return ''
      const flag = `<img src="/assets/flags/${p.data.code}.svg" style="width:16px;height:16px;vertical-align:middle;margin-right:2px" />`
      const dot = (color: string) => `<span style="display:inline-block;width:6px;height:6px;border-radius:50%;background:${color}"></span>`
      const online = `<span style="display:flex;gap:4px;align-items:center">${dot(chartThemeColors.value.dotEmerald)} ${p.data.online}</span>`
      const offline = p.data.offline > 0 ? ` <span style="display:flex;gap:4px;align-items:center">${dot(chartThemeColors.value.dotYellow)} ${p.data.offline}</span>` : ''
      return `<div style="line-height:1.4"><span style="display:flex;gap:2px;align-items:center;">${flag}${p.name}</span><div style="display:flex;gap:8px;align-items:center;color:${chartThemeColors.value.textSecondary}">${online}${offline}</div></div>`
    },
  },
  geo: {
    map: mapName.value,
    roam: false,
    left: 'center',
    top: 'center',
    width: '100%',
    silent: true,
    itemStyle: {
      areaColor: 'transparent',
      borderColor: 'transparent',
    },
    emphasis: {
      itemStyle: {
        areaColor: 'transparent',
        borderColor: 'transparent',
      },
      label: { show: false },
    },
    label: { show: false },
  },
  series: [
    {
      type: 'map',
      map: mapName.value,
      roam: false,
      selectedMode: false,
      left: 'center',
      top: 'center',
      width: '100%',
      tooltip: { show: false },
      emphasis: {
        label: { show: false },
        itemStyle: {
          areaColor: chartThemeColors.value.borderColor,
          borderColor: chartThemeColors.value.hoverBorderColor,
          borderWidth: 0.5,
        },
      },
      itemStyle: {
        areaColor: chartThemeColors.value.areaColor,
        borderColor: chartThemeColors.value.borderColor,
        borderWidth: 0.5,
      },
      data: mapSeriesData.value,
      label: { show: false },
    },
    {
      type: 'scatter',
      coordinateSystem: 'geo',
      data: scatterData.value,
      symbol: 'circle',
      itemStyle: {
        color: chartThemeColors.value.dotEmerald,
        borderColor: '#ffffff',
        borderWidth: 1,
      },
      label: {
        fontSize: 10,
        color: '#ffffff',
        formatter: (params: unknown) => {
          const p = params as { value: Array<number | string> }
          const total = typeof p.value?.[2] === 'number' ? p.value[2] : 0
          return String(total)
        },
      },
      emphasis: {
        scale: 1.3,
      },
    },
  ],
}))

onMounted(async () => {
  loading.value = true
  loadError.value = null

  try {
    mapName.value = await ensureWorldMapRegistered()
  }
  catch (error) {
    loadError.value = error instanceof Error ? error.message : '地图资源加载失败'
  }
  finally {
    loading.value = false
  }
})
</script>

<template>
  <div class="relative h-full border-none" content-class="h-full !p-0">
    <!-- 预加载国旗图片，避免 tooltip 重复请求 -->
    <div class="hidden">
      <img v-for="point in points" :key="point.code" :src="`/assets/flags/${point.code}.svg`">
    </div>
    <div class="relative flex h-88 flex-col items-center">
      <div
        v-if="totalServers > 0"
        class="absolute top-0 right-0 z-2 text-[10px] text-muted-foreground pointer-events-none flex gap-2 items-center backdrop-blur-lg bg-background/60 rounded px-2 py-0.5"
      >
        <div v-if="onlineServers > 0" class="flex items-center gap-1">
          <span class="inline-block size-1.5 rounded-full bg-green-600 animate-pulse" />
          <span class="text-green-600">{{ onlineServers }}</span>
        </div>
        <div v-if="offlineServers > 0" class="flex items-center gap-1">
          <span class="inline-block size-1.5 rounded-full bg-yellow-600 animate-pulse" />
          <span class="text-yellow-600">{{ offlineServers }}</span>
        </div>
      </div>

      <div class="relative flex-1 w-full md:-translate-y-1/6 -translate-y-1/5">
        <Spinner :show="loading" class="h-full w-full" content-class="!bg-transparent">
          <VChart
            v-if="mapName"
            :option="chartOption"
            autoresize
            class="h-full w-full"
          />
          <Empty
            v-else-if="loadError"
            description="地图资源加载失败"
            class="h-full"
          />
        </Spinner>
      </div>
    </div>
  </div>
</template>
