# Vue.js 源码剖析-响应式原理、虚拟 DOM、模板编译和组件化

## 一、简答题

### 1、请简述 Vue 首次渲染的过程。
* Vue 初始化：初始化实例成员和静态成员，然后调用 Vue 的构造函数，在构造函数中调用了 _init 方法，相当于 Vue 的入口，在 _init 中最终调用了两个 $mount 方法
* 第一个 $mount 是在 `src/platforms/web/entry-runtime-with-compiler.js` 中，也就是入口文件的 $mount 核心作用是将模板编译成 render 函数，首先会判断是否传入了 render 如果没有传入，会获取 template 如果 template 也没有的话会把 el 中的内容作为模板，然后通过 compilerToFunction 函数把模板编译成 render 函数，编译成功后会把 render 函数保存到 options.render 中；第二个 $mount 是在 `src/platforms/web/runtime/index.js` 它会重新获取 el 因为如果是运行时版本是不会从第一个 $mount 中获取 el 的，然后会执行 mountComponent 方法
* mountComponent 方法是在 `src/core/instance/lifecycle.js` 中定义的，会首先判断当前是否有 render 选项，如果没有设置 render 但是传入了模板，并且当前是开发环境会发送警告，判断的目的是：如果当前使用的是运行时版本的 Vue 而且没有传入 render 但是传入了模板，此时会发送一个警告，告诉我们运行时版本不支持编译器，然后会触发 beforeMount 中的钩子函数，也就是开始挂载之前
* 然后定义了 updateComponent 此处仅仅是定义了这个函数，在这个函数中调用了 vm._update 和 vm._render 这两个方法
  * vm._render 的作用是生成虚拟 dom 
  * vm._update 的作用是把虚拟 dom 转换成真实 dom 并挂载到页面上
* 之后会创建 Watcher 对象，在创建时传入了 updateComponent 函数，这个函数最终会在 Watcher 对象当中被调用。创建完 watcher 对象会调用一次 get 方法，在 get 方法中会调用 updateComponent 方法
* 然后会触发生命周期中的钩子函数 mounted 于是挂载结束，最终返回 Vue 实例

### 2、请简述 Vue 响应式原理。
* Vue 的响应式是先从 Vue 实例的 init 方法开始的，在 init 方法中先调用 initState 初始化 Vue 实例的状态，在 initState 方法中调用了 initData 方法，它会把 data 属性注入到 Vue 实例上，并且调用 observe -- 响应式的入口 方法把 data 对象转换成响应式的对象
* observe 位置在 `src/core/observe/index.js` 它接收一个参数 value 这个参数就是响应式要处理的对象，首先判断当前接收到的 value 是否是一个对象，如果不是对象直接返回，然后再判断 value 对象是否有 __ob__ 如果有，说明这个对象之前已经做过了响应式处理，所以直接返回，如果没有，为这个对象创建 observe 对象，最后把 observe 对象返回
  * 创建 observe 对象：首先会为当前的 value 对象定义一个不可枚举的 __ob__ 属性，并且把当前 observe 对象记录到 __ob__ 当中，然后再进行数组的响应式处理和对象的响应式处理
    * 数组的响应式处理：设置数组方法，如：push、pop、shift 等，这些方法会改变源数组，所以当这些方法被调用时需要发送通知，发送通知会找到数组对象对应的 __ob__ 既 observe 对象，再找到这个 observe 对象的 dep 并且调用 dep 的 notify 方法，更改完数组的这些特殊方法之后，然后遍历数组中的每一个成员，针对每一个成员再去调用 observe 如果这个成员是对象的话，也会把这个对象转换为响应式对象
    * 对象的响应式处理：如果当前的 value 是对象的话，会调用 walk 方法，这个方法会遍历对象的所有属性，对每一个属性调用 defineReactive ：
      * 位置：`src/core/observe/index.js`
      * 功能：会为每一个属性创建 dep 对象，让 dep 去收集依赖，如果当前属性的值是对象，会调用 observe 把这个对象转换成响应式对象
        * 收集依赖过程：收集依赖时首先去执行 watcher 对象的 get 方法，在 get 方法中会调用 pushTarget 方法，在这个方法中会把当前的 watcher 对象记录到 Dep.target 属性中，然后访问 data 成员的属性的时候去收集依赖，这时，当访问这个属性的值的时候就会触发 defineReactive 中的 getter 这时在 getter 中会去收集依赖，它会把属性对应的 watcher 对象添加到 dep 的 subs 数组中，也就是为属性收集依赖，如果这个属性的值也是对象，此时会创建一个 childOb 对象，要为这个子对象收集依赖，目的是在子对象发生变化时发送通知
        * watcher ：当数据发生变化时，会调用 dep.notify() 发送通知，它会调用 watcher 对象的 update 方法，在这个方法中会去调用 queueWatcher 函数去判断 watcher 是否被处理了，如果 watcher 对象没有被处理的话会被添加到 queue 队列中，并且调用 flushSchedulerQueue 既刷新任务队列函数，在这个函数中会去触发 beforeUpdate 钩子函数，然后调用 watcher.run 方法，在这个方法中通过调用 watcher 的 get 方法调用 getter 在 getter 中存储的就是 updateComponent 这是针对渲染 watcher 当 watcher.run 运行完成之后就把数据更新到了视图上，在页面就能看到最新的数据了，接下来就是一些清理工作：会去清空上一次的依赖还会重置 watcher 中的一些状态，然后去触发 actived 钩子函数，最后去触发 updated 钩子函数
      * 核心：会定义 getter 和 setter 
        * 在 getter 中收集依赖，收集依赖时要为每一个属性收集依赖，如果这个属性的值是对象，也要为这个子对象收集依赖，最终返回这个属性的值
        * 在 setter 中首先把新值保存下来，如果新值是对象的话，调用 observe 把新设置的对象也转换成响应式的对象，在 setter 中数据发生变化所以要发送通知，既调用 dep.notify 方法

### 3、请简述虚拟 DOM 中 Key 的作用和好处。
* 作用：给元素节点添加标记，方便对比节点，找到被修改的元素，只替换被修改的元素，然后生成新的真实 dom 没有修改的元素保持不变
* 好处：减少不必要的 dom 操作，减少渲染时间，提升性能

### 4、请简述 Vue 中模板编译的过程。
* 模板编译入口函数 compileToFunctions 它的内部首先从缓存中加载编译好的 render 函数，如果缓存中没有的话都用 compile 开始编译
* 在 compile 函数中首先去合并 options 然后调用 baseCompile 编译模板 compile 函数的核心是合并选项，真正处理实在 baseCompile 中完成的，把模板和合并好的选项传递给 baseCompile 它的内部完成了模板编译核心的三件事：
  * 首先把模板字符串转换成 AST tree 也就是抽象语法树
  * 然后对抽象语法树进行优化：标记静态语法树中的所有静态根节点，静态根节点不需要每次被重绘，patch 过程中会跳过静态根节点，最后 AST 对象转换成字符串形式的 js 代码
* 当 compile 执行完毕后会编译的入口函数 compileToFunctions 会继续把上一步中生成的字符串形式的 js 代码通过调用 createFunction 转换成函数形式，当 render 和 staticRenderFns 创建完毕，最终它们都会被挂载到 Vue 实例的 options 选项对应的属性上
* 
