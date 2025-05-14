## 4 系统开发

### 4.1 源程序清单

| 文件名 | 所属功能 | 文件描述 |
| --- | --- | --- |
| **前端部分** |  |  |
| index.html | 前端入口 | 应用程序主页面，用户访问的入口点 |
| main.js | 前端入口 | Vue应用程序的主入口文件，负责初始化应用和挂载根组件 |
| App.vue | 前端框架 | 应用程序的根组件，包含全局导航和布局 |
| router.js | 路由管理 | 定义前端路由，负责页面间的导航 |
| store.js | 状态管理 | Vuex状态管理库配置，管理全局应用状态 |
| components/Header.vue | UI组件 | 页面顶部导航栏组件 |
| components/Footer.vue | UI组件 | 页面底部信息组件 |
| components/Sidebar.vue | UI组件 | 侧边栏导航组件 |
| views/Login.vue | 用户管理 | 用户登录页面 |
| views/Register.vue | 用户管理 | 用户注册页面 |
| views/Dashboard.vue | 数据展示 | 用户仪表盘，展示概览信息和统计数据 |
| views/PromptManage.vue | Prompt管理 | Prompt列表和管理页面 |
| views/PromptEditor.vue | Prompt管理 | Prompt编辑界面，支持语法高亮 |
| views/DataImport.vue | 数据导入 | 数据导入界面，支持多种导入方式 |
| views/Visualization.vue | 数据可视化 | 数据可视化页面，展示各类图表 |
| utils/api.js | API工具 | 封装后端API请求方法 |
| utils/auth.js | 用户管理 | 处理用户认证相关逻辑 |
| utils/formatter.js | 数据处理 | 数据格式化工具函数 |
| **后端部分** |  |  |
| app.py | 后端入口 | Flask应用程序主入口，配置路由和中间件 |
| config.py | 系统配置 | 系统配置文件，包含数据库连接等配置 |
| models/user.py | 用户管理 | 用户模型，定义用户数据结构和方法 |
| models/prompt.py | Prompt管理 | Prompt模型，定义Prompt数据结构和方法 |
| models/data.py | 数据存储 | 数据模型，定义AI处理结果数据结构 |
| controllers/user_controller.py | 用户管理 | 用户相关API控制器 |
| controllers/prompt_controller.py | Prompt管理 | Prompt相关API控制器 |
| controllers/data_controller.py | 数据导入 | 数据导入和处理相关API控制器 |
| controllers/visualization_controller.py | 数据可视化 | 可视化数据API控制器 |
| services/auth_service.py | 用户管理 | 用户认证服务 |
| services/file_service.py | 数据导入 | 文件处理服务，处理上传文件 |
| services/cache_service.py | 性能优化 | 缓存服务，优化数据读取 |
| utils/db.py | 数据存储 | 数据库工具类，封装数据库操作 |
| utils/security.py | 安全管理 | 安全相关工具，如加密解密 |
| **数据库** |  |  |
| schema.sql | 数据存储 | 数据库架构定义，包含表结构 |
| **部署文件** |  |  |
| requirements.txt | 环境配置 | Python依赖包列表 |
| nginx.conf | 系统部署 | Nginx服务器配置文件 |
| start.sh | 系统部署 | 系统启动脚本 |

### 4.2 功能实现

#### 4.2.1 Prompt管理功能实现

Prompt管理是本系统的核心功能之一，下面展示关键部分的实现代码：

**后端Prompt模型定义 (models/prompt.py)**

