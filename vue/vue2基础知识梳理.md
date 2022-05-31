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

# 6.vue中，key的原理

