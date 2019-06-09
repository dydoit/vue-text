#### 基本用法
可以使用v-model指令在表单 <input>、<textarea> 及 <select> 元素上创建双向数据绑定，v-model 会忽略所有表单元素的 value、checked、selected 特性的初始值而总是将 Vue 实例的数据作为数据来源。你应该通过 JavaScript 在组件的 data 选项中声明初始值。v-model其实就是一个语法糖，它针对不同的输入元素使用不同的属性并抛出不同的事件
- text和textarea 元素使用value属性和input事件
- checkbox和radio 元素使用checked属性和change事件
- select字段将value作为prop并将change作为事件
```
<input type="text" v-model = "xx"/>
相当于
<input type="text" :value="xx" @input="xx = $event.target.value">
<input type="checkbox" v-model = "xx"/>
相当于
<input type="checkbox" :checked="xx" @change="xx = $event.target.checked">
```
来看源码怎么定义的
```
export default function model (
  el: ASTElement,
  dir: ASTDirective,
  _warn: Function
): ?boolean {
  warn = _warn
  const value = dir.value
  const modifiers = dir.modifiers
  const tag = el.tag
  const type = el.attrsMap.type
  if (el.component) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (tag === 'select') {
    genSelect(el, value, modifiers)
  } else if (tag === 'input' && type === 'checkbox') {
    genCheckboxModel(el, value, modifiers)
  } else if (tag === 'input' && type === 'radio') {
    genRadioModel(el, value, modifiers)
  } else if (tag === 'input' || tag === 'textarea') {
    genDefaultModel(el, value, modifiers)
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers)
    return false
  } 
  // ensure runtime directive metadata
  return true
}

function genCheckboxModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
) {
  const number = modifiers && modifiers.number
  const valueBinding = getBindingAttr(el, 'value') || 'null'
  const trueValueBinding = getBindingAttr(el, 'true-value') || 'true'
  const falseValueBinding = getBindingAttr(el, 'false-value') || 'false'
  // 下面就是会自动给prop上加个checked属性
  addProp(el, 'checked',
    `Array.isArray(${value})` +
    `?_i(${value},${valueBinding})>-1` + (
      trueValueBinding === 'true'
        ? `:(${value})`
        : `:_q(${value},${trueValueBinding})`
    )
  )
  // 添加一个change事件，如果v-model绑定的值是一个数组，选中则会把input的value值push到绑定的数组里，否则则从数组里删除。
  如果绑定的值不是一个数组，则执行else的逻辑，读取的是input的checked值而不是value。
  addHandler(el, 'change',
    `var $$a=${value},` +
        '$$el=$event.target,' +
        `$$c=$$el.checked?(${trueValueBinding}):(${falseValueBinding});` +
    'if(Array.isArray($$a)){' +
      `var $$v=${number ? '_n(' + valueBinding + ')' : valueBinding},` +
          '$$i=_i($$a,$$v);' +
      `if($$el.checked){$$i<0&&(${genAssignmentCode(value, '$$a.concat([$$v])')})}` +
      `else{$$i>-1&&(${genAssignmentCode(value, '$$a.slice(0,$$i).concat($$a.slice($$i+1))')})}` +
    `}else{${genAssignmentCode(value, '$$c')}}`,
    null, true
  )
}

function genRadioModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
) {
  const number = modifiers && modifiers.number
  let valueBinding = getBindingAttr(el, 'value') || 'null'
  valueBinding = number ? `_n(${valueBinding})` : valueBinding
  addProp(el, 'checked', `_q(${value},${valueBinding})`) 
  // radio的逻辑只有一个就是都是操作value
  addHandler(el, 'change', genAssignmentCode(value, valueBinding), null, true)
}

function genSelect (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
) {
  const number = modifiers && modifiers.number
  const selectedVal = `Array.prototype.filter` +
    `.call($event.target.options,function(o){return o.selected})` +
    `.map(function(o){var val = "_value" in o ? o._value : o.value;` +
    `return ${number ? '_n(val)' : 'val'}})`

  const assignment = '$event.target.multiple ? $$selectedVal : $$selectedVal[0]'
  let code = `var $$selectedVal = ${selectedVal};` // 找的是seletced 的value
  code = `${code} ${genAssignmentCode(value, assignment)}`
  addHandler(el, 'change', code, null, true)
}
```
#### 修饰符
- .lazy: 懒响应，表示响应的是change事件不再是input事件
- .trim: 自动过滤用户输入的首尾空白字符
- .number: 自动将用户的输入值转为数值类型
上述我们讲的都是普通input元素的情况，下面来讲一下组件上如何使用v-model
#### 组件上使用v-model
会默认注入value属性和input事件，无论input的类型是什么，同时可以设置model来改变注入的属性和事件名称。
```
<custom-input v-model="serchText"></custom-input>
相当于
<custom-input :value="serchText" @input="searchText = $event"></custom-input>
在组件内部必须用props来接收value，同时在input事件发生时$emit('input')
```
当然如果设置了model时候，相当于是把默认注入的名称改了
```
<custom v-model="serchText"></custom>
// custom组件内部
<template>
  <div>
    <input type="checkbox" :checked="checked" @change="handle"/>
  </div>
</template>
<script>
  export default {
    name: 'custom',
    props: ['checked'], // 默认是value，使用了model后，这里还是必须注入，而且名字和model里prop设置的一致。
    model: {
      prop:'checked',
      event: 'change'
    },
    methods: {
      handle(e) {
        this.$emit('change', e.target.checked)
      }
    }
  }
</script>
```
来看下源码，在创建子组件 vnode 阶段，会执行 createComponent 函数，它的定义在 src/core/vdom/create-component.js 中：
```
export function createComponent (
 Ctor: Class<Component> | Function | Object | void,
 data: ?VNodeData,
 context: Component,
 children: ?Array<VNode>,
 tag?: string
): VNode | Array<VNode> | void {
 // ...
 // transform component v-model data into props & events
 if (isDef(data.model)) { // 判断如果设置了model，就执行transformModel方法
   transformModel(Ctor.options, data)
 }

 // extract props
 const propsData = extractPropsFromVNodeData(data, Ctor, tag)
 // ...
 // extract listeners, since these needs to be treated as
 // child component listeners instead of DOM listeners
 const listeners = data.on
 // ...
 const vnode = new VNode(
   `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
   data, undefined, undefined, undefined, context,
   { Ctor, propsData, listeners, tag, children },
   asyncFactory
 )
 
 return vnode
}
```
其中会对 data.model 的情况做处理，执行 transformModel(Ctor.options, data) 方法：
```
// transform component v-model info (value and callback) into
// prop and event handler respectively.
function transformModel (options, data: any) {
  const prop = (options.model && options.model.prop) || 'value' // 优先读取model里设置的prop
  const event = (options.model && options.model.event) || 'input' // 优先读取model里设置的event
  ;(data.props || (data.props = {}))[prop] = data.model.value
  const on = data.on || (data.on = {})
  if (isDef(on[event])) {
    on[event] = [data.model.callback].concat(on[event])
  } else {
    on[event] = data.model.callback
  }
}
```