```python
import sqlite3
import json
from datetime import datetime
from utils.db import get_db_connection

class Prompt:
    def __init__(self, id=None, title=None, content=None, category=None, tags=None, user_id=None, created_at=None, updated_at=None):
        self.id = id
        self.title = title
        self.content = content
        self.category = category
        self.tags = tags if tags else []
        self.user_id = user_id
        self.created_at = created_at if created_at else datetime.now()
        self.updated_at = updated_at if updated_at else datetime.now()
    
    @staticmethod
    def create(title, content, category, tags, user_id):
        conn = get_db_connection()
        cursor = conn.cursor()
        now = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        
        cursor.execute(
            'INSERT INTO prompts (title, content, category, tags, user_id, created_at, updated_at) VALUES (?, ?, ?, ?, ?, ?, ?)',
            (title, content, category, json.dumps(tags), user_id, now, now)
        )
        
        prompt_id = cursor.lastrowid
        conn.commit()
        conn.close()
        
        return Prompt.get_by_id(prompt_id)
    
    @staticmethod
    def get_by_id(prompt_id):
        conn = get_db_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT * FROM prompts WHERE id = ?', (prompt_id,))
        row = cursor.fetchone()
        conn.close()
        
        if not row:
            return None
        
        return Prompt(
            id=row['id'],
            title=row['title'],
            content=row['content'],
            category=row['category'],
            tags=json.loads(row['tags']),
            user_id=row['user_id'],
            created_at=row['created_at'],
            updated_at=row['updated_at']
        )
    
    @staticmethod
    def update(prompt_id, title, content, category, tags):
        conn = get_db_connection()
        cursor = conn.cursor()
        now = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        
        cursor.execute(
            'UPDATE prompts SET title = ?, content = ?, category = ?, tags = ?, updated_at = ? WHERE id = ?',
            (title, content, category, json.dumps(tags), now, prompt_id)
        )
        
        conn.commit()
        conn.close()
        
        return Prompt.get_by_id(prompt_id)
    
    @staticmethod
    def delete(prompt_id):
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # 删除prompt相关的处理结果数据
        cursor.execute('DELETE FROM prompt_results WHERE prompt_id = ?', (prompt_id,))
        
        # 删除prompt本身
        cursor.execute('DELETE FROM prompts WHERE id = ?', (prompt_id,))
        
        conn.commit()
        conn.close()
        
        return True
```

**Prompt编辑页面实现 (views/PromptEditor.vue)**

```javascript
<template>
  <div class="prompt-editor">
    <h2>{{ isEditing ? '编辑Prompt' : '创建新Prompt' }}</h2>
    
    <div class="form-group">
      <label for="title">标题</label>
      <input id="title" v-model="promptData.title" type="text" class="form-control" />
    </div>
    
    <div class="form-group">
      <label for="category">分类</label>
      <select id="category" v-model="promptData.category" class="form-control">
        <option value="text-generation">文本生成</option>
        <option value="image-generation">图像生成</option>
        <option value="translation">翻译</option>
        <option value="qa">智能问答</option>
        <option value="other">其他</option>
      </select>
    </div>
    
    <div class="form-group">
      <label for="tags">标签</label>
      <div class="tags-input">
        <input
          type="text"
          v-model="tagInput"
          @keydown.enter.prevent="addTag"
          placeholder="输入标签后按回车添加"
        />
        <div class="tags-list">
          <span v-for="(tag, index) in promptData.tags" :key="index" class="tag">
            {{ tag }}
            <button @click="removeTag(index)" class="remove-tag">&times;</button>
          </span>
        </div>
      </div>
    </div>
    
    <div class="form-group">
      <label for="content">Prompt内容</label>
      <div class="code-editor-container">
        <code-editor
          v-model="promptData.content"
          language="markdown"
          :line-numbers="true"
          theme="vs-dark"
        ></code-editor>
      </div>
    </div>
    
    <div class="button-group">
      <button @click="savePrompt" class="btn btn-primary" :disabled="isLoading">
        {{ isLoading ? '保存中...' : '保存' }}
      </button>
      <button @click="goBack" class="btn btn-secondary">取消</button>
    </div>
  </div>
</template>

<script>
import CodeEditor from '@/components/CodeEditor.vue';
import { createPrompt, getPromptById, updatePrompt } from '@/utils/api';

export default {
  components: {
    CodeEditor
  },
  data() {
    return {
      promptId: this.$route.params.id,
      isEditing: !!this.$route.params.id,
      isLoading: false,
      tagInput: '',
      promptData: {
        title: '',
        content: '',
        category: 'text-generation',
        tags: []
      }
    };
  },
  async created() {
    if (this.isEditing) {
      this.isLoading = true;
      try {
        const prompt = await getPromptById(this.promptId);
        this.promptData = {
          title: prompt.title,
          content: prompt.content,
          category: prompt.category,
          tags: prompt.tags
        };
      } catch (error) {
        this.$toast.error('加载Prompt失败');
        console.error(error);
      } finally {
        this.isLoading = false;
      }
    }
  },
  methods: {
    addTag() {
      if (this.tagInput.trim() && !this.promptData.tags.includes(this.tagInput.trim())) {
        this.promptData.tags.push(this.tagInput.trim());
        this.tagInput = '';
      }
    },
    removeTag(index) {
      this.promptData.tags.splice(index, 1);
    },
    async savePrompt() {
      if (!this.validateForm()) return;
      
      this.isLoading = true;
      try {
        if (this.isEditing) {
          await updatePrompt(this.promptId, this.promptData);
          this.$toast.success('Prompt更新成功');
        } else {
          await createPrompt(this.promptData);
          this.$toast.success('Prompt创建成功');
        }
        this.$router.push('/prompts');
      } catch (error) {
        this.$toast.error('保存失败');
        console.error(error);
      } finally {
        this.isLoading = false;
      }
    },
    validateForm() {
      if (!this.promptData.title.trim()) {
        this.$toast.error('请输入标题');
        return false;
      }
      if (!this.promptData.content.trim()) {
        this.$toast.error('请输入Prompt内容');
        return false;
      }
      return true;
    },
    goBack() {
      this.$router.push('/prompts');
    }
  }
};
</script>
```

