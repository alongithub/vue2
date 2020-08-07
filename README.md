## vue响应式源码

vue 源码文件结构

<pre style="background: #000; color: #ccc">
└ src/ ·····································  生成器目录
   ├ compiler/ ·········································  把模板转换成render函数
   ├ core/ ···································  核心代码，与平台无关
   |  ├ componetnts/
   |  |  └ keep-alive.js ······································· vue keep-alive组件
   |  ├ global-api/ ······································· vue静态方法，componet,filter,extend,mixin,use 等方法
   |  ├ instance/ ······································· 创建vue实例，包括vue构造函数，初始化，生命周期等 
   |  ├ observer/ ······································· 响应式机制实现
   |  ├ util/ ······································· 公共成员
   |  └ vdom/  ······································· 虚拟dom
   ├ platforms/ ······································· 平台相关代码
   |  ├ web/
   |  └ weex/
   ├ server/ ······································· 服务端渲染相关文章
   ├ sfc/  ······································· single file component 单文件组件转换成js对象
   └ shared/ ······································· 公共部分代码
</pre>

### Vue初始化过程

先通过`Vue`初始划过成，了解`Vue`源码中，静态方法实例方法等相关逻辑定义的位置

首先要从构造函数开始，找到构造函数的位置可以通过打包入口文件`platforms/web/entry-runtime-with-compiler.js`中引入`Vue`的位置，顺藤摸瓜找到`Vue`构造函数位置

#### Vue实例上的属性和方法

找到`vue`构造函数定义的地方 `/instance/index.js`
构造函数中判断是否是开发环境且通过`new`调用构造方法  
调用 `_init()`,该方法是通过下方`initMixin()`注册的  
初始化`vue`实例的原型和属性

```js
function Vue (options) {
   if (process.env.NODE_ENV !== 'production' &&
      !(this instanceof Vue)
   ) {
      warn('Vue is a constructor and should be called with the `new` keyword')
   }
   // 调用 _init() 方法
   this._init(options)
}

initMixin(Vue)
// 注册 vm 的 $data/$props/$set/$delete/$watch
stateMixin(Vue)
// 初始化事件相关方法
// $on/$once/$off/$emit
eventsMixin(Vue)
// 初始化生命周期相关的混入方法
// _update/$forceUpdate/$destroy
lifecycleMixin(Vue)
// 混入 render
// $nextTick/_render
renderMixin(Vue)
```

#### Vue类的静态成员

`core/index`中引入了`/instance/index.js` 
为`vue`构造函数增加静态方法后导出

```js
import Vue from './instance/index'
initGlobalAPI(Vue)

Vue.version = '__VERSION__'
```

`initGlobalAPI` 在 `global-api`/`index.js` 中
初始化`vue.config`,开发环境不能更改
`vue.util`中定义了一些框架内部使用的方法
定义静态方法 `set`/`delete`/`nextTick`
定义`observable` 可响应方法
初始化`options`,增加`components`/`directives`/`filters`
增加全局`keep-alive`组件
注册`use`,`mixin`,`extend`和`directive`,`component`,`filter`方法

```js
export function initGlobalAPI (Vue) {
   Object.defineProperty(Vue, 'config', configDef)

   Vue.util = {warn,extend,mergeOptions,defineReactive}

   Vue.set = set
   Vue.delete = del
   Vue.nextTick = nextTick

   Vue.observable = <T>(obj: T): T => {
      observe(obj)
      return obj
   }

   Vue.options = Object.create(null)
   ASSET_TYPES.forEach(type => {
      Vue.options[type + 's'] = Object.create(null)
   })

   extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```

#### 平台相关的指令组件和__patch__、$mount

`platforms/web/runtime/index.js  `中引入了`core/index`
这个文件首先添加了一些方法，用于判断是否是保留标签，保留属性等
然后通过`extend`为平台注册全局指令（`v-model`,`v-show`）和组件(`transition`, `transitionGroup`)  
为`vue`注册全局的`__patch__`函数，用于将虚拟`dom`转换成真实`dom  `
为`vue`添加全局的`$mount`方法，用于渲染`dom`

```js
import Vue from 'core/index'
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement

extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)

Vue.prototype.__patch__ = inBrowser ? patch : noop

Vue.prototype.$mount = function () {
  return mountComponent(this, el, hydrating)
}

```

