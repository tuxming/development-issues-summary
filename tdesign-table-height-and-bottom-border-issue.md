# TDesign Table 表格高度动态计算与底部边框缺失问题总结

## 问题描述
在使用 TDesign Vue Next 的 Table（或 EnhancedTable）组件时，如果使用 CSS Flex 布局或百分比高度来控制表格高度，往往会因为内部数据量过大，导致表格容器被撑开，进而使页面的分页组件被挤出可视区域外，出现页面级滚动条，体验不佳。
同时，在给 Table 设定了固定高度或最大高度后，由于 TDesign 内部的滚动容器实现机制，表格最后一行数据的底部边框可能会出现丢失的问题。

## 解决方案

### 1. 动态计算表格安全高度 (终极绝招)
为了保证表格在任何情况下都不会撑破屏幕可视区域，放弃依赖外部 Flex 容器自动计算高度，改为**直接通过浏览器视口高度（`window.innerHeight`）减去固定的偏移量**来得出表格的确切像素高度。

**Vue 3 实现代码：**

```typescript
import { ref, computed, onMounted, onUnmounted } from 'vue';

// 1. 响应式记录视口高度
const windowHeight = ref(window.innerHeight);

const handleResize = () => {
  windowHeight.value = window.innerHeight;
};

onMounted(() => {
  window.addEventListener('resize', handleResize);
});

onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
});

// 2. 动态计算表格可用的受控高度
const tableHeight = computed(() => {
  // 终极绝招：直接用浏览器视口高度（window.innerHeight）减去一个安全距离，
  // 这样无论外部怎么撑开，传给 TDesign 的 height/max-height 永远是一个受控的、绝对不会超出屏幕的数字！
  const vh = windowHeight.value;
  // 减去顶部导航、Tab栏、页面 Padding、分页组件等高度，这里假设为 370px。
  // 实际开发中可根据界面的实际布局微调这个数值，直到分页组件刚好贴合屏幕底部边缘。
  return vh - 370; 
});
```

然后在 `<t-enhanced-table>` 或 `<t-table>` 中绑定计算出来的高度：

```html
<t-enhanced-table
  :height="tableHeight"
  class="need-buttom-border"
  ...
/>
```

### 2. 解决最后一行底部边框丢失问题
表格加上固定高度出现内部滚动条后，最后一行数据的底部边框往往会被遮挡或缺失。通过给 table 附加一个自定义的 CSS class（如 `need-buttom-border`），并在全局样式中通过 `!important` 强制补充最后一行单元格的下边框即可解决。

**CSS 修复代码：**

在全局样式文件（如 `src/style/index.less`）中添加以下代码：

```less
/* 强制为 TDesign Table 的最后一行添加底部边框，解决固定高度下的边框丢失问题 */
.need-buttom-border .t-table__body tr:last-child td {
  border-bottom: 1px solid var(--td-component-border) !important;
}
```

## 总结
通过 `window.innerHeight - offset` 的方式能够最暴力且有效地解决一切由于 DOM 层级复杂、Flex 嵌套导致的表格高度失控问题。配合简单的 CSS 后代选择器，完美兼顾了功能交互与 UI 视觉细节。
