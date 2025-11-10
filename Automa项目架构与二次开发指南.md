# Automa 项目架构与二次开发指南

## 项目概述
Automa 是一个功能强大的浏览器自动化扩展，支持 Chrome 和 Firefox。它采用可视化流程图的方式，让用户通过拖拽连接功能块来创建自动化工作流，实现网页操作、数据抓取、表单填写等功能。

## 技术栈
- **前端框架**: Vue 3 + Vue Router + Pinia
- **构建工具**: Webpack 5
- **UI 组件**: Tailwind CSS + 自定义组件
- **数据存储**: IndexedDB (Dexie) + Chrome Storage API
- **国际化**: Vue I18n
- **代码编辑**: CodeMirror 6
- **图形编辑**: Vue Flow (流程图编辑器)
- **图标**: Remixicon
- **开发语言**: JavaScript (ES6+)

## 项目架构

### 1. 入口文件结构
```
src/
├── background/          # 后台脚本 (Service Worker/Background Page)
├── content/            # 内容脚本 (Content Scripts)
├── newtab/            # 新标签页 (主界面)
├── popup/             # 弹出窗口
├── offscreen/         # 离屏文档 (MV3)
├── sandbox/           # 沙盒环境
├── params/            # 参数输入页面
└── execute/           # 工作流执行页面
```

### 2. 核心模块划分

#### 2.1 后台服务 (`src/background/`)
```javascript
// 主要文件
├── index.js                    # 后台脚本入口
├── BackgroundUtils.js          # 工具函数
├── BackgroundOffscreen.js      # 离屏文档管理
├── BackgroundEventsListeners.js # 事件监听器
├── BackgroundWorkflowUtils.js  # 工作流工具
└── BackgroundWorkflowTriggers.js # 工作流触发器
```

**核心功能：**
- 工作流执行管理
- 浏览器 API 代理
- 消息路由
- 下载管理
- 权限管理
- 定时器和触发器

#### 2.2 内容脚本 (`src/content/`)
```javascript
├── index.js                    # 内容脚本入口
├── blocksHandler/             # 功能块处理器
├── services/                  # 服务模块
│   ├── recordWorkflow/       # 录制服务
│   └── webService.js         # Web 服务
├── elementSelector/          # 元素选择器
└── commandPalette/           # 命令面板
```

**核心功能：**
- DOM 操作和事件处理
- 元素选择器
- 工作流录制
- 页面注入和交互

#### 2.3 工作流引擎 (`src/workflowEngine/`)
```javascript
├── WorkflowEngine.js          # 工作流引擎核心
├── WorkflowWorker.js         # 工作流执行器
├── WorkflowManager.js        # 工作流管理器
├── WorkflowLogger.js         # 日志记录器
├── WorkflowState.js          # 状态管理
├── blocksHandler/            # 功能块处理器
├── templating/               # 模板引擎
└── utils/                    # 工具函数
```

**核心功能：**
- 工作流解析和执行
- 功能块调度
- 状态管理
- 错误处理
- 日志记录

#### 2.4 主界面 (`src/newtab/`)
```javascript
├── App.vue                   # 主应用组件
├── router.js                 # 路由配置
├── pages/                    # 页面组件
│   ├── Workflows.vue        # 工作流列表
│   ├── Recording.vue        # 录制页面
│   ├── Settings.vue         # 设置页面
│   └── logs/               # 日志页面
└── utils/                   # 工具函数
```

#### 2.5 状态管理 (`src/stores/`)
```javascript
├── main.js                  # 主状态
├── workflow.js              # 工作流状态
├── user.js                  # 用户状态
├── folder.js                # 文件夹状态
├── package.js               # 包管理状态
├── teamWorkflow.js          # 团队工作流
├── sharedWorkflow.js        # 共享工作流
└── hostedWorkflow.js        # 托管工作流
```

### 3. 功能块系统

#### 3.1 功能块定义 (`src/utils/shared.js`)
包含所有可用的功能块定义，每个功能块包含：
- 基本信息 (名称、图标、描述)
- 组件类型
- 编辑组件
- 输入输出配置
- 默认数据结构

#### 3.2 功能块处理器 (`src/workflowEngine/blocksHandler/`)
```javascript
├── handlerTakeScreenshot.js    # 截图处理
├── handlerSaveAssets.js        # 资源保存
├── handlerForms.js             # 表单操作
├── handlerJavascriptCode.js    # JavaScript 执行
├── handlerWebhook.js           # HTTP 请求
├── handlerGoogleSheets.js      # Google Sheets 集成
└── ... (50+ 处理器)
```

