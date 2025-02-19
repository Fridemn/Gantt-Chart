<template>
  <div class="container">
    <h1 class="title">项目进度甘特图</h1>
    
    <!-- 时间控制工具栏 -->
    <div class="control-panel">
      <div class="control-group">
        <div class="date-control">
          <label>开始日期:</label>
          <input type="date" 
                 v-model="timeRange.start" 
                 @change="updateTimeRange">
        </div>
        <div class="date-control">
          <label>结束日期:</label>
          <input type="date" 
                 v-model="timeRange.end" 
                 @change="updateTimeRange">
        </div>
        <select v-model="timeScale" 
                @change="updateTimeScale">
          <option value="day">日</option>
          <option value="week">周</option>
          <option value="month">月</option>
          <option value="quarter">季度</option>
          <option value="year">年</option>
        </select>
      </div>
    </div>

    <!-- 工具栏 -->
    <div class="toolbar">
      <button @click="showAddTaskModal = true" class="btn btn-primary">
        添加任务
      </button>
      <button @click="exportToJSON" class="btn btn-secondary">
        导出JSON
      </button>
      <button @click="importFromJSON" class="btn btn-secondary">
        导入JSON
      </button>
    </div>

    <!-- 甘特图容器 -->
    <div ref="ganttContainer" class="gantt-chart-container"></div>

    <!-- 添加任务模态框 -->
    <div v-if="showAddTaskModal" class="modal-overlay">
      <div class="modal-content">
        <h2 class="modal-title">添加新任务</h2>
        <form @submit.prevent="addNewTask">
          <div class="form-group">
            <label>任务名称</label>
            <input v-model="newTask.text" required>
          </div>
          <div class="form-group">
            <label>开始日期</label>
            <input v-model="newTask.start_date" type="date" required>
          </div>
          <div class="form-group">
            <label>持续天数</label>
            <input v-model="newTask.duration" type="number" required>
          </div>
          <div class="form-group">
            <label>负责人</label>
            <input v-model="newTask.owner" required>
          </div>
          <div class="modal-actions">
            <button type="submit" class="btn btn-primary">保存</button>
            <button type="button" @click="showAddTaskModal = false" class="btn btn-cancel">
              取消
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script>
// 更新导入语句
import 'dhtmlx-gantt';
import 'dhtmlx-gantt/codebase/dhtmlxgantt.css';
import { gantt } from 'dhtmlx-gantt';