#### 平台完整版增强$mount增加模板编译

`platforms/web/entry-runtime-with-compiler.js`引入`platforms/web/runtime/index.js `

这个文件时`vue`完整版打包入口，这个文件中，增强了`Vue.$mount`方法  
判断组件是否有`render`方法，如果没有，取`template`变异成渲染函数`render`  
最后增加一个`compile`方法，将`template`变异成`render`函数

```js
import Vue from './runtime/index'

const mount = Vue.prototype.$mount

Vue.prototype.$mount = function() {
   let template = options.template
   if (!options.render) {
      if (template) {

      }
   }
}

Vue.compile = compileToFunctions
```

### 初始化实例成员

#### initMixin 增加_init方法

#### stateMixin $data、$props、 

通过`defineproperty`为`Vue.prototype`定义`$data`和`$props`,避免开发环境用户重新定义这两个值
给原型增加`$set`、`$delete`方法
增加原型`$watch`方法，用于监视数据变化的`api`

```js
export function stateMixin (Vue) {
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function () {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    // 获取 Vue 实例 this
    const vm: Component = this
    if (isPlainObject(cb)) {
      // 判断如果 cb 是对象执行 createWatcher
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    // 标记为用户 watcher
    options.user = true
    // 创建用户 watcher 对象
    const watcher = new Watcher(vm, expOrFn, cb, options)
    // 判断 immediate 如果为 true
    if (options.immediate) {
      // 立即执行一次 cb 回调，并且把当前值传入
      try {
        cb.call(vm, watcher.value)
      } catch (error) {
        handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
      }
    }
    // 返回取消监听的方法
    return function unwatchFn () {
      watcher.teardown()
    }
  }
}
```

#### eventsMixin $on、$once、$off、$emit

为原型添加与事件相关的方法`$on`、`$once`、`$off`、`$emit`  
内部就是通过发布订阅模式  
为`vm`实例添加`_events`属性，用于记录不事件及对应的订阅者方法

#### lifecycleMixin _update、$forceUpdate、$destory

为原型添加生命周期相关方法  
`_update`方法内部调用`vm.__patch__`方法，将返回的真实`dom`保存到`vm.$el `
 
#### renderMixin 

这个方法在原型上增加了很多下划线方法（`_o`、`_n`、`_s`等）,这些方法用于将模板变异成render函数  
在原型上增加`$nextTick`方法  
在原型上增加`_render`函数，内部调用用户传入的`render`属性，并传入h函数


### initGlobalAPI 初始化静态成员

#### initUse 注册Vue.use插件注册方法

`initUse`方法为`Vue`类注册了use静态方法，在函数内部，会子啊首次注册插件时为`Vue`类定义`_installedPlugins`数组用于存储注册的插件，注册插件时先检查是否注册过插件  
如果没有注册过该插件判断参数是否为对象，是对象调用对象的`install`方法，如果是函数直接执行这个函数  
最后将刚注册的插件保存到数组中

```js
export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    // 把数组中的第一个元素(plugin)去除
    const args = toArray(arguments, 1)
    // 把this(Vue)插入第一个元素的位置
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}
```

#### initMixin 定义Vue.mixin混入方法

`initMixin`做的事情就是把参数拷贝到`Vue.options`上

```js
import { mergeOptions } from '../util/index'

export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```

#### initExtend 定义Vue.extend 方法

`Vue.extend()`在自定义组件时会用到，该方法会返回一个集成Vue的类，核心功能代码如下。

```js
export function initExtend (Vue) {

  Vue.extend = function (extendOptions) {

    const Sub = function VueComponent (options) {
      // 调用 _init() 初始化
      this._init(options)
    }
    // 原型继承自 Vue
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    // 合并 options
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    Sub['super'] = Super

    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })

    return Sub
  }
}

```

#### initAssetRegisters 注册Vue.directive、component、filter方法

这三个方法用于注册全局指令，组件和过滤器  
所有注册的指令组件和过滤器存储在`Vue.options`中 
三个方法内,都会判断第二个参数是否有值，没有的话返回之前注册的对应的方法
如果是通过`Vue.componet`注册组件并且第二个参数是一个原始对象`object`（即`json`对象），把第二个参数重定义为一个构造函数（`Vue.extend(definition)`）  
如果通过`Vue.direction`并且第二个参数是一个函数，重定义第二个参数为`{ bind: definition, update: definition }`
最后将definition添加到`Vue.options`中

