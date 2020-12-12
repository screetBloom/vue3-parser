# Vue3 解析
大体结构基本完全兼容 Vue2, 在性能和架构上有了一次比较明显的提升。部分语法有了一些破坏性的改变, 但是应该不会很影响解析
<br>

涉及到解析的一些核心改变:
- [组合式 Api](#一组合式-api)
- [Vue3 模板语法和指令](#二vue3-模板语法和指令)
- 如 `<script setup>` 这样的rfc(暂不考虑支持)

<br>
<br>

## 一、组合式 Api

### 1.1 setup
setup 函数是一个新的组件选项。作为在组件内使用 Composition API 的入口点

<br>

**特性:**
- 创建组件实例初始化 props 后会调用 `setup` 函数, 在 beforeCreate 钩子之前
- 如果 setup 返回一个对象，则对象的属性将会被合并到组件模板的渲染上下文, `ref` 值会被自动展开, 不需要手动 `x.value`

<!-- &nbsp; -->

<div style="margin-top: 10px"></div>

**代码示例:**

``` vue
<template>
  <div>{{ count }} {{ object.foo }}</div>
</template>

<script>
  import { ref, reactive } from 'vue'

  export default {
    setup() {
      const count = ref(0)
      const object = reactive({ foo: 'bar' })

      // 暴露给模板
      return {
        count,
        object,
      }
    },
  }
</script>
```

<br>
<br>

### 1.2 defineComponent
Composition api体验
vue3 parser

<br>
<br>
<br>

## 二、Vue3 模板语法和指令



<br>
<br>
<br>

