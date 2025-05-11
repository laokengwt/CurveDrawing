<template>
  <div class="curve-drawer-container">
    <div class="curve-drawer">
      <h1>曲线绘制工具</h1>
      <div class="controls">
        <button @click="clearCanvas">清除画布</button>
        <span class="points-count">点数: {{ points.length }}</span>
        <span class="hint-text" v-if="isPreviewMode">(预览模式 - 按ESC取消)</span>
        <span class="hint-text" v-else>(固定模式 - 可拖动点调整)</span>
      </div>
      <canvas 
        ref="canvas" 
        @click="handleCanvasClick"
        @mousemove="handleMouseMove"
        @mouseleave="handleMouseLeave"
        @mousedown="handleMouseDown"
        @mouseup="handleMouseUp"
        width="800" 
        height="500"
        tabindex="0"
      ></canvas>
      <div class="hint">
        {{ isPreviewMode ? '点击确认添加新点，按ESC取消预览' : '拖动蓝色控制点调整曲线，点击空白处开始添加新点' }}
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';

export default {
  name: 'CurveDrawer',
  setup() {
    const canvas = ref(null);
    const ctx = ref(null);
    const points = ref([]);
    const mousePos = ref(null);
    const isMouseInCanvas = ref(false);
    const isPreviewMode = ref(false); // 初始为固定模式
    const selectedPointIndex = ref(-1);
    const isDragging = ref(false);
    const isWaitingForSecondClick = ref(false); // 新增：等待第二次点击状态

    // 初始化画布
    const initCanvas = () => {
      if (canvas.value) {
        ctx.value = canvas.value.getContext('2d');
        canvas.value.focus();
        drawGrid();
      }
    };

    // 绘制网格背景（保持不变）
    const drawGrid = () => {
      ctx.value.strokeStyle = 'rgba(100, 100, 100, 0.3)';
      ctx.value.lineWidth = 1;
      
      for (let x = 0; x <= canvas.value.width; x += 20) {
        ctx.value.beginPath();
        ctx.value.moveTo(x, 0);
        ctx.value.lineTo(x, canvas.value.height);
        ctx.value.stroke();
      }
      
      for (let y = 0; y <= canvas.value.height; y += 20) {
        ctx.value.beginPath();
        ctx.value.moveTo(0, y);
        ctx.value.lineTo(canvas.value.width, y);
        ctx.value.stroke();
      }
    };

    // Catmull-Rom样条计算（保持不变）
    const catmullRom = (p0, p1, p2, p3, t) => {
      const t2 = t * t;
      const t3 = t2 * t;
      
      return {
        x: 0.5 * ((2 * p1.x) +
                 (-p0.x + p2.x) * t +
                 (2*p0.x - 5*p1.x + 4*p2.x - p3.x) * t2 +
                 (-p0.x + 3*p1.x - 3*p2.x + p3.x) * t3),
        y: 0.5 * ((2 * p1.y) +
                 (-p0.y + p2.y) * t +
                 (2*p0.y - 5*p1.y + 4*p2.y - p3.y) * t2 +
                 (-p0.y + 3*p1.y - 3*p2.y + p3.y) * t3)
      };
    };

    // 检查是否点击了控制点（保持不变）
    const checkPointHit = (x, y) => {
      if (isPreviewMode.value) return -1;
      
      const hitRadius = 10;
      for (let i = 0; i < points.value.length; i++) {
        const point = points.value[i];
        const distance = Math.sqrt((x - point.x) ** 2 + (y - point.y) ** 2);
        if (distance <= hitRadius) {
          return i;
        }
      }
      return -1;
    };

    // 绘制曲线（修改预览点显示逻辑）
    const drawCurve = () => {
      ctx.value.fillStyle = 'black';
      ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height);
      drawGrid();
      
      if (points.value.length === 0 && !isMouseInCanvas.value) return;

      // 绘制所有已确定的点
      points.value.forEach((point, index) => {
        ctx.value.fillStyle = index === selectedPointIndex.value ? '#f1c40f' : '#3498db';
        ctx.value.beginPath();
        ctx.value.arc(point.x, point.y, 5, 0, Math.PI * 2);
        ctx.value.fill();
        
        ctx.value.font = '12px Arial';
        ctx.value.fillStyle = 'white';
        ctx.value.fillText(`(${Math.round(point.x)},${Math.round(point.y)})`, point.x + 10, point.y - 10);
      });

      // 仅在预览模式或等待第二次点击时显示预览点
      if ((isPreviewMode.value || isWaitingForSecondClick.value) && 
          mousePos.value && 
          isMouseInCanvas.value && 
          points.value.length > 0) {
        ctx.value.fillStyle = 'rgba(52, 152, 219, 0.5)';
        ctx.value.beginPath();
        ctx.value.arc(mousePos.value.x, mousePos.value.y, 5, 0, Math.PI * 2);
        ctx.value.fill();
      }

      // 准备绘制点集（包含预览点）
      let pointsToDraw = [...points.value];
      if ((isPreviewMode.value || isWaitingForSecondClick.value) && 
          mousePos.value && 
          isMouseInCanvas.value) {
        pointsToDraw.push(mousePos.value);
      }

      // 绘制平滑曲线
      if (pointsToDraw.length > 1) {
        ctx.value.strokeStyle = '#e74c3c';
        ctx.value.lineWidth = 2;
        ctx.value.beginPath();
        
        const extendedPoints = [
          { x: pointsToDraw[0].x - (pointsToDraw[1].x - pointsToDraw[0].x), 
            y: pointsToDraw[0].y - (pointsToDraw[1].y - pointsToDraw[0].y) },
          ...pointsToDraw,
          { x: pointsToDraw[pointsToDraw.length-1].x - (pointsToDraw[pointsToDraw.length-2].x - pointsToDraw[pointsToDraw.length-1].x),
            y: pointsToDraw[pointsToDraw.length-1].y - (pointsToDraw[pointsToDraw.length-2].y - pointsToDraw[pointsToDraw.length-1].y) }
        ];
        
        for (let i = 1; i < extendedPoints.length - 2; i++) {
          const p0 = extendedPoints[i-1];
          const p1 = extendedPoints[i];
          const p2 = extendedPoints[i+1];
          const p3 = extendedPoints[i+2];
          
          ctx.value.moveTo(p1.x, p1.y);
          
          const segments = 20;
          for (let j = 1; j <= segments; j++) {
            const t = j / segments;
            const point = catmullRom(p0, p1, p2, p3, t);
            ctx.value.lineTo(point.x, point.y);
          }
        }
        
        ctx.value.stroke();
      }
    };

    // 修改后的点击处理逻辑
    const handleCanvasClick = (event) => {
      if (isDragging.value) {
        isDragging.value = false;
        return;
      }
      
      const rect = canvas.value.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;
      
      const hitIndex = checkPointHit(x, y);
      
      if (hitIndex !== -1) {
        // 点击了控制点（在固定模式下）
        selectedPointIndex.value = hitIndex;
      } else {
        if (isWaitingForSecondClick.value) {
          // 第二次点击空白处 - 确认添加新点
          points.value.push({ x, y });
          isWaitingForSecondClick.value = false;
          isPreviewMode.value = true;
        } else if (!isPreviewMode.value) {
          // 第一次点击空白处 - 进入等待第二次点击状态（显示预览但不添加点）
          isWaitingForSecondClick.value = true;
        } else {
          // 预览模式下点击 - 直接添加点
          points.value.push({ x, y });
        }
      }
      
      drawCurve();
    };

    // 修改后的ESC键处理
    const handleKeyDown = (event) => {
      if (event.key === 'Escape') {
        if (isWaitingForSecondClick.value) {
          // 取消等待第二次点击状态
          isWaitingForSecondClick.value = false;
        } else {
          // 正常切换预览/固定模式
          isPreviewMode.value = !isPreviewMode.value;
        }
        isDragging.value = false;
        selectedPointIndex.value = -1;
        drawCurve();
      }
    };

    // 其他事件处理函数保持不变
    const handleMouseMove = (event) => {
      const rect = canvas.value.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;
      
      mousePos.value = { x, y };
      isMouseInCanvas.value = true;

      if (isDragging.value && selectedPointIndex.value !== -1) {
        points.value[selectedPointIndex.value] = { x, y };
      }
      
      drawCurve();
    };

    const handleMouseLeave = () => {
      isMouseInCanvas.value = false;
      drawCurve();
    };

    const handleMouseDown = (event) => {
      if (isPreviewMode.value || isWaitingForSecondClick.value) return;
      
      const rect = canvas.value.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;
      
      const hitIndex = checkPointHit(x, y);
      if (hitIndex !== -1) {
        selectedPointIndex.value = hitIndex;
        isDragging.value = true;
      }
    };

    const handleMouseUp = () => {
      isDragging.value = false;
      selectedPointIndex.value = -1;
    };

    const clearCanvas = () => {
      points.value = [];
      isPreviewMode.value = false;
      isWaitingForSecondClick.value = false;
      isDragging.value = false;
      selectedPointIndex.value = -1;
      drawCurve();
    };

    onMounted(() => {
      initCanvas();
      canvas.value.addEventListener('keydown', handleKeyDown);
    });

    return {
      canvas,
      points,
      isPreviewMode,
      initCanvas,
      handleCanvasClick,
      handleMouseMove,
      handleMouseLeave,
      handleMouseDown,
      handleMouseUp,
      clearCanvas
    };
  }
};
</script>

<style scoped>
/* 样式保持不变 */
.curve-drawer-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-color: #121212;
  padding: 20px;
}

.curve-drawer {
  display: flex;
  flex-direction: column;
  align-items: center;
  max-width: 800px;
  width: 100%;
}

h1 {
  color: #ffffff;
  margin-bottom: 20px;
}

.controls {
  margin: 15px 0;
  display: flex;
  gap: 10px;
  align-items: center;
}

button {
  padding: 8px 15px;
  cursor: pointer;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #2980b9;
}

.points-count {
  color: #ecf0f1;
  margin-left: 10px;
}

.hint-text {
  color: #f39c12;
  margin-left: 10px;
  font-size: 14px;
}

canvas {
  border: 1px solid #333;
  background-color: black;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.5);
  cursor: crosshair;
  outline: none;
}

.hint {
  margin-top: 15px;
  color: #bdc3c7;
  font-size: 14px;
}
</style>