## AI 聊天助手项目详细教程

## 第一章：环境准备

### 1.1 安装必要软件

1. 安装 Node.js

- 验证安装

  ```bash
  node -v  # v20.19.0  或更高版本
  npm -v   # 10.8.2 或更高版本
  ```

### 1.2 创建项目

##### 1.创建 Vue 项目

```bash
   # 创建项目
   npm create vite chats --template vue-ts
   # 进入项目目录
   cd chats
   # 安装依赖
   npm install
```

##### 2.安装项目依赖

```bash
   # 安装 Element Plus
   npm add element-plus @element-plus/icons-vue
   # 安装路由
   npm add vue-router@4
   # 安装状态管理
   npm add pinia
   # 安装 HTTP 客户端
   npm add axios
   # 安装类型支持
   npm add -D @types/node
```

## 第二章：项目结构搭建

### 2.1 创建目录结构

1. 在 src 目录下创建以下文件夹：

   ```bash
      src/
      ├── assets/          # 静态资源
      ├── components/      # 公共组件
      ├── composables/     # 组合式函数
      ├── router/          # 路由配置
      ├── styles/          # 样式文件
      ├── utils/           # 工具函数
      ├── views/           # 页面组件
   ```

2. 创建基础文件

   ```bash
      # 创建路由配置文件
      touch src/router/index.ts
      # 创建样式文件
      touch src/styles/index.css
      # 创建主页面组件
      touch src/views/portal.vue
      # 创建工具函数文件
      touch src/utils/request.ts
   ```

### 2.2 配置基础文件

1、配置 main.ts，将路由组件、ElementUI 等组件引进来

```typescript
   // src/main.ts
   import { createApp } from 'vue'
   import ElementPlus from 'element-plus'
   import 'element-plus/dist/index.css'
   import App from './App.vue'
   import router from './router'
   import './styles/index.css'

   const app = createApp(App)
   app.use(ElementPlus)
   app.use(router)
   app.mount('#app')
```

2、配置 App.vue

```vue
<!-- src/App.vue -->
 <template>
   <router-view></router-view>
 </template>

 <style>
 #app {
   width: 100%;
   height: 100vh;
 }
 </style>
```

3、配置路由 目前大模型 AI 问答只有一个页面

```typescript
   // src/router/index.ts
   import { createRouter, createWebHistory } from 'vue-router'

   const router = createRouter({
     history: createWebHistory(),
     routes: [
       {
         path: '/',
         name: 'Portal',
         component: () => import('../views/portal.vue')
       }
     ]
   })

   export default router
```

4、配置 HTTP 封装成一个请求工具，方便这个项目调用

```typescript
   // src/utils/request.ts
   import axios from 'axios'
   import { ElMessage } from 'element-plus'

   const request = axios.create({
     baseURL: import.meta.env.VITE_API_BASE_URL,
     timeout: 15000
   })

   // 请求拦截器
   request.interceptors.request.use(
     (config) => {
       const token = localStorage.getItem('token')
       if (token) {
         config.headers.Authorization = `Bearer ${token}`
       }
       return config
     },
     (error) => {
       return Promise.reject(error)
     }
   )

   // 响应拦截器
   request.interceptors.response.use(
     (response) => {
       return response.data
     },
     (error) => {
       ElMessage.error(error.message || '请求失败')
       return Promise.reject(error)
     }
   )

   export default request
```

## 第三章：开发聊天界面

### 3.1 创建基础布局

1. 创建 portal.vue 文件：

   ```typescript
      <!-- src/views/portal.vue -->
      <template>
        <div class="app-container">
          <el-container>
            <el-header>
              <!-- 头部导航 -->
            </el-header>
            <el-main>
              <!-- 聊天内容区 -->
            </el-main>
          </el-container>
        </div>
      </template>
   
      <script setup lang="ts">
      import { ref } from 'vue'
      </script>
   
      <style scoped>
      .app-container {
        min-height: 100vh;
        background: linear-gradient(135deg, #F8F9FA 0%, #EFD3D7 100%);
        background-attachment: fixed;
      }
      </style>
   ```

