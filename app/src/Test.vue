<template>
  <div class="container" ref="container">
    <canvas ref="canvas"></canvas>
    <div class="toolbar">
      <el-radio-group v-model="currentType">
        <el-radio-button label="selection">选择</el-radio-button>
        <el-radio-button label="rectangle">矩形</el-radio-button>
      </el-radio-group>
    </div>
  </div>
</template>

<script setup>
import { onMounted, ref } from "vue";

// 初始化画布
const container = ref(null);
const canvas = ref(null);
let ctx = null;
// 初始化画布
const initCanvas = () => {
  let { width, height } = container.value.getBoundingClientRect();
  canvas.value.width = width;
  canvas.value.height = height;
  ctx = canvas.value.getContext("2d");
  // 将画布的原点由左上角移动到中心点
  ctx.translate(width / 2, height / 2);
};
// 清除画布
const clearCanvas = () => {
  let width = canvas.value.width;
  let height = canvas.value.height;
  ctx.clearRect(-width / 2, -height / 2, width, height);
};
// 绘制矩形
const drawRect = (x, y, width, height) => {
  ctx.beginPath();
  ctx.rect(x, y, width, height);
  ctx.stroke();
};
// 绘制圆形
const drawCircle = (x, y, r) => {
  ctx.beginPath();
  ctx.arc(x, y, r, 0, 2 * Math.PI);
  ctx.stroke();
};

// 监听事件
const bindEvent = () => {
  canvas.value.addEventListener("mousedown", onMousedown);
  canvas.value.addEventListener("mousemove", onMousemove);
  canvas.value.addEventListener("mouseup", onMouseup);
};
// 屏幕坐标转画布坐标
const screenToCanvas = (x, y) => {
  return {
    x: x - canvas.value.width / 2,
    y: y - canvas.value.height / 2,
  };
};
let mousedownX = 0;
let mousedownY = 0;
let isMousedown = false;
// 当前操作模式
const currentType = ref("selection");
// 当前是否正在调整元素
let isAdjustmentElement = false;
// 检测是否击中了某个元素
const checkIsHitElement = (x, y) => {
  let hitElement = null;
  for (let i = 0; i < allElements.length; i++) {
    if (allElements[i].isHit(x, y)) {
      hitElement = allElements[i];
      break;
    }
  }
  // 如果当前已经有激活元素则先将它取消激活
  if (activeElement) {
    activeElement.isActive = false;
  }
  // 更新当前激活元素
  activeElement = hitElement;
  if (hitElement) {
    // 如果当前击中了元素，则将它的状态修改为激活状态
    hitElement.isActive = true;
  }
  // 重新渲染所有元素
  renderAllElements();
};
// 鼠标按下事件
const onMousedown = (e) => {
  mousedownX = e.clientX;
  mousedownY = e.clientY;
  isMousedown = true;
  if (currentType.value === "selection") {
    // 选择模式下进行元素激活检测
    if (activeElement) {
      // 当前存在激活元素则判断是否按住了激活状态的某个区域
      let hitActiveArea = activeElement.isHitActiveArea(mousedownX, mousedownY);
      if (hitActiveArea) {
        // 按住了按住了激活状态的某个区域
        isAdjustmentElement = true;
        alert(hitActiveArea);
      } else {
        // 否则进行激活元素的更新操作
        checkIsHitElement(mousedownX, mousedownY);
      }
    } else {
      checkIsHitElement(mousedownX, mousedownY);
    }
  }
};
// 当前激活的元素
let activeElement = null;
// 所有的元素
let allElements = [];
// 渲染所有元素
const renderAllElements = () => {
  clearCanvas();
  allElements.forEach((element) => {
    element.render();
  });
};
// 鼠标移动事件
const onMousemove = (e) => {
  if (!isMousedown || currentType.value === "selection") {
    return;
  }
  if (!activeElement) {
    activeElement = new Rectangle({
      x: mousedownX,
      y: mousedownY,
    });
    allElements.push(activeElement);
  }
  // 更新矩形的大小
  activeElement.width = e.clientX - mousedownX;
  activeElement.height = e.clientY - mousedownY;
  // 渲染所有的元素
  renderAllElements();
};
// 鼠标松开事件
const onMouseup = (e) => {
  isMousedown = false;
  if (currentType.value !== "selection") {
    activeElement = null;
  }
  mousedownX = 0;
  mousedownY = 0;
};

// 矩形类
class Rectangle {
  constructor(opt) {
    this.x = opt.x || 0;
    this.y = opt.y || 0;
    this.width = opt.width || 0;
    this.height = opt.height || 0;
    this.isActive = false;
  }

  render() {
    let canvasPos = screenToCanvas(this.x, this.y);
    drawRect(canvasPos.x, canvasPos.y, this.width, this.height);
    this.renderActiveState();
  }

