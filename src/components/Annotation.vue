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

const {
  left: canvasLeft,
  top: canvasTop,
  width: canvasWidth,
  height: canvasHeight,
} = useElementBounding(canvasRef);

const clearCanvas = () => {
  const el = canvasRef.value!;
  el.width = el.width;
  el.height = el.height;
};

const drawRect = () => {
  const ctx = canvasRef.value!.getContext("2d")!;
  const { from, to } = currentAnnotationItem.value!;
  const width = Math.abs(from.x - to.x);
  const height = Math.abs(from.y - to.y);
  ctx.beginPath();
  ctx.strokeStyle = "#ff0000";
  ctx.lineWidth = 2;
  ctx.strokeRect(Math.min(from.x, to.x), Math.min(from.y, to.y), width, height);
  ctx.closePath();
};

// const drawArrow = () => {
//   const ctx = canvasRef.value!.getContext("2d")!;
//   const { from, to } = currentAnnotationItem.value!;
//   const diagonal = Math.sqrt(
//     Math.pow(to.x - from.x, 2) + Math.pow(to.y - from.y, 2)
//   );
//   ctx.beginPath();
//   ctx.fillStyle = c;
//   ctx.moveTo(from.x, from.y);
//   ctx.lineTo(_.x, _.y);
//   ctx.lineTo(b.x, b.y);
//   ctx.lineTo(f.x, f.y);
//   ctx.lineTo(w.x, w.y);
//   ctx.lineTo(E.X, E.y);
//   ctx.fill();
//   ctx.closePath();
// };

useEventListener(canvasRef, "mousedown", ({ clientX, clientY }: MouseEvent) => {
  mousedown = true;
  // 记录起始点信息
  currentAnnotationItem.value = {
    type: "rect",
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
  drawRect();
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
</template>

<style scoped>
.canvas {
  border: 1px solid #666;
}
</style>
