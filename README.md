1. Vue 3.0 性能提升主要是通过哪几个方面体现的
   - 响应式系统升级
     - Vue.js 2.x 中响应式系统的核心 `defineProperty`
     - Vue.js 3.0 中使用 Proxy 对象重写响应式系统
       - 可以监听动态新增的属性
       - 可以监听删除的属性
       - 可以监听数组的索引和 length 属性
   - 编译优化
     - Vue.js 2.x 中通过标记静态根节点，优化 diff 的过程
     - Vue.js 3.0 中标记和提升所有的静态根节点，diff 时只需要对比动态节点内容
       - `Fragments` （升级 vetur 插件）
       - 静态提升
       - `Patch flag`
       - 缓存事件处理函数
   - 源码体积的优化
     - Vue.js 3.0 中移除了一些不常用的 API
       - 例如：`inline-template`、`filter` 等
     - `Tree-shaking`
2. Vue 3.0 所采用的 Composition Api 和 Vue 2.x 使用的 Options Api 有什么区别
3. Proxy 相对于 Object.defineProperty 有哪些优点
4. Vue 3.0 在编译方面有哪些优化
5. Vue.js 3.0 响应式系统的实现原理