2. 添加头部导航：

   ```css
      <el-header>
        <div class="header">
          <div class="logo">
            <el-icon class="logo-icon"><ChatDotRound /></el-icon>
            <h1>AI 助手</h1>
          </div>
          <el-menu
            class="nav-menu"
            mode="horizontal"
            :router="true"
          >
            <el-menu-item index="/">首页</el-menu-item>
            <el-menu-item index="/history">历史记录</el-menu-item>
            <el-menu-item index="/settings">设置</el-menu-item>
          </el-menu>
        </div>
      </el-header>
   ```

   ### 3.2 实现聊天功能

   1. 添加消息列表组件

      ```css
         <div class="chat-messages">
           <div
             v-for="(message, index) in messages"
             :key="index"
             class="message"
             :class="{ user: message.type === 'user' }"
           >
             <div class="message-content">
               <el-icon class="message-icon">
                 <component :is="message.type === 'user' ? 'User' : 'Service'" />
               </el-icon>
               <div class="message-text">{{ message.content }}</div>
             </div>
           </div>
         </div>
      ```

 2. 添加输入区域：

    ```css
       <div class="input-container">
         <div class="input-header">
           <div class="online-switch">
             <el-switch
               v-model="isOnline"
               active-text="在线"
               inactive-text="离线"
             />
           </div>
           <div class="input-actions">
             <el-upload
               class="upload-btn"
               action="#"
               :show-file-list="false"
               :before-upload="handleImageUpload"
             >
               <el-button type="primary" :icon="Picture">
                 上传图片
               </el-button>
             </el-upload>
             <el-button
               class="recording-btn"
               :type="isRecording ? 'danger' : 'primary'"
               :icon="Microphone"
               circle
               @click="toggleRecording"
             />
           </div>
         </div>
         <el-input
           v-model="userInput"
           type="textarea"
           :rows="3"
           placeholder="输入消息..."
           @keydown.enter.prevent="sendMessage"
         >
           <template #append>
             <el-button
               class="send-button"
               type="primary"
               :loading="loading"
               @click="sendMessage"
             >
               发送
             </el-button>
           </template>
         </el-input>
       </div>
    ```

 3. 添加业务逻辑：

    ```typescript
       // 在 script setup 中添加
       import { ref, onMounted } from 'vue'
       import { ElMessage } from 'element-plus'
       import request from '../utils/request'
    
       const messages = ref<Array<{ type: string; content: string }>>([])
       const userInput = ref('')
       const loading = ref(false)
       const isOnline = ref(true)
       const isRecording = ref(false)
    
       // 发送消息
       const sendMessage = async () => {
         if (!userInput.value.trim() || loading.value) return
    
         try {
           loading.value = true
           const message = userInput.value
           userInput.value = ''
    
           // 添加用户消息
           messages.value.push({
             type: 'user',
             content: message
           })
    
           // 发送请求
           const response = await request.post('/chat', {
             message,
             token: localStorage.getItem('token')
           })
    
           // 添加助手回复
           messages.value.push({
             type: 'assistant',
             content: response.data
           })
         } catch (error) {
           console.error('发送消息失败:', error)
           ElMessage.error('发送消息失败，请重试')
         } finally {
           loading.value = false
         }
       }
    
       // 处理图片上传
       const handleImageUpload = async (file: File) => {
         // 实现图片上传逻辑
       }
    
       // 切换录音状态
       const toggleRecording = () => {
         isRecording.value = !isRecording.value
         // 实现录音逻辑
       }
    ```

### 3.3 添加样式

1. 添加基础样式：

   ```css
      /* src/styles/index.css */
      :root {
        --primary-color: #8E9AAF;
        --secondary-color: #CBC0D3;
        --accent-color: #EFD3D7;
        --text-primary: #5C6B73;
        --text-secondary: #8E9AAF;
        --background-light: #F8F9FA;
        --background-white: #FFFFFF;
      }
   
      body {
        margin: 0;
        font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
      }
   ```

