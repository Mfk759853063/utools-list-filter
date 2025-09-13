<script lang="ts" setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

const props = defineProps({
  enterAction: {
    type: Object,
    required: true
  }
})

// 列表项数据结构
interface ListItem {
  id: string
  title: string
  subtitle: string
  trigger: string
  data: string
}

const listItems = ref<ListItem[]>([])
const searchKeyword = ref('')
const selectedIndex = ref(0)

// 从本地存储加载数据
const loadData = () => {
  const saved = localStorage.getItem('list-manager-data')
  if (saved) {
    listItems.value = JSON.parse(saved)
  }
}

// 根据触发词和输入内容匹配列表项
const matchedItems = computed(() => {
  if (!searchKeyword.value) return []
  
  const keyword = searchKeyword.value.toLowerCase()
  
  return listItems.value.filter(item => {
    const trigger = item.trigger.toLowerCase()
    const title = item.title.toLowerCase()
    const subtitle = item.subtitle.toLowerCase()
    
    // 优先匹配触发词，然后是标题和副标题
    return trigger.includes(keyword) || 
           title.includes(keyword) || 
           subtitle.includes(keyword)
  }).sort((a, b) => {
    // 触发词完全匹配的排在前面
    const aExactMatch = a.trigger.toLowerCase() === keyword
    const bExactMatch = b.trigger.toLowerCase() === keyword
    if (aExactMatch && !bExactMatch) return -1
    if (!aExactMatch && bExactMatch) return 1
    
    // 触发词开头匹配的排在前面
    const aStartsWith = a.trigger.toLowerCase().startsWith(keyword)
    const bStartsWith = b.trigger.toLowerCase().startsWith(keyword)
    if (aStartsWith && !bStartsWith) return -1
    if (!aStartsWith && bStartsWith) return 1
    
    return 0
  })
})

// 复制选中项目到剪切板
const copySelectedItem = () => {
  if (matchedItems.value.length > 0 && selectedIndex.value < matchedItems.value.length) {
    const selectedItem = matchedItems.value[selectedIndex.value]
    window.utools.copyText(selectedItem.data)
    window.utools.hideMainWindow()
    window.utools.outPlugin()
  }
}

// 键盘导航
const handleKeydown = (event: KeyboardEvent) => {
  switch (event.key) {
    case 'ArrowDown':
      event.preventDefault()
      if (selectedIndex.value < matchedItems.value.length - 1) {
        selectedIndex.value++
      }
      break
    case 'ArrowUp':
      event.preventDefault()
      if (selectedIndex.value > 0) {
        selectedIndex.value--
      }
      break
    case 'Enter':
      event.preventDefault()
      copySelectedItem()
      break
    case 'Escape':
      window.utools.outPlugin()
      break
  }
}

// 点击选择项目
const selectItem = (index: number) => {
  selectedIndex.value = index
  copySelectedItem()
}

onMounted(() => {
  loadData()
  
  // 获取输入的搜索关键词
  if (props.enterAction && props.enterAction.payload) {
    searchKeyword.value = props.enterAction.payload
  }
  
  // 监听键盘事件
  document.addEventListener('keydown', handleKeydown)
  
  // 如果只有一个匹配项且完全匹配，直接复制
  if (matchedItems.value.length === 1 && 
      matchedItems.value[0].trigger.toLowerCase() === searchKeyword.value.toLowerCase()) {
    setTimeout(() => {
      copySelectedItem()
    }, 100)
  }
})

// 组件卸载时移除事件监听
onUnmounted(() => {
  document.removeEventListener('keydown', handleKeydown)
})
</script>