#### 4.2.2 数据可视化功能实现

数据可视化是系统的另一个重要功能，下面展示关键实现代码：

**后端可视化控制器 (controllers/visualization_controller.py)**

```python
from flask import Blueprint, jsonify, request
from services.cache_service import cache_data
from models.prompt import Prompt
from models.data import PromptResult
import json

visualization_bp = Blueprint('visualization', __name__)

@visualization_bp.route('/api/visualization/prompt-stats', methods=['GET'])
@cache_data(timeout=300)  # 缓存5分钟
def get_prompt_stats():
    """获取Prompt使用统计数据"""
    user_id = request.args.get('user_id')
    
    stats = PromptResult.get_stats_by_user(user_id)
    
    return jsonify({
        'success': True,
        'data': stats
    })

@visualization_bp.route('/api/visualization/prompt-performance', methods=['GET'])
@cache_data(timeout=300)  # 缓存5分钟
def get_prompt_performance():
    """获取Prompt性能数据"""
    prompt_id = request.args.get('prompt_id')
    
    if not prompt_id:
        return jsonify({
            'success': False,
            'message': 'prompt_id参数缺失'
        }), 400
    
    performance_data = PromptResult.get_performance_data(prompt_id)
    
    return jsonify({
        'success': True,
        'data': performance_data
    })

@visualization_bp.route('/api/visualization/category-distribution', methods=['GET'])
@cache_data(timeout=600)  # 缓存10分钟
def get_category_distribution():
    """获取Prompt分类分布数据"""
    user_id = request.args.get('user_id')
    
    distribution = Prompt.get_category_distribution(user_id)
    
    # 格式化为ECharts饼图数据格式
    pie_data = [
        {'name': category, 'value': count}
        for category, count in distribution.items()
    ]
    
    return jsonify({
        'success': True,
        'data': pie_data
    })

@visualization_bp.route('/api/visualization/time-series', methods=['GET'])
def get_time_series_data():
    """获取时间序列数据"""
    prompt_id = request.args.get('prompt_id')
    start_date = request.args.get('start_date')
    end_date = request.args.get('end_date')
    
    if not prompt_id:
        return jsonify({
            'success': False,
            'message': 'prompt_id参数缺失'
        }), 400
    
    time_series = PromptResult.get_time_series_data(prompt_id, start_date, end_date)
    
    return jsonify({
        'success': True,
        'data': time_series
    })
```

**前端可视化组件 (views/Visualization.vue)**

