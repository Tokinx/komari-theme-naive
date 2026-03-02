<script setup lang="ts">
import type { NodeData } from '@/stores/nodes'
import { NBadge, NButton, NIcon, NList, NListItem, NModal, NProgress, NTag, NText, NTooltip, useThemeVars } from 'naive-ui'
import { computed, ref } from 'vue'
import PingChart from '@/components/PingChart.vue'
import TrafficProgress from '@/components/TrafficProgress.vue'
import { useAppStore } from '@/stores/app'
import { formatBytesPerSecondWithConfig, formatBytesWithConfig, formatUptimeWithFormat, getStatus } from '@/utils/helper'
import { getOSImage, getOSName } from '@/utils/osImageHelper'
import { getRegionCode, getRegionDisplayName } from '@/utils/regionHelper'
import { formatPriceWithCycle, getDaysUntilExpired, getExpireStatus, parseTags } from '@/utils/tagHelper'

const props = defineProps<{
  nodes: NodeData[]
}>()

const emit = defineEmits<{
  click: [node: NodeData]
}>()

// 检测是否为触摸设备（移动端）
const isTouchDevice = computed(() => {
  if (typeof window === 'undefined')
    return false
  return 'ontouchstart' in window || navigator.maxTouchPoints > 0
})

const appStore = useAppStore()

// 获取 Naive UI 主题变量
const themeVars = useThemeVars()

// 延迟图表弹窗状态
const showPingChart = ref(false)
const selectedNode = ref<NodeData | null>(null)

// 列可见性计算
const columns = computed(() => appStore.listViewColumns)

// 格式化函数
const formatBytes = (bytes: number) => formatBytesWithConfig(bytes, appStore.byteDecimals)
const formatBytesPerSecond = (bytes: number) => formatBytesPerSecondWithConfig(bytes, appStore.byteDecimals)
const formatUptime = (seconds: number) => formatUptimeWithFormat(seconds, appStore.uptimeFormat)

// 动态生成 grid 样式，使用配置的列宽度和间距
const gridStyle = computed(() => {
  const visibleColumns = columns.value
  const columnWidths = appStore.listColumnWidths
  const columnGap = appStore.listColumnGap
  const templateColumns = visibleColumns.map(col => columnWidths[col] || 'auto')
  return {
    gridTemplateColumns: templateColumns.join(' '),
    gap: columnGap,
  }
})

// 获取列的内边距样式
function getColumnPadding(col: string): Record<string, string> {
  const padding = appStore.listColumnPadding[col]
  if (padding) {
    return { padding }
  }
  return {}
}

// 获取列的外边距样式
function getColumnMargin(col: string): Record<string, string> {
  const margin = appStore.listColumnMargin[col]
  if (margin) {
    return { margin }
  }
  return {}
}

// 获取列的完整样式（合并 padding 和 margin）
function getColumnStyle(col: string): Record<string, string> {
  return {
    ...getColumnPadding(col),
    ...getColumnMargin(col),
  }
}

// 计算行高度样式
const rowHeightStyle = computed(() => {
  const height = appStore.listRowHeight
  if (height) {
    return { height, minHeight: height }
  }
  return {}
})

// 是否启用背景模糊
const hasBackgroundBlur = computed(() => {
  return appStore.backgroundEnabled && appStore.cardBlurRadius > 0
})