## 核心流程分析

### 1. 工作流执行流程
```
用户触发 → BackgroundWorkflowUtils → WorkflowEngine → WorkflowWorker → 功能块处理器 → 内容脚本 → DOM 操作
```

### 2. 录制功能流程
```
用户点击录制 → 注入录制脚本 → 监听用户行为 → 生成功能块 → 构建工作流 → 保存到存储
```

### 3. 数据流
```
用户输入 → 模板处理 → 变量替换 → 功能块执行 → 结果存储 → 日志记录
```

## 视频录制功能详细分析

### 1. 工作流录制系统

#### 1.1 录制服务架构 (`src/content/services/recordWorkflow/`)
```javascript
├── index.js              # 录制服务入口
├── recordEvents.js       # 事件录制处理
├── addBlock.js          # 功能块添加逻辑
└── main.js              # 录制主逻辑
```

#### 1.2 录制流程详解

**启动录制：**
```javascript
// src/newtab/utils/startRecordWorkflow.js
export default async function (options = {}) {
  // 1. 设置录制状态
  await browser.storage.local.set({
    isRecording: true,
    recording: {
      flows: [],
      name: 'unnamed',
      activeTab: { id: activeTab?.id, url: activeTab?.url },
      ...options,
    },
  });

  // 2. 设置扩展图标为录制状态
  await action.setBadgeBackgroundColor({ color: '#ef4444' });
  await action.setBadgeText({ text: 'rec' });

  // 3. 向所有标签页注入录制脚本
  for (const tab of tabs) {
    await browser.scripting.executeScript({
      target: { tabId: tab.id, allFrames: true },
      files: ['recordWorkflow.bundle.js'],
    });
  }
}
```

**事件捕获机制：**
```javascript
// src/content/services/recordWorkflow/recordEvents.js
export default async function (mainFrame) {
  if (isRecording) {
    // 注册各种事件监听器
    document.addEventListener('click', onClick, true);
    document.addEventListener('change', onChange, true);
    document.addEventListener('focusin', onFocusIn, true);
    document.addEventListener('keydown', onKeydown, true);
    document.addEventListener('scroll', onScroll, true);
    
    if (mainFrame) {
      window.addEventListener('message', onMessage);
    }
  }
}
```

**支持的录制事件类型：**
- **点击事件**: 按钮、链接、图片等元素点击
- **表单操作**: 输入框填写、下拉选择、复选框勾选
- **键盘操作**: 按键组合、文本输入、快捷键
- **页面导航**: 新标签页创建、标签页切换、页面跳转
- **滚动操作**: 页面滚动、元素滚动
- **文件上传**: 文件选择和上传

#### 1.3 跨框架录制支持
```javascript
// 支持 iframe 内的操作录制
const onMessage = debounce(({ data, source }) => {
  if (data.type !== 'automa:record-events') return;
  
  // 处理来自 iframe 的录制数据
  let { frameSelector } = data;
  if (!frameSelector) {
    const frames = document.querySelectorAll('iframe, frame');
    frames.forEach((frame) => {
      if (frame.contentWindow !== source) return;
      frameSelector = finder(frame);
    });
  }
  
  // 更新选择器以包含框架路径
  data.recording.flows[lastIndex].data.selector = 
    `${frameSelector} |> ${lastFlow.data.selector}`;
}, 100);
```

### 2. 截图和媒体捕获功能

#### 2.1 截图处理器 (`src/workflowEngine/blocksHandler/handlerTakeScreenshot.js`)
```javascript
async function takeScreenshot({ data, id, label }) {
  const options = {
    quality: data.quality,      // 图片质量 (0-100)
    format: data.ext || 'png',  // 格式 (png/jpeg)
  };

  if (data.captureActiveTab) {
    // 捕获活动标签页
    if (data.fullPage || data.type === 'fullpage') {
      // 全页截图 - 通过内容脚本实现
      screenshot = await this._sendMessageToTab({
        label,
        options,
        data: { type: data.type, selector: data.selector },
        tabId: this.activeTab.id,
      });
    } else {
      // 可视区域截图
      screenshot = await BrowserAPIService.tabs.captureVisibleTab(options);
    }
  }

  // 保存截图
  await saveScreenshot(screenshot);
}
```

