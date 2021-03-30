# vue 源码学习总结

## 数据驱动
    vue 的一个核心思想就是数据驱动，所谓数据驱动，是指视图由数据驱动生成的，
    我们对视图的修改不会直接修改dom，而是去修改数据，相对于传统的前端开发，
    大大简化了代码量， 让代码结构清晰，利于后期的代码维护。
    接下来会分析模版和数据如何渲染成最终的dom

### new Vue 发生了什么
  ### 
    Vue.prototype._init = function (options?: Object) {
      const vm: Component = this
      // a uid
      vm._uid = uid++

      let startTag, endTag
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        startTag = `vue-perf-start:${vm._uid}`
        endTag = `vue-perf-end:${vm._uid}`
        mark(startTag)
      }

      // a flag to avoid this being observed
      vm._isVue = true
      // merge options
      if (options && options._isComponent) {
        // optimize internal component instantiation
        // since dynamic options merging is pretty slow, and none of the
        // internal component options needs special treatment.
        initInternalComponent(vm, options)
      } else {
        vm.$options = mergeOptions(
          resolveConstructorOptions(vm.constructor),
          options || {},
          vm
        )
      }
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        initProxy(vm)
      } else {
        vm._renderProxy = vm
      }
      // expose real self
      vm._self = vm
      initLifecycle(vm)
      initEvents(vm)
      initRender(vm)
      callHook(vm, 'beforeCreate')
      initInjections(vm) // resolve injections before data/props
      initState(vm)
      initProvide(vm) // resolve provide after data/props
      callHook(vm, 'created')

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        vm._name = formatComponentName(vm, false)
        mark(endTag)
        measure(`vue ${vm._name} init`, startTag, endTag)
      }

      if (vm.$options.el) {
        vm.$mount(vm.$options.el)
      }
    }

  Vue 实际上是一个类， 类在javascript中是通过function实现的
  new Vue 生成vm实例会执行 Vue.prototype._init 方法

  ### 在_init方法中主要做了
    - 初始化生命周期
    - 初始化事件中心
    - 初始化渲染
    - 调用beforeCreate钩子函数
    - 初始化data、props、computed、watcher provider inject等等
    - 调用created 钩子函数
  
  在初始化的最后，检测options 中配置有el属性 那么调用 vm.$mount 方法执行挂载，
  挂载的目标就是把模版渲染成最终的dom。

  Vue 初始化的逻辑写的非常清晰，将不同的功能逻辑拆分成单独的函数去执行，让主线逻辑一目了然。

