---
title: Vue keep-alive应用与源码分析
date: 2017-12-21 23:00:00
categories:
- Font-end
tags:
- Vue
---

## 前言
在做Vue单页面应用时，经常会遇到页面之间来回切换的情况，每个页面可能会有不少网络请求，但是部分页面的数据不需要每次进入都请求。若每次进入页面都需要进行网络请求，无疑会增加不必要的耗时，并且页面的重新渲染也会造成一定的性能浪费。而Vue 2.0+提供了一个内置组件`keep-alive`用来缓存组件，有效地减少性能消耗，下面谈谈 `keep-alive` 的用法与其源码分析。

## 应用
### 基本用法
如果把切换出去的组件（页面）保留在内存中，可以保留它的状态或避免重新渲染。Vue 2.0+提供了一个内置组件`keep-alive`用来缓存组件，减少性能消耗，一般用法如下：

```
<keep-alive>
  <component>
    <!-- 组件将被缓存 -->
  </component>
</keep-alive>
```

比较常用的方法是搭配 **router** 一起使用，缓存不常用的页面。当组件在 `<keep-alive>` 内被切换，它的 `activated` 和 `deactivated` 这两个生命周期钩子函数将会被对应执行。

```
<keep-alive>
  <router-view/>
</keep-alive>
```

使用 **vue-cli** 新建一个路由demo，新建 CompA 和 CompB 单文件组件。
CompA 与 CompB 可互相切换，并且在 CompB 所有生命周期钩子函数中打印调用 log：

```
<!-- CompB.vue -->
<template>
  <div>
    <h1>{{msg}}</h1>
    <router-link to="/CompA">Go to CompA</router-link>
  </div>
</template>

<script>
export default {
  data () {
    return {
      msg: 'Comp B'
    }
  },
  beforeCreate: function () {
    console.log('1 beforeCreate')
  },
  created: function () {
    console.log('2 created')
  },
  beforeMount: function () {
    console.log('3 beforeMount')
  },
  mounted: function () {
    console.log('4 mounted')
  },
  beforeDestroy: function () {
    console.log('5 beforeDestroy')
  },
  destroyed: function () {
    console.log('6 destroyed')
  },
  activated: function () {
    console.log('activated')
  },
  deactivated: function () {
    console.log('deactivated')
  }
}
</script>
```

若根组件的 `<router-view>` 在没有嵌套在 `<keep-alive>` 标签内，访问路径 CompA -> CompB -> CompA ，控制台将依次打印

```
// CompA -> CompB
1 beforeCreate
2 created
3 beforeMount
4 mounted
// CompB -> CompA
5 beforeDestroy
6 destroyed
```

修改 App.vue，引入 `<keep-alive>`

```
<!-- App.vue -->
<template>
  <div>
    <router-view/>
  </div>
</template>
·····
```

重新访问路径 CompA -> CompB -> CompA，此时控制台将依次打印

```
// CompA -> CompB
1 beforeCreate
2 created
3 beforeMount
4 mounted
activated
// CompB -> CompA
deactivated
```

可见，使用 `keep-alive` 后，CompB的销毁钩子函数没有被执行，CompB 被成功缓存（CompA 亦然）。

### 缓存部分页面或组件
Vue 2.1.0 新增 props: `include` 和 `exclude`，官方解释如下

* `include` - 字符串或正则表达式。只有匹配的组件会被缓存。
* `exclude` - 字符串或正则表达式。任何匹配的组件都不会被缓存。

```
<!-- 逗号分隔字符串 -->
<keep-alive include="a,b">
  <router-view/>
</keep-alive>

<!-- 正则表达式 (使用 `v-bind`) -->
<keep-alive :include="/a|b/">
  <router-view/>
</keep-alive>

<!-- 数组 (使用 `v-bind`) -->
<keep-alive :include="['a', 'b']">
  <router-view/>
</keep-alive>
```

显然，数组类型也是支持的。匹配首先检查组件自身的 name 选项，`include` 和 `exclude` 只作用于具名组件，匿名组件不能被匹配。

