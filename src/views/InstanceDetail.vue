<script setup lang="ts">
import { NButton, NCard, NDivider, NEmpty, NIcon, NTabPane, NTabs, NTag, NText } from 'naive-ui'
import { computed, defineAsyncComponent, onMounted, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useAppStore } from '@/stores/app'
import { useNodesStore } from '@/stores/nodes'
import { formatBytesPerSecondWithConfig, formatBytesWithConfig, formatDateTime, formatUptimeWithFormat } from '@/utils/helper'
import { getOSImage, getOSName } from '@/utils/osImageHelper'
import { getRegionCode, getRegionDisplayName } from '@/utils/regionHelper'

// 异步组件：按需加载图表，减少首屏体积
const LoadChart = defineAsyncComponent(() => import('@/components/LoadChart.vue'))
const PingChart = defineAsyncComponent(() => import('@/components/PingChart.vue'))

const route = useRoute()
const router = useRouter()

const appStore = useAppStore()
const nodesStore = useNodesStore()

// 进入详情页时滚动到顶部
onMounted(() => {
  window.scrollTo({ top: 0, behavior: 'instant' })
})

// 格式化函数
const formatBytes = (bytes: number) => formatBytesWithConfig(bytes, appStore.byteDecimals)
const formatBytesPerSecond = (bytes: number) => formatBytesPerSecondWithConfig(bytes, appStore.byteDecimals)
const formatUptime = (seconds: number) => formatUptimeWithFormat(seconds, appStore.uptimeFormat)

// 视图切换：load 或 ping
const chartView = ref<'load' | 'ping'>('load')

const data = computed(() => {
  return nodesStore.nodes.find(node => node.uuid === route.params.id)
})

// 是否启用模糊背景
const hasBackgroundBlur = computed(() => appStore.backgroundEnabled && appStore.backgroundBlur > 0)

// 计算模糊半径类
const blurClass = computed(() => {
  if (!hasBackgroundBlur.value)
    return ''
  const radius = appStore.cardBlurRadius
  if (radius <= 8)
    return 'glass-8'
  if (radius <= 12)
    return 'glass-12'
  if (radius <= 16)
    return 'glass-16'
  if (radius <= 20)
    return 'glass-20'
  return `glass-${radius}`
})

/** 信息项配置 */
interface InfoItem {
  label: string
  value: string | undefined
  icon?: string
}

/** 硬件信息 */
const hardwareInfo = computed<InfoItem[]>(() => [
  { label: 'CPU', value: data.value ? `${data.value.cpu_name} (x${data.value.cpu_cores})` : '-', icon: 'i-icon-park-outline-cpu' },
  { label: '架构', value: data.value?.arch ?? '-', icon: 'i-icon-park-outline-application-two' },
  { label: '虚拟化', value: data.value?.virtualization ?? '-', icon: 'i-icon-park-outline-server' },
  { label: 'GPU', value: data.value?.gpu_name || '-', icon: 'i-icon-park-outline-video-one' },
])

/** 系统信息 */
const systemInfo = computed<InfoItem[]>(() => [
  { label: '操作系统', value: data.value?.os ?? '-', icon: 'i-icon-park-outline-computer' },
  { label: '内核版本', value: data.value?.kernel_version ?? '-', icon: 'i-icon-park-outline-code' },
  { label: '运行时间', value: formatUptime(data.value?.uptime ?? 0), icon: 'i-icon-park-outline-timer' },
  { label: '最后上报', value: formatDateTime(data.value?.time), icon: 'i-icon-park-outline-time' },
])

/** 存储信息 */
const storageInfo = computed<InfoItem[]>(() => [
  { label: '内存', value: formatBytes(data.value?.mem_total ?? 0), icon: 'i-icon-park-outline-memory' },
  { label: '内存交换', value: formatBytes(data.value?.swap_total ?? 0), icon: 'i-icon-park-outline-switch' },
  { label: '硬盘', value: formatBytes(data.value?.disk_total ?? 0), icon: 'i-icon-park-outline-hard-disk' },
])

// 是否启用亮色模式高对比度
const lightCardContrastEnabled = computed(() => appStore.lightCardContrast && !appStore.isDark)
</script>