// 计算列表模糊半径类
const listBlurClass = computed(() => {
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

// 计算国旗图标路径
function getFlagSrc(region: string): string {
  const code = getRegionCode(region)
  return `/images/flags/${code}.svg`
}

function handleClick(node: NodeData) {
  emit('click', node)
}

function openPingChart(node: NodeData) {
  selectedNode.value = node
  showPingChart.value = true
}

// 计算节点是否显示流量进度条
function showTrafficProgress(node: NodeData): boolean {
  return node.traffic_limit > 0
}

// 计算流量使用百分比
function getTrafficUsedPercentage(node: NodeData): number {
  if (node.traffic_limit <= 0)
    return 0

  const { net_total_up = 0, net_total_down = 0, traffic_limit_type } = node
  let used = 0

  switch (traffic_limit_type) {
    case 'up':
      used = net_total_up
      break
    case 'down':
      used = net_total_down
      break
    case 'min':
      used = Math.min(net_total_up, net_total_down)
      break
    case 'max':
      used = Math.max(net_total_up, net_total_down)
      break
    case 'sum':
    default:
      used = net_total_up + net_total_down
      break
  }

  return Math.min((used / node.traffic_limit) * 100, 100)
}

// 计算已用流量
function getTrafficUsed(node: NodeData): number {
  const { net_total_up = 0, net_total_down = 0, traffic_limit_type } = node
  switch (traffic_limit_type) {
    case 'up':
      return net_total_up
    case 'down':
      return net_total_down
    case 'min':
      return Math.min(net_total_up, net_total_down)
    case 'max':
      return Math.max(net_total_up, net_total_down)
    case 'sum':
    default:
      return net_total_up + net_total_down
  }
}

// 根据过期状态获取颜色
function getExpireBadgeColor(status: string): string {
  switch (status) {
    case 'expired':
    case 'critical':
      return '#E54D2E' // 红色
    case 'warning':
      return '#F97316' // 橙色
    case 'long_term':
      return '#8D8D8D' // 灰色
    case 'normal':
    default:
      return '#30A46C' // 绿色
  }
}

// 计算节点的标签列表（返回颜色）
function getNodeTags(node: NodeData): Array<{ text: string, color: string }> {
  const tags: Array<{ text: string, color: string }> = []
  const lang = appStore.lang

  // 前两个标签：剩余天数和价格（price > 0 时显示）
  if (node.price !== 0) {
    // 剩余天数标签
    const days = getDaysUntilExpired(node.expired_at)
    const status = getExpireStatus(node.expired_at)
    const color = getExpireBadgeColor(status)

    if (status === 'expired') {
      tags.push({ text: lang === 'zh-CN' ? '已过期' : 'Expired', color })
    }
    else if (status === 'long_term') {
      tags.push({ text: lang === 'zh-CN' ? '长期' : 'Long-term', color })
    }
    else {
      tags.push({ text: lang === 'zh-CN' ? `剩余 ${days} 天` : `${days} days left`, color })
    }

    // 价格标签
    const priceText = formatPriceWithCycle(node.price, node.billing_cycle, node.currency, lang)
    tags.push({ text: priceText, color: '#0090FF' }) // 蓝色
  }

  // 后续标签：从 tags 字段解析
  const customTags = parseTags(node.tags)
  for (const tag of customTags) {
    tags.push({ text: tag.text, color: tag.hex })
  }

  return tags
}

// 列标题映射
const columnTitles: Record<string, string> = {
  status: '状态',
  region: '地区',
  name: '节点',
  tags: '标签',
  uptime: '运行时间',
  os: '系统',
  cpu: 'CPU',
  mem: '内存',
  disk: '硬盘',
  traffic: '流量',
}
</script>

<template>
  <div class="node-list-wrapper">
    <NList
      hoverable
      clickable
      bordered
      class="min-w-fit w-full"
      :class="[
        { 'light-list-contrast': appStore.lightCardContrast && !appStore.isDark },
        { 'glass-list-enabled': hasBackgroundBlur },
        listBlurClass,
      ]"
    >
      <template #header>
        <div class="node-list-header" :style="gridStyle">
          <template v-for="col in columns" :key="col">
            <div :class="`node-list-header__${col}`" :style="getColumnStyle(col)">
              <NText :depth="3" class="text-xs">
                {{ columnTitles[col] }}
              </NText>
            </div>
          </template>
        </div>
      </template>
      <NListItem v-for="node in props.nodes" :key="node.uuid" :class="{ 'opacity-50 pointer-events-none': !node.online }" :style="rowHeightStyle" @click="handleClick(node)">
        <div class="node-list-item" :style="gridStyle">
          <template v-for="col in columns" :key="col">
            <!-- 在线状态指示器 -->
            <div v-if="col === 'status'" class="node-list-item__status" :style="getColumnStyle('status')">
              <div class="flex gap-1 items-center">
                <NTooltip v-if="appStore.showPingChartButton">
                  <template #trigger>
                    <NButton
                      quaternary
                      circle
                      size="tiny"
                      class="p-1!"
                      @click.stop="openPingChart(node)"
                    >
                      <template #icon>
                        <div class="i-icon-park-outline-area-map text-sm" />
                      </template>
                    </NButton>
                  </template>
                  查看延迟图表
                </NTooltip>
                <!-- 根据 listStatusStyle 配置选择显示方式 -->
                <NTag v-if="appStore.listStatusStyle === 'tag'" :type="node.online ? 'success' : 'error'" size="small">
                  {{ node.online ? '在线' : '离线' }}
                </NTag>
                <NBadge v-else :type="node.online ? 'success' : 'error'" :value="node.online ? '在线' : '离线'" />
              </div>
            </div>

            <!-- 国旗 -->
            <div v-else-if="col === 'region'" class="node-list-item__region" :style="getColumnStyle('region')">
              <NIcon size="20">
                <img :src="getFlagSrc(node.region)" :alt="getRegionDisplayName(node.region)" class="rounded-sm">
              </NIcon>
            </div>

            <!-- 节点名称 -->
            <div v-else-if="col === 'name'" class="node-list-item__name" :style="getColumnStyle('name')">
              <NText class="text-sm font-semibold">
                {{ node.name }}
              </NText>
            </div>

            <!-- 标签 -->
            <div v-else-if="col === 'tags'" class="node-list-item__tags" :style="getColumnStyle('tags')">
              <div class="flex flex-wrap gap-1 items-center">
                <!-- 根据 listTagsStyle 配置选择显示方式 -->
                <template v-if="appStore.listTagsStyle === 'tag'">
                  <NTag
                    v-for="(tag, index) in getNodeTags(node)"
                    :key="index"
                    :color="{ color: `${tag.color}20`, textColor: tag.color, borderColor: `${tag.color}40` }"
                    size="small"
                  >
                    {{ tag.text }}
                  </NTag>
                </template>
                <template v-else>
                  <NBadge
                    v-for="(tag, index) in getNodeTags(node)"
                    :key="index"
                    :color="tag.color"
                    :value="tag.text"
                  />
                </template>
              </div>
            </div>

            <!-- 运行时间 -->
            <div v-else-if="col === 'uptime'" class="node-list-item__uptime" :style="getColumnStyle('uptime')">
              <NText :depth="3" class="text-xs" :style="{ fontFamily: appStore.numberFontFamily }">
                {{ formatUptime(node.uptime ?? 0) }}
              </NText>
            </div>

            <!-- 操作系统 -->
            <div v-else-if="col === 'os'" class="node-list-item__os" :style="getColumnStyle('os')">
              <div class="flex gap-1 items-center">
                <NIcon size="16">
                  <img :src="getOSImage(node.os)" :alt="getOSName(node.os)">
                </NIcon>
                <NText :depth="3" class="text-xs">
                  {{ getOSName(node.os) }}
                </NText>
              </div>
            </div>

            <!-- CPU -->
            <div v-else-if="col === 'cpu'" class="node-list-item__cpu" :style="getColumnStyle('cpu')">
              <div class="flex flex-col gap-0.5">
                <div class="text-xs flex gap-1 items-center" :style="{ fontFamily: appStore.numberFontFamily }">
                  <NText>{{ (node.cpu ?? 0).toFixed(1) }}%</NText>
                  <NText :depth="3">
                    ({{ node.load.toFixed(2) ?? 0 }}, {{ node.load5.toFixed(2) ?? 0 }}, {{ node.load15.toFixed(2) ?? 0 }})
                  </NText>
                </div>
                <NProgress :show-indicator="false" :percentage="node.cpu ?? 0" :status="getStatus(node.cpu ?? 0)" :height="4" />
              </div>
            </div>

            <!-- 内存 -->
            <div v-else-if="col === 'mem'" class="node-list-item__mem" :style="getColumnStyle('mem')">
              <div class="flex flex-col gap-0.5">
                <div class="text-xs flex gap-1 items-center" :style="{ fontFamily: appStore.numberFontFamily }">
                  <NText>{{ ((node.ram ?? 0) / (node.mem_total || 1) * 100).toFixed(1) }}%</NText>
                  <NText :depth="3">
                    ({{ formatBytes(node.ram ?? 0) }} / {{ formatBytes(node.mem_total ?? 0) }})
                  </NText>
                </div>
                <NProgress :show-indicator="false" :percentage="(node.ram ?? 0) / (node.mem_total || 1) * 100" :status="getStatus((node.ram ?? 0) / (node.mem_total || 1) * 100)" :height="4" />
              </div>
            </div>

            <!-- 硬盘 -->
            <div v-else-if="col === 'disk'" class="node-list-item__disk" :style="getColumnStyle('disk')">
              <div class="flex flex-col gap-0.5">
                <div class="text-xs flex gap-1 items-center" :style="{ fontFamily: appStore.numberFontFamily }">
                  <NText>{{ ((node.disk ?? 0) / (node.disk_total || 1) * 100).toFixed(1) }}%</NText>
                  <NText :depth="3">
                    ({{ formatBytes(node.disk ?? 0) }} / {{ formatBytes(node.disk_total ?? 0) }})
                  </NText>
                </div>
                <NProgress :show-indicator="false" :percentage="(node.disk ?? 0) / (node.disk_total || 1) * 100" :status="getStatus((node.disk ?? 0) / (node.disk_total || 1) * 100)" :height="4" />
              </div>
            </div>

            <!-- 流量 -->
            <div v-else-if="col === 'traffic'" class="node-list-item__traffic" :style="getColumnStyle('traffic')">
              <div class="traffic-cell">
                <!-- 有流量限制时显示进度条版式 -->
                <template v-if="showTrafficProgress(node)">
                  <NTooltip :trigger="isTouchDevice ? 'click' : 'hover'">
                    <template #trigger>
                      <div class="flex flex-col gap-0.5 w-full" :class="{ 'cursor-help': !isTouchDevice }" @click.stop>
                        <div class="text-xs flex gap-1 items-center" :style="{ fontFamily: appStore.numberFontFamily }">
                          <NText>{{ getTrafficUsedPercentage(node).toFixed(1) }}%</NText>
                          <NText :depth="3">
                            ({{ formatBytes(getTrafficUsed(node)) }} / {{ formatBytes(node.traffic_limit) }})
                          </NText>
                        </div>
                        <!-- 统一使用 TrafficProgress 组件，自动根据类型选择颜色 -->
                        <TrafficProgress
                          :upload="node.net_total_up ?? 0"
                          :download="node.net_total_down ?? 0"
                          :traffic-limit="node.traffic_limit"
                          :traffic-limit-type="(node.traffic_limit_type || 'sum')"
                          height="4px"
                        />
                      </div>
                    </template>
                    <div class="text-xs flex flex-col gap-1" :style="{ fontFamily: appStore.numberFontFamily }">
                      <span><span :style="{ color: themeVars.successColor }">↑{{ formatBytesPerSecond(node.net_out ?? 0) }}</span> <span style="opacity: 0.6;">({{ formatBytes(node.net_total_up ?? 0) }})</span></span>
                      <span><span :style="{ color: themeVars.infoColor }">↓{{ formatBytesPerSecond(node.net_in ?? 0) }}</span> <span style="opacity: 0.6;">({{ formatBytes(node.net_total_down ?? 0) }})</span></span>
                    </div>
                  </NTooltip>
                </template>
                <!-- 无流量限制时保持现有版式 -->
                <template v-else>
                  <div class="text-xs flex flex-col gap-1" :style="{ fontFamily: appStore.numberFontFamily }">
                    <NText>
                      <span :style="{ color: themeVars.successColor }">↑{{ formatBytesPerSecond(node.net_out ?? 0) }}</span> <NText :depth="3">
                        ({{ formatBytes(node.net_total_up ?? 0) }})
                      </NText>
                    </NText>
                    <NText>
                      <span :style="{ color: themeVars.infoColor }">↓{{ formatBytesPerSecond(node.net_in ?? 0) }}</span><NText :depth="3">
                        ({{ formatBytes(node.net_total_down ?? 0) }})
                      </NText>
                    </NText>
                  </div>
                </template>
              </div>
            </div>
          </template>
        </div>
      </NListItem>
    </NList>

    <!-- 延迟图表弹窗 -->
    <NModal
      v-model:show="showPingChart"
      preset="card"
      :title="selectedNode ? `${selectedNode.name} - 延迟监控` : '延迟监控'"
      class="w-full sm:w-3/4"
      :bordered="false"
      :segmented="{ content: true, footer: 'soft' }"
    >
      <PingChart v-if="selectedNode" :uuid="selectedNode.uuid" />
    </NModal>
  </div>
</template>

<style scoped lang="scss">
.node-list-wrapper {
  overflow-x: auto;
  min-width: 0;
}

:deep(.n-list__header) {
  padding: 0px !important;
}

:deep(.n-list-item) {
  padding: 8px 16px !important;
}

.node-list-header,
.node-list-item {
  display: grid;
  align-items: center;
}

.node-list-header {
  padding: 8px 16px;
  background-color: var(--n-color-hover);
  border-radius: var(--n-border-radius);
}

.node-list-header__status,
.node-list-item__status {
  display: flex;
  justify-content: center;
  align-items: center;
}

.node-list-header__region,
.node-list-item__region {
  display: flex;
  justify-content: center;
  align-items: center;
}

.node-list-header__name,
.node-list-item__name {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.node-list-header__uptime,
.node-list-item__uptime {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  text-align: left;
}

.node-list-header__os,
.node-list-item__os {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.node-list-header__cpu,
.node-list-item__cpu,
.node-list-header__mem,
.node-list-item__mem,
.node-list-header__disk,
.node-list-item__disk {
  min-width: 0;
}

.node-list-header__traffic,
.node-list-item__traffic {
  min-width: 0;
}

.node-list-header__tags,
.node-list-item__tags {
  min-width: 0;
}

.traffic-cell {
  min-height: 38px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

/* 亮色模式高对比度样式 */
.light-list-contrast {
  box-shadow: 0 2px 8px 0 rgba(0, 0, 0, 0.08);
  border-color: rgba(0, 0, 0, 0.12);

  :deep(.n-list-item) {
    border-color: rgba(0, 0, 0, 0.08);
  }
}

/* 毛玻璃列表样式 */
.glass-list-enabled {
  background-color: rgba(255, 255, 255, 0.7) !important;

  :deep(.n-list-item) {
    background-color: rgba(255, 255, 255, 0.6);
  }
}

html.dark .glass-list-enabled {
  background-color: rgba(24, 24, 28, 0.85) !important;

  :deep(.n-list-item) {
    background-color: rgba(24, 24, 28, 0.7);
  }
}
</style>
