[toc]

# 1.vue的理解

## 1.1vue的定义

是一个用于创建用户界面的开源JavaScript框架，也是一个创建单页应用的Web应用框架

是一套用于构建用户界面的渐进式MVVM框架

ps：渐进式：强制主张最少



## 1.2vue的功能

包含了声明式渲染、组件化系统、客户端路由、大规模状态管理、构建工具、数据持久化、跨平台支持等，但在实际开发中，并没有强制要求开发者之后某一特定功能，而是根据需求逐渐扩展

核心：MVC模式中的视图层，同时，它也能方便地获取数据更新，并通过组件内部特定的方法实现视图与模型的交互



## 1.3vue的工作过程

核心库：只关心视图渲染，且由于渐进式的特性，Vue.js便于与第三方库或既有项目整合。

Vue.js实现了一套**声明式渲染引擎**，并在runtime或者预编译时将声明式的模板编译成**渲染函数**，挂载在观察者**Watcher**中，在渲染函数中(touch),响应式系统使用响应式数据的**getter**方法对观察者进行**依赖收集**(Collect as Dependency)，使用响应式数据的**setter**方法**通知**(notify)所有**观察者进行更新**，此时观察者**Watcher会触发组件的渲染函数**，组件执行的render函数，**生成一个新的Vitural DOM Tree**, 此时Vue会对新老Virtual DOM Tree 进行**Diff**, 查找出需要操作的真实DOM并对其进行更新



# 2.双向绑定的理解

## 2.1什么是双向绑定

单向绑定： model绑定到view   js更新model时，view也会自动更新

双向绑定：单向基础上 用户更新view, model也会更新

## 2.2双向绑定的原理

vue是数据双向绑定的框架，双向绑定由三个重要部分构成

**数据层**(Model)：应用的数据以及业务逻辑

**视图层**(View)：应用的展示效果，各类UI组件

**业务逻辑层**(ViewModel)：框架封装的核心，它负责将数据与视图关联起来



**理解ViewModel**

主要职责：

- 数据变化后更新视图
- 视图变化后更新视图

部分组成

- 监听器(Observer) : 对所有数据的属性进行监听

- 解析器(Compiler): 对每个元素节点的指令进行扫描跟解析，根据指令模板替换数据，以及绑定相应的更新函数

  

## 2.3实现

### 双向绑定流程

- new Vue() 执行初始化，对**data执行响应化处理**，这个过程发生在**Observe**中
- 同时**对模板执行编译**，找到其中动态绑定的数据，**从data中获取并初始化视图**，这个过程发生在**Compile**中
- 同时**定义一个更新函数和Watcher**, 将来对应数据变化时Watcher会调用更新函数
- 由于data的某个key在一个视图中可能出现多次，所以每个**key都需要一个管家Dep来管理多个Watcher**
- 将来**data中数据一旦发生变化，会首先找到对应的Dep, 通知所有Watcher**



### 代码实现

#### **初始化对data进行响应化处理**

```
class Vue {
	constructor(options) {
		this.$options = options;
		this.$data = options.data;
	}
  //对data做响应式处理
	observe(this.$data);
	//代理data到vm上
	proxy(this);
	//执行编译
	new Compile(options.el, this);
}
```

#### **对data进行响应化具体操作** **observer**

```
function observe(obj) {
	if (typeof obj !== "object" || obj == null) {
		return;
	}
	new Observer(obj);
}
//将data里的所有属性包括对象里的属性劫持
class Observer {
  constructor(data) {
    this.observer(data);
  }
  observer(data) {
    if (data && typeof data == "object") {
      for (let key in data) {
        this.defineReactive(data, key, data[key]);
      }
    }
  }
  defineReactive(obj, key, value) {
    //value还是对象的话要继续,才会给全部都赋予get和set方法
    this.observer(value);
    let dep = new Dep(); //给每个属性都加上一个发布订阅功能
    Object.defineProperty(obj, key, {
      get() {
        //创建watcher时候，会取到对应内容，并且把watcher放到全局上
        //添加订阅器
        Dep.target && dep.addSub(Dep.target);
        return value;
      },
      set(newVal) {
        //若赋值的是一个对象，还需要继续监控 订阅器更新
        if (newVal != value) {
          this.observer(newVal);
          value = newVal;
          dep.notify();
        }
      },
    });
  }
}
```

#### **编译Compile**

对元素节点指令扫描解析  根据指令模板替换数据，以及绑定相应的更新函数

```
class Compiler {
  constructor(el, vm) {
    //判断el属性
    this.el = this.isElementNode(el) ? el : document.querySelector(el);
    this.vm = vm;
    //把当前节点中的元素获取到，并放到内存中
    let fragment = this.node2fragment(this.el);
    //把节点中内容进行替换

    //编译模板，用数据编译
    this.compile(fragment);
    //把内容塞回页面
    this.el.appendChild(fragment);
  }
  //判断是不是指令
  isDirective(attrName) {
    return attrName.startsWith("v-"); //开头
  }
  
  //编译元素的方法
  compileElement(node) {
    let attributes = node.attributes; //类数组
    [...attributes].forEach((attr) => {
      let { name, value: expr } = attr;
      if (this.isDirective(name)) {
        //v-model v-html v-bind
        let [, directive] = name.split("-"); //v-on:click
        let [directiveName, eventName] = directive.split(":");
        //调用不同指令来处理
        CompileUtil[directiveName](node, expr, this.vm, eventName);
      }
    });
  }
  
  //编译文本的方法
  compileText(node) {
    //判断文本节点中是否包含{{}}
    let content = node.textContent;
    //(.+?)匹配一个大括号内的，一个及以上，到第一个大括号结束时候结束
    if (/\{\{(.+?)\}\}/.test(content)) {
      CompileUtil["text"](node, content, this.vm);
    }
  }
  //编译的核心方法
  compile(node) {
    let childNodes = node.childNodes;
    [...childNodes].forEach((child) => {
      if (this.isElementNode(child)) {
        this.compileElement(child);
        this.compile(child); //递归，获得内层
      } else {
        this.compileText(child);
      }
    });
  }
  node2fragment(node) {
    //创建一个文本碎片
    let fragment = document.createDocumentFragment();
    let firstChild;
    while ((firstChild = node.firstChild)) {
      //appendChild具有移动性
      fragment.appendChild(firstChild);
    }
    return fragment;
  }

  isElementNode(node) {
    //判断是否为元素节点
    return node.nodeType === 1;
  }
}

//绑定处理事件的各种方法
CompileUtil = {
  //取得对应的数据
  getVal(vm, expr) {
    //vm.$data 'school.name'
    //返回name
    return expr.split(".").reduce((data, current) => {
      return data[current]; //继续取值，取到name
    }, vm.$data);
  },
  setValue(vm, expr, value) {
    expr.split(".").reduce((data, current, index, arr) => {
      if (index == arr.length - 1) {
        return (data[current] = value);
      }
      return data[current];
    }, vm.$data);
  },
  model(node, expr, vm) {
    //node节点，expr是表达式，vm是当前实例
    let fn = this.updater["modeUpdater"];
    //给输入框加一个观察者，稍后数据更新就会触发此方法，将新值给输入框赋予值
    new Watcher(vm, expr, (newVal) => {
      fn(node, newVal);
    });
    node.addEventListener("input", (e) => {
    	let value = e.target.value; //获取用户输入的内容
      this.setValue(vm, expr, value);
    });
    let value = this.getVal(vm, expr);
    fn(node, value);
  },
  html(node, expr, vm) {
    let fn = this.updater["htmlUpdater"];
    new Watcher(vm, expr, (newVal) => {
      fn(node, newVal);
    });
    let value = this.getVal(vm, expr);
    fn(node, value);
  },
  getContentValue(vm, expr) {
    //遍历一个表达式，将内容重新替换成一个完整的内容，返还回去
    return expr.replace(/\{\{(.+?)\}\}/g, (...args) => {
      return this.getVal(vm, args[1]);
    });
  },
  on(node, expr, vm, eventName) {
    //  v-on:click="change"  expr就是change
    node.addEventListener(eventName, (e) => {
      vm[expr].call(vm, e); //this.change
    });
  },
  text(node, expr, vm) {
    let fn = this.updater["textUpdater"];
    let content = expr.replace(/\{\{(.+?)\}\}/g, (...args) => {
      //给表达式每个{{}}都加个观察者
      new Watcher(vm, args[1], (newVal) => {
        fn(node, this.getContentValue(vm, expr)); //返回一个全的字符串
      });
      return this.getVal(vm, args[1]);
    });
    fn(node, content);
  },

  //更新视图
  updater: {
    //把数据插入节点当中
    modeUpdater(node, value) {
      node.value = value;
    },
    textUpdater(node, value) {
      node.textContent = value;
    },
    htmlUpdater(node, value) {
      //xss攻击
      node.innerHTML = value;
    },
  },
};

参考：https://blog.csdn.net/imapig_/article/details/119600987

```

#### **依赖收集**

视图中会用到data中某key,这称为依赖。同一个key可能出现多次，每次都需要收集出一个watcher来维护他们，此过程称为依赖收集；多个watcher需要一个Dep来管理，需要更新时由Dep统一通知

实现思路：

1.defineReactive时为每一个key创建一个Dep实例

2.初始化视图读取name1, 创建一个watcher1

3.由于触发name1的getter方法，便将watcher1添加到name1对应的Dep中

4.当name1更新，setter触发时，便可通过对应Dep通知其管理所有Watcher更新

```
//定义一个容器类 来存放所有的订阅者
class Dep {
  constructor() {
    this.subs = [];  //依赖管理
  }
  //订阅
  addSub(watcher) {
    this.subs.push(watcher);
  }
  //发布
  notify() {
    this.subs.forEach((watcher) => watcher.update());
  }
}
//观察者：将数据劫持和页面联系起来
class Watcher {
  constructor(vm, expr, cb) {
    this.vm = vm;
    this.expr = expr;
    this.cb = cb;
    //默认存放一个老值
    this.oldValue = this.get();
  }
  get() {
    //创建实例时，把当前实例指定到Dep.target静态属性上
    Dep.target = this; 
    //获取值  触发get
    let value = CompileUtil.getVal(this.vm, this.expr);
    //置空
    Dep.target = null;
    return value;
  }
  update() {
    //更新操作，数据变化后会调用观察者update方法
    let newVal = CompileUtil.getVal(this.vm, this.expr);
    if (newVal !== this.oldValue) {
      this.cb(newVal);
    }
  }
}
```

# 3.v-show和v-if的区别

## 3.1共同点

  在 vue 中 v-show 与 v-if 的作用效果是相同的(不含v-else)，都能控制元素在页面是否显示 。

- 当表达式都为 false 时，都不会占据页面位置
- 当表达式结果为 true 时，都会占据页面的位置

## 3.2区别