<template>
  <div class="instance-detail">
    <!-- 节点不存在时的空状态 -->
    <div v-if="!data" class="p-4">
      <NCard>
        <NEmpty description="节点不存在或已被删除">
          <template #extra>
            <NButton @click="router.push('/')">
              返回首页
            </NButton>
          </template>
        </NEmpty>
      </NCard>
    </div>

    <!-- 节点详情 -->
    <template v-else>
      <!-- 头部信息 -->
      <div class="px-4 py-2 flex gap-4 items-center">
        <NButton text @click="router.push('/')">
          <div class="i-icon-park-outline-arrow-left" />
        </NButton>
        <div class="text-lg font-bold flex gap-2 items-center">
          <NIcon size="24">
            <img :src="`/images/flags/${getRegionCode(data.region)}.svg`" :alt="getRegionDisplayName(data.region)">
          </NIcon>
          <NText>
            {{ data.name }}
          </NText>
        </div>
        <NTag :type="data.online ? 'success' : 'error'" size="small">
          {{ data.online ? '在线' : '离线' }}
        </NTag>
        <!-- <NText :depth="3" class="text-xs">
          {{ data.uuid }}
        </NText> -->
      </div>

      <!-- 实例信息卡片 -->
      <div class="p-4 gap-4 grid grid-cols-1 lg:grid-cols-2">
        <!-- 硬件信息 -->
        <NCard title="硬件信息" size="small" :class="[{ 'light-card-contrast': lightCardContrastEnabled }, { 'glass-card-enabled': hasBackgroundBlur }, blurClass]">
          <div class="gap-4 grid grid-cols-1 sm:grid-cols-2">
            <div v-for="item in hardwareInfo" :key="item.label" class="flex flex-col gap-1">
              <div class="flex gap-1 items-center">
                <div v-if="item.icon" :class="item.icon" class="text-gray-400" />
                <NText :depth="3" class="text-sm">
                  {{ item.label }}
                </NText>
              </div>
              <NText class="text-sm break-all">
                {{ item.value }}
              </NText>
            </div>
          </div>
        </NCard>

        <!-- 系统信息 -->
        <NCard title="系统信息" size="small" :class="[{ 'light-card-contrast': lightCardContrastEnabled }, { 'glass-card-enabled': hasBackgroundBlur }, blurClass]">
          <div class="gap-4 grid grid-cols-1 sm:grid-cols-2">
            <div v-for="item in systemInfo" :key="item.label" class="flex flex-col gap-1">
              <div class="flex gap-1 items-center">
                <div v-if="item.icon" :class="item.icon" class="text-gray-400" />
                <NText :depth="3" class="text-sm">
                  {{ item.label }}
                </NText>
              </div>
              <div class="flex gap-2 items-center">
                <NIcon v-if="item.label === '操作系统'" size="20">
                  <img :src="getOSImage(data.os)" :alt="getOSName(data.os)">
                </NIcon>
                <NText class="text-sm break-all" :style="(item.label === '运行时间' || item.label === '最后上报') ? { fontFamily: appStore.numberFontFamily } : {}">
                  {{ item.value }}
                </NText>
              </div>
            </div>
          </div>
        </NCard>

        <!-- 存储信息 -->
        <NCard title="存储信息" size="small" :class="[{ 'light-card-contrast': lightCardContrastEnabled }, { 'glass-card-enabled': hasBackgroundBlur }, blurClass]">
          <div class="gap-4 grid grid-cols-1 sm:grid-cols-3">
            <div v-for="item in storageInfo" :key="item.label" class="flex flex-col gap-1">
              <div class="flex gap-1 items-center">
                <div v-if="item.icon" :class="item.icon" class="text-gray-400" />
                <NText :depth="3" class="text-sm">
                  {{ item.label }}
                </NText>
              </div>
              <NText class="text-sm" :style="{ fontFamily: appStore.numberFontFamily }">
                {{ item.value }}
              </NText>
            </div>
          </div>
        </NCard>

        <!-- 网络信息 -->
        <NCard title="网络信息" size="small" :class="[{ 'light-card-contrast': lightCardContrastEnabled }, { 'glass-card-enabled': hasBackgroundBlur }, blurClass]">
          <div class="gap-4 grid grid-cols-1 sm:grid-cols-2">
            <div class="flex flex-col gap-1">
              <div class="flex gap-1 items-center">
                <div class="i-icon-park-outline-transfer-data text-gray-400" />
                <NText :depth="3" class="text-sm">
                  总流量
                </NText>
              </div>
              <NText class="text-sm break-all" :style="{ fontFamily: appStore.numberFontFamily }">
                ↑ {{ formatBytes(data?.net_total_up ?? 0) }}
                <span class="p-1" />
                ↓ {{ formatBytes(data?.net_total_down ?? 0) }}
              </NText>
            </div>
            <div class="flex flex-col gap-1">
              <div class="flex gap-1 items-center">
                <div class="i-icon-park-outline-dashboard-one text-gray-400" />
                <NText :depth="3" class="text-sm">
                  网络速率
                </NText>
              </div>
              <NText class="text-sm break-all" :style="{ fontFamily: appStore.numberFontFamily }">
                ↑ {{ formatBytesPerSecond(data?.net_out ?? 0) }}
                <span class="p-1" />
                ↓ {{ formatBytesPerSecond(data?.net_in ?? 0) }}
              </NText>
            </div>
          </div>
        </NCard>
      </div>

      <!-- 分割线 -->
      <div>
        <NDivider class="my-0! px-4!" dashed />
      </div>

      <!-- 图表标签页 -->
      <div class="p-4">
        <NTabs v-model:value="chartView" type="segment" animated>
          <NTabPane name="load" tab="负载">
            <LoadChart :uuid="data.uuid" />
          </NTabPane>
          <NTabPane name="ping" tab="延迟">
            <PingChart :uuid="data.uuid" />
          </NTabPane>
        </NTabs>
      </div>
    </template>
  </div>
</template>

<style scoped lang="scss">
/* 亮色模式高对比度样式 */
.light-card-contrast {
  background-color: rgba(250, 250, 252, 1) !important;
  box-shadow: 0 2px 8px 0 rgba(0, 0, 0, 0.08);
  border-color: rgba(0, 0, 0, 0.12) !important;

  &:hover {
    box-shadow: 0 4px 12px 0 rgba(0, 0, 0, 0.12);
  }
}

/* 毛玻璃卡片样式 */
.glass-card-enabled {
  background-color: rgba(255, 255, 255, 0.7) !important;

  &:hover {
    filter: brightness(0.95);
  }
}

html.dark .glass-card-enabled {
  background-color: rgba(24, 24, 28, 0.85) !important;

  &:hover {
    filter: brightness(1.1);
  }
}

/* NTabs segment 类型背景修复 */
:deep(.n-tabs-tab--segment) {
  background-color: rgba(255, 255, 255, 0.9) !important;
}

html.dark :deep(.n-tabs-tab--segment) {
  background-color: rgba(30, 30, 35, 0.95) !important;
}

:deep(.n-tabs-tab--segment.n-tabs-tab--active) {
  background-color: var(--n-tab-color-active) !important;
}
</style>
