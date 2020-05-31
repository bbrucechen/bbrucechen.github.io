---
title: Vue源码之响应式
date: 2020-05-31 22:48:36
tags:Vue
---

## Vue源码之响应式

Vue的另外一个最大特点就是响应实，Model层View层的数据显示都是一致的。接下来就来看一下Vue是如何实现数据的响应式的。

Vue的响应式也是在initMixin函数中执行的。initMixin中调用了initState来对Vue实例中的data进行初始化。

路径:src/core/instance/init.js

```javascript
export function initMixin (Vue: Class<Component>) {
  // 定义初始化函数
  Vue.prototype._init = function (options?: Object) {
   	.
    .
    .
    // 对属性的初始化操作
    initState(vm)
    .
    .
    .
}
```

进入initState看看发生了什么

路径：src/core/instance/state.js

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  // 如果有data属性 ，则开始初始化，执行initData
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

可以看到先是判断有没有data属性，有则调用initData初始化data数据。initData如下：

```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    // 判断data是否有属性和prop或者method重名
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      // 将对vm的_data的属性的获取和设置代理到vm的同名属性上
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  // 再创建一个Observer对象，用于将data所有属性转为响应式 每个Vue实例都有一个observer对象
  observe(data, true /* asRootData */)
}
```

先是对判断data中的属性会不会和prop和method中的属性重名。然后遍历每个data属性并调用proxy并传入当前vue实例和_data字符串以及每个属性名。同样的进入proxy函数中查看发生了什么。

```javascript
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}
// 通过这个函数就可以实现用vm.xxx访问和设置vm._data.xxx
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

可以看到，proxy函数设置了共享的属性描述符对象sharedPropertyDefinition的get和set。

由于data的属性需要vm._data,key才能获取，为了方便用户获取，通过proxy的方法将vm._data.key的属性值挂载到vm.key上。这就是为什么我们能在Vue的options中任何地方通过this.xxx直接获取vm._data.key。

回到initData函数中，执行完proxy之后，就执行了observer方法并将data传参。observer方法定义如下：

路径：src/core/observer/index.js

```javascript
// 创建一个观察者实例并将这个参数下的所有子数据进行响应式化
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建Observer实例
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

observer其实就是根据传入的data创建并返回了一个Observer实例。而new Observer其实就是将data每个属性响应化。具体Observer构造函数如下：

路径：同上

```javascript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    // 新建依赖者实例，用于收集被观察的vue实例
    this.dep = new Dep()
    this.vmCount = 0
    // 把自身实例添加到value的_ob_属性上 这就是为什么输出的data属性上有个_ob_属性
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      // 遍历并响应式化每个data属性
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  // 当data为对象时通过遍历每个属性给每个属性配置setter和getter
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  // 如果是数组
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

Observer构造函数显示把当前实例挂载到初始化的Vue实例上。然后执行walk方法。walk方法就是遍历每个data属性并将data和属性名作为参数传给defineReactive。由名可知defineReactive就是对每个数据进行响应式化的方法。

路径：同上

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 创建当前data属性值的依赖者实例
  const dep = new Dep()


  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 递归调用observe保证所有子对象都能被响应式
  let childOb = !shallow && observe(val)
  // 然后定义getter和setter用于收集依赖和派发更新
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // 依赖收集 当在创建vnode时会获取该属性，说明该属性是会被使用的
    // 在new Wacther时触发
    get: function reactiveGetter () {
      // 获取属性值
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        //把获取了当前属性的wathcer对象添加到这个属性的dep实例 即 依赖收集 依赖即指依赖于某个data属性的vue实例（组件）
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      // 获取旧值
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      // 如果数值一样则直接返回
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      // 然后当前属性对应的依赖对象dep就会通知使用了该属性的vnode进行更新
      dep.notify()
    }
  })
}
```

defineReactive先是创建了当前目标data属性的dep实例。这个实例是用来保存依赖项的。所谓依赖项即使用到了当前属性的vue实例，后面会讲到。接着获取当前属性的属性描述中的get和set赋予getter和setter变量。接下来便是响应式的核心。

### 依赖收集

defineReactive中使用了Object.defineProperty来对当前的属性进行收集依赖和派发更新。我们先来看依赖收集。即getter方法。当Vue实例使用到当前属性时就会触发该函数并进行依赖的收集。那么我们什么时候会使用该属性呢？这时候就要回到Vue初始化时执行的mountComponent函数了。

路径：src/core/instance/lifecycle.js