```javascript
<template>
  <div class="visualization-container">
    <h2>数据可视化</h2>
    
    <div class="filter-section">
      <div class="form-group">
        <label for="prompt-select">选择Prompt</label>
        <select id="prompt-select" v-model="selectedPromptId" class="form-control" @change="loadData">
          <option value="">全部Prompts</option>
          <option v-for="prompt in prompts" :key="prompt.id" :value="prompt.id">
            {{ prompt.title }}
          </option>
        </select>
      </div>
      
      <div class="form-group">
        <label>时间范围</label>
        <div class="date-range">
          <datepicker v-model="startDate" format="yyyy-MM-dd" placeholder="开始日期"></datepicker>
          <span class="date-separator">至</span>
          <datepicker v-model="endDate" format="yyyy-MM-dd" placeholder="结束日期"></datepicker>
          <button @click="loadData" class="btn btn-primary btn-sm">应用</button>
        </div>
      </div>
    </div>
    
    <div class="charts-container">
      <div class="chart-card">
        <h3>成功率统计</h3>
        <div class="chart-wrapper">
          <bar-chart 
            :chart-data="successRateData" 
            :options="barChartOptions"
            height="300px"
          ></bar-chart>
        </div>
      </div>
      
      <div class="chart-card">
        <h3>分类分布</h3>
        <div class="chart-wrapper">
          <pie-chart 
            :chart-data="categoryDistributionData" 
            :options="pieChartOptions"
            height="300px"
          ></pie-chart>
        </div>
      </div>
      
      <div class="chart-card full-width">
        <h3>性能趋势</h3>
        <div class="chart-wrapper">
          <line-chart 
            :chart-data="performanceTrendData" 
            :options="lineChartOptions"
            height="300px"
          ></line-chart>
        </div>
      </div>
    </div>
    
    <div class="loading-overlay" v-if="isLoading">
      <div class="spinner"></div>
    </div>
  </div>
</template>

<script>
import BarChart from '@/components/charts/BarChart.vue';
import PieChart from '@/components/charts/PieChart.vue';
import LineChart from '@/components/charts/LineChart.vue';
import Datepicker from 'vuejs-datepicker';
import { getPrompts, getPromptStats, getCategoryDistribution, getPerformanceTrend } from '@/utils/api';
import { formatDate } from '@/utils/formatter';

export default {
  components: {
    BarChart,
    PieChart,
    LineChart,
    Datepicker
  },
  data() {
    return {
      isLoading: false,
      prompts: [],
      selectedPromptId: '',
      startDate: new Date(new Date().setMonth(new Date().getMonth() - 1)),
      endDate: new Date(),
      
      // 图表数据
      successRateData: null,
      categoryDistributionData: null,
      performanceTrendData: null,
      
      // 图表配置
      barChartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        tooltips: { mode: 'index', intersect: false },
        hover: { mode: 'nearest', intersect: true }
      },
      pieChartOptions: {
        responsive: true,
        maintainAspectRatio: false
      },
      lineChartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        tooltips: { mode: 'index', intersect: false },
        hover: { mode: 'nearest', intersect: true },
        scales: {
          xAxes: [{ display: true, scaleLabel: { display: true, labelString: '日期' } }],
          yAxes: [{ display: true, scaleLabel: { display: true, labelString: '执行时间(ms)' } }]
        }
      }
    };
  },
  async created() {
    this.isLoading = true;
    try {
      // 获取用户的所有Prompts
      const promptsResponse = await getPrompts();
      this.prompts = promptsResponse.data;
      
      // 加载初始数据
      await this.loadData();
    } catch (error) {
      console.error('Failed to load initial data:', error);
      this.$toast.error('加载数据失败');
    } finally {
      this.isLoading = false;
    }
  },
  methods: {
    async loadData() {
      this.isLoading = true;
      try {
        const params = {
          prompt_id: this.selectedPromptId || undefined,
          start_date: formatDate(this.startDate),
          end_date: formatDate(this.endDate)
        };
        
        // 并行请求三个数据源
        const [statsResponse, distributionResponse, trendResponse] = await Promise.all([
          getPromptStats(params),
          getCategoryDistribution(params),
          getPerformanceTrend(params)
        ]);
        
        // 处理成功率数据
        this.successRateData = {
          labels: statsResponse.data.map(item => item.name),
          datasets: [{
            label: '成功率(%)',
            backgroundColor: 'rgba(75, 192, 192, 0.6)',
            data: statsResponse.data.map(item => item.success_rate)
          }]
        };
        
        // 处理分类分布数据
        this.categoryDistributionData = {
          labels: distributionResponse.data.map(item => item.name),
          datasets: [{
            backgroundColor: [
              '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF', '#FF9F40'
            ],
            data: distributionResponse.data.map(item => item.value)
          }]
        };
        
        // 处理性能趋势数据
        this.performanceTrendData = {
          labels: trendResponse.data.map(item => item.date),
          datasets: [{
            label: '平均执行时间(ms)',
            backgroundColor: 'rgba(54, 162, 235, 0.2)',
            borderColor: 'rgba(54, 162, 235, 1)',
            pointRadius: 3,
            fill: true,
            data: trendResponse.data.map(item => item.avg_execution_time)
          }]
        };
      } catch (error) {
        console.error('Failed to load visualization data:', error);
        this.$toast.error('加载可视化数据失败');
      } finally {
        this.isLoading = false;
      }
    }
  }
};
</script>
```