- 控制手段：v-show隐藏则是为该元素添加css--display:none，dom元素依旧还在。v-if显示隐藏是将dom元素整个添加或删除

- 编译过程：v-if切换有一个局部编译/卸载的过程，切换过程中合适地销毁和重建内部的事件监听和子组件；v-show只是简单的基于css切换

- 编译条件：v-if是真正的条件渲染，它会确保在切换过程中条件块内的事件监听器和子组件适当地被销毁和重建。只有渲染条件为假时，并不做操作，直到为真才渲染

  **1）**v-show 由false变为true的时候不会触发组件的生命周期

  **2）**v-if由false变为true的时候，触发组件的beforeCreate、create、beforeMount、mounted钩子，由true变为false的时候触发组件的beforeDestory、destoryed方法

- 性能消耗：v-if有更高的切换消耗；v-show有更高的初始渲染消耗

## 3.3v-show和v-if原理分析

具体解析流程这里不展开讲，大致流程如下

- 将模板template转为ast结构的JS对象

- 用ast得到的JS对象拼装render和staticRenderFns函数

- render和staticRenderFns函数被调用后生成虚拟VNODE节点，该节点包含创建DOM节点所需信息

- vm.patch函数通过虚拟DOM算法利用VNODE节点创建真实DOM节点


### **v-show原理**

不管初始条件是什么，元素总是会被渲染

我们看一下在vue中是如何实现的

代码很好理解，有transition就执行transition，没有就直接设置display属性

```
// https://github.com/vuejs/vue-next/blob/3cd30c5245da0733f9eb6f29d220f39c46518162/packages/runtime-dom/src/directives/vShow.ts
export const vShow: ObjectDirective<VShowElement> = {
  beforeMount(el, { value }, { transition }) {
    el._vod = el.style.display === 'none' ? '' : el.style.display
    if (transition && value) {
      transition.beforeEnter(el)
    } else {
      setDisplay(el, value)
    }
  },
  mounted(el, { value }, { transition }) {
    if (transition && value) {
      transition.enter(el)
    }
  },
  updated(el, { value, oldValue }, { transition }) {
    // ...
  },
  beforeUnmount(el, { value }) {
    setDisplay(el, value)
  }
}
```

### v-if 原理

v-if 在实现上比v-show要复杂的多，因为还有else else-if 等条件需要处理，这里我们也只摘抄源码中处理 v-if 的一小部分

返回一个node节点，render函数通过表达式的值来决定是否生成DOM

```
// https://github.com/vuejs/vue-next/blob/cdc9f336fd/packages/compiler-core/src/transforms/vIf.ts
export const transformIf = createStructuralDirectiveTransform(
  /^(if|else|else-if)$/,
  (node, dir, context) => {
    return processIf(node, dir, context, (ifNode, branch, isRoot) => {
      // ...
      return () => {
        if (isRoot) {
          ifNode.codegenNode = createCodegenNodeForBranch(
            branch,
            key,
            context
          ) as IfConditionalExpression
        } else {
          // attach this branch's codegen node to the v-if root.
          const parentCondition = getParentCondition(ifNode.codegenNode!)
          parentCondition.alternate = createCodegenNodeForBranch(
            branch,
            key + ifNode.branches.length - 1,
            context
          )
        }
      }
    })
  }
)
```

## 3.4 v-show与v-if的使用场景

v-if 与 v-show 都能控制dom元素在页面的显示

v-if 相比 v-show 开销更大的（直接操作dom节点增加与删除）

如果需要非常频繁地切换，则使用 v-show 较好

如果在运行时条件很少改变，则使用 v-if 较好

**参考文献**
https://www.jianshu.com/p/7af8554d8f08
https://juejin.cn/post/6897948855904501768
https://vue3js/docs/zh

https://blog.csdn.net/weixin_57519185/article/details/121168426

# 4.vue实例挂载过程

## 4.1思考

- new Vue() 究竟做了什么
- 如何完成数据绑定，如何将数据渲染到视图的

## 4.2分析

首先找到Vue的构造函数

源码位置: src/core/instance/index.js

```
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```


options是用户传递过来的配置项，如data、methods等常用的方法。

Vue构建函数调用_init方法，但我们发现本文件中并没有此方法，但仔细可以看到文件下方定义了很多初始化方法。

```
initMixin(Vue);     // 定义 _init
stateMixin(Vue);    // 定义 $set $get $delete $watch 等
eventsMixin(Vue);   // 定义事件  $on  $once $off $emit
lifecycleMixin(Vue);// 定义 _update  $forceUpdate  $destroy
renderMixin(Vue);   // 定义 _render 返回虚拟dom

```

首先可以看到initMixin方法，发现该方法在Vue原型上定义了_init方法。

源码位置： src/core/instance/init.js

```
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
```

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    // 合并属性，判断初始化的是否是组件，这里合并主要是 mixins 或 extends 的方法
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else { // 合并vue属性
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      // 初始化proxy拦截器
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    // 初始化组件生命周期标志位
    initLifecycle(vm)
    // 初始化组件事件侦听
    initEvents(vm)
    // 初始化渲染方法
    initRender(vm)
    callHook(vm, 'beforeCreate')
    // 初始化依赖注入内容，在初始化data、props之前
    initInjections(vm) // resolve injections before data/props
    // 初始化props/data/method/watch/methods
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
    
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }
    // 挂载元素
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
    }
    
    仔细阅读上面的代码，我们得到以下的结论：
在调用beforeCreate之前，数据初始化并未完成，像data、props这些属性无法访问到
到了created的时候，数据已经初始化完成，能够访问到data、props这些属性，但这时候并未完成dom的挂载，因此无法访问到dom元素。
挂载方法是调用vm.$mount方法
initState方法是完成 props/data/method/watch/methods的初始化。

源码位置：src/core/instance/state.js

```
export function initState (vm: Component) {
  // 初始化组件的watcher列表
  vm._watchers = []
  const opts = vm.$options
  // 初始化props
  if (opts.props) initProps(vm, opts.props)
  // 初始化methods方法
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    // 初始化data  
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


我们在这里主要看初始化data的方法为initData，它与initState在同一文件上

```
function initData (vm: Component) {
  let data = vm.$options.data
  // 获取到组件上的data
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
    if (process.env.NODE_ENV !== 'production') {
      // 属性名不能与方法名重复
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    // 属性名不能与state名称重复
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) { // 验证key值的合法性
      // 将_data中的数据挂载到组件vm上,这样就可以通过this.xxx访问到组件上的数据
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  // 响应式监听data是数据的变化
  observe(data, true /* asRootData */)
}
```


仔细阅读上面的代码，我们可以得到以下结论：

初始化顺序：props、methods、data
data定义的时候可选择函数形式或者对象形式(组件只能为函数形式)
关于数据响应式在这就不展示详细说明了
上文提到挂载方法是调用vm.$mount方法

源码：

```
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // 获取或查询元素
  el = el && query(el)

  /* istanbul ignore if */
  // vue 不允许直接挂载到body或页面文档上
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
    // 存在template模板，解析vue模板文件
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
      // 通过选择器获取元素内容
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }
      /**
       *  1.将temmplate解析ast tree
       *  2.将ast tree转换成render语法字符串
       *  3.生成render方法
       */
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
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


```

阅读上面代码，我们能得到以下结论：

- 不要将根元素放到body或者html上

- 可以在对象中定义template/render或者直接使用template、el表示元素选择器
- 最终都会解析成render函数，调用compileToFunctions，会将template解析成render函数

对template的解析步骤大致分为以下几步：

- 将html文档片段解析成ast描述符

- 将ast描述符解析成字符串
- 生成render函数
- 生成render函数，挂载到vm上后，会再次调用mount方法

源码位置：src/platforms/web/runtime/index.js

// public mount method

```
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  // 渲染组件
  return mountComponent(this, el, hydrating)
}

```

调用mountComponent渲染组件

```
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  // 如果没有获取解析的render函数，则会抛出警告
  // render是解析模板文件生成的
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
        // 没有获取到vue的模板文件
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  // 执行beforeMount钩子
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
    // 定义更新函数
    updateComponent = () => {
      // 实际调⽤是在lifeCycleMixin中定义的_update和renderMixin中定义的_render
      vm._update(vm._render(), hydrating)
    }
  }
  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  // 监听当前组件状态，当有数据变化时，更新组件
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        // 数据更新引发的组件更新
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
```


阅读上面代码，我们得到以下结论：

- 会触发beforeCreate钩子

- 定义updateComponent渲染页面视图的方法
- 监听组件数据，一旦发生变化，触发beforeUpdate生命钩子
- updateComponent方法主要执行在vue初始化时声明的render, update方法

render的作用主要是生成vnode

源码位置：src/core/instance/render.js

// 定义vue 原型上的render方法

```
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    // render函数来自于组件的option
    const { render, _parentVnode } = vm.$options
```

```
if (_parentVnode) {
    vm.$scopedSlots = normalizeScopedSlots(
        _parentVnode.data.scopedSlots,
        vm.$slots,
        vm.$scopedSlots
    )
}