```js
export function initAssetRegisters (Vue: GlobalAPI) {
  // 遍历 ASSET_TYPES 数组，为 Vue 定义相应方法
  // ASSET_TYPES 包括了directive、 component、filter
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (id,definition) {
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
       
        // Vue.component('comp', { template: '' })
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          // 把组件配置转换为组件的构造函数
          definition = this.options._base.extend(definition)
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        // 全局注册，存储资源并赋值
        // this.options['components']['comp'] = definition
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}
```

### 创建Vue实例

当通过`new Vue(options)`创建`Vue`实例之后，首先会在构造函数中调用`_init`函数，可以通过`vue`实例创建，依次来看执行了哪些操作

#### _init()

之前已经看到 `Vue.prototype._init` 注册的位置，在`initMixin`方法中，看下_init做了哪些事情

1. `_init`中，首先定义常亮`vm`，保存当前`Vue`实例  
2. 标记`vm._isVue = true`, 目的是在之后执行`observe`响应式时，不对`vm`实例做处理
3. 合并用户传入的`options`和`Vue`构造函数的`options`
4. 渲染时的代理对象 `vm._renderProxy = vm`
5. 初始化`vm`生命周期相关属性 `initLifecycle`
   - 将自身添加到父组件的`$children`中
   - `$parent`
   - `$root`
   - `_watcher`
   - `_inactive`
   - `_directInactive`
   - `_isDestroyed`
   - `_isBeingDestroyed`
6. 初始化当前组件的事件 `initEvents` `vm.events` 用于存储事件和事件处理函数，并获取父元素附加的事件注册到当前组件
7. `initRender` 
   - 初始化 `vm`插槽相关属性 `$slots`, `$scopedSlots`
   - 初始化`vm._c` 和 `vm.$createElement` 方法，`_c` 会在当通过模板编译生成`render`函数时被使用，`$createElement` 即h函数，用于将虚拟`dom`转换成真实`dom`
   - 通过`defineProperty` 定义不允许重新赋值的 `$attrs`属性和`$listeners`属性
8. `callHook` 触发生命周期函数 `beforeCreate`
9. `initInjections` 实现依赖注入 `inject`
   - 提取 `reject`数组中存在且vm._provided中存在的属性`keys`保存到常量result中
   - 将`key`设置成`vm`上的响应式数据，生产环境不可更改
   - 
10. `initState` 初始化`props`、`methods`、`data、`computed`、`watch`
    - `initProps` 
      - 初始化`vm._props`对象
      - 遍历参数`props`中所有属性，注入到`_props`,并在`vm`上代理`_props`中的属性，可以通过`vm` 直接访问
    - `initMethods`
      - `判断methods`中的属性是否与`props`重复
      - 生产模式方法名禁止用`_`或`$`开头
    - `initData` 初始化数据
      - 判断`data`中的属性是否与`methods`和`props`中的重复
      - 将不是以`_`和`$`开头的属性注入到`vue`实例，并把`data`转换成响应式对象
    - `initComputed` 初始化计算属性
      - 注入到`vue`实例中
      - ...
    - `initWatch` 初始化侦听器
      - 注入到`vue`实例中
      - ...
11. `initProvide` 实现依赖注入 `provide`
   - `vm.$options.privide` 存储到 `vm._provide`d中
   - 
12. `callHook` 触发生命周期函数 `created`
13. `$mount` 如果存在`$options.el` , 使用 `$mouont`挂载页面


#### vue 首次渲染过程

初始化一个`vue`实例，看下`vue`首次渲染过程

1. 初始化实例成员和静态成员
2. 调用`new vue`，进入`_init`方法
3. 调用带编译器的`$mount`（主要作用是在用户没有传入`render`函数时，编译用户传入的`template`），判断`options`中是否有`render`函数
   - 如果没有`render`，获取`template`，判断`template`是否存在
     - `template`存在
       - 判断`template`是否是字符串,如果是判断是否以`#`开头,如果是通过`id`获取`dom`元素，并把元素的`innerHTML`赋值给`template`
       - 判断`template`是否有`nodeType`属性，如果有把`template`的`innerHTML`赋值给`template`
       - `template`即不是字符串也没有`nodeType`,返回`vue`实例
     - `template`不存在
       - 判断是否有`el`，有的话将`el`的`outerHTML`赋值给`template`(这里会判断`el`是否有`outerHTML`，如果没有创建一个`div`并把`el`克隆到`div`里)
     - 将最终的`template`处理成`render`函数，存储到`options`中
