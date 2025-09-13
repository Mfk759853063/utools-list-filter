<script lang="ts" setup>
import { ref, onMounted, computed } from 'vue'

defineProps({
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
const showAddForm = ref(false)
const editingItem = ref<ListItem | null>(null)
const searchKeyword = ref('')

// 表单数据
const formData = ref({
  title: '',
  subtitle: '',
  trigger: '',
  data: ''
})

// 从本地存储加载数据
const loadData = () => {
  const saved = localStorage.getItem('list-manager-data')
  if (saved) {
    listItems.value = JSON.parse(saved)
  }
}

// 保存数据到本地存储
const saveData = () => {
  localStorage.setItem('list-manager-data', JSON.stringify(listItems.value))
}

// 添加新项目
const addItem = () => {
  if (!formData.value.title || !formData.value.trigger) {
    alert('标题和触发词不能为空')
    return
  }
  
  const newItem: ListItem = {
    id: Date.now().toString(),
    title: formData.value.title,
    subtitle: formData.value.subtitle,
    trigger: formData.value.trigger,
    data: formData.value.data
  }
  
  listItems.value.push(newItem)
  saveData()
  resetForm()
}

// 编辑项目
const editItem = (item: ListItem) => {
  editingItem.value = item
  formData.value = { ...item }
  showAddForm.value = true
}

// 更新项目
const updateItem = () => {
  if (!editingItem.value) return
  
  const index = listItems.value.findIndex(item => item.id === editingItem.value!.id)
  if (index !== -1) {
    listItems.value[index] = {
      ...editingItem.value,
      title: formData.value.title,
      subtitle: formData.value.subtitle,
      trigger: formData.value.trigger,
      data: formData.value.data
    }
    saveData()
    resetForm()
  }
}

// 删除项目
const deleteItem = (id: string) => {
  if (confirm('确定要删除这个项目吗？')) {
    listItems.value = listItems.value.filter(item => item.id !== id)
    saveData()
  }
}

// 重置表单
const resetForm = () => {
  formData.value = {
    title: '',
    subtitle: '',
    trigger: '',
    data: ''
  }
  showAddForm.value = false
  editingItem.value = null
}

// 复制到剪切板
const copyToClipboard = (data: string) => {
  window.utools.copyText(data)
  window.utools.hideMainWindow()
  window.utools.outPlugin()
}



// 导出数据
const exportData = () => {
  try {
    const exportData = {
      version: '1.0',
      timestamp: new Date().toISOString(),
      data: listItems.value
    }
    
    const dataStr = JSON.stringify(exportData, null, 2)
    const blob = new Blob([dataStr], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    
    const link = document.createElement('a')
    link.href = url
    link.download = `list-manager-data-${new Date().toISOString().split('T')[0]}.json`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    URL.revokeObjectURL(url)
    
    alert(`成功导出 ${listItems.value.length} 个项目！`)
  } catch (error) {
    console.error('导出失败：', error)
    alert('导出失败，请重试')
  }
}

// 导入数据
const importData = () => {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = '.json'
  
  input.onchange = (event) => {
    const file = (event.target as HTMLInputElement).files?.[0]
    if (!file) return
    
    const reader = new FileReader()
    reader.onload = (e) => {
      try {
        const content = e.target?.result as string
        const importedData = JSON.parse(content)
        
        console.log('导入的数据：', importedData)
        
        // 验证数据格式
        let dataToImport = null
        
        // 支持多种数据格式
        if (importedData.data && Array.isArray(importedData.data)) {
          // 新格式：包含版本信息的导出文件
          dataToImport = importedData.data
        } else if (Array.isArray(importedData)) {
          // 旧格式：直接是数组
          dataToImport = importedData
        }
        
        if (dataToImport && dataToImport.length >= 0) {
          if (confirm(`确定要导入 ${dataToImport.length} 个项目吗？这将覆盖当前所有数据！`)) {
            listItems.value = dataToImport
            saveData()
            console.log('导入后的数据：', listItems.value)
            alert(`成功导入 ${dataToImport.length} 个项目！`)
          }
        } else {
          console.error('数据格式验证失败：', importedData)
          alert('文件格式不正确，请选择正确的导出文件\n\n支持的格式：\n1. 本应用导出的JSON文件\n2. 包含列表项数组的JSON文件')
        }
      } catch (error) {
        console.error('导入失败：', error)
        alert('导入失败，文件格式可能不正确')
      }
    }
    reader.readAsText(file)
  }
  
  input.click()
}

// 清空所有数据
const clearAllData = () => {
  if (confirm('确定要清空所有数据吗？此操作不可恢复！')) {
    listItems.value = []
    saveData()
  }
}



// 过滤列表项
const filteredItems = computed(() => {
  if (!searchKeyword.value) return listItems.value
  return listItems.value.filter(item => 
    item.title.toLowerCase().includes(searchKeyword.value.toLowerCase()) ||
    item.subtitle.toLowerCase().includes(searchKeyword.value.toLowerCase()) ||
    item.trigger.toLowerCase().includes(searchKeyword.value.toLowerCase())
  )
})

onMounted(() => {
  loadData()
})
</script>

<template>
  <div class="list-manager">
    <div class="header">
      <h1>列表管理器</h1>
      <div class="header-actions">
          <button @click="exportData" class="btn btn-success">导出数据</button>
          <button @click="importData" class="btn btn-info">导入数据</button>
          <button @click="clearAllData" class="btn btn-danger">清空数据</button>
          <button @click="showAddForm = true" class="btn btn-primary">添加新项目</button>
        </div>
    </div>

    <!-- 搜索框 -->
    <div class="search-box">
      <input 
        v-model="searchKeyword" 
        type="text" 
        placeholder="搜索标题、副标题或触发词..."
        class="search-input"
      >
    </div>

    <!-- 添加/编辑表单 -->
    <div v-if="showAddForm" class="form-overlay">
      <div class="form-container">
        <h2>{{ editingItem ? '编辑项目' : '添加新项目' }}</h2>
        <form @submit.prevent="editingItem ? updateItem() : addItem()">
          <div class="form-group">
            <label>标题 *</label>
            <input v-model="formData.title" type="text" required class="form-input">
          </div>
          <div class="form-group">
            <label>副标题</label>
            <input v-model="formData.subtitle" type="text" class="form-input">
          </div>
          <div class="form-group">
            <label>触发词 *</label>
            <input v-model="formData.trigger" type="text" required class="form-input">
          </div>
          <div class="form-group">
            <label>数据</label>
            <textarea v-model="formData.data" class="form-textarea" rows="4"></textarea>
          </div>
          <div class="form-actions">
            <button type="submit" class="btn btn-primary">
              {{ editingItem ? '更新' : '添加' }}
            </button>
            <button type="button" @click="resetForm()" class="btn btn-secondary">
              取消
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- 列表项 -->
    <div class="list-container">
      <div v-if="filteredItems.length === 0" class="empty-state">
        <p>{{ searchKeyword ? '没有找到匹配的项目' : '还没有任何项目，点击上方按钮添加第一个项目' }}</p>
      </div>
      <div v-else>
        <div 
          v-for="item in filteredItems" 
          :key="item.id" 
          class="list-item"
          @click="copyToClipboard(item.data)"
        >
          <div class="item-content">
            <h3 class="item-title">{{ item.title }}</h3>
            <p v-if="item.subtitle" class="item-subtitle">{{ item.subtitle }}</p>
            <div class="item-meta">
              <span class="trigger-word">触发词: {{ item.trigger }}</span>
            </div>
            <p v-if="item.data" class="item-data">{{ item.data.substring(0, 100) }}{{ item.data.length > 100 ? '...' : '' }}</p>
          </div>
          <div class="item-actions">
            <button @click.stop="editItem(item)" class="btn btn-small btn-secondary">编辑</button>
            <button @click.stop="deleteItem(item.id)" class="btn btn-small btn-danger">删除</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.list-manager {
  padding: 20px;
  max-width: 800px;
  margin: 0 auto;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.header-actions {
  display: flex;
  gap: 10px;
}

.header h1 {
  margin: 0;
  color: #333;
}

.search-box {
  margin-bottom: 20px;
}

.search-input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.form-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.form-container {
  background: white;
  padding: 30px;
  border-radius: 8px;
  width: 90%;
  max-width: 500px;
  max-height: 80vh;
  overflow-y: auto;
}

.form-container h2 {
  margin-top: 0;
  margin-bottom: 20px;
  color: #333;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
  color: #555;
}

.form-input, .form-textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  box-sizing: border-box;
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

.form-actions {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.list-container {
  margin-top: 20px;
}

.empty-state {
  text-align: center;
  padding: 40px;
  color: #666;
}

.list-item {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 10px;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  cursor: pointer;
  transition: all 0.2s;
}

.list-item:hover {
  border-color: #007bff;
  box-shadow: 0 2px 8px rgba(0, 123, 255, 0.1);
}

.item-content {
  flex: 1;
}

.item-title {
  margin: 0 0 5px 0;
  color: #333;
  font-size: 16px;
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
  padding: 2px 6px;
  border-radius: 3px;
  font-size: 12px;
  color: #495057;
}

.item-data {
  margin: 8px 0 0 0;
  color: #666;
  font-size: 13px;
  line-height: 1.4;
}

.item-actions {
  display: flex;
  gap: 5px;
  margin-left: 15px;
}

.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.2s;
}

.btn-primary {
  background: #007bff;
  color: white;
}

.btn-primary:hover {
  background: #0056b3;
}

.btn-secondary {
  background: #6c757d;
  color: white;
}

.btn-secondary:hover {
  background: #545b62;
}

.btn-danger {
  background: #dc3545;
  color: white;
}

.btn-danger:hover {
  background: #c82333;
}

.btn-small {
  padding: 4px 8px;
  font-size: 12px;
}

.btn-success {
  background-color: #28a745;
  color: white;
}

.btn-success:hover {
  background-color: #218838;
}

.btn-info {
  background-color: #17a2b8;
  color: white;
}

.btn-info:hover {
  background-color: #138496;
}


</style>