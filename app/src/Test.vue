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
// 当前按住了激活元素激活态的哪个区域
let hitActiveElementArea = "";
// 检测是否击中了某个元素
const checkIsHitElement = (x, y) => {
  let hitElement = null;
  // 从后往前遍历元素，即默认认为新的元素在更上层
  for (let i = allElements.length - 1; i >= 0; i--) {
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
      let hitArea = activeElement.isHitActiveArea(mousedownX, mousedownY);
      if (hitArea) {
        // 按住了按住了激活状态的某个区域
        isAdjustmentElement = true;
        hitActiveElementArea = hitArea;
        activeElement.save();
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
  if (!isMousedown) {
    return;
  }
  if (currentType.value === "selection") {
    if (isAdjustmentElement) {
      // 调整元素中
      let ox = e.clientX - mousedownX;
      let oy = e.clientY - mousedownY;
      if (hitActiveElementArea === "body") {
        // 进行移动操作
        activeElement.moveBy(ox, oy);
      } else if (hitActiveElementArea === "rotate") {
        // 进行旋转操作
        // 矩形的中心点
        let center = getRectangleCenter(activeElement);
        let or = getTowPointRotate(
          center.x,
          center.y,
          mousedownX,
          mousedownY,
          e.clientX,
          e.clientY
        );
        activeElement.rotateBy(or);
      }
      renderAllElements();
    }
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
  if (isAdjustmentElement) {
    isAdjustmentElement = false;
    hitActiveElementArea = "";
  }
};

// 矩形类
class Rectangle {
  constructor(opt) {
    this.x = opt.x || 0;
    this.y = opt.y || 0;
    // 记录矩形的初始位置
    this.startX = 0;
    this.startY = 0;
    // 旋转角度
    this.rotate = opt.rotate || 0;
    // 记录矩形的初始角度
    this.startRotate = 0;
    this.width = opt.width || 0;
    this.height = opt.height || 0;
    this.isActive = false;
  }

  render() {
    ctx.save();
    let canvasPos = screenToCanvas(this.x, this.y);
    // 将画布原点移动到自身的中心
    let halfWidth = this.width / 2;
    let halfHeight = this.height / 2;
    ctx.translate(canvasPos.x + halfWidth, canvasPos.y + halfHeight);
    // 旋转
    ctx.rotate(degToRad(this.rotate));
    // 原点变成自身中心，那么自身的坐标其实变成了-width/2、-height/2
    drawRect(-halfWidth, -halfHeight, this.width, this.height);
    this.renderActiveState();
    ctx.restore();
  }

  // 当激活时渲染激活态
  renderActiveState() {
    if (!this.isActive) {
      return;
    }
    let halfWidth = this.width / 2;
    let halfHeight = this.height / 2;
    // 为了不和矩形重叠，虚线框比矩形大一圈，增加5px的内边距
    let x = -halfWidth - 5;
    let y = -halfHeight - 5;
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
    // 反向旋转矩形的角度
    let center = getRectangleCenter(this);
    let rotatePoint = getRotatedPoint(x0, y0, center.x, center.y, -this.rotate);
    x0 = rotatePoint.x;
    y0 = rotatePoint.y;
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
    // 反向旋转矩形的角度
    let center = getRectangleCenter(this);
    let rotatePoint = getRotatedPoint(x0, y0, center.x, center.y, -this.rotate);
    x0 = rotatePoint.x;
    y0 = rotatePoint.y;
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

  // 保存矩形此刻的状态
  save() {
    this.startX = this.x;
    this.startY = this.y;
    this.startRotate = this.rotate;
  }

  // 移动矩形
  moveBy(ox, oy) {
    this.x = this.startX + ox;
    this.y = this.startY + oy;
  }

  // 旋转矩形
  rotateBy(or) {
    this.rotate = this.startRotate + or;
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

// 弧度转角度
const radToDeg = (rad) => {
  return rad * (180 / Math.PI);
};

// 角度转弧度
const degToRad = (deg) => {
  return deg * (Math.PI / 180);
};

// 计算两个坐标以同一个中心点构成的角度
const getTowPointRotate = (cx, cy, tx, ty, fx, fy) => {
  return radToDeg(Math.atan2(fy - cy, fx - cx) - Math.atan2(ty - cy, tx - cx));
};

// 计算矩形的中心点
const getRectangleCenter = ({ x, y, width, height }) => {
  return {
    x: x + width / 2,
    y: y + height / 2,
  };
};

// 获取坐标经指定中心点旋转指定角度的坐标
const getRotatedPoint = (x, y, cx, cy, rotate) => {
  let deg = radToDeg(Math.atan2(y - cy, x - cx));
  let del = deg + rotate;
  let dis = getTowPointDistance(x, y, cx, cy);
  return {
    x: Math.cos(degToRad(del)) * dis + cx,
    y: Math.sin(degToRad(del)) * dis + cy,
  };
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
