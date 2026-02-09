---
creation_date: 2026-01-04 10:55
type: #Type/Concept
status: #Status/Refactoring
tags: [WebGL, UI, Rendering, DPI]
aliases: [WebGL字体模糊, Canvas Resolution, High DPI]
tech_stack: #Tech/Unity
complexity: ⭐⭐
related_modules: []
---

# Unity WebGL 字体渲染模糊解决方案

## 1. 核心摘要 (Abstract)
> 解决 WebGL 平台下由于逻辑分辨率与物理像素比（Device Pixel Ratio）不匹配导致的 UI/字体渲染模糊问题。

## 2. 详细分析 (Implementation Detail)
- **背景/痛点**：在 WebGL 构建中，Unity 默认的 Canvas 分辨率可能与浏览器的显示分辨率不一致，导致高分屏（Retina/HiDPI）下文字和 UI 边缘出现明显的锯齿或模糊。
- **底层原理**：浏览器渲染机制中，`drawingBufferWidth/Height`（渲染缓冲区大小）与 CSS `width/height`（显示大小）分离。默认情况下两者 1:1，但在高 DPI 设备上需要提升缓冲区大小以匹配物理像素。
- **解决方案/结论**：通过 JS 脚本手动调整 Canvas 的 width/height 属性为 `clientWidth * window.devicePixelRatio`，实现类似超采样（Super-sampling）的效果。需权衡显存（VRAM）占用与渲染性能。

## 3. 关联知识 (References)
- [[Device Pixel Ratio]]
- [[WebGL Memory Management]]
- [[Anti-Aliasing Strategies]]