```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // 挂载目标
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  // 执行beforeMount钩子函数
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      // 通过调用实例的_render方法进行渲染返回VNode
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      // 通过调用实例的_update方法将Vnode挂载到DOM上
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  // 一个Vue实例对应一个wathcer对象。Wacther构造函数里把这个watcher实例赋予了_watcher属性
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  // 函数最后判断为根节点的时候设置 vm._isMounted 为 true， 表示这个实例已经挂载了，同时执行 mounted 钩子函数。 这里注意 vm.$vnode 表示 Vue 实例的父虚拟 Node，所以它为 Null 则表示当前是根 Vue 的实例。
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}

```

可以看到，虽然已经定义了updateComponent函数，但是并没有立即直接执行。那么是哪里在执行呢？其实就在下面的new Watcher里。

Watcher也是一个构造函数，用来创造订阅者实例。每个Vue实例对应一个watcher。来到Watcher构造函数：

路径：src/core/observer/watcher.js

```javascript
constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      // vue实例的_watcher指向其watcher对象
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      // 把updateComponent赋予getter
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      // 最后执行get
      : this.get()
  }
```

可以看到，该构造函数先是将当前实例添加到vue实例的_wathcers中。然后将前面的updateComponent赋予getter变量。最后执行this.get。this.get如下：

```javascript
// 依赖收集函数
get () {
  // 把当前的watcher实例赋值给Dep.target并压栈到targetStack中 Dep.target就是存放当前要被监听的Vue实例
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    // 然后执行updateComponent updateComponent中会获取属性值 就会触发getter 从而完成依赖收集
    value = this.getter.call(vm, vm)
  } catch (e) {
    if (this.user) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    // 收集完之后，继续递归收集
    if (this.deep) {
      traverse(value)
    }
    // 清空收集目标
    popTarget()
    // 将新旧依赖进行交换
    this.cleanupDeps()
  }
  return value
}
```

get方法先是执行了pushTarget：

路径：src/core/observer/dep.js

```javascript
export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}
```

该方法即是将当前watcher实例添加到Dep类的target中。代表了当前渲染的watcher，在收集依赖时就可以通过target快速获取依赖的watcher。

pushTarget之后便执行了this.getter即updateComponent，原来updateComponent就是在这个时候执行的。

我们回到defineReactive中：

```javascript
get: function reactiveGetter () {
  // 获取属性值
  const value = getter ? getter.call(obj) : val
  if (Dep.target) {
    //把获取了当前属性的wathcer对象添加到这个属性的dep实例 即 依赖收集 依赖即指依赖于某个data属性的vue实例（组件）
    dep.depend()
    if (childOb) {
      childOb.dep.depend()
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
  }
  return value
},
```

可以看到，判断了Dep.target是否有值。因为在渲染vue实例前已经将vue实例对应的watcher加入到了target中。所以继续执行dep.depend。

depend方法如下：

路径：src/core/observer/dep.js

```javascript
depend () {
  if (Dep.target) {
    Dep.target.addDep(this)
  }
}
```

此时Dep.target我们已经说过，是当前vue实例的watcher对象。所以查看addDep方法如下：

```javascript
addDep (dep: Dep) {
  const id = dep.id
  // 判断 防止重复收集
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      // 把当前观察者watcher添加到依赖的subs中 这个目的是为后续数据变化时候能通知到哪些 subs 做准备。
      dep.addSub(this)
    }
  }
}
```

addDep便是依赖收集的实现方法。调用了收集目标data属性的dep的addSub方法并将当前watcher对象作为实例：

```javascript
addSub (sub: Watcher) {
  this.subs.push(sub)
}
```

最终将watcher添加到了当前收集data属性的dep实例的subs数组中。subs数组保存的就是使用了其dep对象对应的属性的vue实例的watcher对象。这就是整个依赖收集的过程。

### 派发更新

依赖收集的目的就是为了数据变化时能通知具体的订阅者，让订阅了当前属性的wacther重新渲染，这个过程就叫派发更新。

我们回顾一下defineReactive中的setter部分：

```javascript
set: function reactiveSetter (newVal) {
  // 获取旧值
  const value = getter ? getter.call(obj) : val
  /* eslint-disable no-self-compare */
  // 如果数值一样则直接返回
  if (newVal === value || (newVal !== newVal && value !== value)) {
    return
  }
  /* eslint-enable no-self-compare */
  if (process.env.NODE_ENV !== 'production' && customSetter) {
    customSetter()
  }
  // #7981: for accessor properties without setter
  if (getter && !setter) return
  if (setter) {
    setter.call(obj, newVal)
  } else {
    val = newVal
  }
  childOb = !shallow && observe(newVal)
  // 然后当前属性对应的依赖对象dep就会通知使用了该属性的vnode进行更新
  dep.notify()
}
```

