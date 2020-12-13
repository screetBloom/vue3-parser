# Vue3 解析

大体结构基本完全兼容 Vue2, 在性能和架构上有了一次比较明显的提升。部分语法有了一些破坏性的改变, 但是应该不会很影响解析
<br>

涉及到解析的一些**核心改变**:

- [组合式 Api](#一组合式-api)
- [新 Vue3 模板特性](#二新-Vue3-模板特性和语法)
- 如 `<script setup>` 这样的 rfc 暂不考虑支持

<br>

**如何解析:**

- [需要做哪些动作](#三如何解析)
- [RoadMap](#RoadMap)

<br>
<br>

## 一、组合式 Api

### 1.1 setup

setup 函数是一个新的组件选项。作为在组件内使用 Composition API 的加载入口

<br>

**特性:**

- 创建组件实例初始化 props 后会调用 `setup` 函数, 在 beforeCreate 钩子之前
- 如果 setup 返回一个对象，则对象的属性将会被合并到组件模板的渲染上下文, `ref` 值会被自动展开, 不需要手动 `x.value`

<!-- &nbsp; -->

<div style="margin-top: 30px"></div>

**代码示例:**

```vue
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

主要手动渲染函数、TSX 和 IDE 工具支持

#### 1.2.1 具有组件选项的对象

```ts
import { defineComponent, ref, reactive } from 'vue'

const MyComponent = defineComponent({
  data() {
    return { countV2: 1 }
  },
  methods: {
    increment() {
      this.countV2++
    },
  },
  setup() {
    const count = ref(0)
    const object = reactive({ foo: 'bar' })

    // 暴露给模板
    return {
      count,
      object,
    }
  },
})
```

<br>

#### 1.2.2 函数(视作 setup 函数)

直接传入一个 `setup` 函数，函数名称将作为组件名称来使用

```ts
import { defineComponent, ref } from 'vue'

const HelloWorld = defineComponent(function HelloWorld() {
  const count = ref(0)
  return { count }
})
```

<br>
<br>
<br>

## 二、新 Vue3 模板特性和语法
只考虑需要解析的部分

### Teleport 组件

teleport 组件它只是单纯的把定义在其内部的内容转移到目标元素中，在元素结构上不会产生多余的元素，也不会影响到组件树，主要目的是为了有更好的代码组织体验
比如：有时，组件模板的一部分在逻辑上属于此组件，但从技术角度来看(如：样式化需求），最好将模板的这一部分移动到 DOM 中的其他位置

```vue
// 使用 <teleport> 并告诉 Vue “teleport 内的 HTML 要渲染到‘#modals’下”。
<teleport to="#modals">
  <div>A</div>
</teleport>
<teleport to="#modals">
  <div>B</div>
</teleport>

<!-- 目标渲染区域-->
<div id="modals">
  <div>A</div>
  <div>B</div>
</div>
```

<br>
<br>

### defineAsyncComponent
Vue3 新增的异步组件
``` js
// (1) 作为组件导出, 但是功能由其它组件实现
export default defineAsyncComponent(() =>
  import('./components/AsyncComponent.vue')
)

// (2) resolve 组件对象, 可能会是 jsx 组件
export default defineAsyncComponent(
  () =>
    new Promise((resolve, reject) => {
      resolve({
        name: 'xxx',
        template: '<div>I am async component!</div>'
      })
    })
)
```

<br>
<br>
<br>

## 三、如何解析

### 3.1 哪些新特性需要额外支持

| 特性                    | 是否需要支持 | 原因                                                                                                                   |
| ----------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------- |
| setup                   | 否           | 解析的目的是为了让调用方了解这个是什么组件, setup 主要是声明组件内部的使用方式, 不需要默认解析, 可以提供 ParserOptions |
| defineComponent         | 是           | SFC 组件 export 改变                                                                             |
| defineAsyncComponent    | 是           | 会影响 export 结果, 可能会直接引入组件或者是 resolve 一个对象式组件(多为 jsx 组件) |
| resolveComponent        | 否           | 仅在 render 和 setup 函数中使用, 不影响组件结果                                                                        |
| resolveDynamicComponent | 否           | 仅在 render 和 setup 函数中使用, 不影响组件结果                                                                        |
| resolveDirective        | 否           | 仅在 render 和 setup 函数中使用, 不影响组件结果                                                                        |
| withDirectives          | 否           | 仅在 render 和 setup 函数中使用, 不影响组件结果                                                                        |
| createRenderer          | 否           | 用于制作自定义渲染器的全局 api                                                                                         |
| nextTick                | 否           | 等价于 Vue2 的 this.$nextTick                                                                                          |

<br>

### 3.2 现有解析需要做的兼容

当前的默认解析结果

```ts
interface ParserResult {
  props?: PropsResult[]
  events?: EventResult[]
  slots?: SlotResult[]
  mixIns?: MixInResult[]
  methods?: MethodResult[]
  name?: string
  componentDesc?: CommentResult
}
```
<br>

#### 3.2.1 props

`props`的声明还是以 props 对象选项的形式支持, 只是会传递给 setup
解析能力不需更新

<br>

#### 3.2.2 events

用法上没变，但需要额外多定义 emits 对象; 更多的是为了描述组件上定义的发出事件, 同时提供了对发出事件进行有效性验证的能力

```js
// (1)
this.$emit('submit', data)
```

```js
// (2) 组件 emits 选项
app.component('my-cmp', {
  emits: ['submit'],
})

// 此部分注释需要采集
app.component('my-cmp', {
  emits: {
    // 没有验证
    click: null,
    // 验证submit 事件
    submit: ({ email, password }) => {
      if (email && password) {
        return true
      } else {
        console.warn('Invalid submit event payload!')
        return false
      }
    },
  },
})
```

<br>

#### 3.2.3 slots

合并了 `slot` 和 `slot-scope` 语法

```vue
<!-- vue 2.x -->
<foo>
  <bar slot="one" slot-scope="one">
    <div slot-scope="bar">
      {{ one }} {{ bar }}
    </div>
  </bar>

  <bar slot="two" slot-scope="two">
    <div slot-scope="bar">
      {{ two }} {{ bar }}
    </div>
  </bar>
</foo>

<!-- vue 3.x -->
<foo>
  <template v-slot:one="one">
    <bar v-slot="bar">
      <div>{{ one }} {{ bar }}</div>
    </bar>
  </template>

  <template v-slot:two="two">
    <bar v-slot="bar">
      <div>{{ two }} {{ bar }}</div>
    </bar>
  </template>
</foo>

<!-- 调用 slot  -->
<TestComponent>
  <template #one="{ name }">Hello {{ name }}</template>
</TestComponent>
```

<br>

#### 3.2.4 mixIns

Vue3 推荐以后能不用 mixIns 就不要再用了, 基本上半废弃了, 不需要增加更多的解析能力

<br>

#### 3.2.5 methods
Vue3 中 methods 的一大改变时, 如果在 setup 中返回了函数, 其相当于在 methods 中定义了组件内函数

```js
setup(){
  const fetchData = ()=>{
    console.log('fetchData')
  }

  return {
    // 等价于在 methods 中定义
    fetchData
  }
}
```

<br>

#### 3.2.6 name
有三种方式会定义组件的name
``` js
// (1) 这里的 name 是 xxx
export default {
  name: 'xxx'
}

// (2) 这里的 name 是 xxx
export default defineComponent({
  name: 'xxx'
})

// (3) 这里的 name 是 HelloWorld
export default defineComponent(function HelloWorld() {
  const count = ref(0)
  return { count }
})

// (4) 异步组件, 没有 name
export default defineAsyncComponent(() =>
  import('./components/AsyncComponent.vue')
)

// (5) 异步组件, name 为 xxx
export default defineAsyncComponent(
  () =>
    new Promise((resolve, reject) => {
      resolve({
        name: 'xxx',
        template: '<div>I am async!</div>'
      })
    })
)
```

<br>

#### 3.2.7 componentDesc
理论上还是在 `export default` 上定义, 和之前不变

<br>
<br>
<br>

## RoadMap

| 能力拆分 | 谁来支持 | 预计完成时间点 |
| -------- | -------- | -------------- |
[defineComponent](#12-definecomponent) | xxx | 2月17号
[defineAsyncComponent](#defineAsyncComponent) | xxx | 2月17号
[teleport 组件](#teleport-组件) | xxx | 2月17号
[events](#322-events) | xxx | 2月17号
[slots](#323-slots) | xxx | 2月17号
[methods](#325-methods) | xxx | 2月17号
[name](#326-name) | xxx | 2月17号




<br>
<br>
<br>