// set parent vnode. this allows render functions to have access
// to the data on the placeholder node.
vm.$vnode = _parentVnode
// render self
let vnode
try {
    // There's no need to maintain a stack because all render fns are called
    // separately from one another. Nested component's render fns are called
    // when parent component is patched.
    currentRenderingInstance = vm
    // 调用render方法，自己的独特的render方法， 传入createElement参数，生成vNode
    vnode = render.call(vm._renderProxy, vm.$createElement)
} catch (e) {
    handleError(e, vm, `render`)
    // return error render result,
    // or previous vnode to prevent render error causing blank component
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
        try {
            vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
        } catch (e) {
            handleError(e, vm, `renderError`)
            vnode = vm._vnode
        }
    } else {
        vnode = vm._vnode
    }
} finally {
    currentRenderingInstance = null
}
// if the returned array contains only a single node, allow it
if (Array.isArray(vnode) && vnode.length === 1) {
    vnode = vnode[0]
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
```


_update主要功能是调用patch，将vnode转成为真实DOM，并且更新到页面中

源码位置：src/core/instance/lifecycle.js

```
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    // 设置当前激活的作用域
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      // 执行具体的挂载逻辑
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```



## 4.3结论

- new Vue的时候会调用_init方法
- 定义$set、$get、$delete、$watch等方法
- 定义$on、$off、$emit、$off等事件
- 定义_update、$forceUpdate、$destory生命周期
- 调用$mount进行页面的挂载
- 挂载的时候主要是通过mountComponent方法
- 定义updateComponent更新函数
- 执行render生成虚拟DOM
- _update将虚拟DOM生成真实DOM结构，并且渲染到页面中

参考文献
https://www.cnblogs.com/gerry2019/p/12001661.html
https://github.com/vuejs/vue/tree/dev/src/core/instance
https://vue3js.cn

https://blog.csdn.net/m0_46171043/article/details/12233063

# 5.SPA首屏加载缓慢怎么解决

## 5.1首屏加载的含义

⾸屏时间（First Contentful Paint），指的是浏览器从响应⽤户输⼊⽹址地址，到⾸屏内容渲染完成的时间，此时整个⽹页不⼀定要全部渲染完成，但需要展⽰当前视窗需要的内容

⾸屏加载可以说是⽤户体验中最重要的环节
关于计算⾸屏时间 利⽤performance.timing提供的数据：

通过DOMContentLoad或者performance来计算出⾸屏时间

```
// ⽅案⼀：
document.addEventListener('DOMContentLoaded', (event) => {
    console.log('first contentful painting');
});
// ⽅案⼆：
performance.getEntriesByName("first-contentful-paint")[0].startTime
// performance.getEntriesByName("first-contentful-paint")[0]
// 会返回⼀个 PerformancePaintTiming的实例，结构如下：
{
  name: "first-contentful-paint",
  entryType: "paint",
  startTime: 507.80000002123415,
  duration: 0,

}
```

## 5.2缓慢原因

### 页⾯渲染的过程，导致加载速度慢的因素可能如下：

⽹络延时问题
资源⽂件体积是否过⼤
资源是否重复发送请求去加载了

加载脚本的时候，渲染内容堵塞了

## 5.3解决方案

### 常见的⼏种SPA⾸屏优化⽅式

减⼩⼊⼝⽂件体积
静态资源本地缓存
UI框架按需加载
图⽚资源的压缩
组件重复打包
开启GZip压缩

使⽤SSR

### 减⼩⼊⼝⽂件体积
常⽤的⼿段是路由懒加载，把不同路由对应的组件分割成不同的代码块，待路由被请求的时候会单独打包路由，使得⼊⼝⽂件变⼩，加载速
度⼤⼤增加
在vue-router配置路由的时候，采⽤动态加载路由的形式

```
routes:[ 
    path: 'Blogs',
    name: 'ShowBlogs',
    component: () => import('./components/ShowBlogs.vue')
]
```

以函数的形式加载路由，这样就可以把各⾃的路由⽂件分别打包，只有在解析给定的路由时，才会加载路由组件

### 静态资源本地缓存

后端返回资源问题：
采⽤HTTP缓存，设置Cache-Control，Last-Modified，Etag等响应头
采⽤Service Worker离线缓存
前端合理利⽤localStorage
UI框架按需加载
在⽇常使⽤UI框架，例如element-UI、或者antd，我们经常性直接饮⽤整个UI库

```
import ElementUI from 'element-ui'
Vue.use(ElementUI)
```

但实际上我⽤到的组件只有按钮，分页，表格，输⼊与警告 所以我们要按需引⽤

```
import { Button, Input, Pagination, Table, TableColumn, MessageBox } from 'element-ui';
Vue.use(Button)
Vue.use(Input)
Vue.use(Pagination)
```

### 组件重复打包

假设A.js⽂件是⼀个常⽤的库，现在有多个路由使⽤了A.js⽂件，这就造成了重复下载

解决⽅案：在webpack的config⽂件中，修改CommonsChunkPlugin的配置

```
minChunks: 3
```

minChunks为3表⽰会把使⽤3次及以上的包抽离出来，放进公共依赖⽂件，避免了重复加载组件

### 图⽚资源的压缩

图⽚资源虽然不在编码过程中，但它却是对页⾯性能影响最⼤的因素
对于所有的图⽚资源，我们可以进⾏适当的压缩
对页⾯上使⽤到的icon，可以使⽤在线字体图标，或者雪碧图，将众多⼩图标合并到同⼀张图上，⽤以减轻http请求压⼒。
开启GZip压缩
拆完包之后，我们再⽤gzip做⼀下压缩 安装compression-webpack-plugin

```
cnmp i compression-webpack-plugin -D
```

在vue.config.js中引⼊并修改webpack配置

```
const CompressionPlugin = require('compression-webpack-plugin')

configureWebpack: (config) => {
        if (process.env.NODE_ENV === 'production') {
            // 为⽣产环境修改配置...
            config.mode = 'production'
            return {
                plugins: [new CompressionPlugin({
                    test: /\.js$|\.html$|\.css/, //匹配⽂件名
                    threshold: 10240, //对超过10k的数据进⾏压缩
                    deleteOriginalAssets: false //是否删除原⽂件
                })]
            }
        }
```

在服务器我们也要做相应的配置 如果发送请求的浏览器⽀持gzip，就发送给它gzip格式的⽂件 我的服务器是⽤express框架搭建的 只要安装
⼀下compression就能使⽤

```
const compression = require('compression')
app.use(compression())  // 在其他中间件使⽤之前调⽤
```

### 使⽤SSR

SSR（Server side ），也就是服务端渲染，组件或页⾯通过服务器⽣成html字符串，再发送到浏览器

从头搭建⼀个服务端渲染是很复杂的，vue应⽤建议使⽤Nuxt.js实现服务端渲染

⼩结：
减少⾸屏渲染时间的⽅法有很多，总的来讲可以分成两⼤部分 ：资源加载优化 和 页⾯渲染优化

参考链接：https://wenku.baidu.com/view/9c19c15602f69e3143323968011ca300a6c3f614.html

# 6.Vue中的nextTick有什么作用？

- 什么是nextTick
- 使用场景
- 实现原理

## 6.1什么是NextTick

官方定义：

> 在下次DOM更新循环结束之后执行延迟回调，在修改数据之后立即使用这个方法，获取更新后的DOM

Vue更新DOM是异步执行的。当数据发生变化，Vue将开启一个异步更新队列，视图需要等队列中所有数据变化完成之后，再统一进行更新

举例一下

`Html`结构

```text-html-basic
<div id="app"> {{ message }} </div>
```

构建一个`vue`实例

```source-js
const vm = new Vue({
  el: '#app',
  data: {
    message: '原始值'
  }
})
```

修改`message`

```source-js
this.message = '修改后的值1'
this.message = '修改后的值2'
this.message = '修改后的值3'
这时候想获取页面最新的DOM节点，却发现获取到的是旧值

console.log(vm.$el.textContent) // 原始值
```

这是因为message数据在发现变化的时候，vue并不会立刻去更新Dom，而是将修改数据的操作放在了一个异步操作队列中

如果我们一直修改相同数据，异步操作队列还会进行去重

等待同一事件循环中的所有数据变化完成之后，会将队列中的事件拿来进行处理，进行DOM的更新

## 6.2使用场景

如果想要在修改数据后立刻得到更新后的`DOM`结构，可以使用`Vue.nextTick()`

第一个参数为：回调函数（可以获取最近的`DOM`结构）

第二个参数为：执行函数上下文



```source-js
// 修改数据
vm.message = '修改后的值'
// DOM 还没有更新
console.log(vm.$el.textContent) // 原始的值
Vue.nextTick(function () {
  // DOM 更新了
  console.log(vm.$el.textContent) // 修改后的值
})
```

组件内使用 `vm.$nextTick()` 实例方法只需要通过`this.$nextTick()`，并且回调函数中的 `this` 将自动绑定到当前的 `Vue` 实例上

```source-js
this.message = '修改后的值'
console.log(this.$el.textContent) // => '原始的值'
this.$nextTick(function () {
    console.log(this.$el.textContent) // => '修改后的值'
})
```

`$nextTick()` 会返回一个 `Promise` 对象，可以是用`async/await`完成相同作用的事情

```source-js
this.message = '修改后的值'
console.log(this.$el.textContent) // => '原始的值'
await this.$nextTick()
console.log(this.$el.textContent) // => '修改后的值'
```

## 6.3 实现原理

源码位置：`/src/core/util/next-tick.js`

`callbacks`也就是异步操作队列

`callbacks`新增回调函数后又执行了`timerFunc`函数，`pending`是用来标识同一个时间只能执行一次。

```source-js
export function nextTick(cb?: Function, ctx?: Object) {
  let _resolve;

  // cb 回调函数会经统一处理压入 callbacks 数组
  callbacks.push(() => {
    if (cb) {
      // 给 cb 回调函数执行加上了 try-catch 错误处理
      try {
        cb.call(ctx);
      } catch (e) {
        handleError(e, ctx, 'nextTick');
      }
    } else if (_resolve) {
      _resolve(ctx);
    }
  });

  // 执行异步延迟函数 timerFunc
  if (!pending) {
    pending = true;
    timerFunc();
  }

  // 当 nextTick 没有传入函数参数的时候，返回一个 Promise 化的调用
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve;
    });
  }
}
```

`timerFunc`函数定义，这里是根据当前环境支持什么方法则确定调用哪个，分别有：

```
Promise.then`、`MutationObserver`、`setImmediate`、`setTimeout
```

通过上面任意一种方法，进行降级操作

```source-js
export let isUsingMicroTask = false
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  //判断1：是否原生支持Promise
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  //判断2：是否原生支持MutationObserver
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
  //判断3：是否原生支持setImmediate
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  //判断4：上面都不行，直接用setTimeout
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

无论是微任务还是宏任务，都会放到`flushCallbacks`使用

这里将`callbacks`里面的函数复制一份，同时`callbacks`置空

依次执行`callbacks`里面的函数

