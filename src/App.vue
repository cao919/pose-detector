<template>
  <div class="container">
    <h1>🏓 乒乓球姿势检测</h1>
    
    <div class="upload-section">
      <div 
        class="upload-area" 
        @click="triggerFileInput"
        @drop.prevent="handleDrop"
        @dragover.prevent
        :class="{ 'drag-over': isDragOver }"
        @dragenter.prevent="isDragOver = true"
        @dragleave.prevent="isDragOver = false"
      >
        <input 
          type="file" 
          ref="fileInput" 
          @change="handleFileChange" 
          accept="image/*"
          style="display: none"
        />
        <div v-if="!previewUrl" class="upload-placeholder">
          <span class="icon">📷</span>
          <p>点击或拖拽上传图片</p>
          <p class="hint">支持 JPG、PNG 格式</p>
        </div>
        <img v-else :src="previewUrl" alt="预览" class="preview-image" />
      </div>
      
      <button 
        @click="uploadImage" 
        :disabled="!selectedFile || isLoading"
        class="upload-btn"
      >
        <span v-if="isLoading">检测中...</span>
        <span v-else>开始检测</span>
      </button>
      
      <button 
        @click="useMockData" 
        :disabled="isLoading"
        class="mock-btn"
      >
        使用模拟数据测试
      </button>
    </div>

    <div v-if="result" class="result-section">
      <h2>检测结果</h2>
      <div class="result-card" :class="{ success: result.success, error: !result.success }">
        <div class="score-display">
          <div class="score-circle" :style="{ background: getScoreColor(result.score) }">
            <span class="score-number">{{ result.score }}</span>
            <span class="score-label">分</span>
          </div>
        </div>
        
        <div class="result-info">
          <p><strong>运动类型：</strong>{{ result.sportType }}</p>
          <p><strong>状态：</strong>{{ result.message }}</p>
        </div>

        <div v-if="result.errors && result.errors.length > 0" class="errors-section">
          <h3>发现的问题</h3>
          <div class="error-list">
            <div v-for="(error, index) in result.errors" :key="index" class="error-item">
              <div class="error-header">
                <span class="error-part">{{ error.part }}</span>
                <span class="error-type">{{ error.error }}</span>
              </div>
              <div class="error-details">
                <p><strong>建议：</strong>{{ error.suggestion }}</p>
                <p><strong>当前角度：</strong>{{ typeof error.currentAngle === 'number' ? error.currentAngle.toFixed(2) : error.currentAngle }}°</p>
                <p><strong>标准角度：</strong>{{ error.standardAngle }}</p>
              </div>
            </div>
          </div>
        </div>
        
        <div v-else-if="result.success" class="success-message">
          ✅ 姿势标准，继续保持！
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const fileInput = ref(null)
const selectedFile = ref(null)
const previewUrl = ref('')
const isLoading = ref(false)
const isDragOver = ref(false)
const result = ref(null)

const triggerFileInput = () => {
  fileInput.value?.click()
}

const handleFileChange = (event) => {
  const file = event.target.files[0]
  if (file) {
    processFile(file)
  }
}

const handleDrop = (event) => {
  isDragOver.value = false
  const file = event.dataTransfer.files[0]
  if (file && file.type.startsWith('image/')) {
    processFile(file)
  }
}

const processFile = (file) => {
  selectedFile.value = file
  previewUrl.value = URL.createObjectURL(file)
  result.value = null
}

const getScoreColor = (score) => {
  if (score >= 90) return 'linear-gradient(135deg, #52c41a, #389e0d)'
  if (score >= 80) return 'linear-gradient(135deg, #faad14, #d48806)'
  if (score >= 60) return 'linear-gradient(135deg, #fa8c16, #d46b08)'
  return 'linear-gradient(135deg, #f5222d, #cf1322)'
}

const useMockData = () => {
  isLoading.value = true
  result.value = null
  
  setTimeout(() => {
    result.value = {
      sportType: "乒乓球",
      score: 85,
      errors: [
        {
          part: "右手腕",
          error: "严重过度后仰",
          suggestion: "手腕不要翘起",
          currentAngle: 177.4043,
          standardAngle: "≤165°"
        }
      ],
      success: true,
      message: "检测完成"
    }
    isLoading.value = false
  }, 1000)
}