export default {
  name: 'App',
  data() {
    return {
      showAddTaskModal: false,
      newTask: {
        text: '',
        start_date: '',
        duration: 1,
        owner: '',
        progress: 0
      },
      tasks: {
        data: [
          { id: 1, text: '示例任务1', start_date: '2024-01-01', duration: 5, progress: 0.4, owner: '张三' },
          { id: 2, text: '示例任务2', start_date: '2024-01-03', duration: 4, progress: 0.6, owner: '李四' },
          { id: 3, text: '示例任务3', start_date: '2024-01-06', duration: 6, progress: 0.2, owner: '王五' }
        ],
        links: [
          { id: 1, source: 1, target: 2, type: '0' },
          { id: 2, source: 2, target: 3, type: '0' }
        ]
      },
      timeRange: {
        start: new Date().toISOString().split('T')[0],
        end: new Date(new Date().setMonth(new Date().getMonth() + 6)).toISOString().split('T')[0]
      },
      timeScale: 'month',
      scaleConfigs: {
        day: {
          scale_unit: "day",
          date_scale: "%d %M",
          subscales: [
            {unit: "hour", step: 6, date: "%H:00"}
          ],
          min_column_width: 50
        },
        week: {
          scale_unit: "week",
          date_scale: "%W周",
          subscales: [
            {unit: "day", step: 1, date: "%j %D"}
          ],
          min_column_width: 30
        },
        month: {
          scale_unit: "month",
          date_scale: "%Y年%m月",
          subscales: [
            {unit: "week", step: 1, date: "%j日"}
          ],
          min_column_width: 20
        },
        quarter: {
          scale_unit: "quarter",
          date_scale: "%Y年 Q%q",
          subscales: [
            {unit: "month", step: 1, date: "%M"}
          ],
          min_column_width: 50
        },
        year: {
          scale_unit: "year",
          date_scale: "%Y年",
          subscales: [
            {unit: "month", step: 1, date: "%M"}
          ],
          min_column_width: 30
        }
      }
    };
  },
  mounted() {
    // 确保在组件挂载后初始化甘特图
    this.$nextTick(() => {
      setTimeout(() => {
        this.initGantt();
      }, 100);
    });
  },
  methods: {
    addNewTask() {
      const newId = gantt.getTaskCount() + 1;
      const task = {
        id: newId,
        ...this.newTask
      };
      
      gantt.addTask(task);
      this.showAddTaskModal = false;
      this.resetNewTask();
    },
    
    resetNewTask() {
      this.newTask = {
        text: '',
        start_date: '',
        duration: 1,
        owner: '',
        progress: 0
      };
    },

    exportToJSON() {
      const currentData = gantt.serialize();
      if (!currentData.data || currentData.data.length === 0) {
        alert('当前没有可导出的任务数据');
        return;
      }

      const data = {
        tasks: currentData,
        timestamp: new Date().toISOString()
      };
      
      const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `gantt-export-${new Date().toISOString()}.json`;
      a.click();
      window.URL.revokeObjectURL(url);
    },

    importFromJSON() {
      const input = document.createElement('input');
      input.type = 'file';
      input.accept = '.json';
      input.onchange = (e) => {
        const file = e.target.files[0];
        if (!file) return;

        const reader = new FileReader();
        reader.onload = (event) => {
          try {
            const data = JSON.parse(event.target.result);
            
            // 验证数据结构
            if (!data.tasks || !data.tasks.data) {
              throw new Error('无效的任务数据格式');
            }

            // 清空现有数据
            gantt.clearAll();

            // 确保所有日期格式正确
            const processedData = {
              data: data.tasks.data.map(task => ({
                ...task,
                start_date: new Date(task.start_date).toISOString().split('T')[0],
                end_date: task.end_date ? new Date(task.end_date).toISOString().split('T')[0] : undefined
              })),
              links: data.tasks.links || []
            };

            // 导入数据
            gantt.parse(processedData);
            alert('数据导入成功！');
          } catch (error) {
            console.error('导入失败:', error);
            alert(`导入失败: ${error.message}`);
          }
        };
        reader.onerror = () => {
          alert('文件读取失败');
        };
        reader.readAsText(file);
      };
      input.click();
    },

    updateTimeRange() {
      gantt.config.start_date = new Date(this.timeRange.start);
      gantt.config.end_date = new Date(this.timeRange.end);
      gantt.render();
    },

    updateTimeScale() {
      const config = this.scaleConfigs[this.timeScale];
      gantt.config.scale_unit = config.scale_unit;
      gantt.config.date_scale = config.date_scale;
      gantt.config.subscales = config.subscales;
      gantt.config.min_column_width = config.min_column_width;
      gantt.render();
    },

    initGantt() {
      // 确保容器存在
      if (!this.$refs.ganttContainer) {
        console.error('ganttContainer ref not found');
        return;
      }

      // 基础配置
      gantt.config.date_format = "%Y-%m-%d";
      
      // 设置初始宽度和高度
      gantt.config.autosize = "y";
      gantt.config.min_column_width = 40;
      
      // 应用初始时间刻度
      this.updateTimeScale();
      
      // 应用初始时间范围
      this.updateTimeRange();

      // 配置表格列
      gantt.config.columns = [
        { name: "text", label: "任务名称", tree: true, width: 200, resize: true },
        { name: "start_date", label: "开始时间", align: "center", width: 100, resize: true },
        { name: "duration", label: "持续时间(天)", align: "center", width: 100, resize: true },
        { name: "owner", label: "负责人", align: "center", width: 80, resize: true },
        { name: "progress", label: "进度", align: "center", width: 80, template: (obj) => (obj.progress * 100) + "%" }
      ];

      // 启用调整列宽
      gantt.config.layout = {
        css: "gantt_container",
        rows: [
          {
            cols: [
              {view: "grid", width: 500, scrollX: "scrollHor", scrollY: "scrollVer"},
              {resizer: true, width: 1},
              {view: "timeline", scrollX: "scrollHor", scrollY: "scrollVer"},
              {view: "scrollbar", id: "scrollVer"}
            ]
          },
          {view: "scrollbar", id: "scrollHor", height: 20}
        ]
      };

      // 添加编辑功能
      gantt.config.readonly = false;
      gantt.config.drag_progress = true;

      // 任务更新事件
      gantt.attachEvent("onAfterTaskUpdate", (id, task) => {
        console.log("任务已更新:", task);
      });

      // 添加错误处理事件
      gantt.attachEvent("onError", (errorMessage) => {
        console.error('甘特图错误:', errorMessage);
        alert(`操作失败: ${errorMessage}`);
      });

      // 初始化甘特图
      try {
        gantt.init(this.$refs.ganttContainer);
        gantt.clearAll();
        gantt.parse(this.tasks);
        console.log('甘特图初始化成功');
      } catch (error) {
        console.error('甘特图初始化失败:', error);
      }
    }
  }
};
</script>

<style>
.container {
  width: 100%;
  max-width: 100%;
  margin: 0;
  padding: 20px;
  box-sizing: border-box;
  height: 90vh;
}

.title {
  font-size: 24px;
  font-weight: bold;
  margin-bottom: 20px;
  color: #2c3e50;
}

.control-panel {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.control-group {
  display: flex;
  gap: 20px;
  align-items: center;
}

.date-control {
  display: flex;
  align-items: center;
  gap: 10px;
}

.toolbar {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.btn {
  padding: 8px 16px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.3s ease;
}

.btn-primary {
  background: #3498db;
  color: white;
}

.btn-primary:hover {
  background: #2980b9;
}

.btn-secondary {
  background: #95a5a6;
  color: white;
}

.btn-secondary:hover {
  background: #7f8c8d;
}

.btn-cancel {
  background: #e74c3c;
  color: white;
}

.btn-cancel:hover {
  background: #c0392b;
}

.gantt-chart-container {
  height: calc(100vh - 200px);
  width: 100%;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  margin-top: 20px;
  position: relative;
}

/* 模态框样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  width: 400px;
}

.modal-title {
  font-size: 20px;
  margin-bottom: 20px;
  color: #2c3e50;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  color: #34495e;
}

.form-group input {
  width: 90%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
}

/* 甘特图样式重写 */
.gantt_task_line {
  background-color: #3498db;
  border-color: #2980b9;
  border-radius: 4px;
}

.gantt_task_progress {
  background-color: #2980b9;
  border-radius: 4px;
}

.gantt_grid_head_cell {
  font-weight: bold;
  color: #2c3e50;
  background-color: #f8f9fa;
}

/* 输入控件样式 */
input[type="date"], 
select {
  padding: 6px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

input[type="date"]:focus, 
select:focus {
  border-color: #3498db;
  outline: none;
  box-shadow: 0 0 0 2px rgba(52, 152, 219, 0.2);
}
</style>
