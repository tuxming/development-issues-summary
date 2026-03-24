# TDesign Vue Next 树形表格动态更新与懒加载追加子节点问题总结

## 1. 背景与问题现象

在使用 TDesign Vue Next 的 `<t-enhanced-table>` 组件开发树形表格（Tree Table）时，当需要进行**懒加载（lazy-load）**并且在前端**动态追加子节点**或**动态将叶子节点转换为父节点**时，遇到了以下严重问题：

1. **叶子节点转换 Bug**：
   当一个原本没有子节点的行（叶子节点），在后端添加了子节点后，前端试图通过 `tableRef.value.setData()` 修改其 `hasChildren` 状态并追加子节点时，表格渲染崩溃，节点行显示出 `[object Object]`。
2. **`appendTo` 定位错乱**：
   在复杂的懒加载场景下，直接对刚刚转换为父节点的行使用 `appendTo` 方法追加数据，新节点会被错误地追加到表格根部，而不是指定的父节点下。
3. **全表刷新与展开状态丢失**：
   为了规避上述 Bug，尝试过全局刷新 `fetchData()` 或深拷贝重置数据源 `JSON.parse(JSON.stringify(data.value))`，这导致了表格所有已展开的层级状态全部丢失，且破坏了 Vue 响应式引用，体验极差。

## 2. 问题核心原因分析

经过深入排查 TDesign 的 API 和内部机制，发现问题的根源在于对 **`TableRowState` 代理对象**与**原始业务数据 (`row`)** 的混用：

- **`getData(id)` 的返回值误区**：
  调用 `tableRef.value.getData(id)` 返回的并不是传入的纯净业务数据，而是一个被 TDesign 包装过的代理状态对象 `TableRowState`。它内部包含了大量的组件渲染状态（如 `expanded`, `level`, `rowIndex`, `parent`，甚至包含对自身的循环引用）。
- **`setData` 的覆盖灾难**：
  如果直接将从 `getData()` 获取到的代理对象（或修改了该对象）再次传给 `setData(id, newData)`，会导致 TDesign 内部的树形响应式追踪发生死循环或结构破坏，从而渲染出 `[object Object]` 或引发深拷贝的 `Circular structure` 报错。

## 3. 终极完美解决方案

彻底抛弃复杂的全表轮询和深拷贝，**严格区分“代理状态”与“原始数据”，利用组件自身的懒加载机制完成更新**。

### 3.1 提取原始数据更新
当需要把叶子节点变成父节点时，必须从 `rowState` 中提取纯净的 `row`（业务数据），修改后进行 `setData`。

```javascript
// 错误做法 ❌
const rowState = tableRef.value.getData(id);
rowState.hasChildren = true;
tableRef.value.setData(id, rowState); // 导致 [object Object]

// 正确做法 ✅
const rowState = tableRef.value.getData(id);
const rawData = { ...rowState.row }; // 提取纯净业务数据
rawData.hasChildren = true;
rawData.children = true; // 设置为 true 触发懒加载标识
tableRef.value.setData(id, rawData);
```

### 3.2 巧妙利用内置懒加载机制
当节点被正确标记为父节点后，**不要手动去查数据并 `appendTo`**。而是利用代码模拟点击“展开”，让组件自身的 `onExpandedTreeNodesChange` 去接管后续的网络请求和数据挂载。

```javascript
// 修改状态为父节点后，等待 DOM 刷新，模拟触发展开
setTimeout(() => {
  const newState = tableRef.value.getData(id);
  if (newState && !newState.expanded) {
    // 触发组件内置事件，自动拉取子节点并追加
    tableRef.value.toggleExpandData(newState.row); 
  }
}, 100);
```

### 3.3 已展开父节点的局部刷新
如果节点本来就是父节点且已经展开，新增了子节点后如何局部刷新？
最稳妥的方式是：重新请求该节点的最新子级列表 -> `removeChildren` 清空旧列表 -> `appendTo` 追加新列表。

```javascript
// 重新从后台拉取该父节点的所有子节点
const result = await request.post(api.list, { parentId: id });
if (result.status && result.data && result.data.list) {
  const children = result.data.list.map((item) => {
    if (item.hasChildren) return { ...item, children: true };
    return item;
  });
  // 移除旧的子节点，追加新的子节点，完成局部刷新！
  tableRef.value.removeChildren(id);
  tableRef.value.appendTo(id, children);
}
```

## 4. 总结经验

在操作 Vue 复杂组件（如 TDesign 的树形表格）时：
1. **警惕框架注入的属性**：永远不要把组件暴露出来的内部状态对象（包含 `$` 或嵌套引用的代理对象）当做业务数据去修改和覆盖。
2. **顺应组件生命周期**：与其自己写复杂的逻辑去组装树结构，不如利用组件自带的“懒加载事件触发机制”，改变状态后让组件自己去完成闭环。