  // 当激活时渲染激活态
  renderActiveState() {
    if (!this.isActive) {
      return;
    }
    let canvasPos = screenToCanvas(this.x, this.y);
    // 为了不和矩形重叠，虚线框比矩形大一圈，增加5px的内边距
    let x = canvasPos.x - 5;
    let y = canvasPos.y - 5;
    let width = this.width + 10;
    let height = this.height + 10;
    // 主体的虚线框
    ctx.save();
    ctx.setLineDash([5]);
    drawRect(x, y, width, height);
    ctx.restore();

    // 左上角的操作手柄
    drawRect(x - 10, y - 10, 10, 10);

    // 右上角的操作手柄
    drawRect(x + width, y - 10, 10, 10);

    // 右下角的操作手柄
    drawRect(x + width, y + height, 10, 10);

    // 左下角的操作手柄
    drawRect(x - 10, y + height, 10, 10);

    // 旋转操作手柄
    drawCircle(x + width / 2, y - 10, 10);
  }

  // 检测是否被击中
  isHit(x0, y0) {
    let { x, y, width, height } = this;
    // 矩形四条边的线段
    let segments = [
      [x, y, x + width, y],
      [x + width, y, x + width, y + height],
      [x + width, y + height, x, y + height],
      [x, y + height, x, y],
    ];
    for (let i = 0; i < segments.length; i++) {
      let segment = segments[i];
      if (
        checkIsAtSegment(x0, y0, segment[0], segment[1], segment[2], segment[3])
      ) {
        return true;
      }
    }
    return false;
  }

  // 检测是否击中了激活状态的某个区域
  isHitActiveArea(x0, y0) {
    let x = this.x - 5;
    let y = this.y - 5;
    let width = this.width + 10;
    let height = this.height + 10;
    if (checkPointIsInRectangle(x0, y0, x, y, width, height)) {
      // 在中间的虚线框
      return "body";
    } else if (getTowPointDistance(x0, y0, x + width / 2, y - 10) <= 10) {
      // 在旋转手柄
      return "rotate";
    } else if (checkPointIsInRectangle(x0, y0, x + width, y + height, 10, 10)) {
      // 在右下角操作手柄
      return "bottomRight";
    }
  }
}

// 计算点到直线的距离
const getPointToLineDistance = (x, y, x1, y1, x2, y2) => {
  // 直线公式y=kx+b不适用于直线垂直于x轴的情况，所以对于直线垂直于x轴的情况单独处理
  if (x1 === x2) {
    return Math.abs(x - x1);
  } else {
    let k, b;
    // y1 = k * x1 + b  // 0式
    // b = y1 - k * x1  // 1式

    // y2 = k * x2 + b    // 2式
    // y2 = k * x2 + y1 - k * x1  // 1式代入2式
    // y2 - y1 = k * x2 - k * x1
    // y2 - y1 = k * (x2 -  x1)
    k = (y2 - y1) / (x2 - x1); // 3式

    b = y1 - k * x1; // 3式代入0式

    return Math.abs((k * x - y + b) / Math.sqrt(1 + k * k));
  }
};

// 计算两点之间的距离
const getTowPointDistance = (x1, y1, x2, y2) => {
  return Math.sqrt(Math.pow(x1 - x2, 2) + Math.pow(y1 - y2, 2));
};

// 检查是否点击到了一条线段
const checkIsAtSegment = (x, y, x1, y1, x2, y2, dis = 10) => {
  // 点到直线的距离不满足直接返回
  if (getPointToLineDistance(x, y, x1, y1, x2, y2) > dis) {
    return false;
  }
  // 点两个端点的距离
  let dis1 = getTowPointDistance(x, y, x1, y1);
  let dis2 = getTowPointDistance(x, y, x2, y2);
  // 线段两个端点的距离
  let dis3 = getTowPointDistance(x1, y1, x2, y2);
  // 根据勾股定理计算斜边长度，也就是最远的距离
  let max = Math.sqrt(dis * dis + dis3 * dis3);
  // 点距离两个端点的距离都需要小于这个最远距离
  if (dis1 <= max && dis2 <= max) {
    return true;
  }
  return false;
};

// 判断一个坐标是否在一个矩形内
const checkPointIsInRectangle = (x, y, rx, ry, rw, rh) => {
  return x >= rx && x <= rx + rw && y >= ry && y <= ry + rh;
};

onMounted(() => {
  initCanvas();
  bindEvent();
});
</script>

<style lang="less">
.container {
  position: fixed;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;

  .canvasBox {
    position: absolute;
    left: 50%;
    top: 50%;
    width: 100%;
    height: 100%;
    transform: translate(-50%, -50%);
    background-color: #fff;
  }

  .toolbar {
    position: absolute;
    left: 50%;
    top: 10px;
    transform: translateX(-50%);
    z-index: 2;
    display: flex;
    justify-content: center;
  }
}
</style>