## vm.$mount 方法

    Vue中我们通过$mount方法挂载vm的，$mount方法在多个文件中都有定义，这个方法的实现与平台和构建方式相关的，接下来主要分析带compile版本的$mount实现，
  ##
    const mount = Vue.prototype.$mount
    Vue.prototype.$mount = function (
      el?: string | Element,
      hydrating?: boolean
    ): Component {
      el = el && query(el)

      /* istanbul ignore if */
      if (el === document.body || el === document.documentElement) {
        process.env.NODE_ENV !== 'production' && warn(
          `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
        )
        return this
      }

      const options = this.$options
      // resolve template/el and convert to render function
      if (!options.render) {
        let template = options.template
        if (template) {
          if (typeof template === 'string') {
            if (template.charAt(0) === '#') {
              template = idToTemplate(template)
              /* istanbul ignore if */
              if (process.env.NODE_ENV !== 'production' && !template) {
                warn(
                  `Template element not found or is empty: ${options.template}`,
                  this
                )
              }
            }
          } else if (template.nodeType) {
            template = template.innerHTML
          } else {
            if (process.env.NODE_ENV !== 'production') {
              warn('invalid template option:' + template, this)
            }
            return this
          }
        } else if (el) {
          template = getOuterHTML(el)
        }
        if (template) {
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
            mark('compile')
          }

          const { render, staticRenderFns } = compileToFunctions(template, {
            shouldDecodeNewlines,
            shouldDecodeNewlinesForHref,
            delimiters: options.delimiters,
            comments: options.comments
          }, this)
          options.render = render
          options.staticRenderFns = staticRenderFns

          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
            mark('compile end')
            measure(`vue ${this._name} compile`, 'compile', 'compile end')
          }
        }
      }
      return mount.call(this, el, hydrating)
    }

  ###
      这段代码首先缓存了vue 原型上的mount 方法， 然后又重写了mount 方法。
      这样做完全是为了复用，因为run-time-only版本mount 方法可以直接使用。
      首先他对el 做了限制，vue 实例不能直接挂载到body html 这样的根结点元素上，
      然后如果没有定义render方法，那么会把el或者template 字符串转换为
      render 方法， 无论我们写的是.vue 文件还是写了el 或者template 属性
      都会转换成render 方法。 这个过程是vue的一个在线编译的过程， 调用compileToFunction 方法实现。
      而mount 方法挂载实例通过调用 mountComponent 方法

##
    export function mountComponent (
      vm: Component,
      el: ?Element,
      hydrating?: boolean
    ): Component {
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
          const vnode = vm._render()
          mark(endTag)
          measure(`vue ${name} render`, startTag, endTag)

          mark(startTag)
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
      new Watcher(vm, updateComponent, noop, {
        before () {
          if (vm._isMounted) {
            callHook(vm, 'beforeUpdate')
          }
        }
      }, true /* isRenderWatcher */)
      hydrating = false

      // manually mounted instance, call mounted on self
      // mounted is called for render-created child components in its inserted hook
      if (vm.$vnode == null) {
        vm._isMounted = true
        callHook(vm, 'mounted')
      }
      return vm
    }

###
    以上代码可以看出， mountComponent 方法中会首先初始化一个渲染watcher，
    传入vm实例， updateComponent 回调函数。在此方法中调用vm._render 方法生成vnode，
    然后调用vm_.update方法更新dom
    Watcher 在这里起到两个作用，一个是初始化的时候会执行回调函数，另一个是当 
    vm 实例中的监测的数据发生变化的时候执行回调函数。

### vm._render 
    Vue.prototype._render = function (): VNode {
      const vm: Component = this
      const { render, _parentVnode } = vm.$options

      // reset _rendered flag on slots for duplicate slot check
      if (process.env.NODE_ENV !== 'production') {
        for (const key in vm.$slots) {
          // $flow-disable-line
          vm.$slots[key]._rendered = false
        }
      }

      if (_parentVnode) {
        vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
      }

      // set parent vnode. this allows render functions to have access
      // to the data on the placeholder node.
      vm.$vnode = _parentVnode
      // render self
      let vnode
      try {
        vnode = render.call(vm._renderProxy, vm.$createElement)
      } catch (e) {
        handleError(e, vm, `render`)
        // return error render result,
        // or previous vnode to prevent render error causing blank component
        /* istanbul ignore else */
        if (process.env.NODE_ENV !== 'production') {
          if (vm.$options.renderError) {
            try {
              vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
            } catch (e) {
              handleError(e, vm, `renderError`)
              vnode = vm._vnode
            }
          } else {
            vnode = vm._vnode
          }
        } else {
          vnode = vm._vnode
        }
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
        vnode = createEmptyVNode()
      }
      // set parent
      vnode.parent = _parentVnode
      return vnode
    }
 ###
    vm._render 方法实际上会获取compiltToFunction转换生成的render函数，执行这个render函数
    绑定 vm._renderProxy 传入createElement 方法生成vnode 并返回这个vnode, 而这个createElement、
    方法是在initRender 函数中绑定到vm实例上的。
### virtual dom
    virtual dom 使用一个原生的js对象去描述一个dom节点，浏览器在设计dom元素时设计的相当复杂，
    因此操作多个dom成本较高，会来带一定的性能问题。

###
    export default class VNode {
      tag: string | void;
      data: VNodeData | void;
      children: ?Array<VNode>;
      text: string | void;
      elm: Node | void;
      ns: string | void;
      context: Component | void; // rendered in this component's scope
      key: string | number | void;
      componentOptions: VNodeComponentOptions | void;
      componentInstance: Component | void; // component instance
      parent: VNode | void; // component placeholder node

      // strictly internal
      raw: boolean; // contains raw HTML? (server only)
      isStatic: boolean; // hoisted static node
      isRootInsert: boolean; // necessary for enter transition check
      isComment: boolean; // empty comment placeholder?
      isCloned: boolean; // is a cloned node?
      isOnce: boolean; // is a v-once node?
      asyncFactory: Function | void; // async component factory function
      asyncMeta: Object | void;
      isAsyncPlaceholder: boolean;
      ssrContext: Object | void;
      fnContext: Component | void; // real context vm for functional nodes
      fnOptions: ?ComponentOptions; // for SSR caching
      fnScopeId: ?string; // functional scope id support

      constructor (
        tag?: string,
        data?: VNodeData,
        children?: ?Array<VNode>,
        text?: string,
        elm?: Node,
        context?: Component,
        componentOptions?: VNodeComponentOptions,
        asyncFactory?: Function
      ) {
        this.tag = tag
        this.data = data
        this.children = children
        this.text = text
        this.elm = elm
        this.ns = undefined
        this.context = context
        this.fnContext = undefined
        this.fnOptions = undefined
        this.fnScopeId = undefined
        this.key = data && data.key
        this.componentOptions = componentOptions
        this.componentInstance = undefined
        this.parent = undefined
        this.raw = false
        this.isStatic = false
        this.isRootInsert = true
        this.isComment = false
        this.isCloned = false
        this.isOnce = false
        this.asyncFactory = asyncFactory
        this.asyncMeta = undefined
        this.isAsyncPlaceholder = false
      }

      // DEPRECATED: alias for componentInstance for backwards compat.
      /* istanbul ignore next */
      get child (): Component | void {
        return this.componentInstance
      }
    }
###
  其实vnode 是对真是dom的一种抽象描述， 他的核心定义无非几个关键属性， tag
  child data key 其他的属性是为了拓展vue 特性 加入的 vue特色的东西。
  virtual dom 除了他数据结构的定义 映射到真实dom 实际上需要经过vnode 的
  create diff patch 等过程。 vode 的create 阶段是通过createElement 
  方法实现的。

### createElement
  