#### 2.2 媒体资源保存 (`src/workflowEngine/blocksHandler/handlerSaveAssets.js`)
```javascript
export default async function ({ data, id, label }) {
  let sources = [data.url];
  
  if (data.type === 'element') {
    // 从页面元素提取媒体 URL
    sources = await this._sendMessageToTab({
      id, data, label,
      tabId: this.activeTab.id,
    });
  }
  
  // 下载文件
  const downloadIds = await Promise.all(
    sources.map((url) => downloadFile(url))
  );
  
  return {
    data: { sources, downloadIds },
    nextBlockId: this.getBlockConnections(id),
  };
}
```

**支持的媒体类型：**
- 图片 (IMG 标签)
- 视频 (VIDEO 标签) 
- 音频 (AUDIO 标签)
- 任意文件 URL

### 3. 录制界面组件 (`src/newtab/pages/Recording.vue`)

#### 3.1 实时预览功能
```vue
<template>
  <ui-list class="space-y-1">
    <ui-list-item v-for="(item, index) in state.flows" :key="index">
      <v-remixicon :name="tasks[item.id].icon" />
      <div class="mx-2 flex-1 overflow-hidden">
        <p>{{ t(`workflow.blocks.${item.id}.name`) }}</p>
        <p class="text-sm text-gray-600">
          {{ item.data.description || item.description }}
        </p>
      </div>
      <v-remixicon 
        name="riDeleteBin7Line"
        @click="removeBlock(index)"
      />
    </ui-list-item>
  </ui-list>
</template>
```

#### 3.2 录制状态管理
```javascript
// 监听录制数据变化
function onStorageChanged({ recording }) {
  if (!recording) return;
  Object.assign(state, recording.newValue);
}

// 监听浏览器事件
browser.tabs.onCreated.addListener(browserEvents.onTabCreated);
browser.tabs.onActivated.addListener(browserEvents.onTabsActivated);
browser.webNavigation.onCommitted.addListener(browserEvents.onCommitted);
```

## 二次开发指南

### 1. 开发环境搭建

```bash
# 克隆项目
git clone https://github.com/AutomaApp/automa.git
cd automa

# 安装依赖
pnpm install

# 创建必需文件
echo "export default function() { return 'your-pass-key'; }" > src/utils/getPassKey.js

# 开发模式 (Chrome)
pnpm dev

# 开发模式 (Firefox)
pnpm dev:firefox

# 构建生产版本
pnpm build
```

### 2. 添加新功能块

#### 2.1 定义功能块 (`src/utils/shared.js`)
```javascript
export const tasks = {
  'your-block': {
    name: 'Your Block Name',
    description: 'Block description',
    icon: 'riYourIcon',
    component: 'BlockBasic',
    editComponent: 'EditYourBlock',
    category: 'interaction',
    inputs: 1,
    outputs: 1,
    allowedInputs: true,
    maxConnection: 1,
    refDataKeys: ['selector', 'variableName'],
    autocomplete: ['variableName'],
    data: {
      disableBlock: false,
      description: '',
      selector: '',
      variableName: '',
      // 其他默认配置
    }
  }
}
```

#### 2.2 创建处理器 (`src/workflowEngine/blocksHandler/handlerYourBlock.js`)
```javascript
export default async function ({ data, id, label }) {
  try {
    // 验证输入数据
    if (!data.selector) {
      throw new Error('Selector is required');
    }
    
    // 执行功能逻辑
    const result = await this._sendMessageToTab({
      id,
      data,
      label,
      tabId: this.activeTab.id,
    });
    
    // 处理变量赋值
    if (data.assignVariable) {
      await this.setVariable(data.variableName, result);
    }
    
    // 处理数据列存储
    if (data.saveData) {
      this.addDataToColumn(data.dataColumn, result);
    }
    
    return {
      data: result,
      nextBlockId: this.getBlockConnections(id),
    };
  } catch (error) {
    throw error;
  }
}
```

#### 2.3 创建编辑组件 (`src/components/block/edit/EditYourBlock.vue`)
```vue
<template>
  <div class="space-y-4">
    <ui-input
      v-model="data.selector"
      label="CSS Selector"
      placeholder="Enter CSS selector"
    />
    
    <ui-input
      v-model="data.description"
      label="Description"
      placeholder="Optional description"
    />
    
    <!-- 变量赋值选项 -->
    <ui-expand>
      <template #header>
        <ui-checkbox v-model="data.assignVariable">
          Assign to variable
        </ui-checkbox>
      </template>
      
      <ui-input
        v-if="data.assignVariable"
        v-model="data.variableName"
        label="Variable name"
        placeholder="variableName"
      />
    </ui-expand>
    
    <!-- 数据存储选项 -->
    <ui-expand>
      <template #header>
        <ui-checkbox v-model="data.saveData">
          Save to table
        </ui-checkbox>
      </template>
      
      <ui-select
        v-if="data.saveData"
        v-model="data.dataColumn"
        :options="dataColumns"
        label="Column"
      />
    </ui-expand>
  </div>
</template>

<script setup>
import { computed } from 'vue';

const props = defineProps(['data']);
const emit = defineEmits(['change']);

// 监听数据变化
const data = computed({
  get: () => props.data,
  set: (value) => emit('change', value),
});

// 获取数据列选项
const dataColumns = computed(() => {
  // 从工作流获取数据列
  return workflowStore.workflow?.dataColumns || [];
});
</script>
```