4. 调用`$mount`
   - 开发环境会判断是否有`render`函数，没有的话运行时版本不包含编译器的警告(运行时版本未传入`render`函数，`vue`不会把`template`编译成`render`函数)
   - 触发`beforeMount`生命周期函数
   - 定义`updateComponent`方法
     - 调用`vm._render()`方法,把用户传入或编译器编译的`render`转换成虚拟`dom`,并作为参数传递给`vm._update`方法，`vm._update`内部调用`vm.__patch__`方法，把虚拟`dom`转换成真实`dom`，更新到界面上，并记录到`vm.$el`中
   - 创建一个渲染`Watcher`实例,数据改变时调用`get()`,`get`内部调用`updateComponent`
   - 触发`mounted`生命周期函数

### Vue响应式原理 

在`vue`构造函数中，调用了`_init`方法，这个方法中调用`initState`初始化`_data`,`_props`,`methods`等,`initState`调用了`initData`,这个函数对用户传入的数据进行响应式处理


```js

function initState() {
  ...
  if (opts.data) {
    initDate(vm)
  } else {
    observe(vm._data = {}, true)
  }
  ...
}

```

在`initState`方法中,首先判断用户是否传入`data`
- 如果传入`data`调用`initData`
  - 检查`data`中的属性是否与`props`,`methods`重名，不重名设置代理，将`data`成员注入到`vue`实例
  - 调用`observe(data, true)`函数，内部视图创建`Observe`对象，如果创建成功返回，或者返回已存在的`Observe`对象
    - 首先判断，如果不是对象或者是`VNode`，直接`return`
    - 如果`data`中有`__ob__`属性并且改属性是`Observer`实例
      - 直接返回该属性值
      - 否则，判断`data`是数组或者`Object`,并且不是`vue`实例，成立时返回一个`Observer(data)`对象,该对象会把data中所有属性转换为`getter`，`setter`
- 如果没传入直接给vm注册`_data`为一个响应式对象

`Observer`类构造函数中
- 将`data`设置属性`__ob__`,赋值为`Observer`实例，并设置`__ob__`为不可枚举
- 判断`data`是否是数组
  - 是，对数组进行特殊响应式处理
    - 判断浏览器是否支持`__proto__`
      - 支持，调用`protoAugment`
        - 修改当前数组的原型指向一个新的对象，将原型上涉及修改数组的方法增强，如果数组增加了元素，将新加元素设置成响应式。最后调用`dep.notify`派发通知，返回原始方法的执行结果
      - 不支持，调用`copyAugment`
        - 类似`protoAugment`,维数组增强方法
    - 调用`observeArray`,遍历数组中的成员，如果是对象的换转换成响应式对象 
  - 不是，调用`observer.walk`方法，对`data`中的每一个属性，都转换成`getter`和`setter`
    - 获取`data`中的所有属性，分别调用`defineReactive`方法，将属性转换成`getter`和`setter`，同时进行收集依赖，发送通知等处理
      - 函数内部创建`Dep`对象用于收集依赖
      - 获取当前属性的属性描述符，考虑到处理用户传入对象时设置了`configurable`、`set`和`get`的情况，如果用户设置了当前属性不可配置`configurable:false`，直接`return`
      - 如果属性值是对象，会递归调用`observe`函数,实现深度监听，并将调用`observe`返回的`Observer`队形缓存到`childOb`中
      - 通过`defineproperty`设置属性的`getter`和`setter`，
        - `get`, 如果用户设置了`get`,首先调用用户传入的`getter`，接下来判断`Dep.target`属性，存在把该属性值（`watcher`）通过`dep.depend()`添加到`dep`中,如果值是对象，则通过`childOb.dep.depend()`把`target`上的`watcher`也添加到子对象的`dep`中
        - `set`, 
          - 判断新旧值是否相等，（考虑`NaN !== NaN`的情况）,如果新旧址相等或者都为`NaN`，直接返回
          - 如果属性是只读属性直接返回
          - 接下来如果用户设置了`setter`，调用`setter`，否则直接赋值`val = newValue`
          - 如果新值是对象，递归调用`observe`方法，把新对象设置为响应式属性
          - 最后通过`dep.notify`派发更新

