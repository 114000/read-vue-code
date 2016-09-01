# Vue/core/instance/render.js

## ☆ [fn] initRender

``` javascript
function initRender (vm: Component) {
  vm.$vnode = null // the placeholder node in parent tree
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null
  vm.$slots = resolveSlots(vm.$options._renderChildren)
  // bind the public createElement fn to this instance
  // so that we get proper render context inside it.
  vm.$createElement = bind(createElement, vm)
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

### [fn] renderMixin

添加实例方法

### Vue.prototype.$nextTick

``` javascript
Vue.prototype.$nextTick = function (fn: Function) {
  // util/env.js
  nextTick(fn, this)
}
```

### ☆ Vue.prototype.\_render

``` javascript
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const {
    render,
    staticRenderFns,
    _parentVnode
  } = vm.$options

  if (staticRenderFns && !vm._staticTrees) {
    vm._staticTrees = []
  }
  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Error when rendering ${formatComponentName(vm)}:`)
    }
    /* istanbul ignore else */
    if (config.errorHandler) {
      config.errorHandler.call(null, e, vm)
    } else {
      if (config._isServer) {
        throw e
      } else {
        setTimeout(() => { throw e }, 0)
      }
    }
    // return previous vnode to prevent render error causing blank component
    vnode = vm._vnode
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
      warn(
        'Multiple root nodes returned from render function. Render function ' +
        'should return a single root node.',
        vm
      )
    }
    vnode = emptyVNode()
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}
```

### ☆ Vue.prototype.\_h/\_s/\_n/\_m/\_f/\_l/\_b/\_k

``` javascript
function renderMixin (Vue: Class<Component>) {
  // shorthands used in render functions
  Vue.prototype._h = createElement
  // toString for mustaches
  Vue.prototype._s = _toString
  // number conversion
  Vue.prototype._n = toNumber

  // render static tree by index
  Vue.prototype._m = function renderStatic (
    index: number,
    isInFor?: boolean
  ): VNode | VNodeChildren {
    let tree = this._staticTrees[index]
    // if has already-rendered static tree and not inside v-for,
    // we can reuse the same tree by indentity.
    if (tree && !isInFor) {
      return tree
    }
    // otherwise, render a fresh tree.
    tree = this._staticTrees[index] = this.$options.staticRenderFns[index].call(this._renderProxy)
    if (Array.isArray(tree)) {
      for (let i = 0; i < tree.length; i++) {
        tree[i].isStatic = true
        tree[i].key = `__static__${index}_${i}`
      }
    } else {
      tree.isStatic = true
      tree.key = `__static__${index}`
    }
    return tree
  }

  // filter resolution helper
  const identity = _ => _
  Vue.prototype._f = function resolveFilter (id) {
    return resolveAsset(this.$options, 'filters', id, true) || identity
  }

  // render v-for
  Vue.prototype._l = function renderList (
    val: any,
    render: () => VNode
  ): ?Array<VNode> {
    let ret: ?Array<VNode>, i, l, keys, key
    if (Array.isArray(val)) {
      ret = new Array(val.length)
      for (i = 0, l = val.length; i < l; i++) {
        ret[i] = render(val[i], i)
      }
    } else if (typeof val === 'number') {
      ret = new Array(val)
      for (i = 0; i < val; i++) {
        ret[i] = render(i + 1, i)
      }
    } else if (isObject(val)) {
      keys = Object.keys(val)
      ret = new Array(keys.length)
      for (i = 0, l = keys.length; i < l; i++) {
        key = keys[i]
        ret[i] = render(val[key], key, i)
      }
    }
    return ret
  }

  // apply v-bind object
  Vue.prototype._b = function bindProps (
    vnode: VNodeWithData,
    value: any,
    asProp?: boolean) {
    if (value) {
      if (!isObject(value)) {
        process.env.NODE_ENV !== 'production' && warn(
          'v-bind without argument expects an Object or Array value',
          this
        )
      } else {
        if (Array.isArray(value)) {
          value = toObject(value)
        }
        const data: any = vnode.data
        for (const key in value) {
          if (key === 'class' || key === 'style') {
            data[key] = value[key]
          } else {
            const hash = asProp || config.mustUseProp(key)
              ? data.domProps || (data.domProps = {})
              : data.attrs || (data.attrs = {})
            hash[key] = value[key]
          }
        }
      }
    }
  }

  // expose v-on keyCodes
  Vue.prototype._k = function getKeyCodes (key: string): any {
    return config.keyCodes[key]
  }
}
```

### ☆ [fn] resolveSlots

解析 slots

``` javascript
function resolveSlots (
  renderChildren: ?VNodeChildren
): { [key: string]: Array<VNode> } {
  const slots = {}
  if (!renderChildren) {
    return slots
  }
  const children = normalizeChildren(renderChildren) || []
  const defaultSlot = []
  let name, child
  for (let i = 0, l = children.length; i < l; i++) {
    child = children[i]
    if (child.data && (name = child.data.slot)) {
      delete child.data.slot
      const slot = (slots[name] || (slots[name] = []))
      if (child.tag === 'template') {
        slot.push.apply(slot, child.children)
      } else {
        slot.push(child)
      }
    } else {
      defaultSlot.push(child)
    }
  }
  // ignore single whitespace
  if (defaultSlot.length && !(
    defaultSlot.length === 1 &&
    defaultSlot[0].text === ' '
  )) {
    slots.default = defaultSlot
  }
  return slots
}
```