const uploadImage = async () => {
  if (!selectedFile.value) return
  
  isLoading.value = true
  result.value = null
  
  try {
    const formData = new FormData()
    formData.append('imageFile', selectedFile.value)
    
    const response = await fetch('http://localhost:5003/api/pose/correct', {
      method: 'POST',
      body: formData
    })
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const data = await response.json()
    result.value = data
  } catch (error) {
    console.error('上传失败:', error)
    let errorMsg = error.message
    if (error.message === 'Failed to fetch') {
      errorMsg = '无法连接到检测服务，请确保后端服务已启动 (http://localhost:5003)'
    }
    result.value = {
      success: false,
      message: '检测失败：' + errorMsg,
      score: 0,
      sportType: '未知',
      errors: []
    }
  } finally {
    isLoading.value = false
  }
}
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
  font-size: 28px;
}

.upload-section {
  background: #fff;
  border-radius: 12px;
  padding: 30px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  margin-bottom: 30px;
}

.upload-area {
  border: 2px dashed #d9d9d9;
  border-radius: 8px;
  padding: 40px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.upload-area:hover,
.upload-area.drag-over {
  border-color: #1890ff;
  background: #f0f7ff;
}

.upload-placeholder .icon {
  font-size: 48px;
  display: block;
  margin-bottom: 16px;
}

.upload-placeholder p {
  margin: 8px 0;
  color: #666;
  font-size: 16px;
}

.upload-placeholder .hint {
  color: #999;
  font-size: 14px;
}

.preview-image {
  max-width: 100%;
  max-height: 400px;
  border-radius: 8px;
  object-fit: contain;
}

.upload-btn {
  width: 100%;
  padding: 14px 24px;
  margin-top: 20px;
  background: linear-gradient(135deg, #1890ff, #096dd9);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s;
}

.upload-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #40a9ff, #1890ff);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(24, 144, 255, 0.4);
}

.upload-btn:disabled {
  background: #d9d9d9;
  cursor: not-allowed;
}

.mock-btn {
  width: 100%;
  padding: 10px 20px;
  margin-top: 10px;
  background: #f5f5f5;
  color: #666;
  border: 1px solid #d9d9d9;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s;
}

.mock-btn:hover:not(:disabled) {
  background: #e8e8e8;
  border-color: #bfbfbf;
}

.mock-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.result-section {
  animation: fadeIn 0.5s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.result-section h2 {
  color: #333;
  margin-bottom: 20px;
  font-size: 22px;
}

.result-card {
  background: #fff;
  border-radius: 12px;
  padding: 30px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.result-card.success {
  border-left: 4px solid #52c41a;
}

.result-card.error {
  border-left: 4px solid #f5222d;
}

.score-display {
  display: flex;
  justify-content: center;
  margin-bottom: 24px;
}

.score-circle {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.score-number {
  font-size: 42px;
  font-weight: bold;
  line-height: 1;
}

.score-label {
  font-size: 14px;
  margin-top: 4px;
}

.result-info {
  text-align: center;
  margin-bottom: 24px;
  padding-bottom: 20px;
  border-bottom: 1px solid #f0f0f0;
}

.result-info p {
  margin: 8px 0;
  color: #666;
  font-size: 16px;
}

.errors-section h3 {
  color: #333;
  margin-bottom: 16px;
  font-size: 18px;
}

.error-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.error-item {
  background: #fff2f0;
  border: 1px solid #ffccc7;
  border-radius: 8px;
  padding: 16px;
}

.error-header {
  display: flex;
  gap: 12px;
  margin-bottom: 12px;
  flex-wrap: wrap;
}

.error-part {
  background: #ff4d4f;
  color: white;
  padding: 4px 12px;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
}

.error-type {
  background: #ffa39e;
  color: #820014;
  padding: 4px 12px;
  border-radius: 4px;
  font-size: 14px;
}

.error-details {
  color: #666;
  font-size: 14px;
}

.error-details p {
  margin: 6px 0;
}

.success-message {
  text-align: center;
  padding: 40px;
  color: #52c41a;
  font-size: 18px;
  font-weight: 500;
}
</style>
