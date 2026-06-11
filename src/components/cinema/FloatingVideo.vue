<script setup lang="ts">
import { ref, watch, onMounted, onBeforeUnmount } from "vue";

const props = withDefaults(
  defineProps<{
    stream?: MediaStream;
    label?: string;
    // 'screen' 用 16:9 较大默认尺寸,'camera' 用 4:3 较小尺寸
    variant?: "screen" | "camera";
    // 多个浮窗按序号错开初始位置,避免完全重叠
    index?: number;
  }>(),
  {
    label: "",
    variant: "camera",
    index: 0
  }
);

const panel = ref<HTMLElement | null>(null);
const video = ref<HTMLVideoElement | null>(null);

// 浮窗位置(fixed 定位,单位 px)
const x = ref(0);
const y = ref(0);
const collapsed = ref(false);

// 把 MediaStream 绑定到 video 元素
const bindStream = () => {
  if (video.value) {
    video.value.srcObject = props.stream ?? null;
  }
};
watch(() => props.stream, bindStream);

// 默认尺寸
const defaultWidth = props.variant === "screen" ? 360 : 220;
const defaultHeight = props.variant === "screen" ? 203 : 165;

const clamp = (val: number, min: number, max: number) =>
  Math.min(Math.max(val, min), max);

// 拖拽逻辑(指针事件,兼容鼠标/触摸)
let dragging = false;
let startPointerX = 0;
let startPointerY = 0;
let startX = 0;
let startY = 0;

const onPointerMove = (e: PointerEvent) => {
  if (!dragging) return;
  const w = panel.value?.offsetWidth ?? defaultWidth;
  const h = panel.value?.offsetHeight ?? defaultHeight;
  // 至少保留窗口在可视区域内(留出标题栏可点)
  x.value = clamp(startX + (e.clientX - startPointerX), 0, window.innerWidth - 40);
  y.value = clamp(startY + (e.clientY - startPointerY), 0, window.innerHeight - 40);
  // 防止整窗完全滑出右/下边
  x.value = clamp(x.value, -(w - 80), window.innerWidth - 40);
  y.value = clamp(y.value, 0, window.innerHeight - 40);
};

const onPointerUp = (e: PointerEvent) => {
  dragging = false;
  try {
    (e.target as HTMLElement).releasePointerCapture?.(e.pointerId);
  } catch {
    // ignore
  }
  window.removeEventListener("pointermove", onPointerMove);
  window.removeEventListener("pointerup", onPointerUp);
};

const onHeaderPointerDown = (e: PointerEvent) => {
  // 仅左键/触摸触发拖拽
  if (e.button !== 0) return;
  dragging = true;
  startPointerX = e.clientX;
  startPointerY = e.clientY;
  startX = x.value;
  startY = y.value;
  try {
    (e.target as HTMLElement).setPointerCapture?.(e.pointerId);
  } catch {
    // ignore
  }
  window.addEventListener("pointermove", onPointerMove);
  window.addEventListener("pointerup", onPointerUp);
};

// 双击视频在全屏/还原间切换
const toggleFullscreen = () => {
  const el = video.value;
  if (!el) return;
  if (document.fullscreenElement) {
    document.exitFullscreen().catch((err) => console.error("退出全屏失败:", err));
  } else {
    el.requestFullscreen().catch((err) => console.error("进入全屏失败:", err));
  }
};

onMounted(() => {
  bindStream();
  // 初始停靠右上区域,按 index 纵向错开
  const offset = props.index * 24;
  x.value = clamp(
    window.innerWidth - defaultWidth - 24 - offset,
    8,
    window.innerWidth - defaultWidth - 8
  );
  y.value = clamp(88 + offset, 8, window.innerHeight - 120);
});

onBeforeUnmount(() => {
  window.removeEventListener("pointermove", onPointerMove);
  window.removeEventListener("pointerup", onPointerUp);
});
</script>

<template>
  <div
    ref="panel"
    class="floating-video"
    :class="[variant, { collapsed }]"
    :style="{
      left: x + 'px',
      top: y + 'px',
      width: defaultWidth + 'px',
      height: collapsed ? 'auto' : defaultHeight + 'px'
    }"
  >
    <div class="fv-header" @pointerdown="onHeaderPointerDown">
      <span class="fv-label" :title="label">{{ label }}</span>
      <span class="fv-actions">
        <button
          class="fv-btn"
          :title="collapsed ? '展开' : '收起'"
          @click.stop="collapsed = !collapsed"
        >
          {{ collapsed ? "▢" : "—" }}
        </button>
      </span>
    </div>
    <video
      v-show="!collapsed"
      ref="video"
      class="fv-video"
      autoplay
      playsinline
      muted
      title="双击全屏"
      @dblclick="toggleFullscreen"
    ></video>
  </div>
</template>

<style scoped lang="scss">
.floating-video {
  position: fixed;
  z-index: 2000;
  display: flex;
  flex-direction: column;
  min-width: 140px;
  min-height: 48px;
  background-color: #000;
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 8px;
  box-shadow: 0 6px 24px rgba(0, 0, 0, 0.45);
  overflow: hidden;
  // 右下角原生缩放手柄
  resize: both;

  &.collapsed {
    resize: none;
  }
}

.fv-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 26px;
  padding: 0 6px;
  background-color: rgba(30, 30, 30, 0.92);
  color: #eee;
  font-size: 12px;
  cursor: move;
  user-select: none;
  touch-action: none;
  flex: 0 0 auto;
}

.fv-label {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

.fv-actions {
  display: flex;
  gap: 4px;
  flex: 0 0 auto;
}

.fv-btn {
  width: 20px;
  height: 18px;
  line-height: 1;
  padding: 0;
  border: none;
  border-radius: 4px;
  background-color: rgba(255, 255, 255, 0.12);
  color: #eee;
  cursor: pointer;
  font-size: 12px;

  &:hover {
    background-color: rgba(255, 255, 255, 0.28);
  }
}

.fv-video {
  flex: 1 1 auto;
  width: 100%;
  min-height: 0;
  background-color: #000;
  // 摄像头铺满裁切,屏幕完整显示
  object-fit: contain;
}

.floating-video.camera .fv-video {
  object-fit: cover;
}
</style>