通过 `defineReactive`,为属性设置`getter`和`setter`时，会进行依赖收集，收集`Dep`上的`target`到`Dep`的`subs`数组中,而`Dep`上`target`是什么时候被赋值的呢

- `_init` => `Vue.$mount` => `mountComponent`
- `mountComponent`方法中，创建了`Watcher`对象，`Watcher`对象中，为`Dep.target`赋值为当前`Watcher`对象
- 当数据改变时，`watcher`对象的`get`方法被调用，内部会触发`updateComponente`（`vm._update(vm._render(), hydrating)`）函数，在`vm._render`函数中，会调用用户传入的`render`或者模板编译的`render`,而`render`函数内部，会访问挂载到`vue`实例上的`data`中的属性，访问属性时会触发`getter`，因为`vue`实例上代理了`_data`中的属性，因此又会再次触发`_data`属性的`getter`方法，也就是`defineReactive`中设置的`getter`
- 在`defineReactive`中的`getter`被触发时，会执行`dep.depend`放法，该方法内部，调用了`target`上`watcher`实例的`addDep`方法
  - `addDep`方法缓存了已添加的`dep`对象的`id`，如果该`id`不存在，添加到`watcher`的`newDeps`数组中，并把`watcher`添加到`dep`对象的`subs`数组中


`watcher`类

`watcher` 类分为三种，计算属性`watcher`,用户侦听器`watcher`,渲染`watcher`

watcher中构造函数执行内容
- `constructor`参数
  - `vm`, `vue`实例
  - `expOrFn`, 字符串或者函数
  - `cb`, 回调函数
  - `options`? 配置对象
  - `isRenderWatcher`? 是否是渲染`watcher`  
- 记录`vue`实例到`this.vm`
- 判断用户传入的`isRenderWatcher`,如果`true` 代表 渲染`watcher`,将当前`watcher`添加到`vue`实例的`_watcher`属性上
- 将`watcherpush` 到`vm._watchers`数组中，`_watchers`保存了`vm`实例的所有类型的`watcher`
- 根据`options`初始化配置项，默认都是`false`，
- 声明 `deps` `newDeps` `depIds` `newDepIds` 用于记录与`watcher`相关的`Dep`对象 
- 判断`expOrFn` 是否是函数，如果是函数，赋值给`wather`对象的`getter`属性 ,如果是字符串,说明是侦听器，通过`parsePath(expOrFn)`赋值给`getter`一个获取`expOrFn`值的内容```// 例如 watch: {'person.name': fn},parsePath返回一个获取person.name的函数```
- 判断options中的lazy属性，如果不存在或者为false,执行watcher实例的get方法，渲染watcher会立即执行get方法
- get方法内容
  - 首先调用pushTarget，将watcher入栈，并赋值给Dep.target
  - 调用watcher实例的getter方法，（如果是渲染watcher,getter中保存的是updateComponent,该方法执行后，会将VNode渲染成真实dom并更新页面上）
  - 调用popTarget,清理watcher队列，将栈顶watcher出栈 
  
## 24 25 26

### 动态添加属性

如果要动态添加或删除属性，需要用到set,delete两个方法,不能直接修改对象

#### set

涉及`set`函数的有`Vue.set`静态方法,和`vm.$set`实例方法，分别在`initGlobalApi`函数和`stateMixin`函数中被赋值，这两个方法都指向`instance`/`observer`/`index.js`里的`set`方法

- `set`方法接收3个参数，目标对象`target`, 新增的属性`key`和值`val`
- 在方法内部，首先会判断`target`是否是`undefined`或者原始值，如果是会发出警告
- 接下来判断`target`如果是数组，会判断`key`是否是合法的索引，将`Max(target.length, key)`赋值给`target.length`,然后借助数组的`splice(key, 1, val)`,对目标索引值进行替换，注意此时的`splice`方法是经过`defineProperty`处理过的，被调用后会下发通知
- 如果`target`是对象，判断属性`key`如果是`target`已经存在的属性并且不是原型上的属性，直接赋值，`target.key=val`
- 如果以上条件都不满足，获取`target`上的`observer`对象
- 如果`target`是`vue`实例或者`$data`，发出警告并返回
- 判断`target`上不存在`observer`实例，直接赋值并返回，不必做响应式处理
- 如果`observer`实例存在，通过`defineReactive`将`target`增加响应式属性`key`,值为`value`
- 发送通知