当我们对属性赋值时，就会触发set即reactiveSetter。先是调用getter获取旧值。然后新旧值进行对比，如果没有改变则结束。否则调用setter赋予当前属性新值。最后调用当前改变属性对应的dep对象的notify方法。

我们来看看dep.notify这个方法。

路径：src/core/observer/dep.js

```javascript
 notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      // 通知watcher重新渲染
      subs[i].update()
    }
  }
```

可以看到先是将当前的subs中的watcher按照id从大到小排序，这样的目的是为了保证按照先父后子的顺序更新渲染组件。然后便利subs并执行他们的update方法即watcher对象的update方法。我们来看看这个方法。

路径：src/core/observer/watcher.js

```javascript
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

先忽略其他判断条件，可以看到最后执行了queueWatcher方法并将当前watcher作为参数。

queueWatcher方法定义于src/core/observer/scheduler.js，代码如下：

```javascript
const queue: Array<Watcher> = []
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      // 不会每次都更新watcher，而是先放到queue队列中
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      // 异步执行flushScheduleQueue
      nextTick(flushSchedulerQueue)
    }
  }
}
```

可以看到，watcher并没有立即重新渲染，而是放到了队列数组queue中。这样做的目的是为了异步一次性即nextTick之后更新所有需要重新渲染的watcher。nextTick放到后面再说，先来看lushSchedulerQueue方法的实现：

路径：src/core/observer/scheduler.js

```javascript
// 遍历wather并执行run
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  // 将queue排序 保证组件的更新顺序是先父后子
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    // 将当前轮到的watcher从queue出去并执行其run方法
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

```

显示依旧对queue中的wachter进行排列保证先父组件后子组件的顺序进行渲染。然后执行wacther对象的run方法。

```javascript
run () {
  if (this.active) {
    // 执行get方法 就会触发getter方法 即updateComponent 完成重新渲染
    const value = this.get()
    if (
      value !== this.value ||
      // Deep watchers and watchers on Object/Arrays should fire even
      // when the value is the same, because the value may
      // have mutated.
      isObject(value) ||
      this.deep
    ) {
      // set new value
      const oldValue = this.value
      this.value = value
      if (this.user) {
        try {
          // 执行回调
          this.cb.call(this.vm, value, oldValue)
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        this.cb.call(this.vm, value, oldValue)
      }
    }
  }
}
```

即执行了watcher的get方法即完成了重新渲染。这样就完成了DOM的更新。

### nextTick的实现

前面我们说到DOM的更新是异步的，即通过nextTick执行flushSchedulerQueue。我们来看看nextTick函数的定义：

路径：src/core/util/next-tick.js

```javascript
const callbacks = []
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 把传入的回调函数压入callbacks堆栈 即flushSchedulerQueue函数
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  // 这是让我们可以使用this.$nextick来获取渲染后的
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }

  // 所以当我们在某个组件对多个数据作了修改时，想要获取变化后的dom,就必须通过调用Vue.nexttick或者this.$nextTick来获取变化后的DOM
}
```

可以看到，nextTick只是将回调函数的执行传入一个匿名函数，并将匿名函数传入callback栈中。

那么nextTick到底是怎么实现异步的呢？看下nextTick之前的代码就知道了：

```javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

原来是通过Promise或者messagechannel或者setImmediate或者setTimeout来实现的。具体原则是因为JS的单线程以及事件循环，这里不再赘述。

可以看到最后都执行了flushCallbacks函数。这个函数非常简单：

```
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

就是遍历执行`callback`栈中的函数，从而执行前面的`flushSchedulerQueue`更新视图。

这里使用 `callbacks` 而不是直接在 `nextTick` 中执行回调函数的原因是保证在同一个 tick 内多次执行 `nextTick`，不会开启多个异步任务，而把这些异步任务都压成一个同步任务，在下一个 tick 执行完毕。

除此之外，在nextTick函数最后有这么一段：

```javascript
if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
```

这是为了让用户可以调用this,$nextTick。当没有传入回调函数时就会返回一个Promise。

所以当我们在某个组件对多个数据作了修改时，想要获取变化后的dom,就必须通过调用Vue.nexttick或者this.$nextTick来获取变化后的DOM

### 总结

整个Vue的响应式原理就是Observer、Dep、Watcher三个对象的操作

先是用Obeserver对象对每个data属性定义setter、getter。getter用于收集依赖并返回属性值，执行当前属性的dep实例的depend方法将Dep.target传入dep的subs数组中。setter用于派发更新，当属性变化时，就会赋予新值并执行该属性对应dep实例的notify方法，遍历其subs数组，将所有需要更新的watcher放进queue中。并异步执行对watcher的重新渲染。

### 原理图

![img](https://ustbhuangyi.github.io/vue-analysis/assets/reactive.png)