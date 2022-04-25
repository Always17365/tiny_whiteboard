<template>
  <div class="container" ref="container">
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script setup>
import { onMounted, ref } from "vue";

// 初始化画布
const container = ref(null);
const canvas = ref(null);
let ctx = null;
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

// 监听事件
const bindEvent = () => {
  canvas.value.addEventListener("mousedown", onMousedown);
  canvas.value.addEventListener("mousemove", onMousemove);
  canvas.value.addEventListener("mouseup", onMouseup);
};
const screenToCanvas = (x, y) => {
  return {
    x: x - canvas.value.width / 2,
    y: y - canvas.value.height / 2,
  };
};
let mousedownX = 0;
let mousedownY = 0;
let isMousedown = false;
const onMousedown = (e) => {
  mousedownX = e.clientX;
  mousedownY = e.clientY;
  isMousedown = true;
};
let activeElement = null;
const onMousemove = (e) => {
  if (!isMousedown) {
    return;
  }
  if (!activeElement) {
    activeElement = new Rectangle({
      x: mousedownX,
      y: mousedownY,
    });
  }
  // 更新矩形的大小
  activeElement.width = e.clientX - mousedownX;
  activeElement.height = e.clientY - mousedownY;
  clearCanvas();
  activeElement.render();
};
const onMouseup = (e) => {
  isMousedown = false;
  activeElement = null;
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
  }

  render() {
    ctx.beginPath();
    let canvasPos = screenToCanvas(this.x, this.y);
    ctx.rect(canvasPos.x, canvasPos.y, this.width, this.height);
    ctx.stroke();
  }
}

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
}
</style>