#### 4.2.3 性能优化实现

以下是系统性能优化的关键实现代码：

**缓存服务 (services/cache_service.py)**

```python
from functools import wraps
from flask import request, jsonify
import json
import time
import os

# 简单的内存缓存实现
cache = {}

def cache_data(timeout=300):
    """
    缓存装饰器，用于缓存API响应数据
    
    Args:
        timeout (int): 缓存超时时间，单位为秒，默认300秒(5分钟)
    """
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            # 生成缓存键（基于URL和查询参数）
            cache_key = f"{request.path}_{hash(frozenset(request.args.items()))}"
            
            # 检查缓存是否存在且未过期
            if cache_key in cache:
                cached_data, timestamp = cache[cache_key]
                if time.time() - timestamp < timeout:
                    return cached_data
            
            # 执行原始函数
            result = f(*args, **kwargs)
            
            # 如果结果是JSON响应，则缓存它
            if isinstance(result, tuple) and len(result) == 2:
                response, status_code = result
                if status_code == 200:
                    cache[cache_key] = (result, time.time())
            else:
                cache[cache_key] = (result, time.time())
            
            return result
        return decorated_function
    return decorator

def clear_cache():
    """清除所有缓存"""
    global cache
    cache = {}

def clear_cache_by_prefix(prefix):
    """
    清除指定前缀的缓存
    
    Args:
        prefix (str): 缓存键前缀
    """
    global cache
    keys_to_delete = [key for key in cache.keys() if key.startswith(prefix)]
    for key in keys_to_delete:
        del cache[key]
```

**前端异步加载实现 (main.js)**

```javascript
import Vue from 'vue';
import App from './App.vue';
import router from './router';
import store from './store';
import VueToast from 'vue-toast-notification';
import 'vue-toast-notification/dist/theme-default.css';

// 使用Toast插件
Vue.use(VueToast, {
  position: 'top-right',
  duration: 3000
});

// 异步组件加载
Vue.component('CodeEditor', () => import(/* webpackChunkName: "code-editor" */ './components/CodeEditor.vue'));
Vue.component('BarChart', () => import(/* webpackChunkName: "charts" */ './components/charts/BarChart.vue'));
Vue.component('PieChart', () => import(/* webpackChunkName: "charts" */ './components/charts/PieChart.vue'));
Vue.component('LineChart', () => import(/* webpackChunkName: "charts" */ './components/charts/LineChart.vue'));

// 全局防抖函数
Vue.prototype.$debounce = function(fn, delay = 300) {
  let timer = null;
  return function() {
    let context = this;
    let args = arguments;
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(() => {
      fn.apply(context, args);
    }, delay);
  };
};

// 全局节流函数
Vue.prototype.$throttle = function(fn, delay = 300) {
  let lastCall = 0;
  return function() {
    const now = new Date().getTime();
    if (now - lastCall < delay) {
      return;
    }
    lastCall = now;
    return fn.apply(this, arguments);
  };
};

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app');
```

