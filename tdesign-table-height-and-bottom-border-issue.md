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

### 3. 表格实际高度与设置高度不一致导致无法出现滚动条问题
在给 `<t-enhanced-table>` 或 `<t-table>` 设置了具体的 `height` 属性后，有时会发现表格并没有按照预期的高度渲染（实际高度超出了设置的高度），导致应该在表格内部出现的滚动条没有出现，反而撑开了外层容器出现了页面级的滚动条。

**原因分析：**
这通常是因为表格处于某个 CSS 容器（例如 Flex 布局或者某个特定的 wrapper 容器）中，该外层容器没有限制其自身的最大高度，导致被表格内部的数据直接撑开。即便我们通过计算得到了一个安全的 `tableHeight` 并传递给 TDesign，但如果外层容器允许无限扩展，TDesign 内部的一些高度计算逻辑或者虚拟滚动逻辑就无法正确感知到边界，从而导致滚动条失效。

**解决办法：**
确保包裹表格的外层 DOM 元素（或组件）不能被子元素无限撑开。
最直接的方法是在包裹表格的外层 div 上也加上高度限制，或者更简单的，给表格外层的包裹容器（比如 `<div class="table-wrap">`）添加绝对的高度或者 `overflow: hidden;`，以强制划定边界。如果是在 Flex 布局中，需要给外层 Flex 容器加上 `min-height: 0;`。

**示例代码：**
如果是普通的 div 包裹，可以直接将计算好的 `tableHeight` 也应用到外层：
```html
<div class="table-wrap" :style="{ height: tableHeight + 'px', overflow: 'hidden' }">
  <t-enhanced-table
    :height="tableHeight"
    :scroll="{ type: 'virtual', rowHeight: 48, bufferSize: 20 }"
    ...
  />
</div>
```

## 总结
通过 `window.innerHeight - offset` 的方式能够最暴力且有效地解决一切由于 DOM 层级复杂、Flex 嵌套导致的表格高度失控问题。配合简单的 CSS 后代选择器，完美兼顾了功能交互与 UI 视觉细节。
