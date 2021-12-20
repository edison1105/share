---
# try also 'default' to start simple
theme: seriph
themeConfig: '#41b883'
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---
# Vue3 Virtual DOM 性能优化

<div>

[戴威@edison1105 / Vue.js team member ](https://github.com/edison1105) 

</div>

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

---

# Virtual DOM 性能优化

- **组件的工作原理**
    - 编译阶段
    - 运行时
- **传统 Diff 算法** 
    - 性能的浪费
    - Vue2中 diff 的优化
- **Vue3 Virtual DOM 优化**
    - 什么是 PatchFlags
    - 什么是 BlockTree
    - stable fragment
- **Patch时的改进**
    - 优化模式
    - diff 算法的改进

---

# 组件的工作原理
<div class="flex">
<div class="flex-1">

- 编译阶段
  - Parse: 模板字符串 -> AST
  - Transform: 对 AST 进行转换
  - Generate: AST -> render 函数

</div>
<div class="flex-1 mx-10">

  template

  ```html
  <template>
    <div>{{ msg }}</div>
  </template>
  ```

  [AST](https://astexplorer.net/#/gist/8a0bd75dd94ca7093bde998a0dfccd43/c9e0e68e6f504a333180760ac227b8d88ab640bb)

  render function
    
  ```javascript
  function render(ctx) {
    return h('div', ctx.msg)
  }
  ```

</div>
</div>


---

# 组件的工作原理
<div class="flex">
<div class="flex-1">

  - 运行时
    - 数据响应式处理
    - render 阶段，render函数执行返回 VNode
    - mount 阶段，VNode 渲染成 HTML

</div>

<div class="flex-1 mx-10">

reactive
```javascript
new Proxy(ctx, {
  get(target, key, receiver) {/* 收集依赖 */},
  set(target, key, value, receiver) {/* 触发依赖 */}
})
```

render function
```javascript
function render(ctx) {
  return h('div', ctx.msg)
}
```

VNode
```javascript
const vnode = {
  tag:'div',
  children: [{ text: ctx.msg }]
}
```
</div>
</div>

---

# 组件的工作原理
  - 运行时
    - 响应式数据变化，render 函数重新执行，得到新的 VNode
    - patch 阶段：新、旧 VNode diff，更新 HTML

<div class="flex flex-row">
  <div class="flex-1">
  <div class="flex-1">

  old VNode
  ```javascript
  const vnode = {
    tag:'div',
    children: [{ text: 'hello' }]
  }
  ```
  </div>
  <div class="flex-1">

  new VNode
  ```javascript
  const newVNode = {
    tag:'div',
    children: [{ text: 'world' }]
  }
  ```
  </div>
  </div>
  <div class="flex-1 mt-30 mx-10px">
  
  new HTML

  ```html
  <div>world</div>
  ```

  </div>
</div>

---

# 传统 Diff 算法
  
<div class="flex">
  <div class="flex-1">
  template

  ```html
  <template>
    <div id="content">
      <p class="test">content</p>
      <p class="test">content</p>
      <p class="test">{{ msg }}</p>
      <p class="test">content</p>
      <p class="test">content</p>
    </div>
  </template>
  ```
  </div>
  <div class="flex-1 mx-20px">
  Diff 的过程

  - diff DIV
    - diff props of DIV
    - diff children of DIV
      - diff P (重复多次)
        - diff props of P
        - diff children of P
  </div>
</div>

<div class="mt-30px">

- diff 算法对比颗粒度是组件
- 遍历整个VNode，**深度优先，同层比较**
- 将差异 patch 到真实 DOM 上，减少回流与重绘

</div>
---

# 传统 diff 算法
- 存在性能损耗
  - 需要遍历整个 VNode
  - 静态不会变的节点也参与了 diff

```html {all|1-4,6-9}
  <template>
    <div id="content">
      <p class="test">content</p>
      <p class="test">content</p>
      <p class="test">{{ msg }}</p>
      <p class="test">content</p>
      <p class="test">content</p>
    </div>
  </template>
```

- 思考
  - 那么才能避免性能的浪费？
---

# Vue2 中的优化
Vue2为了向下兼容，采取了比较保守的做法：静态标记
<div class="flex flex-row">
  <div>
  
- 编译期标记静态节点

```javascript
var ast = parse(template.trim(), options);
if (options.optimize !== false) {
  //标记静态节点
  optimize(ast, options);
}
var code = generate(ast, options);
```
- render function
```javascript
{
  render() { return _m(0) },
  staticRenderFns: [function(){ return VNode }]
}
// renderStatic
_m = function (index) {
  const node = staticRenderFns[index]()
  node.isStatic = true
  return node
}

```

  </div>
  <div class="flex-1 mx-10">
  
  - 有哪些节点是静态的？
    - 节点类型是3(纯文本)
    - 节点使用了 v-pre
    - 其他满足以下条件的节点
      - 没有绑定的指令、事件等
      - 不包含v-if、v-for、v-else
      - 不是slot、component
      - 不是组件类型
      - 静态节点的祖先节点，不是带有 v-for 指令的 template 节点
      - 节点只包含staticClass,staticStyle + [基础属性](https://astexplorer.net/#/gist/51a08f2ce6aa6422d7794b7992e7b4da/f48e7715ca6b8feabc1e35ea6e85ba79c2f6903a)

  </div>
</div>

---

# Vue2 中的优化
- 优化后
  - patch 时跳过静态节点 diff
  - 但依然需要遍历整个VNode

```html {all|5}
  <template>
    <div id="content">
      <p class="test">content</p>
      <p class="test">content</p>
      <p class="test">{{ msg }}</p>
      <p class="test">content</p>
      <p class="test">content</p>
    </div>
  </template>
```

- 重新思考
  - 我们其实关心的是动态节点，并不关心静态节点。
  - 有没有办法像标记静态节点那样，找出动态节点，运行时只更新动态节点？
---

<!--
只更新动态节点跟不更新静态节点，其实是有区别的。
1）只更新动态节点可以不按照层级对比，因为在首次渲染之后我们已经将
el缓存到VNode上了，可以直接 patch。
-->

#  Vue3 中的优化
在编译阶段给动态节点标记增加 PatchFlag


<div class="flex flex-row">
  <div class="flex-1">

- template
```html {all|5}
  <template>
    <div id="content">
      <p class="test">content</p>
      <p class="test">content</p>
      <p class="test">{{ msg }}</p>
      <p class="test">content</p>
      <p class="test">content</p>
    </div>
    
  </template>
```

  </div>
  <div class="flex-1 mx-20px">

- render function
```javascript {all|6}
function render() {
  // 此处对代码进行了简化
  return createVNode('div',{ id: "content" }, [
    createVNode('p', { class: "test" }, 'content'),
    createVNode('p', { class: "test" }, 'content'),
    createVNode('p', { class: "test" }, this.msg, PatchFlags.TEXT),
    createVNode('p', { class: "test" }, 'content'),
    createVNode('p', { class: "test" }, 'content'),
  ])
}
```

  </div>
</div>


- [AST](https://astexplorer.net/#/gist/51a08f2ce6aa6422d7794b7992e7b4da/5e22b9e652d7a825b2e9c05327375e45805ff64a)
- PatchFlags 可以简单理解为一个枚举，每个枚举值有不同的含义
  - 1：动态的文本
  - 2：动态的 class
  - //...
---

#  Vue3 中的优化

<div class="flex flex-row">
  <div class="flex-1">

- render function
```javascript
function render() {
  // 此处对代码进行了简化
  return createVNode('div',{ id: "content" }, [
    createVNode('p', { /**/ }, 'content'),
    createVNode('p', { /**/ }, 'content'),
    createVNode('p', { /**/ }, this.msg, PatchFlags.TEXT),
    createVNode('p', { /**/ }, 'content'),
    createVNode('p', { /**/ }, 'content'),
  ])
}
```

  </div>
  <div class="flex-1 mx-20px">

  - VNode

  ```javascript {all|10-12}
  const vnode = {
    type: 'div',
    children: [
      { type: 'p', children: 'content' },
      { type: 'p', children: 'content' },
      { type: 'p', children: ctx.msg, patchFlag: 1 /* TEXT */ }, 
      { type: 'p', children: 'content' },
      { type: 'p', children: 'content' },
    ],
    dynamicChildren: [
      { type: 'p', children: ctx.msg, patchFlag: 1 /* TEXT */ }, 
    ]
  }
  ```

  </div>
</div>

- 在首次渲染时，可以通过一个数组，将动态节点收集起来
- 在 patch 时，就可以只 diff 动态节点
- 拥有 dynamicChildren 属性的 VNode，就是一个 Block

---

#  Vue3 中的优化

<div class="flex flex-row">
  <div class="flex-1">

- template
```html
<template>
  <div>
    <p class="test">{{ foo }}</p>
    <div>
      <span class="test">{{ bar }}</span>
    </div>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - VNode

  ```javascript {all}
  const vnode = {
    type: 'div',
    children: [
      { type: 'p', children: ctx.foo, patchFlag: 1 /* TEXT */ }, 
      { type: 'div', children: [
          { type: 'span', children: ctx.bar, patchFlag: 1 /* TEXT */ }
        ], 
      },
    ],
    dynamicChildren: [
      { type: 'p', children: ctx.foo, patchFlag: 1 /* TEXT */ },
      { type: 'span', children: ctx.bar, patchFlag: 1 /* TEXT */ } 
    ]
  }
  ```

  </div>
</div>

- dynamicChildren 是忽略层级的，会收集所有子代的动态节点（patch 时无需遍历整个 VNode）
- 什么样节点可以作为 Block?
---


#  Block 的特点
什么样的节点可以作为 Block?

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html
<template>
  <div>
    <p class="test">{{ foo }}</p>
    <div>
      <span class="test">{{ bar }}</span>
    </div>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block 的特点（节点的内部结构不会变化）
    - 子节点的数量不会变化
    - 子节点顺序不会变化
  
  </div>
</div>

- 有哪些操作为导致节点的内部结构发生变化
  - v-if 
  - v-for
---

#  节点的内部结构不稳定 v-if

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html
<template>
  <div>
    <div v-if="foo">
      <p>hello</p>
      <div>{{ bar }}</div>
    </div>
    <div v-else>
      <p>world</p>
      <div>{{ bar }}</div>
      <div>{{ baz }}</div>
    </div>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript {all|3-6|7-11}
  const block = {// baz = true
    type: 'div',
    dynamicChildren:[
      { type: 'div',  children: ctx.bar, patchFlag: 1 /* TEXT */ },
    ]
  }
  const block = { // baz = false
    type: 'div',
    dynamicChildren:[
      { type: 'div',  children: ctx.bar, patchFlag: 1 /* TEXT */ },
      { type: 'div',  children: ctx.baz, patchFlag: 1 /* TEXT */ },
    ]
  }

  ```
  </div>
</div>

  - 当 v-if 的值发生变化的时候，动态节点的数量会不一致
  - 会导致 diff 不正确

---

#  v-if 节点作为 Block

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html
<template>
  <div>
    <div v-if="foo">
      <p>hello</p>
      <div>{{ bar }}</div>
    </div>
    <div v-else>
      <p>world</p>
      <div>{{ bar }}</div>
      <div>{{ baz }}</div>
    </div>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript
  Block(div)
    - Block(div v-if)
    - Block(div v-else)

  const block = {
    type: 'div',
    dynamicChildren: [
      //  Block(v-if)
      { type: 'div', { key: 0 }, children:[...], dynamicChildren:[...] },
      //  Block(v-else)
      { type: 'div', { key: 1 }, children:[...], dynamicChildren:[...] },
    ]
  }
  ```
  </div>
</div>

- v-if,v-else 会有不同的key
- key 不相同，会进行 full diff（diff children）
- 多个 Block 嵌套，就构成了 Block Tree
---


#  节点结构不稳定 v-for

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html {all}
<template>
  <div>
    <div v-for="item in list">
      <span>title:</span>
      <span>{{ item }}</span>
    </div>
    <i>{{ foo }}</i>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript
  // list = ['a', 'b']
  const block = {
    type: 'div',
    dynamicChildren:[
      { type: 'span', children: 'a', patchFlag: 1 /* TEXT */},
      { type: 'span', children: 'b', patchFlag: 1 /* TEXT */},
      { type: 'i', children: ctx.foo, patchFlag: 1 /* TEXT */},
    ]
  }
  // list = ['a']
  const block = {
    type: 'div',
    dynamicChildren:[
      { type: 'span', children: 'a', patchFlag: 1 /* TEXT */},
      { type: 'i', children: ctx.foo, patchFlag: 1 /* TEXT */},
    ]
  }
  ```
  </div>
</div>

- 动态节点的数量不一致，怎么 diff?
    - 能将 dynamicChildren 进行传统 diff?

---

#  v-for 作为 Block

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html {3-6}
<template>
  <div>
    <div v-for="item in list">
      <span>title:</span>
      <span>{{ item }}</span>
    </div>
    <i>{{ foo }}</i>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript {all|4}
  const block = {
    type: 'div',
    dynamicChildren: [
      { type: Fragment, dynamicChildren:[...] },// v-for 节点
      { type: 'i', children: ctx.foo, patchFlag: 1 /* TEXT */},
    ]
  }
  ```
  </div>
</div>

- 将整个 v-for 作为一个 Block
- 无论 list 怎么改变，都不会导致外部结构变化

---

#  内部结构不稳定的Fragment

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html
<template>
  <div v-for="item in list">
    <span>title:</span>
    <span>{{ item }}</span>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript
  // list = ['a', 'b']
  const block = {
    type: Fragment,
    children:[...],
    dynamicChildren: [
      { type: 'span', children: 'a', patchFlag: 1 /* TEXT */},
      { type: 'span', children: 'b', patchFlag: 1 /* TEXT */},
    ]
  }

  // list = ['a']
  const block = {
    type: Fragment,
    children:[...],
    dynamicChildren: [
      { type: 'span', children: 'a', patchFlag: 1 /* TEXT */},
    ]
  }
  ```
  </div>
</div>

- 不稳定的 Fragment 需要 full diff
---

#  stable fragment
v-for 的表达式是常量

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html
<template>
  <div v-for="item in [1,2,3]">
    <span>title:</span>
    <span>{{ foo }}</span>
  </div>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript
  const block = {
    type: Fragment,
    dynamicChildren:[{ 
      type: Fragment, 
      dynamicChildren:[
        { type:'span', children: ctx.foo, patchFlag: 1 /* TEXT */},
        { type:'span', children: ctx.foo, patchFlag: 1 /* TEXT */},
        { type:'span', children: ctx.foo, patchFlag: 1 /* TEXT */}
      ]
    }]
  }
  ```
  </div>
</div>

stable fragment 需要 diff dynamicChildren

---

# stable fragment
template v-for

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html {all|2-5}
<template>
  <template v-for="item in list">
    <p>{{ item.name }}</P>
    <p>{{ item.age }}</P>
  </template>
</template> 
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript {all|5-8}
  const block = {
    type: Fragment,
    dynamicChildren:[{ 
      type: Fragment, 
      dynamicChildren:[
        { type:'p', children: item.name, patchFlag: 1 /* TEXT */},
        { type:'p', children: item.age, patchFlag: 1 /* TEXT */}
      ],
      patchFlag: 64 /* STABLE_FRAGMENT */
    },{ 
      type: Fragment, 
      dynamicChildren:[
        { type:'p', children: item.name, patchFlag: 1 /* TEXT */},
        { type:'p', children: item.age, patchFlag: 1 /* TEXT */}
      ],
      patchFlag: 64 /* STABLE_FRAGMENT */
    },...],
    patchFlag: 256 /* UNKEYED_FRAGMENT */
  }
  ```
  </div>
</div>


---

#  stable fragment
多个根节点

<div class="flex flex-row">
  <div class="flex-1">
  
- template

```html
<template>
  <span v-if="x">a</span>
  <div>b</div>
  <p>c</p>
</template>
```

  </div>
  <div class="flex-1 mx-20px">

  - Block
  
  ```javascript
  Block(Fragment)
    - Block(span)
    - VNode(div)
    - VNode(p)
  ```
  </div>
</div>

---

# Block Tree

![BlockTree](/assets/blocktree.jpg)

- 将模板基于动态节点指令切割为嵌套的区块。
- 每个区块内部的节点结构是固定的。
- 每个区块只需要以一个 Array 追踪自身包含的动态节点。

优化后 Virtual DOM 更新性能能由与模板整体大小相关提升为与动态内容的数量相关。

---

# 优化模式

- patchBlockChildren


---
layout: center
class: text-center
---
# THANKS

[戴威 / Vue.js team member ](https://github.com/edison1105) 

