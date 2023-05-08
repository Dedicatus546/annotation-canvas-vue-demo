<script setup lang="ts">
import { ref } from "vue";
import { useEventListener, useElementBounding } from "@vueuse/core";

interface AnnotationItem {
  type: "rect" | "arrow";
  from: {
    x: number;
    y: number;
  };
  to: {
    x: number;
    y: number;
  };
  // 一些其他的字段
}

const canvasRef = ref<HTMLCanvasElement | null>(null);
let mousedown = false;
const annotationList = ref<AnnotationItem[]>([]);
const currentAnnotationItem = ref<AnnotationItem | null>(null);
const type = ref<"arrow" | "rect">("arrow");

const { left: canvasLeft, top: canvasTop } = useElementBounding(canvasRef);

const clearCanvas = () => {
  const el = canvasRef.value!;
  el.width = el.width;
  el.height = el.height;
};

const drawRect = (annotationItem: AnnotationItem) => {
  const ctx = canvasRef.value!.getContext("2d")!;
  const { from, to } = annotationItem;
  const width = Math.abs(from.x - to.x);
  const height = Math.abs(from.y - to.y);
  ctx.beginPath();
  ctx.strokeStyle = "#ff0000";
  ctx.lineWidth = 2;
  ctx.strokeRect(Math.min(from.x, to.x), Math.min(from.y, to.y), width, height);
  ctx.closePath();
};

const drawArrow = (annotationItem: AnnotationItem) => {
  const ctx = canvasRef.value!.getContext("2d")!;
  const { from, to } = annotationItem;
  const d = Math.sqrt(Math.pow(from.y - to.y, 2) + Math.pow(from.x - to.x, 2));
  const h = Math.min(d * 0.2, 35);
  const v = Math.PI / 4;
  const y = Math.atan(Math.abs(to.y - from.y) / Math.abs(to.x - from.x));
  const p = 0.7 * h;
  const m = to.x > from.x ? 1 : -1;
  const g = to.y > from.y ? 1 : -1;
  const p1 = {
    x: to.x - h * Math.cos(v - y) * m,
    y: to.y + h * Math.sin(v - y) * g,
  };
  const p2 = {
    x: to.x - h * Math.cos(v + y) * m,
    y: to.y - h * Math.sin(v + y) * g,
  };
  const p3 = {
    x: to.x - p * Math.cos(v - y - v / 2) * m,
    y: to.y + p * Math.sin(v - y - v / 2) * g,
  };
  const p4 = {
    x: to.x - p * Math.cos(v + y - v / 2) * m,
    y: to.y - p * Math.sin(v + y - v / 2) * g,
  };
  ctx.beginPath();
  ctx.fillStyle = "#ff0000";
  ctx.moveTo(from.x, from.y);
  ctx.lineTo(p3.x, p3.y);
  ctx.lineTo(p1.x, p1.y);
  ctx.lineTo(to.x, to.y);
  ctx.lineTo(p2.x, p2.y);
  ctx.lineTo(p4.x, p4.y);
  ctx.fill();
  ctx.closePath();
};

const drawAnnotationList = () => {
  // 可以使用离屏 canvas 来缓存这些已存在的批注来降低 api 调用次数
  annotationList.value.forEach((item) => {
    if (item.type === "arrow") {
      drawArrow(item);
    } else {
      drawRect(item);
    }
  });
};

const clear = () => {
  annotationList.value = [];
  clearCanvas();
};

useEventListener(canvasRef, "mousedown", ({ clientX, clientY }: MouseEvent) => {
  mousedown = true;
  // 记录起始点信息
  currentAnnotationItem.value = {
    type: type.value,
    from: {
      x: clientX - canvasLeft.value,
      y: clientY - canvasTop.value,
    },
    to: {
      x: 0,
      y: 0,
    },
  };
});
useEventListener("mousemove", ({ clientX, clientY }: MouseEvent) => {
  if (!mousedown) {
    return;
  }
  // 实时更新当前结束点坐标
  Object.assign(currentAnnotationItem.value!.to, {
    x: clientX - canvasLeft.value,
    y: clientY - canvasTop.value,
  });
  clearCanvas();
  drawAnnotationList();
  if (type.value === "rect") {
    drawRect(currentAnnotationItem.value!);
  } else {
    drawArrow(currentAnnotationItem.value!);
  }
});
useEventListener("mouseup", ({ clientX, clientY }: MouseEvent) => {
  if (mousedown) {
    mousedown = false;
    // 记录结束点信息
    Object.assign(currentAnnotationItem.value!.to, {
      x: clientX - canvasLeft.value,
      y: clientY - canvasTop.value,
    });
    // 存到数组中
    annotationList.value.push(Object.assign({}, currentAnnotationItem.value));
    currentAnnotationItem.value = null;
  }
});
</script>

<template>
  <canvas ref="canvasRef" width="1280" height="720" class="canvas"></canvas>
  <br />
  标注类型：<select v-model="type">
    <option value="arrow">箭头</option>
    <option value="rect">方框</option>
  </select>
  |
  <button @click="clear">清除画布</button>
</template>

<style scoped>
.canvas {
  border: 1px solid #666;
}
</style>