### 3. 扩展录制功能

#### 3.1 添加新的录制事件 (`src/content/services/recordWorkflow/recordEvents.js`)
```javascript
// 添加自定义事件录制
function onYourCustomEvent(event) {
  if (isAutomaInstance(event.target)) return;
  
  const selector = findSelector(event.target);
  
  addBlock({
    id: 'your-block',
    description: `Custom action on ${event.target.tagName}`,
    data: {
      selector,
      eventType: event.type,
      customData: extractCustomData(event),
      waitForSelector: true,
    },
  });
}

// 在录制初始化时注册事件
export default async function (mainFrame) {
  const { isRecording } = await browser.storage.local.get('isRecording');
  
  if (isRecording) {
    // 注册自定义事件
    document.addEventListener('yourcustomevent', onYourCustomEvent, true);
    
    // 其他事件监听器...
  }
  
  // 返回清理函数
  return function cleanUp() {
    document.removeEventListener('yourcustomevent', onYourCustomEvent, true);
    // 其他清理...
  };
}
```

#### 3.2 扩展录制数据结构
```javascript
// src/content/services/recordWorkflow/addBlock.js
export default async function (detail, save = true) {
  const { isRecording, recording } = await browser.storage.local.get([
    'isRecording',
    'recording',
  ]);

  if (!isRecording || !recording) return null;

  let addedBlock = null;

  if (typeof detail === 'function') {
    addedBlock = detail(recording);
  } else {
    // 扩展功能块数据
    const enhancedDetail = {
      ...detail,
      timestamp: Date.now(),
      tabId: await getCurrentTabId(),
      frameSelector: getFrameSelector(),
    };
    
    recording.flows.push(enhancedDetail);
    addedBlock = enhancedDetail;
  }

  if (save) await browser.storage.local.set({ recording });

  return { recording, addedBlock };
}
```

### 4. 添加新的数据源

#### 4.1 扩展数据库结构 (`src/db/storage.js`)
```javascript
import Dexie from 'dexie';

export const dbStorage = new Dexie('automa-db');

dbStorage.version(1).stores({
  // 现有表...
  yourTable: '++id, name, data, createdAt, updatedAt',
  yourRelatedTable: '++id, yourTableId, metadata, createdAt',
});

// 添加数据访问方法
export const yourTableOperations = {
  async create(data) {
    return await dbStorage.yourTable.add({
      ...data,
      createdAt: Date.now(),
      updatedAt: Date.now(),
    });
  },
  
  async getAll() {
    return await dbStorage.yourTable.toArray();
  },
  
  async getById(id) {
    return await dbStorage.yourTable.get(id);
  },
  
  async update(id, data) {
    return await dbStorage.yourTable.update(id, {
      ...data,
      updatedAt: Date.now(),
    });
  },
  
  async delete(id) {
    return await dbStorage.yourTable.delete(id);
  },
};
```

#### 4.2 创建对应的 Store (`src/stores/yourStore.js`)
```javascript
import { defineStore } from 'pinia';
import { yourTableOperations } from '@/db/storage';

export const useYourStore = defineStore('yourStore', {
  state: () => ({
    items: {},
    loading: false,
    error: null,
  }),
  
  getters: {
    getAllItems: (state) => Object.values(state.items),
    getItemById: (state) => (id) => state.items[id],
  },
  
  actions: {
    async loadData() {
      this.loading = true;
      try {
        const items = await yourTableOperations.getAll();
        this.items = items.reduce((acc, item) => {
          acc[item.id] = item;
          return acc;
        }, {});
      } catch (error) {
        this.error = error;
        console.error('Failed to load your data:', error);
      } finally {
        this.loading = false;
      }
    },
    
    async createItem(data) {
      const id = await yourTableOperations.create(data);
      const newItem = { ...data, id };
      this.items[id] = newItem;
      return newItem;
    },
    
    async updateItem(id, data) {
      await yourTableOperations.update(id, data);
      this.items[id] = { ...this.items[id], ...data };
    },
    
    async deleteItem(id) {
      await yourTableOperations.delete(id);
      delete this.items[id];
    },
  },
});
```

