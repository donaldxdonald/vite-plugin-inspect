<script setup lang="ts">
import { inspectSSR, metricDisplayHook, onRefetch } from '../../logic'
import { getHot } from '../../logic/hot'
import { isStaticMode, rpc } from '../../logic/rpc'

const data = ref(await rpc.getPluginMetrics(inspectSSR.value))

const selectedPlugin = ref('')

const displayHookOptions = ['transform', 'resolveId', isStaticMode ? '' : 'server'].filter(Boolean).map(h => ({
  label: {
    transform: 'plugin.transform',
    resolveId: 'plugin.resolveId',
    server: 'server.middleware',
  }[h as string]!,
  value: h,
}))

const plugins = computed(() => {
  if (metricDisplayHook.value === 'server')
    return []

  return data.value.map((info) => {
    if (metricDisplayHook.value === 'transform') {
      return {
        name: info.name,
        enforce: info.enforce,
        totalTime: info.transform.totalTime,
        invokeCount: info.transform.invokeCount,
      }
    }
    else {
      return {
        name: info.name,
        enforce: info.enforce,
        totalTime: info.resolveId.totalTime,
        invokeCount: info.resolveId.invokeCount,
      }
    }
  })
    .sort((a, b) => b.invokeCount - a.invokeCount)
    .sort((a, b) => b.totalTime - a.totalTime)
})

function getLatencyColor(latency: number) {
  if (latency > 1000)
    return 'text-red-400'
  if (latency > 500)
    return 'text-orange-400'
  if (latency > 200)
    return 'text-yellow-400'
  return ''
}

async function refetch() {
  data.value = await rpc.getPluginMetrics(inspectSSR.value)
}

onRefetch.on(async () => {
  await refetch()
})

function selectPlugin(plugin: string) {
  selectedPlugin.value = plugin
}

function clearPlugin() {
  selectedPlugin.value = ''
}

watch(metricDisplayHook, clearPlugin)

getHot().then((hot) => {
  if (hot) {
    hot.on('vite-plugin-inspect:update', () => {
      refetch()
    })
  }
})
</script>

<template>
  <NavBar>
    <RouterLink class="my-auto icon-btn !outline-none" to="/">
      <div i-carbon-arrow-left />
    </RouterLink>
    <div my-auto text-sm font-mono>
      Metrics
    </div>
    <SegmentControl
      v-model="metricDisplayHook"
      :options="displayHookOptions"
    />
    <div flex-auto />
  </NavBar>
  <Container v-if="data" of-auto>
    <PluginChart v-if="selectedPlugin && metricDisplayHook !== 'server'" :plugin="selectedPlugin" :hook="metricDisplayHook" :exit="clearPlugin" />
    <ServerChart v-if="metricDisplayHook === 'server'" />
    <div v-else class="grid grid-cols-[1fr_max-content_max-content_max-content_max-content_max-content_1fr] mb-4 mt-2 whitespace-nowrap children:(border-main border-b px-4 py-2 align-middle) text-sm font-mono">
      <div />
      <div class="text-xs font-bold">
        Name ({{ plugins.length }})
      </div>
      <div class="text-center text-xs font-bold">
        Type
      </div>
      <div class="text-right text-xs font-bold">
        Passes
      </div>

      <div class="text-right text-xs font-bold">
        Average Time
      </div>
      <div class="text-right text-xs font-bold">
        Total Time
      </div>
      <div />

      <template v-for="{ name, totalTime, invokeCount, enforce } in plugins" :key="name">
        <div />
        <div v-if="totalTime > 0" class="cursor-pointer text-lime-600 dark:text-lime-200 hover:underline" @click="selectPlugin(name)">
          <PluginName :name="name" />
        </div>
        <div v-else>
          <PluginName :name="name" />
        </div>
        <div class="flex items-center text-center p0!">
          <Badge
            v-if="enforce"
            class="m-auto text-xs"
            :class="[enforce === 'post' ? 'bg-purple5/10 text-purple5' : 'bg-green5/10 text-green5']"
          >
            {{ enforce }}
          </Badge>
        </div>
        <template v-if="invokeCount">
          <div class="text-right">
            {{ invokeCount }}
          </div>
          <div class="text-right" :class="getLatencyColor(totalTime / invokeCount)">
            {{ +(totalTime / invokeCount).toFixed(1) }}<span ml-1 text-xs op50>ms</span>
          </div>
          <div class="text-right" :class="getLatencyColor(totalTime / invokeCount)">
            {{ totalTime }}<span ml-1 text-xs op50>ms</span>
          </div>
        </template>
        <template v-else>
          <div class="text-right">
            -
          </div>
          <div class="text-right">
            -
          </div>
          <div class="text-right">
            -
          </div>
        </template>
        <div />
      </template>
    </div>
  </Container>
</template>