## 源码分析
`keep-alive` 是Vue的内置组件，源码位于 [src/core/components/keep-alive.js](https://github.com/vuejs/vue/blob/dev/src/core/components/keep-alive.js)。

### `created` 与 `destroyed` 钩子
`created` 钩子设置2个关键属性 `cache` 和 `keys`，其中`cache`用于保存缓存的组件，`keys`记录缓存组件的顺序
`destroyed` 钩子对每一个被缓存的组件执行 `pruneCacheEntry` 方法，该方法内部将调用每一个组件实例的 `$destroy` 生命周期钩子进行销毁

```
······
created () {
  this.cache = Object.create(null)
  this.keys = []
},

destroyed () {
  for (const key in this.cache) {
    pruneCacheEntry(this.cache, key, this.keys)
  }
}
······
```

### render 函数

```javascript
// src/core/components/keep-alive.js
······
export default {
  ......

  render () {
    const vnode: VNode = getFirstComponentChild(this.$slots.default)
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) {
      // check pattern for include/exclude
      ······
      
      const { cache, keys } = this
      const key: ?string = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (#3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance
        // make current key freshest
        remove(keys, key)
        keys.push(key)
      } else {
        cache[key] = vnode
        keys.push(key)
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
    }
    return vnode
  }
}
```

render 函数只会返回其包裹的子组件，意味着本身作为一个抽象组件，不会渲染一个 DOM 元素，也不会出现在父组件链中。注意到，render 函数返回的是 VNode 对象，即 Virtual DOM 对象，用 JavaScript 对象来代替 DOM 节点。若子组件已被缓存，则直接返回，并刷新组件顺序；若未被缓存，则将其插入 cache，此时如果缓存的组件数量达到设置的上限，就移除最早缓存的组件，缓存组件的数量默认不设置，即无上限。

### `include` 与 `exclude`
在上面的应用中说到：`include` 和 `exclude` 只作用于具名组件，匿名组件不能被匹配。这里的不能被匹配是指 `include` 或 `exclude` 在匿名组件上不起作用。
依旧以 CompA 与 CompB 的 DEMO 为例，设置 `include` 属性，并对 CompA 设置 `name` 属性

```
<keep-alive include="CompA">
  <router-view/>
</keep-alive>
```

从 CompA -> CompB，控制台显示 CompB 依然执行了 `activated` 钩子，即被缓存了。若对 CompB 设置 `name` 属性，则不会执行 `activated` 钩子。render 函数中关键代码如下，匿名组件无法读取 name 属性，将参与缓存处理。

```
render () {
······
  const name: ?string = getComponentName(componentOptions)
  if (name && (
    (this.exclude && matches(this.exclude, name)) ||
    (this.include && !matches(this.include, name))
  )) {
    return vnode
  }
······
}
```

### 最大缓存组件数量
`keep-alive` 组件定义了 `props` 属性

```
props: {
  include: patternTypes,
  exclude: patternTypes,
  max: [String, Number]
}
```

官方文档没有提供 `max` 属性的使用说明。上面分析 render 函数可知如果缓存的组件数量达到设置的上限，就移除最早缓存的组件。
在上面 CompA 与 CompB 的基础上，增加组件 CompC，并设置 `max` 属性为 2 

```
<!-- App.vue -->
<keep-alive max=2>
  <router-view/>
</keep-alive>
```

因为最大缓存组件数为 2，则内存中缓存的组件始终只有 2 个。依次访问 CompB -> CompC -> CompA -> CompB，控制台输出

```
// CompB -> CompC
1 beforeCreate
2 created
3 beforeMount
4 mounted
activated

// CompC -> CompA
deactivated
5 beforeDestroy
6 destroyed

// CompA -> CompB
1 beforeCreate
2 created
3 beforeMount
4 mounted
activated
```

CompC 切换至 CompA，内存中已有 [ CompB, CompC ]，已达到上限。当 CompA 需要被缓存时，最先被缓存的 CompB 将会被销毁。
有个有趣的现象，如果将 `max` 属性设为 1，进行组件切换，发现组件的 `beforeDestory` 和 `destroyed` 钩子不会被调用，原因在 `keep-alive` 的移除实例的函数 **pruneCacheEntry**。例如，从 CompB -> CompC，则 CompC 需要添加至缓存，此时判断 CompB 为当前实例，因此 `$destroy` 没有被执行。虽然 CompB 已经被移出 `keep-alive` 的缓存，但是其实例componentInstance并没有被销毁，这会导致内存泄露。

```
function pruneCacheEntry (cache: VNodeCache, key: string, keys: Array<string>, current?: VNode) {
  const cached = cache[key]
  if (cached && cached !== current) {
    cached.componentInstance.$destroy()
  }
  cache[key] = null
  remove(keys, key)
}
```


### vnode.data.keepAlive
成功缓存的子组件 data 选项的 keepAlive 属性将会被设为 true。那么这个属性有什么作用呢？
引入一个概念 **虚拟 DOM（Virtual DOM）**，Vue 使用 JavaScript 对象表示 DOM 信息和结构，所包含的信息会告诉 Vue 页面上需要渲染什么样的节点，及其子节点。我们把这样的节点描述为**虚拟节点 (Virtual Node)**，也常简写它为`VNode`。**虚拟 DOM**是我们对由 Vue 组件树建立起来的整个 `VNode` 树的称呼。当状态发生变更时，可以使用新渲染的对象树与旧的树进行对比**（DOM diff）**，记录两棵树差异，接下来对页面真正的 DOM 操作，然后把它们应用在真正的 DOM 树上，页面就变更了。
将 **DOM diff** 得到的差异去修改 DOM 树的关键操作为 `patch` 函数，源码传送 [src/core/vdom/patch.js](https://github.com/vuejs/vue/blob/dev/src/core/vdom/patch.js)：

```
// oldVnode: 旧的Virtual DOM或者旧的真实DOM
// vnode：新的Virtual DOM
function patch(oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {
  // ...
}
```

假设有 CompA -> CompB 的切换路径，切换时将对 CompA 和 CompB 的虚拟 DOM 树进行差异化比较，将所得到的差异作用于真正 DOM 树完成视图更新。在 `patch` 函数中，或调用多个组件钩子，钩子源码传送 [src/core/vdom/create-component.js](https://github.com/vuejs/vue/blob/dev/src/core/vdom/create-component.js)。
初始化 CompB 节点，初始化钩子**（init hook）**将会调用：

```
init (vnode: VNodeWithData, hydrating: boolean, parentElm: ?Node, refElm: ?Node): ?boolean {
  if (!vnode.componentInstance || vnode.componentInstance._isDestroyed) {
    const child = vnode.componentInstance = createComponentInstanceForVnode(
      vnode,
      activeInstance,
      parentElm,
      refElm
    )
    child.$mount(hydrating ? vnode.elm : undefined, hydrating)
  } else if (vnode.data.keepAlive) {
    // kept-alive components, treat as a patch
    const mountedNode: any = vnode // work around flow
    componentVNodeHooks.prepatch(mountedNode, mountedNode)
  }
}
```

因为是首次创建，调用 `createComponentInstanceForVnode` 方法返回实例属性。若再次进入，由于设置了 `keepAlive` 属性，`prepatch` 钩子会被调用，进行实例复制，避免重复创建实例：

```
prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  }
```

插入 CompB 节点，**insert** 钩子将会调用：

```
insert (vnode: MountedComponentVNode) {
  const { context, componentInstance } = vnode
  if (!componentInstance._isMounted) {
    componentInstance._isMounted = true
    callHook(componentInstance, 'mounted')
  }
  if (vnode.data.keepAlive) {
    if (context._isMounted) {
      queueActivatedComponent(componentInstance)
    } else {
      activateChildComponent(componentInstance, true /* direct */)
    }
  }
}
```

首先 CompB 是新路径，其生命周期的 `mounted` 钩子会被执行；其次，因为设置了 `keepAlive`  属性，在 `activateChildComponent` 方法中会执行生命周期的 `activated` 钩子。
CompA 虚拟节点会被移出虚拟 DOM 树，执行**destroy** hook：

```
destroy (vnode: MountedComponentVNode) {
  const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
    if (!vnode.data.keepAlive) {
      componentInstance.$destroy()
    } else {
      deactivateChildComponent(componentInstance, true /* direct */)
    }
  }
}
```

若没有设置 `keepAlive` 属性，则 CompA 组件将会调用 `$destroy` 函数进行销毁。而设置了 `keepAlive` 属性，则只会调用组件生命周期的 `deactivated` 钩子函数，组件实例得以保存在内存中。
**综上所述，虚拟节点的 `keepAlive` 属性是关键，通过该属性在调用 `init` 钩子时避免实例被多次创建，同时在切换时调用`destroy` 钩子避免被销毁。**


## References
1. 戴嘉华 [深度剖析：如何实现一个 Virtual DOM 算法](https://segmentfault.com/a/1190000004029168)