```source-js
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

# 7.说说对mixin的理解，以及有哪些应用场景

## 7.1、mixin是什么

Mixin是面向对象程序设计语言中的类，提供了方法的实现。其他类可以访问mixin类的方法而不必成为其子类

Mixin类通常作为功能模块使用，在需要该功能时“混入”，有利于代码复用又避免了多继承的复杂

### Vue中的mixin

先来看一下官方定义

> mixin（混入），提供了一种非常灵活的方式，来分发 Vue 组件中的可复用功能。

本质其实就是一个js对象，它可以包含我们组件中任意功能选项，如data、components、methods、created、computed等等

我们只要将共用的功能以对象的方式传入 mixins选项中，当组件使用 mixins对象时所有mixins对象的选项都将被混入该组件本身的选项中来

在Vue中我们可以**局部混入**跟**全局混入**

### 局部混入

定义一个mixin对象，有组件options的data、methods属性

```
var myMixin = {  created: function () {    this.hello()  },  methods: {    hello: function () {      console.log('hello from mixin!')    }  }}
```

组件通过mixins属性调用mixin对象

```
Vue.component('componentA',{  mixins: [myMixin]})
```

该组件在使用的时候，混合了mixin里面的方法，在自动执行create生命钩子，执行hello方法

### 全局混入

通过Vue.mixin()进行全局的混入

```
Vue.mixin({  created: function () {      console.log("全局混入")    }})
```

使用全局混入需要特别注意，因为它会影响到每一个组件实例（包括第三方组件）

PS：全局混入常用于插件的编写

### 注意事项：

当组件存在与mixin对象相同的选项的时候，进行递归合并的时候组件的选项会覆盖mixin的选项

但是如果相同选项为生命周期钩子的时候，会合并成一个数组，先执行mixin的钩子，再执行组件的钩子

## 7.2使用场景

在日常的开发中，我们经常会遇到在不同的组件中经常会需要用到一些相同或者相似的代码，这些代码的功能相对独立

这时，可以通过Vue的mixin功能将相同或者相似的代码提出来

举个例子

定义一个modal弹窗组件，内部通过isShowing来控制显示

```
const Modal = {  template: '#modal',  data() {    return {      isShowing: false    }  },  methods: {    toggleShow() {      this.isShowing = !this.isShowing;    }  }}
```

定义一个tooltip提示框，内部通过isShowing来控制显示

```
const Tooltip = {  template: '#tooltip',  data() {    return {      isShowing: false    }  },  methods: {    toggleShow() {      this.isShowing = !this.isShowing;    }  }}
```

通过观察上面两个组件，发现两者的逻辑是相同，代码控制显示也是相同的，这时候mixin就派上用场了

首先抽出共同代码，编写一个mixin

```
const toggle = {  data() {    return {      isShowing: false    }  },  methods: {    toggleShow() {      this.isShowing = !this.isShowing;    }  }}
```

两个组件在使用上，只需要引入mixin

```
const Modal = {  template: '#modal',  mixins: [toggle]}; const Tooltip = {  template: '#tooltip',  mixins: [toggle]}
```

通过上面小小的例子，让我们知道了Mixin对于封装一些可复用的功能如此有趣、方便、实用

## 7.3源码分析

首先从Vue.mixin入手

源码位置：/src/core/global-api/mixin.js

```
export function initMixin (Vue: GlobalAPI) {  Vue.mixin = function (mixin: Object) {    this.options = mergeOptions(this.options, mixin)    return this  }}
```

主要是调用merOptions方法

源码位置：/src/core/util/options.js

```
export function mergeOptions (  parent: Object,  child: Object,  vm?: Component): Object {if (child.mixins) { // 判断有没有mixin 也就是mixin里面挂mixin的情况 有的话递归进行合并    for (let i = 0, l = child.mixins.length; i < l; i++) {    parent = mergeOptions(parent, child.mixins[i], vm)    }}  const options = {}   let key  for (key in parent) {    mergeField(key) // 先遍历parent的key 调对应的strats[XXX]方法进行合并  }  for (key in child) {    if (!hasOwn(parent, key)) { // 如果parent已经处理过某个key 就不处理了      mergeField(key) // 处理child中的key 也就parent中没有处理过的key    }  }  function mergeField (key) {    const strat = strats[key] || defaultStrat    options[key] = strat(parent[key], child[key], vm, key) // 根据不同类型的options调用strats中不同的方法进行合并  }  return options}
```

从上面的源码，我们得到以下几点：

- 优先递归处理 mixins
- 先遍历合并parent 中的key，调用mergeField方法进行合并，然后保存在变量options
- 再遍历 child，合并补上 parent 中没有的key，调用mergeField方法进行合并，保存在变量options
- 通过 mergeField 函数进行了合并

下面是关于Vue的几种类型的合并策略

- 替换型
- 合并型
- 队列型
- 叠加型

### 替换型

替换型合并有props、methods、inject、computed

```
strats.props =strats.methods =strats.inject =strats.computed = function (  parentVal: ?Object,  childVal: ?Object,  vm?: Component,  key: string): ?Object {  if (!parentVal) return childVal // 如果parentVal没有值，直接返回childVal  const ret = Object.create(null) // 创建一个第三方对象 ret  extend(ret, parentVal) // extend方法实际是把parentVal的属性复制到ret中  if (childVal) extend(ret, childVal) // 把childVal的属性复制到ret中  return ret}strats.provide = mergeDataOrFn
```

同名的props、methods、inject、computed会被后来者代替

### 合并型

和并型合并有：data

```
strats.data = function(parentVal, childVal, vm) {        return mergeDataOrFn(        parentVal, childVal, vm    )};function mergeDataOrFn(parentVal, childVal, vm) {        return function mergedInstanceDataFn() {                var childData = childVal.call(vm, vm) // 执行data挂的函数得到对象        var parentData = parentVal.call(vm, vm)                if (childData) {                        return mergeData(childData, parentData) // 将2个对象进行合并                                         } else {                        return parentData // 如果没有childData 直接返回parentData        }    }}function mergeData(to, from) {        if (!from) return to        var key, toVal, fromVal;        var keys = Object.keys(from);       for (var i = 0; i < keys.length; i++) {        key = keys[i];        toVal = to[key];        fromVal = from[key];            // 如果不存在这个属性，就重新设置        if (!to.hasOwnProperty(key)) {            set(to, key, fromVal);        }              // 存在相同属性，合并对象        else if (typeof toVal =="object" && typeof fromVal =="object") {            mergeData(toVal, fromVal);        }    }        return to}
```

mergeData函数遍历了要合并的 data 的所有属性，然后根据不同情况进行合并：

- 当目标 data 对象不包含当前属性时，调用 set 方法进行合并（set方法其实就是一些合并重新赋值的方法）
- 当目标 data 对象包含当前属性并且当前值为纯对象时，递归合并当前对象值，这样做是为了防止对象存在新增属性

### 队列性

队列性合并有：全部生命周期和watch

```
function mergeHook (  parentVal: ?Array<Function>,  childVal: ?Function | ?Array<Function>): ?Array<Function> {  return childVal    ? parentVal      ? parentVal.concat(childVal)      : Array.isArray(childVal)        ? childVal        : [childVal]    : parentVal}LIFECYCLE_HOOKS.forEach(hook => {  strats[hook] = mergeHook})// watchstrats.watch = function (  parentVal,  childVal,  vm,  key) {  // work around Firefox's Object.prototype.watch...  if (parentVal === nativeWatch) { parentVal = undefined; }  if (childVal === nativeWatch) { childVal = undefined; }  /* istanbul ignore if */  if (!childVal) { return Object.create(parentVal || null) }  {    assertObjectType(key, childVal, vm);  }  if (!parentVal) { return childVal }  var ret = {};  extend(ret, parentVal);  for (var key$1 in childVal) {    var parent = ret[key$1];    var child = childVal[key$1];    if (parent && !Array.isArray(parent)) {      parent = [parent];    }    ret[key$1] = parent      ? parent.concat(child)      : Array.isArray(child) ? child : [child];  }  return ret};
```

生命周期钩子和watch被合并为一个数组，然后正序遍历一次执行

### 叠加型

叠加型合并有：component、directives、filters

```
strats.components=strats.directives=strats.filters = function mergeAssets(    parentVal, childVal, vm, key) {        var res = Object.create(parentVal || null);        if (childVal) {         for (var key in childVal) {            res[key] = childVal[key];        }       }     return res}
```

叠加型主要是通过原型链进行层层的叠加

### 小结：

- 替换型策略有props、methods、inject、computed，就是将新的同名参数替代旧的参数
- 合并型策略是data, 通过set方法进行合并和重新赋值
- 队列型策略有生命周期函数和watch，原理是将函数存入一个数组，然后正序遍历依次执行
- 叠加型有component、directives、filters，通过原型链进行层层的叠加

# 8.说说你对slot的理解？使用场景有哪些？

## 8.1 什么是slot

在HTML中slot元素，作为Web Components技术套件的一部分，是Web组件内的一个占位符

该占位符可以在后期使用自己的标记语言填充

举个栗子

```
<template id="element-details-template">
	<slot name="element-name">Slot template</slot>
</template>
<element-details>
	<span slot="element-name">1</span>
</element-details>
<element-details>
	<span slot="element-name">2</span>
</element-details>
```

template不会展示到页面中，需要用先获取它的引用，然后添加到DOM中

```
customElements.define('element-details',
	class extends HTMLElement{
		constructor(){
			super();
			const template = document
				.getElementById('element-details-template')
				.content;
			const shadowRoot = this.attachShadow({mode:'open'})
				.appendChild(template.cloneNode(true))			  										
	}
})
```


在Vue中的概念也是如此

Slot艺名插槽，花名"占坑"，我们可以理解为slot在组件模板中占好了位置，当使用该组件标签时候，组件标签里面的内容就会自动填充（替换组件模板中slot位置），作为承载分发内容的出口

可以将其类比为插卡式的FC游戏机，游戏机暴露卡槽（插槽）让用户插入不同的游戏磁条（自定义内容）

## 8.2使用场景

- 通过插槽可以让用户可以拓展组件，去更好地复用组件和对其做定制化处理
- 如果父组件在使用到一个复用组件的时候，获取这个组件在不同地方有少量的更改，如果去重写组件是一件不明智的事情

- 通过slot插槽向组件内部指定位置传递内容，完成这个复用组件在不同场景的应用


​      比如布局组件、表格列、下拉选、弹框显示内容等

slot可以分为一下三种：

- 默认插槽

- 具名插槽
- 作用域插槽
- 默认插槽

子组件用< slot >标签来确定渲染位置，标签里面可以放DOM结构，当父组件使用的时候没有往插槽传入内容，标签内DOM结构就会显示在页面

父组件在使用的时候，直接在子组件的标签内写入内容即可

子组件Child.vue

<template>
	<slot>
		<p>插槽后备的容</p>
	</slot>
</template>

父组件

```
<Child>
	<div>默认插槽</div>
</Child>
```

**具名插槽**
子组件用name属性来表示插槽的名字，不传为默认插槽

父组件中在使用时在默认插槽的基础上加上slot属性，值为子组件插槽name属性值
子组件Child.vue

<template>
	<slot>插槽后备的内容</slot>
	<slot name="content">插槽后备的内容</slot>
</tamplate>

```
<Child>
	<template v-slot:default>具名插槽
</template>
	<!--具名插槽 插槽名做参数-->
	<template v-slot:content>内容....</template>
</Child>
```

**作用域插槽**
子组件在作用域上绑定属性来将自组件的信息传给父组件使用，这些属性会被挂在父组件v-slot接收的对象上

父组件中在使用时通过 v-slot: (简写：#)获取子组件的信息，在内容中使用

子组件Child.vue

<template>
	<slot name="footer" testProps="子组件的值">
		<h3>没传footer插槽</h3>
</slot>
</template>
父组件

```
<Child>
	<!--把v-slot的值指定为作用域上下文对象-->
	<template v-slot:default="slotProps">
		来自子组件数据:{{slotProps.testProps}}
</template>
	<template #default="slotProps">
		来自子组件数据:{{slotProps.testProps}}
</template>
</Child>

```

**小结：**

- v-slot属性只能在< template >上使用，但在只有默认插槽时可以在组加标签上使用

- 默认插槽名为 default ,可以省略 default 直接写 v-slot
- 缩写为 # 时不能不写参数，写成 #default
- 可以通过解构获取 v-slot={user} ,还可以重命名 v-slot="{user:newName}" 和定义默认值 v-slot="{user = ‘默认值’}"

## 8.3原理分析

slot 本质上时返回 VNode 的函数，一般情况下， Vue 中的组件要渲染到页面上需要经过 template -> render function -> VNode -> DOM 过程，这里看看 slot 如何实现：

编写一个 buttonCounter 组件，使用匿名插槽

```
Vue.component('button-counter',{
	template:'<div><slot>我是默认内容</slot></div>'
})
```

使用该组件

```
new Vue({
	el:"#app",
	template:"<button-counter><span>我是slot传入的内容</span></button-counter>",
	components:{buttonCounter}
})
```

获取 buttonCounter 组件渲染函数

```
(function anonymous){
	with(this){return _c('div',[_t("default",[_v('我是默认内容')])],2)}
}
_v 表示创建普通文本节点，_
```

t 表示渲染插槽的函数

渲染插槽函数 renderSlot (做了简化)

```
function renderSlot(name,fallback,props,bindObject){
	//得到渲染插槽内容的函数
	var scopedSlotFn = this.$scopedSlots[name];
	var nodes;
	//如果存在插槽渲染函数，则执行插槽渲染函数，生成nodes节点返回
	//否则使用默认值
	nodes = scopedSlotFn(props) || fallback;
	return nodes;
}
```

name 属性表示定义插槽的名字，默认值为 default ,fallback 表示子组件中的 slot 节点的默认值

关于 this.$scopedSlots 是什么，我们可以先看看 vm.slot

```
function initRender(vm){
	...
	vm.$slots = resolveSlots(options._renderChildren,renderContext);
}
```

resolveSlots 函数会对 children 节点做归类和过滤处理，返回 slots

```
function resolveSlots(children,context){
	if(!children || !children.length){
		return {}
	}
	var slots = {};
	for(var i = 0,l = children.length;i < l;i++){
		var child = children[];
		var data = child.data;
		//remove slot attribute if the node is resolved as a Vue slot node
		if(data && data.attrs && data.attrs.slot){
			delete data.attrs.slot;
		}
		//named slots should noly be respected if the vnode was rendered in the same context
		if((child.context === context || child.fnContext === context) && data && data.slot != null){
			//如果slot存在(slot="header")则拿对应的值作为key
			var name = data.slot;
			var slot = (slots[name] || (slots[name] = []));
			//如果是template元素，则把template的children添加进数组中，这也就是为什么你写的template标签并不会被渲染成另一个标签到页面
			if(child.tag === 'template'){
				slot.push.apply(slot, child.children || [])
			}else{
				slot.push(child)
			}
		}else{
			//如果没有就默认是default
			(slots.default || (slots.default = [])).push(child);
		}
	}
	//ignore slots that contains only whitespace
	for(var name$1 in slots){
		if(slots[name$1].every(isWhitespace)){
			delete slots[name$1]
		}
	}
	return slots
}
```

_render 渲染函数通过 normalizeScopedSlots 得到 vm.$scopedSlots

```
vm.$scopedSlots = normalizeScopedSlots(
	_parenVnode.data.scopedSlots,
	vm.$slots,
	vm.$scopedSlots
)
```

作用域插槽中父组件能够得到子组件的值是因为在 renderSlot 的时候执行会传入 props ,也就是上述 _t 第三个参数，父组件则能够得到子组件传递过来的值



# 9.Vue.observable是什么？

observable翻译过来可以理解成 可观察的

## vue 中的定义

返回的对象可以直接用于渲染函数和计算属性内，并且会在发生变更时触发相应的更新。也可以作为最小化的跨组件状态存储器，用于简单的场景：

```
Vue.observable({ count : 1})
```

## 使用场景

在非父子组件通信是， 通常可以使用vuex 或者bus 但是使用的功能不太复杂 使用这两个又有点繁琐，这是observable 就是个一个很好的选择

创建一个js文件

```
import Vue from ‘vue’
export const state = Vue。observable({
	user:{name:'龚'}
})

export const mutations = {
	updateUser(payload) {
		state.user = Object.assign({},payload)	
	}
}
```

在页面中使用


<template>
  <div>
    <button @click="changeUser">改变用户</button>
  </div>
</template>
```
import { state, mutations } from '../store.js'
export default {
```

	computed:{
		user(){
			return state.user
		}
	},
	
	methods:{
		// 调用murations 中的方法更新数据
		changeUser(){
			mutations.update.user({name:'晔'})
		}	
	}
	
	}
	
# 10.说说Vue中,key的原理

## 1.key值的作用

> 主要用于v-for语法中

- 主要用在 Vue 的虚拟 DOM 算法，在新旧 nodes 对比时辨识 VNodes，相当于唯一标识ID
- Vue 会尽可能高效地渲染元素，通常会复用已有元素而不是从头开始渲染， 因此使用key值可以提高渲染效率，同理，改变某一元素的key值会使该元素重新被渲染
- vue中在使用相同标签名元素的过渡切换时，也会使用到key属性，其目的也是为了让vue可以区分它们，否则vue只会替换其内部属性而不会触发过渡效果

## 2.diff算法部分代码

通过sameVnode方法来判断vNode和oldNode是否是同一个节点，主要通过key值来判断

```jsx
 // src\core\vdom\patch.js
function sameVnode (a, b) {
    return (
        // 判断a, b两个Vnode上的key值是否相等
        a.key === b.key && (
            (
                a.tag === b.tag &&
                a.isComment === b.isComment &&
                isDef(a.data) === isDef(b.data) &&
                sameInputType(a, b)
            ) || (
                isTrue(a.isAsyncPlaceholder) &&
                a.asyncFactory === b.asyncFactory &&
                isUndef(b.asyncFactory.error)
            )
        )
    )
}
```

# 11.说说Vue中CSS scoped的原理

## 1 什么是 Scope CSS

Scope CSS 即作用域 CSS，组件化所密不可分的一部分。Scope CSS 使得我们可以在各组件中定义的 CSS 不产生污染。例如，我们在 Vue 中定义一个组件：

```
scoped css
```

通常情况下，在开发环境我们的组件会先经过 vue-loader 的处理，然后结合运行时的框架代码渲染到页面上。相应地，它们对应的 HTML 和 CSS 分别会是这样：

HTML 部分：

```
scoped css
```

CSS 部分：

```
.box[data-v-992092a6] {
  width: 200px;
  height: 200px;
  background: #aff;
}
```

可以看到 Scope CSS 的本质是基于 HTML 和 CSS 选择器的属性，通过分别给 HTML 标签和 CSS 选择器添加 data-v-xxxx 属性的方式实现。

## 2 vue-loader 处理组件（.vue 文件）

前面，我们也提及了在开发环境下一个组件（.vue 文件）会先由 vue-loader 来处理。那么，针对 Scope CSS 而言，vue-loader 会做这 3 件事：

- 解析组件，提取出 template、script、style 对应的代码块
- 构造并导出 export 组件实例，在组件实例的选项上绑定 ScopId
- 对 style 的 CSS 代码进行编译转化，应用 ScopId 生成选择器的属性

然而，之所以 vue-loader 有这么多的能力，主要是因为 vue-loader 的底层使用了 Vue 官方提供的包（package） @vue/component-compiler-utils，其提供了解析组件（.vue 文件）、编译模版 template、编译 style等 3 种能力。

那么，下面我们就先来看一下 vue-loader 是如何使用 @vue/component-compiler-utils 来解析组件提取 template、script、style 的。

### 2.1 提取 template、script、style

vue-loader 提取 template、script、style 的过程主要是使用了 @vue/component-compiler-utils 包的 parse 方法，这个过程对应的代码（伪代码）会是这样：

// vue-loader/lib/index.js
const { parse } = require("@vue/component-compiler-utils");

module.exports = function (source) {
  const loaderContext = this;
  const { sourceMap, rootContext, resourcePath } = loaderContext;
  const sourceRoot = path.dirname(path.relative(context, resourcePath));
  const descriptor = parse({
    source,
    compiler: require("vue-template-compiler"),
    filename,
    sourceRoot,
    needMap: sourceMap,
  });
};
我们来逐点分析一下这段代码，首先，会获取当前上下文 loaderContext，它会包含 webpack 打包过程核心对象 compiler、compilation 等。

其次，再构建文件资源入口 sourceRoot，一般情况下它指的是 src 文件目录，它主要用于构建 source-map 使用。

最后，则会使用 @vue/component-compiler-utils 提供的 parse 方法来解析 source（组件代码）。这里，我们先来看一下 parse 方法的几个参数：

- soruce 源代码块，这里是组件对应的代码，即包含了 template、style、script
- compiler 编译核心对象，它是一个 CommonJS 模块（vue-template-compiler），parse 方法内部会使用它提供的 parseComponent 方法来解析组件
- filename 当前组件的文件名，例如 App.vue
- sourceRoot 文件资源入口，用于构建 source-map 使用
- needMap 是否需要 source-map，parse 方法内部会根据 needMap 的值（true 或 false，默认为 true）来判断是否生成 script、style 对应的 source-map
  而 parse 方法的执行则会返回一个对象给 desciptor，它会包含 template、style、script 分别对应的代码块。

那么，可以看到的是 vue-loader 解析组件的过程，几乎外包给了 Vue 提供的工具包（package）。并且，我想这个时候肯定会有同学问：这些和 Vue 的 Scope CSS 有几毛钱关系 ????️？

有很大的关系！因为 Vue 的 Scope CSS 可不是无米之炊，它实现的前提是组件被解析了，然后再分别处理 template 和 style 部分的代码！

那么，显然到这里我们已经完成了对组件的解析。接着，则需要构造和导出组件实例～

### 2.2 构造和导出组件实例

vue-loader 在解析完组件后，会分别处理并生成 template、script、style 的导入 import 语句，再调用 normalizer 方法正常化（normalizer）组件，最后将它们拼接成代码字符串：

```
let templateImport = `var render, staticRenderFns`;
if (descriptor.template) {
  // 构造 template 的 import 语句
}
let scriptImport = `var script = {}`;
if (descriptor.script) {
  // 构造 script 的 import 语句
}
let stylesCode = ``;
if (descriptor.styles.length) {
  // 构造 style 的 import 语句
}
let code =
  `
${templateImport}
${scriptImport}
${stylesCode}

import normalizer from ${stringifyRequest(`!${componentNormalizerPath}`)}

var component = normalizer(
  script,
  render,
  staticRenderFns,
  ${hasFunctional ? `true` : `false`},
  ${/injectStyles/.test(stylesCode) ? `injectStyles` : `null`},
  ${hasScoped ? JSON.stringify(id) : `null`},
  ${isServer ? JSON.stringify(hash(request)) : `null`}
  ${isShadow ? `,true` : ``}
)

  `.trim() + `\n`;
```

其中，templateImport、scriptImport、stylesCode 等构造好的 template、script、style 部分的导入 import 语句看起来会是这样：

```
import {
  render,
  staticRenderFns,
} from "./App.vue?vue&type=template&id=7ba5bd90&scoped=true&";
import script from "./App.vue?vue&type=script&lang=js&";
// 兼容命名方式的导出
export * from "./App.vue?vue&type=script&lang=js&";
import style0 from "./App.vue?vue&type=style&index=0&id=7ba5bd90&scoped=true&lang=css&";
```

不知道同学们注意 ⚠️ 到没，template 和 style 的导入 import 语句都有这么一个共同的部分 id=7ba5bd90&scoped=true，这表示此时组件的 template 和 style 是需要 Scope CSS 的，并且 scopeId 为 7ba5bd90。

当然，这仅仅是告知后续的 template 和 style 编译时需要注意生成 Scope CSS，也就是 Scope CSS 的第一步！那么，接着则会调用 normalizer 方法来对该组件进行正常化（Normalizer）处理：

```
import normalizer from "!../node_modules/vue-loader/lib/runtime/componentNormalizer.js";
var component = normalizer(
  script,
  render,
  staticRenderFns,
  false,
  null,
  "7ba5bd90",
  null
);
export default component.exports;
注意，normalizer 是重命名了原方法 normalizeComponent，以下统称 normalizeComponent~

我想同学们应该都注意到了，此时 scopeId 会作为参数传给 normalizeComponent 方法，而传给 normalizeComponent 的目的则是为了在组件实例的 options 上绑定 scopeId。那么，我们来看一下 normalizeComponent 方法（伪代码）：

function normalizeComponent (
  scriptExports,
  render,
  staticRenderFns,
  functionalTemplate,
  injectStyles,
  scopeId,
  moduleIdentifier, /* server only */
  shadowMode /* vue-cli only */
) {
  ...
  var options = typeof scriptExports === 'function'
    ? scriptExports.options
    : scriptExports
  // scopedId
  if (scopeId) {
    options._scopeId = 'data-v-' + scopeId
  }
  ...

}
```

可以看到，这里的 options._scopeId 会等于 data-v-7ba5bd90，而它的作用主要是用于在 patch 的时候，为当前组件的 HTML 标签添加名为 data-v-7ba5bd90 的属性。因此，这也是 template 为什么会形成带有 scopeId 的真正所在！

### 2.3 编译样式 Style，应用 ScopId 生成选择器的属性

在构造完 Style 对应的导入语句后，由于此时 import 语句中的 query 包含 vue，则会被 vue-loader 内部的 Pitching Loader 处理。而 Pitching Loader 则会重写 import 语句，拼接上内联（inline）的 Loader，这看起来会是这样：

```
export * from '
"-!../node_modules/vue-style-loader/index.js??ref--6-oneOf-1-0
!../node_modules/css-loader/dist/cjs.js??ref--6-oneOf-1-1
!../node_modules/vue-loader/lib/loaders/stylePostLoader.js
!../node_modules/postcss-loader/src/index.js??ref--6-oneOf-1-2
!../node_modules/cache-loader/dist/cjs.js??ref--0-0
!../node_modules/vue-loader/lib/index.js??vue-loader-options!./App.vue?vue&type=style&index=0&id=7ba5bd90&scoped=true&lang=css&"
'
```

然后，webpack 会解析出模块所需要的 Loader，显然这里会解析出 6 个 Loader：

```
[
  { loader: "vue-style-loader", options: "?ref--6-oneOf-1-0" },
  { loader: "css-loader", options: "?ref--6-oneOf-1-1" },
  { loader: "stylePostLoader", options: undefined },
  { loader: "postcss-loader", options: "?ref--6-oneOf-1-2" },
  { loader: "cache-loader", options: "?ref--0-0" },
  { loader: "vue-loader", options: "?vue-loader-options" }
]
```

那么，此时 webpack 则会执行这 6 个 Loader（当然还有解析模块本身）。并且，这里会忽略 webpack.config.js 中符合规则的 Normal Loader（vue-style-loader 还会忽略前置 Loader）。

不了解内联 Loader 的同学，可以看一下这篇文章【webpack进阶】你真的掌握了loader么？- loader十问

而对于 Scope CSS 而言，最核心的就是 stylePostLoader。下面，我们来看一下 stylePostLoader 的定义：

```
const { compileStyle } = require("@vue/component-compiler-utils");
module.exports = function (source, inMap) {
  const query = qs.parse(this.resourceQuery.slice(1));
  const { code, map, errors } = compileStyle({
    source,
    filename: this.resourcePath,
    id: `data-v-${query.id}`,
    map: inMap,
    scoped: !!query.scoped,
    trim: true,
  });

  if (errors.length) {
    this.callback(errors[0]);
  } else {
    this.callback(null, code, map);
  }
};
```

从 stylePostLoader 的定义中，我们知道它是使用了 @vue/component-compiler-utils 提供的 compileStyle 方法来完成对组件 style 的编译。并且，此时会传入参数 id 为 data-v-${query.id}，即 data-v-7ba5bd90，而这也是 style 中声明的选择器的属性为 scopeId 的关键点！

而在 compileStyle 函数内部，则是使用的我们所熟知 postcss 来完成对 style 代码的编译和构造选择器的 scopeId 属性。至于如何使用 postcss 完成这个过程，这里就不做过多介绍，有兴趣的同学自行了解哈～

## 3 Patch 阶段应用 ScopeId 生成 HTML 的属性

不知道同学们是否还记得在 3.2 构造并导出组件实例的时候，我们讲了在组件实例的 options 上绑定 _scopeId 是实现 template 的 Scope 的关键点！但是，当时我们并没有介绍这个 _scopeId 到底是如何应用到 template 上的元素的 ????？

如果，你想在 vue-loader 或者 @vue/component-compiler-utils 的代码中找到这个答案，我可以和你说找一万年都找不到！ 因为，真正应用 _scopeId 的过程是发生在 Vue 运行时的框架代码中（没想到吧 ????）。

了解过 Vue 模版编译过程的同学，我想应该都知道 template 会被编译成 render 函数，然后会根据 render 函数创建对应的 VNode，最后再由 VNode 渲染成真实的 DOM 在页面上：

而 VNode 到真实 DOM 这个过程是由 patch 方法完成的。假设，此时我们是第一次渲染 DOM，这在 patch 方法中会命中 isUndef(oldVnode) 为 true 的逻辑：

```
function patch (oldVnode, vnode, hydrating, removeOnly) {
  if (isUndef(oldVnode)) {
    // empty mount (likely as component), create new root element
    isInitialPatch = true
    createElm(vnode, insertedVnodeQueue)
  } 
}
```

因为第一次渲染 DOM，所以压根不存在什么 oldVnode ????

可以看到，此时会执行 createElm 方法。而在 createElm 方法中则会创建 VNode 对应的真实 DOM，并且它还做了一件很重要的事，调用 setScope 方法应用 _scopeId 在 DOM 上生成 data-v-xxx 的属性！对应的代码（伪代码）：

```
// packages/src/core/vdom/patch.js
function createElm(
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  ...
  setScope(vnode);
  ...
}
```

在 setScope 方法中则会使用组件实例的 options._scopeId 作为属性来添加到 DOM 上，从而生成了 template 中的 HTML 标签上名为 data-v-xxx 的属性。并且，这个过程会由 Vue 封装好的工具函数 nodeOps.setStyleScope 完成，它的本质是调用 DOM 对象的 setAttribute 方法：

```
// src/platforms/web/runtime/node-ops.js
export function setStyleScope (node: Element, scopeId: string) {
  node.setAttribute(scopeId, '')
}
```

结语
如果，有在网上查找过关于 vue-loader 和 Scope CSS 的文章的同学会发现，很多文章都是说在 @vue/component-compiler-utils 包的 compilerTemplate 方法中应用 scopeId 生成了 template 中 HTML 标签的属性。但是，通过阅读本文，我们会发现这两者压根没有任何关系（SSR 的情况除外）！

并且，我想同学们也注意到了一点，本文中提及的 Vue 运行时框架的代码是 Vue 2.x 版本的（不是 Vue3）。所以，有兴趣的同学可以借助本文提供的路线推敲一下 Vue3 中 Scope CSS 的过程，相信你会收获满满。



# 12.vue模板是如何编译的

## 写在开头

写过 Vue 的同学肯定体验过， `.vue` 这种单文件组件有多么方便。但是我们也知道，Vue 底层是通过虚拟 DOM 来进行渲染的，那么 `.vue` 文件的模板到底是怎么转换成虚拟 DOM 的呢？这一块对我来说一直是个黑盒，之前也没有深入研究过，今天打算一探究竟。

![Virtual Dom](https://file.shenfq.com/ipic/2020-08-19-032238.jpg)

Vue 3 发布在即，本来想着直接看看 Vue 3 的模板编译，但是我打开 Vue 3 源码的时候，发现我好像连 Vue 2 是怎么编译模板的都不知道。从小鲁迅就告诉我们，不能一口吃成一个胖子，那我只能回头看看 Vue 2 的模板编译源码，至于 Vue 3 就留到正式发布的时候再看。

## Vue 的版本

很多人使用 Vue 的时候，都是直接通过 vue-cli 生成的模板代码，并不知道 Vue 其实提供了两个构建版本。

- `vue.js`： 完整版本，包含了模板编译的能力；
- `vue.runtime.js`： 运行时版本，不提供模板编译能力，需要通过 vue-loader 进行提前编译。

![Vue不同构建版本](https://file.shenfq.com/ipic/2020-08-19-033601.png)

![完整版与运行时版区别](https://file.shenfq.com/ipic/2020-08-19-033941.png)

简单来说，就是如果你用了 vue-loader ，就可以使用 `vue.runtime.min.js`，将模板编译的过程交过 vue-loader，如果你是在浏览器中直接通过 `script` 标签引入 Vue，需要使用 `vue.min.js`，运行的时候编译模板。

## 编译入口

了解了 Vue 的版本，我们看看 Vue 完整版的入口文件（`src/platforms/web/entry-runtime-with-compiler.js`）。

```js
// 省略了部分代码，只保留了关键部分
import { compileToFunctions } from './compiler/index'

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (el) {
  const options = this.$options
  
  // 如果没有 render 方法，则进行 template 编译
  if (!options.render) {
    let template = options.template
    if (template) {
      // 调用 compileToFunctions，编译 template，得到 render 方法
      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      // 这里的 render 方法就是生成生成虚拟 DOM 的方法
      options.render = render
    }
  }
  return mount.call(this, el, hydrating)
}
复制代码
```

再看看 `./compiler/index` 文件的 `compileToFunctions` 方法从何而来。

```js
import { baseOptions } from './options'
import { createCompiler } from 'compiler/index'

// 通过 createCompiler 方法生成编译函数
const { compile, compileToFunctions } = createCompiler(baseOptions)
export { compile, compileToFunctions }
复制代码
```

后续的主要逻辑都在 `compiler` 模块中，这一块有些绕，因为本文不是做源码分析，就不贴整段源码了。简单看看这一段的逻辑是怎么样的。

```js
export function createCompiler(baseOptions) {
  const baseCompile = (template, options) => {
    // 解析 html，转化为 ast
    const ast = parse(template.trim(), options)
    // 优化 ast，标记静态节点
    optimize(ast, options)
    // 将 ast 转化为可执行代码
    const code = generate(ast, options)
    return {
      ast,
      render: code.render,
      staticRenderFns: code.staticRenderFns
    }
  }
  const compile = (template, options) => {
    const tips = []
    const errors = []
    // 收集编译过程中的错误信息
    options.warn = (msg, tip) => {
      (tip ? tips : errors).push(msg)
    }
    // 编译
    const compiled = baseCompile(template, options)
    compiled.errors = errors
    compiled.tips = tips

    return compiled
  }
  const createCompileToFunctionFn = () => {
    // 编译缓存
    const cache = Object.create(null)
    return (template, options, vm) => {
      // 已编译模板直接走缓存
      if (cache[template]) {
        return cache[template]
      }
      const compiled = compile(template, options)
    	return (cache[key] = compiled)
    }
  }
  return {
    compile,
    compileToFunctions: createCompileToFunctionFn(compile)
  }
}
复制代码
```

## 主流程

可以看到主要的编译逻辑基本都在 `baseCompile` 方法内，主要分为三个步骤：

1. 模板编译，将模板代码转化为 AST；
2. 优化 AST，方便后续虚拟 DOM 更新；
3. 生成代码，将 AST 转化为可执行的代码；

```js
const baseCompile = (template, options) => {
  // 解析 html，转化为 ast
  const ast = parse(template.trim(), options)
  // 优化 ast，标记静态节点
  optimize(ast, options)
  // 将 ast 转化为可执行代码
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}
复制代码
```

### parse

#### AST

首先看到 parse 方法，该方法的主要作用就是解析 HTML，并转化为 AST（抽象语法树），接触过 ESLint、Babel 的同学肯定对 AST 不陌生，我们可以先看看经过 parse 之后的 AST 长什么样。

下面是一段普普通通的 Vue 模板：

```js
new Vue({
  el: '#app',
  template: `
    <div>
      <h2 v-if="message">{{message}}</h2>
      <button @click="showName">showName</button>
    </div>
  `,
  data: {
    name: 'shenfq',
    message: 'Hello Vue!'
  },
  methods: {
    showName() {
      alert(this.name)
    }
  }
})
复制代码
```

经过 parse 之后的 AST：

![Template AST](https://file.shenfq.com/ipic/2020-08-19-063252.png)

AST 为一个树形结构的对象，每一层表示一个节点，第一层就是 `div`（`tag: "div"`）。`div` 的子节点都在 children 属性中，分别是 `h2` 标签、空行、`button` 标签。我们还可以注意到有一个用来标记节点类型的属性：type，这里 `div` 的 type 为 1，表示是一个元素节点，type 一共有三种类型：

1. 元素节点；
2. 表达式；
3. 文本；

在 `h2` 和 `button` 标签之间的空行就是 type 为 3 的文本节点，而 `h2` 标签下就是一个表达式节点。

![节点类型](https://file.shenfq.com/ipic/2020-08-19-065127.png)

#### 解析HTML

parse 的整体逻辑较为复杂，我们可以先简化一下代码，看看 parse 的流程。

```js
import { parseHTML } from './html-parser'

export function parse(template, options) {
  let root
  parseHTML(template, {
    // some options...
    start() {}, // 解析到标签位置开始的回调
    end() {}, // 解析到标签位置结束的回调
    chars() {}, // 解析到文本时的回调
    comment() {} // 解析到注释时的回调
  })
  return root
}
复制代码
```

可以看到 parse 主要通过 parseHTML 进行工作，这个 parseHTML 本身来自于开源库：[htmlparser.js](https://link.juejin.cn?target=https%3A%2F%2Fjohnresig.com%2Ffiles%2Fhtmlparser.js)，只不过经过了 Vue 团队的一些修改，修复了相关 issue。

![HTML parser](https://file.shenfq.com/ipic/2020-08-19-065917.png)

下面我们一起来理一理 parseHTML 的逻辑。

```js
export function parseHTML(html, options) {
  let index = 0
  let last,lastTag
  const stack = []
  while(html) {
    last = html
    let textEnd = html.indexOf('<')

    // "<" 字符在当前 html 字符串开始位置
    if (textEnd === 0) {
      // 1、匹配到注释: <!-- -->
      if (/^<!\--/.test(html)) {
        const commentEnd = html.indexOf('-->')
        if (commentEnd >= 0) {
          // 调用 options.comment 回调，传入注释内容
          options.comment(html.substring(4, commentEnd))
          // 裁切掉注释部分
          advance(commentEnd + 3)
          continue
        }
      }

      // 2、匹配到条件注释: <![if !IE]>  <![endif]>
      if (/^<!\[/.test(html)) {
        // ... 逻辑与匹配到注释类似
      }

      // 3、匹配到 Doctype: <!DOCTYPE html>
      const doctypeMatch = html.match(/^<!DOCTYPE [^>]+>/i)
      if (doctypeMatch) {
        // ... 逻辑与匹配到注释类似
      }

      // 4、匹配到结束标签: </div>
      const endTagMatch = html.match(endTag)
      if (endTagMatch) {}

      // 5、匹配到开始标签: <div>
      const startTagMatch = parseStartTag()
      if (startTagMatch) {}
    }
    // "<" 字符在当前 html 字符串中间位置
    let text, rest, next
    if (textEnd > 0) {
      // 提取中间字符
      rest = html.slice(textEnd)
      // 这一部分当成文本处理
      text = html.substring(0, textEnd)
      advance(textEnd)
    }
    // "<" 字符在当前 html 字符串中不存在
    if (textEnd < 0) {
      text = html
      html = ''
    }
    
    // 如果存在 text 文本
    // 调用 options.chars 回调，传入 text 文本
    if (options.chars && text) {
      // 字符相关回调
      options.chars(text)
    }
  }
  // 向前推进，裁切 html
  function advance(n) {
    index += n
    html = html.substring(n)
  }
}
复制代码
```

上述代码为简化后的 parseHTML，`while` 循环中每次截取一段 html 文本，然后通过正则判断文本的类型进行处理，这就类似于编译原理中常用的有限状态机。每次拿到 `"<"` 字符前后的文本，`"<"` 字符前的就当做文本处理，`"<"` 字符后的通过正则判断，可推算出有限的几种状态。

![html的几种状态](https://file.shenfq.com/ipic/2020-08-19-120759.png)

其他的逻辑处理都不复杂，主要是开始标签与结束标签，我们先看看关于开始标签与结束标签相关的正则。

```js
const ncname = '[a-zA-Z_][\\w\\-\\.]*'
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const startTagOpen = new RegExp(`^<${qnameCapture}`)
复制代码
```

这段正则看起来很长，但是理清之后也不是很难。这里推荐一个[正则可视化工具](https://link.juejin.cn?target=https%3A%2F%2Fjex.im%2Fregulex%2F)。我们到工具上看看startTagOpen：

![startTagOpen](https://file.shenfq.com/ipic/2020-08-19-122932.png)

这里比较疑惑的点就是为什么 tagName 会存在 `:`，这个是 XML 的 [命名空间](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3school.com.cn%2Fxml%2Fxml_namespaces.asp)，现在已经很少使用了，我们可以直接忽略，所以我们简化一下这个正则：

```js
const ncname = '[a-zA-Z_][\\w\\-\\.]*'
const startTagOpen = new RegExp(`^<${ncname}`)
const startTagClose = /^\s*(\/?)>/
const endTag = new RegExp(`^<\\/${ncname}[^>]*>`)
复制代码
```

![startTagOpen](https://file.shenfq.com/ipic/2020-08-19-123411.png)

![endTag](https://file.shenfq.com/ipic/2020-08-19-123859.png)

除了上面关于标签开始和结束的正则，还有一段用来提取标签属性的正则，真的是又臭又长。

```js
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
复制代码
```

把正则放到工具上就一目了然了，以 `=` 为分界，前面为属性的名字，后面为属性的值。

![attribute](https://file.shenfq.com/ipic/2020-08-19-124918.png)

理清正则后可以更加方便我们看后面的代码。

```js
while(html) {
  last = html
  let textEnd = html.indexOf('<')

  // "<" 字符在当前 html 字符串开始位置
  if (textEnd === 0) {
    // some code ...

    // 4、匹配到标签结束位置: </div>
    const endTagMatch = html.match(endTag)
    if (endTagMatch) {
      const curIndex = index
      advance(endTagMatch[0].length)
      parseEndTag(endTagMatch[1], curIndex, index)
      continue
    }

    // 5、匹配到标签开始位置: <div>
    const startTagMatch = parseStartTag()
    if (startTagMatch) {
      handleStartTag(startTagMatch)
      continue
    }
  }
}
// 向前推进，裁切 html
function advance(n) {
  index += n
  html = html.substring(n)
}

// 判断是否标签开始位置，如果是，则提取标签名以及相关属性
function parseStartTag () {
  // 提取 <xxx
  const start = html.match(startTagOpen)
  if (start) {
    const [fullStr, tag] = start
    const match = {
      attrs: [],
      start: index,
      tagName: tag,
    }
    advance(fullStr.length)
    let end, attr
    // 递归提取属性，直到出现 ">" 或 "/>" 字符
    while (
      !(end = html.match(startTagClose)) &&
      (attr = html.match(attribute))
    ) {
      advance(attr[0].length)
      match.attrs.push(attr)
    }
    if (end) {
      // 如果是 "/>" 表示单标签
      match.unarySlash = end[1]
      advance(end[0].length)
      match.end = index
      return match
    }
  }
}

// 处理开始标签
function handleStartTag (match) {
  const tagName = match.tagName
  const unary = match.unarySlash
  const len = match.attrs.length
  const attrs = new Array(len)
  for (let i = 0; i < l; i++) {
    const args = match.attrs[i]
    // 这里的 3、4、5 分别对应三种不同复制属性的方式
    // 3: attr="xxx" 双引号
    // 4: attr='xxx' 单引号
    // 5: attr=xxx   省略引号
    const value = args[3] || args[4] || args[5] || ''
    attrs[i] = {
      name: args[1],
      value
    }
  }

  if (!unary) {
    // 非单标签，入栈
    stack.push({
      tag: tagName,
      lowerCasedTag:
      tagName.toLowerCase(),
      attrs: attrs
    })
    lastTag = tagName
  }

  if (options.start) {
    // 开始标签的回调
    options.start(tagName, attrs, unary, match.start, match.end)
  }
}

// 处理闭合标签
function parseEndTag (tagName, start, end) {
  let pos, lowerCasedTagName
  if (start == null) start = index
  if (end == null) end = index

  if (tagName) {
    lowerCasedTagName = tagName.toLowerCase()
  }

  // 在栈内查找相同类型的未闭合标签
  if (tagName) {
    for (pos = stack.length - 1; pos >= 0; pos--) {
      if (stack[pos].lowerCasedTag === lowerCasedTagName) {
        break
      }
    }
  } else {
    pos = 0
  }

  if (pos >= 0) {
    // 关闭该标签内的未闭合标签，更新堆栈
    for (let i = stack.length - 1; i >= pos; i--) {
      if (options.end) {
        // end 回调
        options.end(stack[i].tag, start, end)
      }
    }

    // 堆栈中删除已关闭标签
    stack.length = pos
    lastTag = pos && stack[pos - 1].tag
  }
}
复制代码
```

在解析开始标签的时候，如果该标签不是单标签，会将该标签放入到一个堆栈当中，每次闭合标签的时候，会从栈顶向下查找同名标签，直到找到同名标签，这个操作会闭合同名标签上面的所有标签。接下来我们举个例子：

```html
<div>
  <h2>test</h2>
  <p>
  <p>
</div>
复制代码
```

在解析了 div 和 h2 的开始标签后，栈内就存在了两个元素。h2 闭合后，就会将 h2 出栈。然后会解析两个未闭合的 p 标签，此时，栈内存在三个元素（div、p、p）。如果这个时候，解析了 div 的闭合标签，除了将 div 闭合外，div 内两个未闭合的 p 标签也会跟随闭合，此时栈被清空。

为了便于理解，特地录制了一个动图，如下：

![入栈与出栈](https://file.shenfq.com/ipic/2020-08-19-134036.gif)

理清了 parseHTML 的逻辑后，我们回到调用 parseHTML 的位置，调用该方法的时候，一共会传入四个回调，分别对应标签的开始和结束、文本、注释。

```js
parseHTML(template, {
  // some options...

  // 解析到标签位置开始的回调
  start(tag, attrs, unary) {},
  // 解析到标签位置结束的回调
  end(tag) {},
  // 解析到文本时的回调
  chars(text: string) {},
  // 解析到注释时的回调
  comment(text: string) {}
})
复制代码
```

#### 处理开始标签

首先看解析到开始标签时，会生成一个 AST 节点，然后处理标签上的属性，最后将 AST 节点放入树形结构中。

```js
function makeAttrsMap(attrs) {
  const map = {}
  for (let i = 0, l = attrs.length; i < l; i++) {
    const { name, value } = attrs[i]
    map[name] = value
  }
  return map
}
function createASTElement(tag, attrs, parent) {
  const attrsList = attrs
  const attrsMap = makeAttrsMap(attrsList)
  return {
    type: 1,       // 节点类型
    tag,           // 节点名称
    attrsMap,      // 节点属性映射
    attrsList,     // 节点属性数组
    parent,        // 父节点
    children: [],  // 子节点
  }
}

const stack = []
let root // 根节点
let currentParent // 暂存当前的父节点
parseHTML(template, {
  // some options...

  // 解析到标签位置开始的回调
  start(tag, attrs, unary) {
    // 创建 AST 节点
    let element = createASTElement(tag, attrs, currentParent)

    // 处理指令: v-for v-if v-once
    processFor(element)
    processIf(element)
    processOnce(element)
    processElement(element, options)

    // 处理 AST 树
    // 根节点不存在，则设置该元素为根节点
   	if (!root) {
      root = element
      checkRootConstraints(root)
    }
    // 存在父节点
    if (currentParent) {
      // 将该元素推入父节点的子节点中
      currentParent.children.push(element)
      element.parent = currentParent
    }
    if (!unary) {
    	// 非单标签需要入栈，且切换当前父元素的位置
      currentParent = element
      stack.push(element)
    }
  }
})
复制代码
```

#### 处理结束标签

标签结束的逻辑就比较简单了，只需要去除栈内最后一个未闭合标签，进行闭合即可。

```js
parseHTML(template, {
  // some options...

  // 解析到标签位置结束的回调
  end() {
    const element = stack[stack.length - 1]
    const lastNode = element.children[element.children.length - 1]
    // 处理尾部空格的情况
    if (lastNode && lastNode.type === 3 && lastNode.text === ' ') {
      element.children.pop()
    }
    // 出栈，重置当前的父节点
    stack.length -= 1
    currentParent = stack[stack.length - 1]
  }
})
复制代码
```

#### 处理文本

处理完标签后，还需要对标签内的文本进行处理。文本的处理分两种情况，一种是带表达式的文本，还一种就是纯静态的文本。

```js
parseHTML(template, {
  // some options...

  // 解析到文本时的回调
  chars(text) {
    if (!currentParent) {
      // 文本节点外如果没有父节点则不处理
      return
    }
    
    const children = currentParent.children
    text = text.trim()
    if (text) {
      // parseText 用来解析表达式
      // delimiters 表示表达式标识符，默认为 ['{{', '}}']
      const res = parseText(text, delimiters))
      if (res) {
        // 表达式
        children.push({
          type: 2,
          expression: res.expression,
          tokens: res.tokens,
          text
        })
      } else {
        // 静态文本
        children.push({
          type: 3,
          text
        })
      }
    }
  }
})
复制代码
```

下面我们看看 parseText 如何解析表达式。

```js
// 构造匹配表达式的正则
const buildRegex = delimiters => {
  const open = delimiters[0]
  const close = delimiters[1]
  return new RegExp(open + '((?:.|\\n)+?)' + close, 'g')
}

function parseText (text, delimiters){
  // delimiters 默认为 {{ }}
  const tagRE = buildRegex(delimiters || ['{{', '}}'])
  // 未匹配到表达式，直接返回
  if (!tagRE.test(text)) {
    return
  }
  const tokens = []
  const rawTokens = []
  let lastIndex = tagRE.lastIndex = 0
  let match, index, tokenValue
  while ((match = tagRE.exec(text))) {
    // 表达式开始的位置
    index = match.index
    // 提取表达式开始位置前面的静态字符，放入 token 中
    if (index > lastIndex) {
      rawTokens.push(tokenValue = text.slice(lastIndex, index))
      tokens.push(JSON.stringify(tokenValue))
    }
    // 提取表达式内部的内容，使用 _s() 方法包裹
    const exp = match[1].trim()
    tokens.push(`_s(${exp})`)
    rawTokens.push({ '@binding': exp })
    lastIndex = index + match[0].length
  }
  // 表达式后面还有其他静态字符，放入 token 中
  if (lastIndex < text.length) {
    rawTokens.push(tokenValue = text.slice(lastIndex))
    tokens.push(JSON.stringify(tokenValue))
  }
  return {
    expression: tokens.join('+'),
    tokens: rawTokens
  }
}

复制代码
```

首先通过一段正则来提取表达式：

![提取表达式](https://file.shenfq.com/ipic/2020-08-20-052158.png)

看代码可能有点难，我们直接看例子，这里有一个包含表达式的文本。

```HTML
<div>是否登录：{{isLogin ? '是' : '否'}}</div>
复制代码
```

![运行结果](https://file.shenfq.com/ipic/2020-08-20-035633.png)

![解析文本](https://file.shenfq.com/ipic/2020-08-20-053140.png)

### optimize

通过上述一些列处理，我们就得到了 Vue 模板的 AST。由于 Vue 是响应式设计，所以拿到 AST 之后还需要进行一系列优化，确保静态的数据不会进入虚拟 DOM 的更新阶段，以此来优化性能。

```js
export function optimize (root, options) {
  if (!root) return
  // 标记静态节点
  markStatic(root)
}
复制代码
```

简单来说，就是把所以静态节点的 static 属性设置为 true。

```js
function isStatic (node) {
  if (node.type === 2) { // 表达式，返回 false
    return false
  }
  if (node.type === 3) { // 静态文本，返回 true
    return true
  }
  // 此处省略了部分条件
  return !!(
    !node.hasBindings && // 没有动态绑定
    !node.if && !node.for && // 没有 v-if/v-for
    !isBuiltInTag(node.tag) && // 不是内置组件 slot/component
    !isDirectChildOfTemplateFor(node) && // 不在 template for 循环内
    Object.keys(node).every(isStaticKey) // 非静态节点
  )
}

function markStatic (node) {
  node.static = isStatic(node)
  if (node.type === 1) {
    // 如果是元素节点，需要遍历所有子节点
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      if (!child.static) {
        // 如果有一个子节点不是静态节点，则该节点也必须是动态的
        node.static = false
      }
    }
  }
}
复制代码
```

### generate

得到优化的 AST 之后，就需要将 AST 转化为 render 方法。还是用之前的模板，先看看生成的代码长什么样：

```html
<div>
  <h2 v-if="message">{{message}}</h2>
  <button @click="showName">showName</button>
</div>
复制代码
{
  render: "with(this){return _c('div',[(message)?_c('h2',[_v(_s(message))]):_e(),_v(" "),_c('button',{on:{"click":showName}},[_v("showName")])])}"
}
复制代码
```

将生成的代码展开：

```js
with (this) {
    return _c(
      'div',
      [
        (message) ? _c('h2', [_v(_s(message))]) : _e(),
        _v(' '),
        _c('button', { on: { click: showName } }, [_v('showName')])
      ])
    ;
}
复制代码
```

看到这里一堆的下划线肯定很懵逼，这里的 `_c` 对应的是虚拟 DOM 中的 `createElement` 方法。其他的下划线方法在 `core/instance/render-helpers` 中都有定义，每个方法具体做了什么不做展开。

![render-helpers`](https://file.shenfq.com/ipic/2020-08-20-061905.png)

具体转化方法就是一些简单的字符拼接，下面是简化了逻辑的部分，不做过多讲述。

```js
export function generate(ast, options) {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}

export function genElement (el, state) {
  let code
  const data = genData(el, state)
  const children = genChildren(el, state, true)
  code = `_c('${el.tag}'${
    data ? `,${data}` : '' // data
  }${
    children ? `,${children}` : '' // children
  })`
  return code
}
复制代码
```

## 总结

理清了 Vue 模板编译的整个过程，重点都放在了解析 HTML 生成 AST 的部分。本文只是大致讲述了主要流程，其中省略了特别多的细节，比如：对 template/slot 的处理、指令的处理等等，如果想了解其中的细节可以直接阅读源码。希望大家在阅读这篇文章后有所收获。