### 5. 集成外部服务

#### 5.1 添加 API 服务 (`src/utils/yourServiceAPI.js`)
```javascript
import { fetchApi } from './api';

export class YourServiceAPI {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.baseURL = 'https://api.yourservice.com/v1';
  }
  
  async authenticate() {
    const response = await fetchApi(`${this.baseURL}/auth`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
    });
    
    if (!response.ok) {
      throw new Error('Authentication failed');
    }
    
    return await response.json();
  }
  
  async getData(params = {}) {
    const queryString = new URLSearchParams(params).toString();
    const url = `${this.baseURL}/data?${queryString}`;
    
    const response = await fetchApi(url, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
      },
    });
    
    if (!response.ok) {
      throw new Error(`API request failed: ${response.status}`);
    }
    
    return await response.json();
  }
  
  async uploadData(data) {
    const response = await fetchApi(`${this.baseURL}/upload`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data),
    });
    
    if (!response.ok) {
      throw new Error('Upload failed');
    }
    
    return await response.json();
  }
}
```

#### 5.2 创建服务处理器 (`src/workflowEngine/blocksHandler/handlerYourService.js`)
```javascript
import { YourServiceAPI } from '@/utils/yourServiceAPI';

export default async function ({ data, id }) {
  try {
    // 验证 API 密钥
    if (!data.apiKey) {
      throw new Error('API key is required');
    }
    
    const api = new YourServiceAPI(data.apiKey);
    
    // 根据操作类型执行不同逻辑
    let result;
    switch (data.operation) {
      case 'fetch':
        result = await api.getData(data.params);
        break;
        
      case 'upload':
        result = await api.uploadData(data.uploadData);
        break;
        
      default:
        throw new Error(`Unsupported operation: ${data.operation}`);
    }
    
    // 处理结果
    if (data.assignVariable) {
      await this.setVariable(data.variableName, result);
    }
    
    if (data.saveData) {
      this.addDataToColumn(data.dataColumn, result);
    }
    
    return {
      data: result,
      nextBlockId: this.getBlockConnections(id),
    };
  } catch (error) {
    throw new Error(`Your Service API Error: ${error.message}`);
  }
}
```

## 关键开发注意事项

### 1. 消息传递系统

#### 1.1 消息监听器 (`src/utils/message.js`)
```javascript
export class MessageListener {
  constructor(context) {
    this.context = context;
    this.handlers = new Map();
    this.listener = this.listener.bind(this);
  }
  
  on(event, handler) {
    this.handlers.set(event, handler);
  }
  
  async listener(message, sender, sendResponse) {
    const handler = this.handlers.get(message.type);
    if (!handler) return;
    
    try {
      const result = await handler(message.data, sender);
      if (sendResponse) sendResponse({ success: true, data: result });
      return result;
    } catch (error) {
      console.error(`Message handler error for ${message.type}:`, error);
      if (sendResponse) sendResponse({ success: false, error: error.message });
      throw error;
    }
  }
  
  static async sendMessage(type, data, context) {
    return await browser.runtime.sendMessage({ type, data, context });
  }
}
```

#### 1.2 跨上下文通信示例
```javascript
// 后台脚本
const message = new MessageListener('background');
message.on('your-action', async (data, sender) => {
  // 处理来自内容脚本的消息
  const result = await processYourAction(data);
  return result;
});
browser.runtime.onMessage.addListener(message.listener);

// 内容脚本
const result = await MessageListener.sendMessage('your-action', { 
  param1: 'value1' 
}, 'background');
```

### 2. 权限管理

#### 2.1 动态权限检查
```javascript
import BrowserAPIService from '@/service/browser-api/BrowserAPIService';

export async function checkRequiredPermissions(blockData) {
  const requiredPermissions = getRequiredPermissions(blockData);
  
  for (const permission of requiredPermissions) {
    const hasPermission = await BrowserAPIService.permissions.contains({
      permissions: [permission],
    });
    
    if (!hasPermission) {
      throw new Error(`Missing permission: ${permission}`);
    }
  }
}

function getRequiredPermissions(blockData) {
  const permissions = [];
  
  if (blockData.id === 'save-assets') {
    permissions.push('downloads');
  }
  
  if (blockData.id === 'take-screenshot') {
    permissions.push('activeTab');
  }
  
  // 添加其他权限检查...
  
  return permissions;
}
```

