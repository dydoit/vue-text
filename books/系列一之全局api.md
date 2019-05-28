#### 一、MVVM（Model-view-viewmodel）
- M：模型，内容的数据访问层
- V：视图，用户在屏幕上看到的结构、布局和外观（UI）
- viewmodel：视图模型，是暴露公共属性和命令的视图的抽象，内部有一个绑定器，绑定器在视图和数据绑定器之间进行通信。
**Vue框架虽然没有完全遵循MVVM，但是大部分借鉴了MVVM的设计思想，VM层就是vue的内部机制，vue开发就是面向数据层的开发，全部精力在操作M层也就是数据层。**
#### 二、Vue是个什么鬼
Vue其实就是一个构造函数对象，我们来看一段源码，在Vue源码src/core/instance/index.js中有定义
```
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```
这里先声明一个构造类Vue，后面传入Vue调用一系列Mixin方法，在其他的文件继续给Vue和Vue.prototype添加全局属性和全局方法。源码里有调用initGlobalAPI(Vue)
```
export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue) // 定义了Vue.use
  initMixin(Vue) //定义了Vue.mixin
  initExtend(Vue) // 定义了Vue.extend
  initAssetRegisters(Vue)
}
```
#### 三、 Vue源码构建
Vue源码是基于Rollup构建的，源码工程里一样有个package.json 文件，看部分代码
```
{
  "build": "node scripts/build.js",
  "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer",
  "build:weex": "npm run build -- weex",
}
```
当运行npm bun build时执行“node scripts/build.js”生成Vue这个库的js代码。在scripts/build.js中又去require了config
```
let builds = require('./config').getAllBuilds()

// filter builds via command line arg
if (process.argv[2]) {
  const filters = process.argv[2].split(',')
  builds = builds.filter(b => {
    return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
  })
} else {
  // filter out weex builds by default
  builds = builds.filter(b => {
    return b.output.file.indexOf('weex') === -1
  })
}

build(builds)
```
这段代码逻辑非常简单，先从配置文件读取配置，再通过命令行参数对构建配置做过滤，这样就可以构建出不同用途的 Vue.js 了。接下来我们看一下配置文件，在 scripts/config.js 中：(注意这是最终的boss，不同版本的vue入口文件和出口文件都定义了)
```
const builds = {
  // Runtime only (CommonJS). Used by bundlers e.g. Webpack & Browserify
  'web-runtime-cjs': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.common.js'),
    format: 'cjs',
    banner
  },
  // Runtime+compiler CommonJS build (CommonJS)
  'web-full-cjs': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.common.js'),
    format: 'cjs',
    alias: { he: './entity-decoder' },
    banner
  },
  // Runtime only (ES Modules). Used by bundlers that support ES Modules,：不能直接在浏览器中引入，不带编译代码
  // e.g. Rollup & Webpack 2
  'web-runtime-esm': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.esm.js'),
    format: 'es',
    banner
  },
  // Runtime+compiler CommonJS build (ES Modules)
  'web-full-esm': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.esm.js'),
    format: 'es',
    alias: { he: './entity-decoder' },
    banner
  },
  // runtime-only build (Browser) ：可以直接在浏览器引入，未压缩的，不带编译代码
  'web-runtime-dev': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.js'),
    format: 'umd',
    env: 'development',
    banner
  },
  // runtime-only production build (Browser)：可以直接在浏览器引入，压缩的，不带编译代码
  'web-runtime-prod': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.min.js'),
    format: 'umd',
    env: 'production',
    banner
  },
  // Runtime+compiler development build (Browser)：可以直接在浏览器引入，未压缩最全版
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.js'),
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
  // Runtime+compiler production build  (Browser)：可以直接在浏览器引入，压缩最全版
  'web-full-prod': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.min.js'),
    format: 'umd',
    env: 'production',
    alias: { he: './entity-decoder' },
    banner
  },
  // ...
}
```
生成dist目录下不同的版本js文件，怎么区分
- 可直接在浏览器引入使用的： Vue.js()
#### 四、Vue全局属性和全局方法
##### 全局属性
1. 全局配置对象config，包含 Vue 的全局配置，可以在启动应用前修改里面的属性。源码在src/core/config.js里有定义，包含
- silent： 默认值是false，表示开启vue日记和警告
- optionMergeStrategies: 默认值是{},表示继承时如何合并属性，默认是没有的属性直接继承，有的属性继承时替换掉。可以自定义，很少用
```
Vue.config.optionMergeStrategies._my_option = function (parent, child, vm) {
  return child + 1
}
const Profile = Vue.extend({
  _my_option: 1
})
// Profile.options._my_option = 2
```
- devtools: 默认值为true（生产版默认为false），表示是否允许 vue-devtools 检查代码
- errorHandler：默认值为undefined，开启这个函数可以来捕捉错误，很少用
- warnHandler:默认值为undefined
- ignoredElements：默认值是[]
- keyCodes:默认值是{},用来自定义键位别名，有时候有用
```
Vue.config.keyCodes = {
  v: 86,
  f1: 112,
  // camelCase 不可用
  mediaPlayPause: 179,
  // 取而代之的是 kebab-case 且用双引号括起来
  "media-play-pause": 179,
  up: [38, 87]
}
```
```
<input type="text" @keyup.media-play-pause="method">
```
- performance: 默认值为false
- productionTip: 默认值为true,一般会设置为false以阻止 vue 在启动时浏览器控制台生成生产提示。
##### 全局方法API
1. Vue.extend(objOptions)
用来创建一个子类，objOptions是一个包含组件选项的对象。来看下源码
```
Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }

    const name = extendOptions.name || Super.options.name
    if (process.env.NODE_ENV !== 'production' && name) {
      validateComponentName(name)
    }

    const Sub = function VueComponent (options) {
      this._init(options)
    }
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid++
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    Sub['super'] = Super

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    // allow further extension/mixin/plugin usage
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    // enable recursive self-lookup
    if (name) {
      Sub.options.components[name] = Sub
    }

    // keep a reference to the super options at extension time.
    // later at instantiation we can check if Super's options have
    // been updated.
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // cache constructor
    cachedCtors[SuperId] = Sub
    return Sub
  }
}
```
注意objOptions里的data必须是一个方法，和组件里的data一样。objOptions如果想修改继承过来的props里的属性，必须通过设置propsData来重新定义。还有一点，组件内部使用时是extends，不是extend。**其实全局api中filter、component、extend在Vue源码内部都直接在他们尾部拼了一个‘s’，和组件内部使用时一致。**
2. Vue.use(plugin)
来看源码
```
 Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) { //如果安装过，就直接return。
      return this
    }
    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
```
如果插件是一个对象，必须提供 install 方法。如果插件是一个函数，它会被作为 install 方法。install 方法调用时，会将 Vue 作为参数传入。相同的插件只会安装一次。
#### new Vue()
![生命周期示意图](https://cn.vuejs.org/images/lifecycle.png "Optional title")