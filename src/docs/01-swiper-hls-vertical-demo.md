# Vue3 + Swiper + HLS.js 实现全屏垂直无限滚动视频轮播实战总结

## 1. 技术选型与依赖

- **Vue3**：主流前端框架，支持 Composition API。
- **Swiper**：官方推荐的轮播库，支持 Vue3，功能强大，支持模块化扩展。
- **HLS.js**：浏览器端播放 HLS（.m3u8）流的最佳选择，兼容性好。

### 依赖安装

```bash
pnpm add swiper hls.js
```

## 2. Swiper 官方 Vue 组件用法

### 2.1 模块注册

- Swiper v10+ 推荐通过 `:modules` 属性注册扩展模块（如 Mousewheel、Keyboard、Pagination）。
- 不再使用 `SwiperCore.use()` 或 `swiper/core`。

```js
import { Swiper, SwiperSlide } from 'swiper/vue';
import { Mousewheel, Keyboard, Pagination } from 'swiper/modules';

const modules = [Mousewheel, Keyboard, Pagination];
```

### 2.2 样式引入

- Swiper v10+ 样式需单独引入（如 `import 'swiper/css'`、`import 'swiper/css/pagination'`）。
- 注意：部分编辑器可能报“找不到模块”，但实际运行无影响。

## 3. 组件实现要点

### 3.1 组件结构

- 使用 `<Swiper>` 和 `<SwiperSlide>` 组件包裹每个视频。
- 通过 `v-for` 渲染视频列表。

### 3.2 HLS 视频初始化

- 用 `ref` 绑定所有 video 元素，`onMounted` 后批量初始化 HLS.js。
- 组件销毁时需销毁所有 HLS 实例，防止内存泄漏。

### 3.3 Swiper 配置

- `direction: 'vertical'` 实现垂直滑动。
- `loop: true` 无限循环。
- `mousewheel: true` 支持鼠标滚轮。
- `keyboard: { enabled: true, onlyInViewport: true }` 支持键盘切换。
- `pagination: { clickable: true }` 显示可点击分页器。
- `:modules="modules"` 注册扩展模块。

### 3.4 样式处理

- `.video-swiper.vertical-swiper` 需 `position: fixed; left: 0; top: 0; width: 100vw; height: 100vh;`，确保全屏无空白。
- `.swiper-wrapper`、`.swiper-slide`、`.video-container`、`.hls-video` 都需 100vw/100vh，且无 margin/padding。
- `object-fit: contain` 保证视频自适应居中。

## 4. 易错点与排查

### 4.1 Swiper 版本与用法不兼容

- Swiper v8 及以下用 `SwiperCore.use()`，v10+ 推荐 `:modules` 属性。
- 误用 vue-awesome-swiper@5 只会 re-export swiper/vue，建议直接用官方组件。

### 4.2 样式未生效导致内容溢出或留白

- Swiper、slide、video 必须全部 100vw/100vh，且无多余 margin/padding。
- Swiper 容器建议用 `position: fixed`，避免父级影响。

### 4.3 HLS.js 多实例未销毁

- 组件销毁时需遍历所有 hls 实例调用 `destroy()`。

### 4.4 编辑器类型声明报错

- Swiper 的类型声明有时因包导出方式报错，不影响实际运行，可忽略。

### 4.5 slot/pagination 兼容

- Swiper v10+ 分页器无需 slot，直接用 `:pagination` 配置即可。

## 5. 完整代码片段参考

```vue
<template>
  <Swiper
    :direction="'vertical'"
    :loop="true"
    :slides-per-view="1"
    :mousewheel="true"
    :keyboard="{ enabled: true, onlyInViewport: true }"
    class="video-swiper vertical-swiper"
    ref="swiperRef"
    :pagination="{ clickable: true }"
    :modules="modules"
  >
    <SwiperSlide v-for="(video, idx) in videos" :key="idx">
      <div class="video-container">
        <video
          ref="videoEls"
          :data-idx="idx"
          class="hls-video"
          controls
          playsinline
          preload="metadata"
        ></video>
      </div>
    </SwiperSlide>
  </Swiper>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue';
import { Swiper, SwiperSlide } from 'swiper/vue';
import { Mousewheel, Keyboard, Pagination } from 'swiper/modules';
import 'swiper/css';
import 'swiper/css/pagination';
import Hls from 'hls.js';

const modules = [Mousewheel, Keyboard, Pagination];

const videos = [
  'https://test-streams.mux.dev/x36xhzz/x36xhzz.m3u8',
  'https://test-streams.mux.dev/test_001/stream.m3u8',
  'https://test-streams.mux.dev/dai-discontinuity-daterange/dai.m3u8',
];

const videoEls = ref([]);
const hlsInstances: Hls[] = [];
const swiperRef = ref();

function initHls(videoEl: HTMLVideoElement, src: string) {
  if (videoEl.canPlayType('application/vnd.apple.mpegurl')) {
    videoEl.src = src;
  } else if (Hls.isSupported()) {
    const hls = new Hls();
    hls.loadSource(src);
    hls.attachMedia(videoEl);
    hlsInstances.push(hls);
  }
}

onMounted(async () => {
  await nextTick();
  videoEls.value.forEach((el: HTMLVideoElement, idx: number) => {
    if (el) initHls(el, videos[idx]);
  });
  if (swiperRef.value && swiperRef.value.swiper) {
    swiperRef.value.swiper.update();
  }
});

onBeforeUnmount(() => {
  hlsInstances.forEach((hls) => hls.destroy());
});
</script>

<style scoped>
.video-swiper.vertical-swiper {
  width: 100vw;
  height: 100vh;
  background: #000;
  position: fixed;
  left: 0;
  top: 0;
  margin: 0;
  padding: 0;
}
.video-swiper.vertical-swiper .swiper-wrapper,
.video-swiper.vertical-swiper .swiper-slide {
  width: 100vw !important;
  height: 100vh !important;
}
.video-container {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
}
.hls-video {
  width: 100vw;
  height: 100vh;
  object-fit: contain;
  background: #000;
  margin: 0;
  padding: 0;
  display: block;
}
</style>
```

---

## 6. 总结

- 推荐直接用 Swiper 官方 Vue 组件，避免社区桥接库的兼容性问题。
- 样式细节决定体验，务必让所有容器和内容 100vw/100vh。
- HLS.js 多实例管理要规范，防止内存泄漏。
- 遇到类型声明报错可忽略，重点关注实际运行效果。

希望本文能帮助你快速、高质量实现基于 Vue3 + Swiper + HLS.js 的全屏垂直无限滚动视频轮播！