2. 添加组件样式：

   ```css
      /* 在 portal.vue 的 style 标签中添加 */
      .header {
        background: rgba(255, 255, 255, 0.95);
        backdrop-filter: blur(10px);
        border-bottom: 1px solid rgba(142, 154, 175, 0.1);
        padding: 0 24px;
        display: flex;
        align-items: center;
        justify-content: space-between;
        height: 64px;
      }
   
      .chat-container {
        background: rgba(255, 255, 255, 0.95);
        backdrop-filter: blur(10px);
        border-radius: 24px;
        box-shadow: 0 8px 32px rgba(142, 154, 175, 0.08);
        height: calc(100vh - 120px);
        display: flex;
        flex-direction: column;
        overflow: hidden;
      }
   
      .message {
        max-width: 85%;
        animation: fadeIn 0.4s ease-out;
      }
   
      .message.user {
        margin-left: auto;
      }
   
      .message-content {
        display: flex;
        gap: 16px;
        align-items: flex-start;
        padding: 16px 20px;
        border-radius: 16px;
        background: white;
        box-shadow: 0 4px 12px rgba(142, 154, 175, 0.05);
      }
   
      .input-container {
        background: white;
        padding: 24px;
        border-top: 1px solid rgba(142, 154, 175, 0.1);
      }
   ```

   ## 第四章：功能完善

   ### 4.1 添加语音识别功能

   1. 创建语音识别工具：

      ```typescript
         // src/utils/speech.ts
         export class SpeechRecognition {
           private recognition: any
           private isListening: boolean = false
      
           constructor() {
             if ('webkitSpeechRecognition' in window) {
               this.recognition = new (window as any).webkitSpeechRecognition()
               this.recognition.continuous = true
               this.recognition.interimResults = true
               this.recognition.lang = 'zh-CN'
             }
           }
      
           start(callback: (text: string) => void) {
             if (!this.recognition) return
      
             this.recognition.onresult = (event: SpeechRecognitionEvent) => {
               const result = event.results[event.results.length - 1]
               if (result.isFinal) {
                 callback(result[0].transcript)
               }
             }
      
             this.recognition.start()
             this.isListening = true
           }
      
           stop() {
             if (this.recognition && this.isListening) {
               this.recognition.stop()
               this.isListening = false
             }
           }
         }
      ```

   2. 在组件中使用：

      ```typescript
       // 在 portal.vue 中添加
       import { SpeechRecognition } from '../utils/speech'
      
       const speechRecognition = new SpeechRecognition()
      
       const toggleRecording = () => {
         isRecording.value = !isRecording.value
         if (isRecording.value) {
           speechRecognition.start((text) => {
             userInput.value = text
             sendMessage()
           })
         } else {
           speechRecognition.stop()
         }
       }
      ```

      

### 4.2 添加图片上传功能

1. 创建图片上传工具：

   ```typescript
      // src/utils/upload.ts
      import request from './request'
   
      export const uploadImage = async (file: File): Promise<string> => {
        const formData = new FormData()
        formData.append('file', file)
   
        const response = await request.post('/upload', formData, {
          headers: {
            'Content-Type': 'multipart/form-data'
          }
        })
   
        return response.data.url
      }
   ```

2. 在组件中使用：

   ```vue
      // 在 portal.vue 中添加
      import { uploadImage } from '../utils/upload'
   
      const handleImageUpload = async (file: File) => {
        try {
          const url = await uploadImage(file)
          messages.value.push({
            type: 'user',
            content: `<img src="${url}" alt="uploaded image" />`
          })
        } catch (error) {
          console.error('上传图片失败:', error)
          ElMessage.error('上传图片失败，请重试')
        }
      }
   ```

   

## 第五章：项目运行和部署

### 5.1 开发环境运行

1. 创建环境配置文件：

   ```
   # .env.development
   VITE_API_BASE_URL=http://localhost:3000
   ```

2. 启动开发服务器：

   ```
      npm run dev
   ```

### 5.2 生产环境部署

1. 创建生产环境配置：

   ```
   # .env.production
   VITE_API_BASE_URL=https://api.example.com
   ```

2. 构建项目

   ```bash
   npm build
   ```

3. 预览构建结果：

   ```bash
   npm preview
   ```

## 第六章：项目优化

### 6.1 性能优化

1. 添加消息虚拟滚动：

   ```css
      <el-scrollbar>
        <div class="chat-messages">
          <virtual-list
            :data-key="'id'"
            :data-sources="messages"
            :data-component="MessageItem"
            :estimate-size="80"
            :direction="'vertical'"
          />
        </div>
      </el-scrollbar>
   ```

2. 添加图片懒加载：

   ```css
      <img
        v-lazy="imageUrl"
        alt="message image"
      />
   ```

### 6.2 用户体验优化

1. 添加消息发送状态：

   ```css
   <div class="message-status">
        <el-icon v-if="message.status === 'sending'">
          <Loading />
        </el-icon>
        <el-icon v-else-if="message.status === 'sent'">
          <Check />
        </el-icon>
      </div>
   ```

2. 添加输入提示：

   ```vue
      <el-input
        v-model="userInput"
        type="textarea"
        :rows="3"
        placeholder="输入消息..."
        @keydown.enter.prevent="sendMessage"
        :maxlength="500"
        show-word-limit
      />
   ```

   