#### 2.2 权限请求界面
```vue
<!-- src/components/newtab/shared/SharedPermissionsModal.vue -->
<template>
  <ui-modal v-model="show" title="Permissions Required">
    <div class="space-y-4">
      <p>This workflow requires the following permissions:</p>
      
      <div class="space-y-2">
        <div 
          v-for="permission in permissions" 
          :key="permission.name"
          class="flex items-center space-x-2"
        >
          <ui-checkbox :checked="permission.granted" disabled />
          <span>{{ permission.name }}</span>
          <span class="text-sm text-gray-500">{{ permission.reason }}</span>
        </div>
      </div>
      
      <div class="flex space-x-2">
        <ui-button @click="requestPermissions" variant="accent">
          Grant Permissions
        </ui-button>
        <ui-button @click="show = false">
          Cancel
        </ui-button>
      </div>
    </div>
  </ui-modal>
</template>
```

### 3. 错误处理

#### 3.1 统一错误处理
```javascript
// src/utils/errorHandler.js
export class WorkflowError extends Error {
  constructor(message, blockId, blockName, data = {}) {
    super(message);
    this.name = 'WorkflowError';
    this.blockId = blockId;
    this.blockName = blockName;
    this.data = data;
  }
}

export function handleBlockError(error, blockData) {
  if (error instanceof WorkflowError) {
    return error;
  }
  
  return new WorkflowError(
    error.message,
    blockData.id,
    blockData.name,
    { originalError: error }
  );
}

// 全局错误捕获
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled promise rejection:', event.reason);
  // 可以在这里添加错误上报逻辑
});
```

#### 3.2 错误重试机制
```javascript
// src/utils/retry.js
export async function retryOperation(
  operation, 
  maxRetries = 3, 
  delay = 1000,
  backoff = 2
) {
  let lastError;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;
      
      if (i < maxRetries - 1) {
        await new Promise(resolve => 
          setTimeout(resolve, delay * Math.pow(backoff, i))
        );
      }
    }
  }
  
  throw lastError;
}

// 使用示例
const result = await retryOperation(
  async () => await fetch('/api/data'),
  3,  // 最多重试3次
  1000, // 初始延迟1秒
  2   // 指数退避
);
```

### 4. 性能优化

#### 4.1 延迟加载组件
```javascript
// src/router/index.js
const routes = [
  {
    path: '/workflows',
    component: () => import('@/pages/Workflows.vue'),
  },
  {
    path: '/recording',
    component: () => import('@/pages/Recording.vue'),
  },
  // 其他路由...
];
```

#### 4.2 虚拟滚动优化
```vue
<!-- src/components/ui/VirtualList.vue -->
<template>
  <div ref="container" class="virtual-list" @scroll="onScroll">
    <div :style="{ height: totalHeight + 'px' }">
      <div 
        :style="{ transform: `translateY(${offsetY}px)` }"
        class="virtual-list-items"
      >
        <div
          v-for="item in visibleItems"
          :key="item.id"
          :style="{ height: itemHeight + 'px' }"
          class="virtual-list-item"
        >
          <slot :item="item" />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

const props = defineProps({
  items: Array,
  itemHeight: { type: Number, default: 50 },
});

const container = ref(null);
const scrollTop = ref(0);
const containerHeight = ref(0);

const visibleCount = computed(() => 
  Math.ceil(containerHeight.value / props.itemHeight) + 2
);

const startIndex = computed(() => 
  Math.max(0, Math.floor(scrollTop.value / props.itemHeight) - 1)
);

const visibleItems = computed(() => 
  props.items.slice(
    startIndex.value, 
    startIndex.value + visibleCount.value
  )
);

const totalHeight = computed(() => 
  props.items.length * props.itemHeight
);

const offsetY = computed(() => 
  startIndex.value * props.itemHeight
);

function onScroll() {
  scrollTop.value = container.value.scrollTop;
}
</script>
```

#### 4.3 内存管理
```javascript
// src/utils/memoryManager.js
export class MemoryManager {
  static cache = new Map();
  static maxCacheSize = 100;
  
  static set(key, value) {
    if (this.cache.size >= this.maxCacheSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, {
      value,
      timestamp: Date.now(),
    });
  }
  
  static get(key, maxAge = 5 * 60 * 1000) {
    const cached = this.cache.get(key);
    
    if (!cached) return null;
    
    if (Date.now() - cached.timestamp > maxAge) {
      this.cache.delete(key);
      return null;
    }
    
    return cached.value;
  }
  
  static clear() {
    this.cache.clear();
  }
}

// 定期清理过期缓存
setInterval(() => {
  MemoryManager.clear();
}, 10 * 60 * 1000); // 每10分钟清理一次
```