<template>
  <div class="list-search">
    <div class="search-header">
      <h2>搜索结果</h2>
      <div class="search-keyword">
        搜索关键词: <span class="keyword">{{ searchKeyword }}</span>
      </div>
    </div>

    <div v-if="matchedItems.length === 0" class="no-results">
      <p>没有找到匹配的项目</p>
      <p class="hint">请尝试其他关键词，或者先在列表管理中添加项目</p>
    </div>

    <div v-else class="results-container">
      <div class="results-hint">
        找到 {{ matchedItems.length }} 个匹配项目，使用 ↑↓ 键选择，回车键复制
      </div>
      
      <div class="results-list">
        <div 
          v-for="(item, index) in matchedItems" 
          :key="item.id"
          :class="['result-item', { 'selected': index === selectedIndex }]"
          @click="selectItem(index)"
        >
          <div class="item-content">
            <h3 class="item-title">{{ item.title }}</h3>
            <p v-if="item.subtitle" class="item-subtitle">{{ item.subtitle }}</p>
            <div class="item-meta">
              <span class="trigger-word">{{ item.trigger }}</span>
            </div>
            <p v-if="item.data" class="item-preview">
              {{ item.data.substring(0, 150) }}{{ item.data.length > 150 ? '...' : '' }}
            </p>
          </div>
          <div class="copy-indicator">
            <span class="copy-text">点击复制</span>
          </div>
        </div>
      </div>
    </div>

    <div class="footer-hint">
      <p>ESC 键退出插件</p>
    </div>
  </div>
</template>

<style scoped>
.list-search {
  padding: 20px;
  max-width: 600px;
  margin: 0 auto;
  min-height: 300px;
}

.search-header {
  margin-bottom: 20px;
  text-align: center;
}

.search-header h2 {
  margin: 0 0 10px 0;
  color: #333;
}

.search-keyword {
  color: #666;
  font-size: 14px;
}

.keyword {
  background: #e3f2fd;
  padding: 2px 8px;
  border-radius: 4px;
  color: #1976d2;
  font-weight: bold;
}

.no-results {
  text-align: center;
  padding: 40px 20px;
  color: #666;
}

.no-results p {
  margin: 10px 0;
}

.hint {
  font-size: 14px;
  color: #999;
}

.results-container {
  margin-bottom: 20px;
}

.results-hint {
  background: #f8f9fa;
  padding: 10px;
  border-radius: 4px;
  font-size: 14px;
  color: #495057;
  margin-bottom: 15px;
  text-align: center;
}

.results-list {
  max-height: 400px;
  overflow-y: auto;
}

.result-item {
  border: 2px solid #e9ecef;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 10px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.result-item:hover {
  border-color: #007bff;
  box-shadow: 0 2px 8px rgba(0, 123, 255, 0.1);
}

.result-item.selected {
  border-color: #007bff;
  background: #f8f9ff;
  box-shadow: 0 4px 12px rgba(0, 123, 255, 0.15);
}

.item-content {
  flex: 1;
}

.item-title {
  margin: 0 0 5px 0;
  color: #333;
  font-size: 16px;
  font-weight: 600;
}

.item-subtitle {
  margin: 0 0 8px 0;
  color: #666;
  font-size: 14px;
}

.item-meta {
  margin-bottom: 8px;
}

.trigger-word {
  background: #e9ecef;
  padding: 3px 8px;
  border-radius: 12px;
  font-size: 12px;
  color: #495057;
  font-weight: 500;
}

.item-preview {
  margin: 8px 0 0 0;
  color: #666;
  font-size: 13px;
  line-height: 1.4;
  background: #f8f9fa;
  padding: 8px;
  border-radius: 4px;
}

.copy-indicator {
  margin-left: 15px;
  display: flex;
  align-items: center;
}

.copy-text {
  background: #28a745;
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
}

.result-item.selected .copy-text {
  background: #007bff;
}

.footer-hint {
  text-align: center;
  margin-top: 20px;
  padding-top: 15px;
  border-top: 1px solid #e9ecef;
}

.footer-hint p {
  margin: 0;
  color: #999;
  font-size: 13px;
}

/* 滚动条样式 */
.results-list::-webkit-scrollbar {
  width: 6px;
}

.results-list::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.results-list::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.results-list::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style>