## 5 系统测试

### 5.1 测试方法

本系统采用以下测试方法：

1. **单元测试**：针对系统各个模块的独立功能进行测试，确保每个模块的功能正常。使用Python的unittest框架和JavaScript的Jest框架进行测试。

2. **集成测试**：测试模块之间的交互和协作，确保系统各部分能够正常协同工作。使用Python的pytest框架进行后端集成测试，使用Vue Test Utils进行前端集成测试。

3. **功能测试**：从用户视角出发，测试系统的各项功能是否符合需求规格说明书的要求。采用基于用例的测试方法，针对每个功能点设计测试用例并执行。

4. **性能测试**：测试系统在不同负载条件下的性能表现，包括响应时间、资源占用等指标。使用JMeter工具进行压力测试和负载测试。

5. **兼容性测试**：测试系统在不同浏览器、不同设备上的兼容性表现。覆盖Chrome、Firefox、Safari等主流浏览器，以及PC、平板、手机等常见设备。

### 5.2 测试实现

#### 5.2.1 单元测试示例

**后端用户模型测试 (tests/test_user_model.py)**

```python
import unittest
from models.user import User
from utils.db import get_db_connection
import os

class TestUserModel(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        # 使用测试数据库
        os.environ['FLASK_ENV'] = 'testing'
        
        # 创建测试表
        conn = get_db_connection()
        with open('tests/test_schema.sql', 'r') as f:
            conn.executescript(f.read())
        conn.close()
    
    def setUp(self):
        # 每个测试前清空用户表
        conn = get_db_connection()
        conn.execute('DELETE FROM users')
        conn.commit()
        conn.close()
    
    def test_create_user(self):
        # 测试创建用户功能
        user = User.create(
            username='testuser',
            password='password123',
            email='test@example.com'
        )
        
        self.assertIsNotNone(user.id)
        self.assertEqual(user.username, 'testuser')
        self.assertEqual(user.email, 'test@example.com')
        self.assertTrue(User.verify_password('password123', user.password_hash))
    
    def test_get_by_username(self):
        # 测试通过用户名获取用户
        User.create(
            username='testuser',
            password='password123',
            email='test@example.com'
        )
        
        user = User.get_by_username('testuser')
        self.assertIsNotNone(user)
        self.assertEqual(user.username, 'testuser')
        
        # 测试不存在的用户名
        user = User.get_by_username('nonexistent')
        self.assertIsNone(user)
    
    def test_verify_password(self):
        # 测试密码验证
        user = User.create(
            username='testuser',
            password='password123',
            email='test@example.com'
        )
        
        self.assertTrue(User.verify_password('password123', user.password_hash))
        self.assertFalse(User.verify_password('wrongpassword', user.password_hash))
```

**前端Prompt编辑组件测试 (tests/components/PromptEditor.spec.js)**