### 5. 兼容性考虑

#### 5.1 浏览器差异处理
```javascript
// src/utils/browserCompat.js
export const IS_FIREFOX = /Firefox/.test(navigator.userAgent);
export const IS_CHROME = /Chrome/.test(navigator.userAgent);
export const IS_EDGE = /Edg/.test(navigator.userAgent);

export function getBrowserAPI() {
  if (typeof browser !== 'undefined') {
    return browser; // Firefox
  } else if (typeof chrome !== 'undefined') {
    return chrome;  // Chrome
  }
  throw new Error('Unsupported browser');
}

// Manifest V2/V3 兼容
export const manifestVersion = getBrowserAPI().runtime.getManifest().manifest_version;

export function executeScript(tabId, details) {
  const api = getBrowserAPI();
  
  if (manifestVersion === 3) {
    return api.scripting.executeScript({
      target: { tabId },
      ...details,
    });
  } else {
    return api.tabs.executeScript(tabId, details);
  }
}
```

#### 5.2 CSP (内容安全策略) 处理
```javascript
// src/utils/cspHandler.js
export async function injectScriptWithCSP(tabId, script) {
  try {
    // 尝试常规注入
    await browser.scripting.executeScript({
      target: { tabId },
      func: new Function(script),
    });
  } catch (error) {
    if (error.message.includes('Content Security Policy')) {
      // CSP 阻止，使用 debugger API
      await injectWithDebugger(tabId, script);
    } else {
      throw error;
    }
  }
}

async function injectWithDebugger(tabId, script) {
  await chrome.debugger.attach({ tabId }, '1.3');
  
  try {
    await chrome.debugger.sendCommand(
      { tabId },
      'Runtime.evaluate',
      { expression: script }
    );
  } finally {
    await chrome.debugger.detach({ tabId });
  }
}
```

## 调试技巧

### 1. 开发者工具使用

#### 1.1 后台脚本调试
```bash
# Chrome
chrome://extensions -> 开发者模式 -> 检查视图: Service Worker

# Firefox  
about:debugging -> 此Firefox -> 检查
```

#### 1.2 内容脚本调试
```javascript
// 在内容脚本中添加断点
debugger;

// 使用 console 分组
console.group('Automa Content Script');
console.log('Element found:', element);
console.groupEnd();
```

#### 1.3 工作流执行调试
```javascript
// src/workflowEngine/WorkflowEngine.js
if (this.workflow.settings.debugMode) {
  console.log('Executing block:', blockData);
  
  // 添加执行时间统计
  const startTime = performance.now();
  const result = await executeBlock(blockData);
  const endTime = performance.now();
  
  console.log(`Block ${blockData.id} took ${endTime - startTime}ms`);
}
```

### 2. 日志系统

#### 2.1 结构化日志
```javascript
// src/utils/logger.js
export class Logger {
  static levels = {
    ERROR: 0,
    WARN: 1,
    INFO: 2,
    DEBUG: 3,
  };
  
  static level = this.levels.INFO;
  
  static log(level, message, data = {}) {
    if (level > this.level) return;
    
    const logEntry = {
      timestamp: new Date().toISOString(),
      level: Object.keys(this.levels)[level],
      message,
      data,
      stack: new Error().stack,
    };
    
    console.log(JSON.stringify(logEntry, null, 2));
    
    // 可以在这里添加远程日志上报
    this.sendToRemote(logEntry);
  }
  
  static error(message, data) {
    this.log(this.levels.ERROR, message, data);
  }
  
  static warn(message, data) {
    this.log(this.levels.WARN, message, data);
  }
  
  static info(message, data) {
    this.log(this.levels.INFO, message, data);
  }
  
  static debug(message, data) {
    this.log(this.levels.DEBUG, message, data);
  }
  
  static async sendToRemote(logEntry) {
    // 实现远程日志上报逻辑
    try {
      await fetch('/api/logs', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(logEntry),
      });
    } catch (error) {
      console.error('Failed to send log to remote:', error);
    }
  }
}
```

### 3. 测试策略

