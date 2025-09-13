<script lang="ts" setup>
import { onMounted, ref } from 'vue';
import ListManager from './ListManager/index.vue'
import ListSearch from './ListSearch/index.vue'

const route = ref('')
const enterAction = ref({})

onMounted(() => {
  // 开发环境模拟utools对象
  if (!window.utools) {
    window.utools = {
      onPluginEnter: (callback) => {
        // 开发环境默认进入列表管理页面
        setTimeout(() => {
          callback({ code: 'list-manager', payload: '' })
        }, 100)
      },
      onPluginOut: (callback) => {},
      copyText: (text) => {
        navigator.clipboard.writeText(text).then(() => {
          console.log('已复制到剪切板:', text)
        })
      },
      hideMainWindow: () => {
        console.log('隐藏主窗口')
      },
      outPlugin: () => {
        console.log('退出插件')
      }
    }
  }
  
  window.utools.onPluginEnter((action) => {
    route.value = action.code
    
    // 处理搜索功能的正则匹配
    if (action.code === 'list-search' && action.payload) {
      // 从"VBNB xxx"中提取搜索关键词
      const match = action.payload.match(/^wj\s+(.+)$/)
      if (match && match[1]) {
        action.payload = match[1].trim()
      }
    }
    
    enterAction.value = action
  })
  window.utools.onPluginOut((isKill) => {
    route.value = ''
  })
})
</script>

<template>
  <template v-if="route === 'list-manager'">
    <ListManager :enterAction="enterAction"></ListManager>
  </template>
  <template v-if="route === 'list-search'">
    <ListSearch :enterAction="enterAction"></ListSearch>
  </template>
</template>