```javascript
import { mount, createLocalVue } from '@vue/test-utils';
import PromptEditor from '@/views/PromptEditor.vue';
import VueRouter from 'vue-router';
import * as api from '@/utils/api';

// Mock API
jest.mock('@/utils/api', () => ({
  createPrompt: jest.fn(),
  getPromptById: jest.fn(),
  updatePrompt: jest.fn()
}));

const localVue = createLocalVue();
localVue.use(VueRouter);
const router = new VueRouter();

describe('PromptEditor.vue', () => {
  let wrapper;
  
  beforeEach(() => {
    // 重置API模拟
    jest.clearAllMocks();
    
    // 模拟$toast
    const mockToast = {
      success: jest.fn(),
      error: jest.fn()
    };
    
    wrapper = mount(PromptEditor, {
      localVue,
      router,
      mocks: {
        $toast: mockToast
      },
      stubs: {
        CodeEditor: true // 存根CodeEditor组件
      }
    });
  });
  
  test('初始状态正确', () => {
    expect(wrapper.vm.isEditing).toBe(false);
    expect(wrapper.vm.isLoading).toBe(false);
    expect(wrapper.vm.promptData.title).toBe('');
    expect(wrapper.vm.promptData.content).toBe('');
    expect(wrapper.vm.promptData.category).toBe('text-generation');
    expect(wrapper.vm.promptData.tags).toEqual([]);
  });
  
  test('添加标签功能', async () => {
    wrapper.vm.tagInput = 'test-tag';
    await wrapper.vm.addTag();
    
    expect(wrapper.vm.promptData.tags).toContain('test-tag');
    expect(wrapper.vm.tagInput).toBe('');
    
    // 测试重复标签不会被添加
    wrapper.vm.tagInput = 'test-tag';
    await wrapper.vm.addTag();
    expect(wrapper.vm.promptData.tags.filter(tag => tag === 'test-tag').length).toBe(1);
  });
  
  test('移除标签功能', async () => {
    wrapper.vm.promptData.tags = ['tag1', 'tag2', 'tag3'];
    await wrapper.vm.removeTag(1);
    
    expect(wrapper.vm.promptData.tags).toEqual(['tag1', 'tag3']);
  });
  
  test('表单验证功能', () => {
    // 空标题验证
    wrapper.vm.promptData.title = '';
    wrapper.vm.promptData.content = 'test content';
    expect(wrapper.vm.validateForm()).toBe(false);
    expect(wrapper.vm.$toast.error).toHaveBeenCalledWith('请输入标题');
    
    // 空内容验证
    wrapper.vm.promptData.title = 'test title';
    wrapper.vm.promptData.content = '';
    expect(wrapper.vm.validateForm()).toBe(false);
    expect(wrapper.vm.$toast.error).toHaveBeenCalledWith('请输入Prompt内容');
    
    // 有效表单验证
    wrapper.vm.promptData.title = 'test title';
    wrapper.vm.promptData.content = 'test content';
    expect(wrapper.vm.validateForm()).toBe(true);
  });
  
  test('创建新Prompt', async () => {
    api.createPrompt.mockResolvedValue({ id: 1 });
    
    wrapper.vm.promptData = {
      title: 'Test Prompt',
      content: 'This is a test prompt',
      category: 'text-generation',
      tags: ['test', 'example']
    };
    
    await wrapper.vm.savePrompt();
    
    expect(api.createPrompt).toHaveBeenCalledWith(wrapper.vm.promptData);
    expect(wrapper.vm.$toast.success).toHaveBeenCalledWith('Prompt创建成功');
  });
});
```

#### 5.2.2 性能测试示例

**后端性能测试脚本 (tests/performance/test_api_performance.py)**

```python
import requests
import time
import statistics
import matplotlib.pyplot as plt
from concurrent.futures import ThreadPoolExecutor

BASE_URL = 'http://localhost:5000/api'
TEST_USER = {'username': 'perftest', 'password': 'perftest123'}
AUTH_TOKEN = None

def setup():
    """设置测试环境，创建测试用户并获取认证令牌"""
    global AUTH_TOKEN
    resp = requests.post(f'{BASE_URL}/auth/login', json=TEST_USER)
    AUTH_TOKEN = resp.json()['data']['token']

def test_endpoint(endpoint, method='GET', data=None, params=None, concurrency=1, requests_count=100):
    """测试指定端点的性能"""
    url = f'{BASE_URL}/{endpoint}'
    headers = {'Authorization': f'Bearer {AUTH_TOKEN}'} if AUTH_TOKEN else {}
    
    response_times = []
    errors = 0
    
    def make_request():
        try:
            start_time = time.time()
            if method == 'GET':
                response = requests.get(url, headers=headers, params=params)
            elif method == 'POST':
                response = requests.post(url, headers=headers, json=data)
            elif method == 'PUT':
                response = requests.put(url, headers=headers, json=data)
            elif method == 'DELETE':
                response = requests.delete(url, headers=headers)
            
            elapsed = time.time() - start_time
            
            if response.status_code == 200:
                return elapsed, None
            else:
                return elapsed, f"Error {response.status_code}: {response.text}"
        except Exception as e:
            return 0, str(e)
    
    with ThreadPoolExecutor(max_workers=concurrency) as executor:
        futures = [executor.submit(make_request)