#### 3.1 单元测试
```javascript
// tests/unit/blockHandler.test.js
import { describe, it, expect, vi } from 'vitest';
import handlerYourBlock from '@/workflowEngine/blocksHandler/handlerYourBlock';

describe('handlerYourBlock', () => {
  it('should execute successfully with valid data', async () => {
    const mockContext = {
      _sendMessageToTab: vi.fn().mockResolvedValue('test result'),
      setVariable: vi.fn(),
      getBlockConnections: vi.fn().mockReturnValue('next-block-id'),
    };
    
    const blockData = {
      data: { selector: '.test', assignVariable: true, variableName: 'testVar' },
      id: 'test-block',
      label: 'test',
    };
    
    const result = await handlerYourBlock.call(mockContext, blockData);
    
    expect(result.data).toBe('test result');
    expect(result.nextBlockId).toBe('next-block-id');
    expect(mockContext.setVariable).toHaveBeenCalledWith('testVar', 'test result');
  });
});
```

#### 3.2 集成测试
```javascript
// tests/integration/workflow.test.js
import { describe, it, expect } from 'vitest';
import WorkflowEngine from '@/workflowEngine/WorkflowEngine';

describe('Workflow Integration', () => {
  it('should execute complete workflow', async () => {
    const workflow = {
      drawflow: {
        nodes: [
          { id: 'trigger', label: 'trigger' },
          { id: 'action', label: 'your-block' },
        ],
        edges: [
          { source: 'trigger', target: 'action' },
        ],
      },
    };
    
    const engine = new WorkflowEngine(workflow, {
      // mock dependencies
    });
    
    await engine.init();
    
    // 验证工作流执行结果
    expect(engine.history).toHaveLength(2);
  });
});
```

## 部署指南

### 1. 构建配置

#### 1.1 环境变量配置
```bash
# .env.development
NODE_ENV=development
BROWSER=chrome
ASSET_PATH=/

# .env.production
NODE_ENV=production
BROWSER=chrome
ASSET_PATH=/
```

#### 1.2 自定义构建脚本
```javascript
// scripts/build-custom.js
const { execSync } = require('child_process');
const fs = require('fs-extra');

async function buildForAllBrowsers() {
  const browsers = ['chrome', 'firefox'];
  
  for (const browser of browsers) {
    console.log(`Building for ${browser}...`);
    
    process.env.BROWSER = browser;
    execSync(`npm run build`, { stdio: 'inherit' });
    
    // 复制构建结果
    await fs.copy('build', `dist/${browser}`);
    
    console.log(`${browser} build completed`);
  }
}

buildForAllBrowsers().catch(console.error);
```

### 2. 版本管理

#### 2.1 自动版本更新
```javascript
// scripts/update-version.js
const fs = require('fs');
const path = require('path');

function updateVersion() {
  const packagePath = path.join(__dirname, '../package.json');
  const package = JSON.parse(fs.readFileSync(packagePath, 'utf8'));
  
  const manifestPaths = [
    'src/manifest.chrome.json',
    'src/manifest.firefox.json',
  ];
  
  manifestPaths.forEach(manifestPath => {
    const manifest = JSON.parse(fs.readFileSync(manifestPath, 'utf8'));
    manifest.version = package.version;
    fs.writeFileSync(manifestPath, JSON.stringify(manifest, null, 2));
  });
}

updateVersion();
```

### 3. 商店发布

#### 3.1 Chrome Web Store
```bash
# 使用 Chrome Web Store API
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
     -H "x-goog-api-version: 2" \
     -X PUT \
     -T automa.zip \
     https://www.googleapis.com/upload/chromewebstore/v1.1/items/$APP_ID
```

#### 3.2 Firefox Add-ons
```bash
# 使用 web-ext 工具
npm install -g web-ext

web-ext build --source-dir=build --artifacts-dir=dist
web-ext sign --source-dir=build --api-key=$API_KEY --api-secret=$API_SECRET
```

## 总结

这份指南涵盖了 Automa 项目的完整架构分析和二次开发方案：

1. **项目架构**：详细介绍了各个模块的职责和交互关系
2. **录制功能**：深入分析了视频录制脚本的实现原理
3. **开发指南**：提供了添加新功能、扩展系统的具体方法
4. **最佳实践**：涵盖了错误处理、性能优化、兼容性等关键要点
5. **调试部署**：给出了完整的测试、调试和发布流程

基于这个指南，您可以快速上手 Automa 项目的二次开发，添加自定义功能块、集成外部服务，或者创建自己的浏览器自动化解决方案。

## 相关资源

- [Automa 官方文档](https://docs.automa.site/)
- [Chrome 扩展开发文档](https://developer.chrome.com/docs/extensions/)
- [Firefox 扩展开发文档](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions)
- [Vue 3 官方文档](https://vuejs.org/)
- [Pinia 状态管理](https://pinia.vuejs.org/)
- [Tailwind CSS](https://tailwindcss.com/)