```javascript
export function set (target: Array<any> | Object, key: any, val: any): any {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  // 判断 target 是否是对象，key 是否是合法的索引
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    // 通过 splice 对key位置的元素进行替换
    // splice 在 array.js 进行了响应化的处理
    target.splice(key, 1, val)
    return val
  }
  // 如果 key 在对象中已经存在直接赋值
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  // 获取 target 中的 observer 对象
  const ob = (target: any).__ob__
  // 如果 target 是 vue 实例或者 $data 直接返回
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  // 如果 ob 不存在，target 不是响应式对象直接赋值
  if (!ob) {
    target[key] = val
    return val
  }
  // 把 key 设置为响应式属性
  defineReactive(ob.value, key, val)
  // 发送通知
  ob.dep.notify()
  return val
}
```

#### delete

delete方法与set方法类似都有两种方式静态方法Vue.delete()或者实例方法vm.$delete()
这两个方法都来自于/core/observer/index.js的del函数

- 判断目标target是否是undefined或者原始值，如果是警告
- 判断是否是数组，并判断key是否是有效的下标值，如果是通过splice方法删除数组中的元素，返回
- 缓存target上的observer实例
- 判断target如果是vue实例或者是否是$data对象，发出警告并返回
- 接下来如果target上ownproperty不存在key属性，直接返回
- 通过delete删除属性 delete target[key]
- 如果target上不存在observer实例，直接返回，如果存在发送通知

### $watch

$watch没有静态方法
watch执行顺序，计算属性watch，用户watch,渲染watch

```js
vm = new Vaue({
  el: '#app',
  data: {
    user: {
      firstname: 'a',
      lastname: 'long',
      fullname: '',
    }
  }
})

vm.$watch('user', function(newvalue, oldvalue) {
  this.user.fullname = newvalue.firstname + newvalue.lastname
}, {
  immediate: true, //首次渲染时立即执行监听函数，默认只在目标改变时才监听
  deep: true, // 深度监听，当user和user的属性有变化都会触发监听函数
})
```

/core/instance/state.js/initWatch

通过initWatch,获取用户传入的watch对象，遍历属性，并处理属性对应的处理函数，如果处理函数是多个，为同一属性绑定多个处理函数

```js
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

createWatcher中会判断handler是否是对象，是的话,赋值handler为handler.handler  
接下来判断，如果handler是字符串，将handle赋值为vue实例上的方法，vm[handler]  
最后返回vm.$watch(...)

```js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

$watch方法定义，（expOrFn,cb,options）
首先获取vue实例
判断cb如果是原始对象，通过createWatcher重新解析
增加options属性user为true,标记为用户watcher，Watcher类中会对用户传入的回调函数try,catch
创建Watcher对象
如果options.immediate为true,立即执行一次回调函数
返回取消监听的方法

```js
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    // 获取 Vue 实例 this
    const vm: Component = this
    if (isPlainObject(cb)) {
      // 判断如果 cb 是对象执行 createWatcher
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    // 标记为用户 watcher
    options.user = true
    // 创建用户 watcher 对象
    const watcher = new Watcher(vm, expOrFn, cb, options)
    // 判断 immediate 如果为 true
    if (options.immediate) {
      // 立即执行一次 cb 回调，并且把当前值传入
      try {
        cb.call(vm, watcher.value)
      } catch (error) {
        handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
      }
    }
    // 返回取消监听的方法
    return function unwatchFn () {
      watcher.teardown()
    }
  }
}
```

### nextTick

当响应式数据发生变化时，如果需要获取变化后的dom内容，需要借助nextTick方法

nextTick有两种使用方式Vue.nextTick和vm.$nextTick  
两种方式最终都来自于core/util/next-tick.js的nextTick函数  
 
nextTick接收两个参数
- 回调函数
- 执行上下文，一般是vm
  

- 函数内部首先在全局数组中添加回调函数
- 然后通过timerFunc调用数组中的函数
  - timerFunc 内部优先通过微任务的方式执行回调函数的内容

```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 把 cb 加上异常处理存入 callbacks 数组中
  callbacks.push(() => {
    if (cb) {
      try {
        // 调用 cb()
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
    // 调用
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    // 返回 promise 对象
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```