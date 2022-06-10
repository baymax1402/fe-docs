# 1.说下vite的原理

Vite有如下特点：

- 快速的**冷启动**: **No Bundle + esbuild 预构建**
- 即时的**模块热更新**: **基于ESM的HMR，同时利用浏览器缓存策略提升速度**
- 真正的**按需加载**: 利用浏览器ESM支持，实现真正的按需加载



# 2.Vue3中的响应式原理







# 3.用Vue3.0 写过组件吗？如果想实现一个 Modal你会怎么设计？

## **一、组件设计**

组件就是把图形、非图形的各种逻辑均抽象为一个统一的概念（组件）来实现开发的模式

现在有一个场景，点击新增与编辑都弹框出来进行填写，功能上大同小异，可能只是标题内容或者是显示的主体内容稍微不同

这时候就没必要写两个组件，只需要根据传入的参数不同，组件显示不同内容即可

这样，下次开发相同界面程序时就可以写更少的代码，意味着更高的开发效率，更少的 `Bug`和更少的程序体积

## **二、需求分析**

实现一个`Modal`组件，首先确定需要完成的内容：

- 遮罩层
- 标题内容
- 主体内容
- 确定和取消按钮

主体内容需要灵活，所以可以是字符串，也可以是一段 `html` 代码

特点是它们在当前`vue`实例之外独立存在，通常挂载于`body`之上

除了通过引入`import`的形式，我们还可通过`API`的形式进行组件的调用

还可以包括配置全局样式、国际化、与`typeScript`结合

## **三、实现流程**

首先看看大致流程：

- 目录结构
- 组件内容
- 实现 API 形式
- 事件处理
- 其他完善

### **目录结构**

`Modal`组件相关的目录结构

```javascript
├── plugins
│   └── modal
│       ├── Content.tsx // 维护 Modal 的内容，用于 h 函数和 jsx 语法
│       ├── Modal.vue // 基础组件
│       ├── config.ts // 全局默认配置
│       ├── index.ts // 入口
│       ├── locale // 国际化相关
│       │   ├── index.ts
│       │   └── lang
│       │       ├── en-US.ts
│       │       ├── zh-CN.ts
│       │       └── zh-TW.ts
│       └── modal.type.ts // ts类型声明相关
```

因为 Modal 会被 `app.use(Modal)` 调用作为一个插件，所以都放在`plugins`目录下

### **组件内容**

首先实现`modal.vue`的主体显示内容大致如下

```javascript
<Teleport to="body" :disabled="!isTeleport">
    <div v-if="modelValue" class="modal">
        <div
             class="mask"
             :style="style"
             @click="maskClose && !loading && handleCancel()"
             ></div>
        <div class="modal__main">
            <div class="modal__title line line--b">
                <span>{{ title || t("r.title") }}</span>
                <span
                      v-if="close"
                      :title="t('r.close')"
                      class="close"
                      @click="!loading && handleCancel()"
                      >✕</span
                    >
            </div>
            <div class="modal__content">
                <Content v-if="typeof content === 'function'" :render="content" />
                <slot v-else>
                    {{ content }}
                </slot>
            </div>
            <div class="modal__btns line line--t">
                <button :disabled="loading" @click="handleConfirm">
                    <span class="loading" v-if="loading"> ❍ </span>{{ t("r.confirm") }}
                </button>
                <button @click="!loading && handleCancel()">
                    {{ t("r.cancel") }}
                </button>
            </div>
        </div>
    </div>
</Teleport>
```

最外层上通过Vue3 `Teleport` 内置组件进行包裹，其相当于传送门，将里面的内容传送至`body`之上

并且从`DOM`结构上来看，把`modal`该有的内容（遮罩层、标题、内容、底部按钮）都实现了

关于主体内容

```javascript
<div class="modal__content">
    <Content v-if="typeof content==='function'"
             :render="content" />
    <slot v-else>
        {{content}}
    </slot>
</div>
```

可以看到根据传入`content`的类型不同，对应显示不同得到内容

最常见的则是通过调用字符串和默认插槽的形式

```javascript
// 默认插槽
<Modal v-model="show"
       title="演示 slot">
    <div>hello world~</div>
</Modal>

// 字符串
<Modal v-model="show"
       title="演示 content"
       content="hello world~" />
```

通过 API 形式调用`Modal`组件的时候，`content`可以使用下面两种

- h 函数

```javascript
$modal.show({
  title: '演示 h 函数',
  content(h) {
    return h(
      'div',
      {
        style: 'color:red;',
        onClick: ($event: Event) => console.log('clicked', $event.target)
      },
      'hello world ~'
    );
  }
});
```

复制

- JSX

```javascript
$modal.show({
  title: '演示 jsx 语法',
  content() {
    return (
      <div
        onClick={($event: Event) => console.log('clicked', $event.target)}
      >
        hello world ~
      </div>
    );
  }
});
```

### **实现 API 形式**

那么组件如何实现`API`形式调用`Modal`组件呢？

在`Vue2`中，我们可以借助`Vue`实例以及`Vue.extend`的方式获得组件实例，然后挂载到`body`上

```javascript
import Modal from './Modal.vue';
const ComponentClass = Vue.extend(Modal);
const instance = new ComponentClass({ el: document.createElement("div") });
document.body.appendChild(instance.$el);
```

虽然`Vue3`移除了`Vue.extend`方法，但可以通过`createVNode`实现

```javascript
import Modal from './Modal.vue';
const container = document.createElement('div');
const vnode = createVNode(Modal);
render(vnode, container);
const instance = vnode.component;
document.body.appendChild(container);
```

在`Vue2`中，可以通过`this`的形式调用全局 API

```javascript
export default {
    install(vue) {
       vue.prototype.$create = create
    }
}
```

而在 Vue3 的 `setup` 中已经没有 `this`概念了，需要调用`app.config.globalProperties`挂载到全局

```javascript
export default {
    install(app) {
        app.config.globalProperties.$create = create
    }
}
```

### **事件处理**

下面再看看看`Modal`组件内部是如何处理「确定」「取消」事件的，既然是`Vue3`，我们可以采用`Compositon API` 形式

```javascript
// Modal.vue
setup(props, ctx) {
  let instance = getCurrentInstance(); // 获得当前组件实例
  onBeforeMount(() => {
    instance._hub = {
      'on-cancel': () => {},
      'on-confirm': () => {}
    };
  });

  const handleConfirm = () => {
    ctx.emit('on-confirm');
    instance._hub['on-confirm']();
  };
  const handleCancel = () => {
    ctx.emit('on-cancel');
    ctx.emit('update:modelValue', false);
    instance._hub['on-cancel']();
  };

  return {
    handleConfirm,
    handleCancel
  };
}
```

在上面代码中，可以看得到除了使用传统`emit`的形式使父组件监听，还可通过`_hub`属性中添加 `on-cancel`，`on-confirm`方法实现在`API`中进行监听

```javascript
app.config.globalProperties.$modal = {
   show({}) {
     /* 监听 确定、取消 事件 */
   }
}
```

下面再来目睹下`_hub`是如何实现

```javascript
// index.ts
app.config.globalProperties.$modal = {
    show({
        /* 其他选项 */
        onConfirm,
        onCancel
    }) {
        /* ... */

        const { props, _hub } = instance;

        const _closeModal = () => {
            props.modelValue = false;
            container.parentNode!.removeChild(container);
        };
        // 往 _hub 新增事件的具体实现
        Object.assign(_hub, {
            async 'on-confirm'() {
            if (onConfirm) {
                const fn = onConfirm();
                // 当方法返回为 Promise
                if (fn && fn.then) {
                    try {
                        props.loading = true;
                        await fn;
                        props.loading = false;
                        _closeModal();
                    } catch (err) {
                        // 发生错误时，不关闭弹框
                        console.error(err);
                        props.loading = false;
                    }
                } else {
                    _closeModal();
                }
            } else {
                _closeModal();
            }
        },
            'on-cancel'() {
                onCancel && onCancel();
                _closeModal();
            }
    });
}
};
```

### **其他完善**

关于组件实现国际化、与`typsScript`结合，大家可以根据自身情况在此基础上进行更改

## **参考文献**

- https://segmentfault.com/a/1190000038928664
- https://vue3js.cn/docs/zh

# 4.Vue3.0中Treeshaking特性是什么，并举例进行说明

## 一、是什么

Tree shaking **是一种通过清除多余代码方式来优化项目打包体积的技术**，专业术语叫 Dead code elimination

简单来讲，就是在保持代码运行结果不变的前提下，去除无用的代码

如果把代码打包比作制作蛋糕，传统的方式是把鸡蛋（带壳）全部丢进去搅拌，然后放入烤箱，最后把（没有用的）蛋壳全部挑选并剔除出去

而treeshaking则是一开始就把有用的蛋白蛋黄（import）放入搅拌，最后直接作出蛋糕

也就是说 ，tree shaking 其实是找出使用的代码

在Vue2中，无论我们使用什么功能，它们最终都会出现在生产代码中。主要原因是Vue实例在项目中是单例的，捆绑程序无法检测到该对象的哪些属性在代码中被使用到

```
import Vue from 'vue'

Vue.nextTick(() => {})
```

而Vue3源码引入tree shaking特性，将全局 API 进行分块。如果你不使用其某些功能，它们将不会包含在你的基础包中

```
import { nextTick, observable } from 'vue'

nextTick(() => {})
```

## 二、如何做

Tree shaking是基于ES6模板语法（import与exports），主要是借助ES6模块的静态编译思想，在编译时就能确定模块的依赖关系，以及输入和输出的变量

Tree shaking无非就是做了两件事：

- 编译阶段利用ES6 Module判断哪些模块已经加载

- 判断那些模块和变量未被使用或者引用，进而删除对应代码


下面就来举个例子：

通过脚手架vue-cli安装Vue2与Vue3项目

```
vue create vue-demo
```

Vue2 项目
组件中使用data属性

<script>
    export default {
        data: () => ({
            count: 1,
        }),
    };
</script>
对项目进行打包，体积如下图

为组件设置其他属性（compted、watch）

```
export default {
    data: () => ({
        question:"", 
        count: 1,
    }),
    computed: {
        double: function () {
            return this.count * 2;
        },
    },
    watch: {
        question: function (newQuestion, oldQuestion) {
            this.answer = 'xxxx'
        }
};
```

再一次打包，发现打包出来的体积并没有变化

Vue3 项目
组件中简单使用

```
import { reactive, defineComponent } from "vue";
export default defineComponent({
  setup() {
    const state = reactive({
      count: 1,
    });
    return {
      state,
    };
  },
});
```

将项目进行打包

在组件中引入computed和watch

```
import { reactive, defineComponent, computed, watch } from "vue";
export default defineComponent({
  setup() {
    const state = reactive({
      count: 1,
    });
    const double = computed(() => {
      return state.count * 2;
    });

    watch(
      () => state.count,
      (count, preCount) => {
        console.log(count);
        console.log(preCount);
      }
    );
    return {
      state,
      double,
    };

  },
});
```

再次对项目进行打包，可以看到在引入computer和watch之后，项目整体体积变大了

## 三、作用

通过Tree shaking，Vue3给我们带来的好处是：

- 减少程序体积（更小）

- 减少程序执行时间（更快）

- 便于将来对程序架构进行优化（更友好）

## 参考链接

https://blog.csdn.net/qq_43299703/article/details/121213453



# 5.Vue3.0所采用的Composition Api 与 Vue2.x使用的Options Api有什么不同？

## 开始之前

Composition API 可以说是Vue3的最大特点，那么为什么要推出Composition Api，解决了什么问题？

通常使用Vue2开发的项目，普遍会存在以下问题：

- 代码的可读性随着组件变大而变差

- 每一种代码复用的方式，都存在缺点
- TypeScript支持有限
- 以上通过使用Composition Api都能迎刃而解

## 正文

### 一、Options Api

Options API，即大家常说的选项API，即以vue为后缀的文件，通过定义methods，computed，watch，data等属性与方法，共同处理页面逻辑

可以看到Options代码编写方式，如果是组件状态，则写在data属性上，如果是方法，则写在methods属性上…

用组件的选项 (data、computed、methods、watch) 组织逻辑在大多数情况下都有效

然而，当组件变得复杂，导致对应属性的列表也会增长，这可能会导致组件难以阅读和理解

### 二、Composition Api

在 Vue3 Composition API 中，组件根据逻辑功能来组织的，一个功能所定义的所有 API 会放在一起（更加的高内聚，低耦合）

即使项目很大，功能很多，我们都能快速的定位到这个功能所用到的所有 API

### 三、对比

下面对Composition Api与Options Api进行两大方面的比较  逻辑组织和逻辑复用

#### 逻辑组织

- 逻辑复用
- 逻辑组织

##### Options API

假设一个组件是一个大型组件，其内部有很多处理逻辑关注点（对应下图不用颜色）

可以看到，这种碎片化使得理解和维护复杂组件变得困难

选项的分离掩盖了潜在的逻辑问题。此外，在处理单个逻辑关注点时，我们必须不断地“跳转”相关代码的选项块

##### Compostion API

而Compositon API正是解决上述问题，将某个逻辑关注点相关的代码全都放在一个函数里，这样当需要修改一个功能时，就不再需要在文件中跳来跳去

下面举个简单例子，将处理count属性相关的代码放在同一个函数了

```
function useCount() {
    let count = ref(10);
    let double = computed(() => {
        return count.value * 2;
    });

    const handleConut = () => {
        count.value = count.value * 2;
    };
    
    console.log(count);
    
    return {
        count,
        double,
        handleConut,
    };

}


```

组件上中使用count

```
export default defineComponent({
    setup() {
        const { count, double, handleConut } = useCount();
        return {
            count,
            double,
            handleConut
        }
    },
});
```

再来一张图进行对比，可以很直观地感受到 Composition API在逻辑组织方面的优势，以后修改一个属性功能的时候，只需要跳到控制该属性的方法中即可

#### 逻辑复用

在Vue2中，我们是用过mixin去复用相同的逻辑

下面举个例子，我们会另起一个mixin.js文件

```
export const MoveMixin = {
  data() {
    return {
      x: 0,
      y: 0,
    };
  },

  methods: {
    handleKeyup(e) {
      console.log(e.code);
      // 上下左右 x y
      switch (e.code) {
        case "ArrowUp":
          this.y--;
          break;
        case "ArrowDown":
          this.y++;
          break;
        case "ArrowLeft":
          this.x--;
          break;
        case "ArrowRight":
          this.x++;
          break;
      }
    },
  },

  mounted() {
    window.addEventListener("keyup", this.handleKeyup);
  },

  unmounted() {
    window.removeEventListener("keyup", this.handleKeyup);
  },
};



```


然后在组件中使用

<template>
  <div>
    Mouse position: x {{ x }} / y {{ y }}
  </div>
</template>
<script>
import mousePositionMixin from './mouse'
export default {
  mixins: [mousePositionMixin]
}
</script>

使用单个mixin似乎问题不大，但是当我们一个组件混入大量不同的 mixins 的时候

```
mixins: [mousePositionMixin, fooMixin, barMixin, otherMixin]
```


会存在两个非常明显的问题：

- 命名冲突

- 数据来源不清晰

现在通过Compositon API这种方式改写上面的代码

```
import { onMounted, onUnmounted, reactive } from "vue";
export function useMove() {
  const position = reactive({
    x: 0,
    y: 0,
  });

  const handleKeyup = (e) => {
    console.log(e.code);
    // 上下左右 x y
    switch (e.code) {
      case "ArrowUp":
        // y.value--;
        position.y--;
        break;
      case "ArrowDown":
        // y.value++;
        position.y++;
        break;
      case "ArrowLeft":
        // x.value--;
        position.x--;
        break;
      case "ArrowRight":
        // x.value++;
        position.x++;
        break;
    }
  };

  onMounted(() => {
    window.addEventListener("keyup", handleKeyup);
  });

  onUnmounted(() => {
    window.removeEventListener("keyup", handleKeyup);
  });

  return { position };
}
```

在组件中使用

```
<template>
  <div>
    Mouse position: x {{ x }} / y {{ y }}
  </div>
</template>


<script>
import { useMove } from "./useMove";
import { toRefs } from "vue";
export default {
  setup() {
    const { position } = useMove();
    const { x, y } = toRefs(position);
    return {
      x,
      y,
    };

  },
};
</script>
```

可以看到，整个数据来源清晰了，即使去编写更多的 hook 函数，也不会出现命名冲突的问题

## 小结

在逻辑组织和逻辑复用方面，Composition API是优于Options API
因为Composition API几乎是函数，会有更好的类型推断。
Composition API对 tree-shaking 友好，代码也更容易压缩
Composition API中见不到this的使用，减少了this指向不明的情况
如果是小型组件，可以继续使用Options API，也是十分友好的
————————————————
参考链接：https://blog.csdn.net/qq_43392573/article/details/123556182

# 6.Vue3.0里为什么要用 Proxy API 替代 defineProperty API ？

## **一、Object.defineProperty**

定义：`Object.defineProperty()` 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回此对象

### **为什么能实现响应式**

通过`defineProperty` 两个属性，`get`及`set`

- get

属性的 getter 函数，当访问该属性时，会调用此函数。执行时不传入任何参数，但是会传入 this 对象（由于继承关系，这里的this并不一定是定义该属性的对象）。该函数的返回值会被用作属性的值

- set

属性的 setter 函数，当属性值被修改时，会调用此函数。该方法接受一个参数（也就是被赋予的新值），会传入赋值时的 this 对象。默认为 undefined

下面通过代码展示：

定义一个响应式函数`defineReactive`

```javascript
function update() {
    app.innerText = obj.foo
}

function defineReactive(obj, key, val) {
    Object.defineProperty(obj, key, {
        get() {
            console.log(`get ${key}:${val}`);
            return val
        },
        set(newVal) {
            if (newVal !== val) {
                val = newVal
                update()
            }
        }
    })
}
```

复制

调用`defineReactive`，数据发生变化触发`update`方法，实现数据响应式

```javascript
const obj = {}
defineReactive(obj, 'foo', '')
setTimeout(()=>{
    obj.foo = new Date().toLocaleTimeString()
},1000)
```

复制

在对象存在多个`key`情况下，需要进行遍历

```javascript
function observe(obj) {
    if (typeof obj !== 'object' || obj == null) {
        return
    }
    Object.keys(obj).forEach(key => {
        defineReactive(obj, key, obj[key])
    })
}
```

复制

如果存在嵌套对象的情况，还需要在`defineReactive`中进行递归

```javascript
function defineReactive(obj, key, val) {
    observe(val)
    Object.defineProperty(obj, key, {
        get() {
            console.log(`get ${key}:${val}`);
            return val
        },
        set(newVal) {
            if (newVal !== val) {
                val = newVal
                update()
            }
        }
    })
}
```

复制

当给`key`赋值为对象的时候，还需要在`set`属性中进行递归

```javascript
set(newVal) {
    if (newVal !== val) {
        observe(newVal) // 新值是对象的情况
        notifyUpdate()
    }
}
```

复制

上述例子能够实现对一个对象的基本响应式，但仍然存在诸多问题

现在对一个对象进行删除与添加属性操作，无法劫持到

```javascript
const obj = {
    foo: "foo",
    bar: "bar"
}
observe(obj)
delete obj.foo // no ok
obj.jar = 'xxx' // no ok
```

复制

当我们对一个数组进行监听的时候，并不那么好使了

```javascript
const arrData = [1,2,3,4,5];
arrData.forEach((val,index)=>{
    defineProperty(arrData,index,val)
})
arrData.push() // no ok
arrData.pop()  // no ok
arrDate[0] = 99 // ok
```

复制

可以看到数据的`api`无法劫持到，从而无法实现数据响应式，

所以在`Vue2`中，增加了`set`、`delete` API，并且对数组`api`方法进行一个重写

还有一个问题则是，如果存在深层的嵌套对象关系，需要深层的进行监听，造成了性能的极大问题

### **小结**

- 检测不到对象属性的添加和删除
- 数组`API`方法无法监听到
- 需要对每个属性进行遍历监听，如果嵌套对象，需要深层监听，造成性能问题

## **二、proxy**

`Proxy`的监听是针对一个对象的，那么对这个对象的所有操作会进入监听操作，这就完全可以代理所有属性了

在`ES6`系列中，我们详细讲解过`Proxy`的使用，就不再述说了

下面通过代码进行展示：

定义一个响应式方法`reactive`

```javascript
function reactive(obj) {
    if (typeof obj !== 'object' && obj != null) {
        return obj
    }
    // Proxy相当于在对象外层加拦截
    const observed = new Proxy(obj, {
        get(target, key, receiver) {
            const res = Reflect.get(target, key, receiver)
            console.log(`获取${key}:${res}`)
            return res
        },
        set(target, key, value, receiver) {
            const res = Reflect.set(target, key, value, receiver)
            console.log(`设置${key}:${value}`)
            return res
        },
        deleteProperty(target, key) {
            const res = Reflect.deleteProperty(target, key)
            console.log(`删除${key}:${res}`)
            return res
        }
    })
    return observed
}
```

复制

测试一下简单数据的操作，发现都能劫持

```javascript
const state = reactive({
    foo: 'foo'
})
// 1.获取
state.foo // ok
// 2.设置已存在属性
state.foo = 'fooooooo' // ok
// 3.设置不存在属性
state.dong = 'dong' // ok
// 4.删除属性
delete state.dong // ok
```

复制

再测试嵌套对象情况，这时候发现就不那么 OK 了

```javascript
const state = reactive({
    bar: { a: 1 }
})

// 设置嵌套对象属性
state.bar.a = 10 // no ok
```

复制

如果要解决，需要在`get`之上再进行一层代理

```javascript
function reactive(obj) {
    if (typeof obj !== 'object' && obj != null) {
        return obj
    }
    // Proxy相当于在对象外层加拦截
    const observed = new Proxy(obj, {
        get(target, key, receiver) {
            const res = Reflect.get(target, key, receiver)
            console.log(`获取${key}:${res}`)
            return isObject(res) ? reactive(res) : res
        },
    return observed
}
```

复制

## **三、总结**

`Object.defineProperty`只能遍历对象属性进行劫持

```javascript
function observe(obj) {
    if (typeof obj !== 'object' || obj == null) {
        return
    }
    Object.keys(obj).forEach(key => {
        defineReactive(obj, key, obj[key])
    })
}
```

复制

`Proxy`直接可以劫持整个对象，并返回一个新对象，我们可以只操作新的对象达到响应式目的

```javascript
function reactive(obj) {
    if (typeof obj !== 'object' && obj != null) {
        return obj
    }
    // Proxy相当于在对象外层加拦截
    const observed = new Proxy(obj, {
        get(target, key, receiver) {
            const res = Reflect.get(target, key, receiver)
            console.log(`获取${key}:${res}`)
            return res
        },
        set(target, key, value, receiver) {
            const res = Reflect.set(target, key, value, receiver)
            console.log(`设置${key}:${value}`)
            return res
        },
        deleteProperty(target, key) {
            const res = Reflect.deleteProperty(target, key)
            console.log(`删除${key}:${res}`)
            return res
        }
    })
    return observed
}
```

复制

`Proxy`可以直接监听数组的变化（`push`、`shift`、`splice`）

```javascript
const obj = [1,2,3]
const proxtObj = reactive(obj)
obj.psuh(4) // ok
```

复制

`Proxy`有多达13种拦截方法,不限于`apply`、`ownKeys`、`deleteProperty`、`has`等等，这是`Object.defineProperty`不具备的

正因为`defineProperty`自身的缺陷，导致`Vue2`在实现响应式过程需要实现其他的方法辅助（如重写数组方法、增加额外`set`、`delete`方法）

```javascript
// 数组重写
const originalProto = Array.prototype
const arrayProto = Object.create(originalProto)
['push', 'pop', 'shift', 'unshift', 'splice', 'reverse', 'sort'].forEach(method => {
  arrayProto[method] = function () {
    originalProto[method].apply(this.arguments)
    dep.notice()
  }
});

// set、delete
Vue.set(obj,'bar','newbar')
Vue.delete(obj),'bar')
```

复制

`Proxy` 不兼容IE，也没有 `polyfill`, `defineProperty` 能支持到IE9

### **参考文献**

- https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty

# 7.Vue3.0性能提升主要是通过哪几方面体现的？

## 前言

如果是 Vue 的忠实er，会知道 Vue.js 2.x 已经是足够优秀的前端框架，并且性能也不错，但是在升级的 Vue 3.0 版本中，让这个性能更上一层楼，那 Vue3.0 性能又有了哪些方面的突破了？

Vue 3 与 Vue 2 相比：

- 在 bundle 包大小方面（tree-shaking 减少了 41% 的体积）
- 初始渲染速度方面（快了 55%）
- 更新速度方面（快了 133%）
- 内存占用方面（减少了 54%）

这篇文章你将了解 Vue 3.0 性能优化的三个方法：

- 源码体积优化
- 数据劫持优化
- 编译优化

## 1. 源码体积优化

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/521e3a59956d4a0db86a28130d2df601~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

tree-shaking，它的原理很简单，tree-shaking 依赖 ES2015 模块语法的静态结构（即 import 和 export），通过编译阶段的静态分析，找到没有引入的模块并打上标记。利用 tree-shaking 技术，如果你在项目中没有引入无关的组件，那么它们对应的代码就不会打包，这样也就间接达到了减少项目引入的 Vue.js 包体积的目的。

Vue3.0 中最直接使用 tree-shaking 技术的一个例子，在 createApp 时会通过 ensureRenderer 创建渲染器对象，但是这里并不是直接创建渲染器对象，而是延时创建渲染器，目的是当用户只依赖响应式包的时候，可以通过 tree-shaking 移除核心渲染逻辑相关的代码。

## 2. 数据劫持优化

我们先来回忆一下，在 Vue 3.0 之前的数据劫持，Vue.js 1.x 和 Vue.js 2.x 内部都是通过 Object.defineProperty 这个 API 去劫持数据的 getter 和 setter，具体是这样的：

```
Object.defineProperty(data, 'x',{
  get(){
    // 收集
  },
  set(){
    // 更新
  }
})
复制代码
```

核心就是 Object.defineProperty，其实 Vue.js 1.x 和 Vue.js 2.x 不支持 IE 8 及以下也是因为 Object.defineProperty 不支持。数据劫持是 Vue.js 区别于 React 的一大特色，Vue 框架中 DOM 是数据的一种映射，数据发生变化后可以自动更新 DOM，用户只需要专注于数据的修改，没有其余的心智负担。

在 Vue.js 3.0 使用了 Proxy API 做数据劫持，它是这样的：

```
observed = new Proxy(data, {
  get() {
    // 收集
  },
  set() {
    // 更新
  }
});
复制代码
```

### 为什么替换 Object.defineProperty 到 Proxy

Object.defineProperty 和 Proxy 都可以进行数据的劫持，那为什么还要将 Object.defineProperty 替换为 Proxy 了。原因有两个：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1244484032d447ea5f779841f60716f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### Proxy 是如何解决 Object.defineProperty API 的缺陷

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff94922f23854ac68b3294b1023c377b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

## 3. 编译优化

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f551113bc5c34995a38ce314cc35eb98~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

通过数据劫持和依赖收集，Vue.js 2.x 的数据更新并触发重新渲染的粒度是组件级的：

```
<template>
  <div id="content">
    <p class="text">1</p>
    <p class="text">2</p>
    <p class="text">3</p>
    <p class="text">{{message}}</p>
    <p class="text">4</p>
    <p class="text">5</p>
    <p class="text">6</p>
  </div>
</template>
复制代码
```

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab456a5e27674058963a5fc6feffd5ad~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

虽然这段代码中只有一个动态节点，但在如果 message 发生改变，单个组件内部依然需要遍历该组件的整个 vnode 树，所以这里有很多 diff 和遍历其实都是不需要的，这就会导致 vnode 的性能跟模版大小正相关，跟动态节点的数量无关，当一些组件的整个模版内只有少量动态节点时，这些遍历都是性能的浪费。

Vue.js 3.0 做到了，它通过编译阶段对静态模板的分析，编译生成了 Block tree。Block tree 是一个将模版基于动态节点指令切割的嵌套区块，每个区块内部的节点结构是固定的，而且每个区块只需要以一个 Array 来追踪自身包含的动态节点。

借助 Block tree，Vue.js 将 vnode **更新性能**由与模版整体大小相关提升为**与动态内容的数量相关**，这是一个非常大的性能突破。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5203db3517b646b6a34d0310646fdbdf~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

除此之外，Vue.js 3.0 在编译阶段还包含了对 Slot 的编译优化、事件侦听函数的缓存优化，并且在运行时重写了 diff 算法。

## 参考

- [docs.google.com/spreadsheet…](https://link.juejin.cn?target=https%3A%2F%2Fdocs.google.com%2Fspreadsheets%2Fd%2F1VJFx-kQ4KjJmnpDXIEaig-cVAAJtpIGLZNbv3Lr4CR0%2Fedit%23gid%3D0)
- [github.com/vuejs/vue-n…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fvuejs%2Fvue-next%2Freleases)
- [zhuanlan.zhihu.com/p/254219538](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F254219538)
- [kaiwu.lagou.com/course/cour…](https://link.juejin.cn?target=https%3A%2F%2Fkaiwu.lagou.com%2Fcourse%2FcourseInfo.htm%3FcourseId%3D326%23%2Fdetail%2Fpc%3Fid%3D4054)
- https://juejin.cn/post/6983856130535473183
- [note.youdao.com/web/#/file/…](https://link.juejin.cn/?target=https%3A%2F%2Fnote.youdao.com%2Fweb%2F%23%2Ffile%2F3B4DFEA7A26D4FC0BE54FCA3212EA247%2Fmarkdown%2F2F16679FEBE94D108BE38D5BB88132D7%2F)

# 8.Vue3.0的设计目标是什么？做了哪些优化?

## 一、设计目标

不以解决实际业务痛点的更新都是耍流氓，下面我们来列举一下Vue3之前我们或许会面临的问题

- 随着功能的增长，复杂组件的代码变得越来越难以维护

- 缺少一种比较「干净」的在多个组件之间提取和复用逻辑的机制

- 类型推断不够友好

- bundle的时间太久了


而 Vue3 经过长达两三年时间的筹备，做了哪些事情？

我们从结果反推

- 更小

- 更快

- TypeScript支持

- API设计一致性

- 提高自身可维护性

- 开放更多底层功能


一句话概述，就是更小更快更友好了

**更小**
Vue3移除一些不常用的 API

引入tree-shaking，可以将无用模块“剪辑”，仅打包需要的，使打包的整体体积变小了

**更快**
主要体现在编译方面：

- diff算法优化

- 静态提升

- 事件监听缓存

- SSR优化


下篇文章我们会进一步介绍

**更友好**
vue3在兼顾vue2的options API的同时还推出了composition API，大大增加了代码的逻辑组织和代码复用能力

这里代码简单演示下：

存在一个获取鼠标位置的函数

```
import { toRefs, reactive } from 'vue';
function useMouse(){
    const state = reactive({x:0,y:0});
    const update = e=>{
        state.x = e.pageX;
        state.y = e.pageY;
    }
    onMounted(()=>{
        window.addEventListener('mousemove',update);
    })
    onUnmounted(()=>{
        window.removeEventListener('mousemove',update);
    })

    return toRefs(state);

}
```

我们只需要调用这个函数，即可获取x、y的坐标，完全不用关注实现过程

试想一下，如果很多类似的第三方库，我们只需要调用即可，不必关注实现过程，开发效率大大提高

同时，VUE3是基于typescipt编写的，可以享受到自动的类型定义提示

## 三、优化方案

vue3从很多层面都做了优化，可以分成三个方面：

- 源码

- 性能

- 语法 API


### 源码

源码可以从两个层面展开：

- 源码管理

- TypeScript


**源码管理**

vue3整个源码是通过 monorepo的方式维护的，根据功能将不同的模块拆分到packages目录下面不同的子目录中

这样使得模块拆分更细化，职责划分更明确，模块之间的依赖关系也更加明确，开发人员也更容易阅读、理解和更改所有模块源码，提高代码的可维护性

另外一些 package（比如 reactivity 响应式库）是可以独立于 Vue 使用的，这样用户如果只想使用 Vue3的响应式能力，可以单独依赖这个响应式库而不用去依赖整个 Vue

**TypeScript**

Vue3是基于typeScript编写的，提供了更好的类型检查，能支持复杂的类型推导

**性能**
vue3是从什么哪些方面对性能进行进一步优化呢？

- 体积优化

- 编译优化

- 数据劫持优化


这里讲述数据劫持：

在vue2中，数据劫持是通过Object.defineProperty，这个 API 有一些缺陷，并不能检测对象属性的添加和删除

```
Object.defineProperty(data, 'a',{
  get(){
    // track
  },
  set(){
    // trigger
  }
})
```

尽管Vue为了解决这个问题提供了 set和delete实例方法，但是对于用户来说，还是增加了一定的心智负担

同时在面对嵌套层级比较深的情况下，就存在性能问题

```
default {
  data: {
    a: {
      b: {
          c: {
          d: 1
        }
      }
    }
  }
}
```

相比之下，vue3是通过proxy监听整个对象，那么对于删除还是监听当然也能监听到

同时Proxy 并不能监听到内部深层次的对象变化，而 Vue3 的处理方式是在getter 中去递归响应式，这样的好处是真正访问到的内部对象才会变成响应式，而不是无脑递归

### 语法 API

这里当然说的就是composition API，其两大显著的优化：

- 优化逻辑组织

- 优化逻辑复用


**逻辑组织**

一张图，我们可以很直观地感受到 Composition API在逻辑组织方面的优势

相同功能的代码编写在一块，而不像options API那样，各个功能的代码混成一块

**逻辑复用**

在vue2中，我们是通过mixin实现功能混合，如果多个mixin混合，会存在两个非常明显的问题：命名冲突和数据来源不清晰

而通过composition这种形式，可以将一些复用的代码抽离出来作为一个函数，只要的使用的地方直接进行调用即可

同样是上文的获取鼠标位置的例子

```
import { toRefs, reactive, onUnmounted, onMounted } from 'vue';
function useMouse(){
    const state = reactive({x:0,y:0});
    const update = e=>{
        state.x = e.pageX;
        state.y = e.pageY;
    }
    onMounted(()=>{
        window.addEventListener('mousemove',update);
    })
    onUnmounted(()=>{
        window.removeEventListener('mousemove',update);
    })

    return toRefs(state);

}
```

组件使用

```
import useMousePosition from './mouse'
export default {
    setup() {
        const { x, y } = useMousePosition()
        return { x, y }
    }
}
```

可以看到，整个数据来源清晰了，即使去编写更多的hook函数，也不会出现命名冲突的问题
————————————————
参考链接：https://blog.csdn.net/qq_43299703/article/details/121213347

# 9.Vue3有了解过吗？能说说跟Vue2的区别吗？

## **一、Vue3介绍**

关于`vue3`的重构背景，看看尤大怎么说：

「Vue 新版本的理念成型于 2018 年末，当时 Vue 2 的代码库已经有两岁半了。比起通用软件的生命周期来这好像也没那么久，但在这段时期，前端世界已经今昔非比了

在我们更新（和重写）Vue 的主要版本时，主要考虑两点因素：首先是新的 JavaScript 语言特性在主流浏览器中的受支持水平；其次是当前代码库中随时间推移而逐渐暴露出来的一些设计和架构问题」

简要就是：

- 利用新的语言特性(es6)
- 解决架构问题

## **哪些变化**

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/758sp5q8j6.png?imageView2/2/w/1620)

从上图中，我们可以概览`Vue3`的新特性，如下：

- 速度更快
- 体积减少
- 更易维护
- 更接近原生
- 更易使用

### **速度更快**

```
vue3`相比`vue2
```

- 重写了虚拟`Dom`实现
- 编译模板的优化
- 更高效的组件初始化
- `undate`性能提高1.3~2倍
- `SSR`速度提高了2~3倍

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/t1yunfd7u1.png?imageView2/2/w/1620)

### **体积更小**

通过`webpack`的`tree-shaking`功能，可以将无用模块“剪辑”，仅打包需要的

能够`tree-shaking`，有两大好处：

- 对开发人员，能够对`vue`实现更多其他的功能，而不必担忧整体体积过大
- 对使用者，打包出来的包体积变小了

`vue`可以开发出更多其他的功能，而不必担忧`vue`打包出来的整体体积过多

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/t88cw05egi.png?imageView2/2/w/1620)

### **更易维护**

#### **compositon Api**

- 可与现有的`Options API`一起使用
- 灵活的逻辑组合与复用
- `Vue3`模块可以和其他框架搭配使用

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/tsx5lkn48x.png?imageView2/2/w/1620)

#### **更好的Typescript支持**

`VUE3`是基于`typescipt`编写的，可以享受到自动的类型定义提示

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/vhk7cp32gb.png?imageView2/2/w/1620)

#### **编译器重写**

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/5kyiwib9nz.png?imageView2/2/w/1620)

### **更接近原生**

可以自定义渲染 API

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/07i2hx1m1d.png?imageView2/2/w/1620)

### **更易使用**

响应式 `Api` 暴露出来

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/i3ikvylt4x.png?imageView2/2/w/1620)

轻松识别组件重新渲染原因

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/quy9sbq2xx.png?imageView2/2/w/1620)

## **二、Vue3新增特性**

Vue 3 中需要关注的一些新功能包括：

- framents
- Teleport
- composition Api
- createRenderer

### **framents**

在 `Vue3.x` 中，组件现在支持有多个根节点

```javascript
<!-- Layout.vue -->
<template>
  <header>...</header>
  <main v-bind="$attrs">...</main>
  <footer>...</footer>
</template>
```

复制

### **Teleport**

`Teleport` 是一种能够将我们的模板移动到 `DOM` 中 `Vue app` 之外的其他位置的技术，就有点像哆啦A梦的“任意门”

在`vue2`中，像 `modals`,`toast` 等这样的元素，如果我们嵌套在 `Vue` 的某个组件内部，那么处理嵌套组件的定位、`z-index` 和样式就会变得很困难

通过`Teleport`，我们可以在组件的逻辑位置写模板代码，然后在 `Vue` 应用范围之外渲染它

```javascript
<button @click="showToast" class="btn">打开 toast</button>
<!-- to 属性就是目标位置 -->
<teleport to="#teleport-target">
    <div v-if="visible" class="toast-wrap">
        <div class="toast-msg">我是一个 Toast 文案</div>
    </div>
</teleport>
```

复制

### **createRenderer**

通过`createRenderer`，我们能够构建自定义渲染器，我们能够将 `vue` 的开发模型扩展到其他平台

我们可以将其生成在`canvas`画布上

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/5jzptplxjf.png?imageView2/2/w/1620)

关于`createRenderer`，我们了解下基本使用，就不展开讲述了

```javascript
import { createRenderer } from '@vue/runtime-core'

const { render, createApp } = createRenderer({
  patchProp,
  insert,
  remove,
  createElement,
  // ...
})

export { render, createApp }

export * from '@vue/runtime-core'
```

复制

### **composition Api**

composition Api，也就是组合式`api`，通过这种形式，我们能够更加容易维护我们的代码，将相同功能的变量进行一个集中式的管理

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/3ai7idga0j.png?imageView2/2/w/1620)

关于`compositon api`的使用，这里以下图展开

![img](https://ask.qcloudimg.com/http-save/yehe-2451713/qcio1ucmao.png?imageView2/2/w/1620)

简单使用:

```javascript
export default {
    setup() {
        const count = ref(0)
        const double = computed(() => count.value * 2)
        function increment() {
            count.value++
        }
        onMounted(() => console.log('component mounted!'))
        return {
            count,
            double,
            increment
        }
    }
}
```

复制

### **三、非兼容变更**

### **Global API**

- 全局 `Vue API` 已更改为使用应用程序实例
- 全局和内部 `API` 已经被重构为可 `tree-shakable`

### **模板指令**

- 组件上 `v-model` 用法已更改
- `<template v-for>`和 非 `v-for`节点上`key`用法已更改
- 在同一元素上使用的 `v-if` 和 `v-for` 优先级已更改
- `v-bind="object"` 现在排序敏感
- `v-for` 中的 `ref` 不再注册 `ref` 数组

### **组件**

- 只能使用普通函数创建功能组件
- `functional` 属性在单文件组件 `(SFC)`
- 异步组件现在需要 `defineAsyncComponent` 方法来创建

### **渲染函数**

- 渲染函数`API`改变

- scopedSlotsproperty已删除，所有插槽都通过slots 作为函数暴露

- 自定义指令 API 已更改为与组件生命周期一致

- 一些转换 

  ```
  class
  ```

   被重命名了：

  - `v-enter` -> `v-enter-from`
  - `v-leave` -> `v-leave-from`

- 组件 `watch` 选项和实例方法 `$watch`不再支持点分隔字符串路径，请改用计算函数作为参数

- 在 `Vue 2.x` 中，应用根[容器](https://cloud.tencent.com/product/tke?from=10680)的 `outerHTML` 将替换为根组件模板 (如果根组件没有模板/渲染选项，则最终编译为模板)。`VUE3.x` 现在使用应用程序容器的 `innerHTML`。

### **其他小改变**

- `destroyed` 生命周期选项被重命名为 `unmounted`
- `beforeDestroy` 生命周期选项被重命名为 `beforeUnmount`
- `[prop default`工厂函数不再有权访问 `this` 是上下文
- 自定义指令 API 已更改为与组件生命周期一致
- `data` 应始终声明为函数
- 来自 `mixin` 的 `data` 选项现在可简单地合并
- `attribute` 强制策略已更改
- 一些过渡 `class` 被重命名
- 组建 watch 选项和实例方法 `$watch`不再支持以点分隔的字符串路径。请改用计算属性函数作为参数。
- `<template>` 没有特殊指令的标记 (`v-if/else-if/else`、`v-for` 或 `v-slot`) 现在被视为普通元素，并将生成原生的 `<template>` 元素，而不是渲染其内部内容。
- 在`Vue 2.x` 中，应用根容器的 `outerHTML` 将替换为根组件模板 (如果根组件没有模板/渲染选项，则最终编译为模板)。`Vue 3.x` 现在使用应用容器的 `innerHTML`，这意味着容器本身不再被视为模板的一部分。

### **移除 API**

- `keyCode` 支持作为 `v-on` 的修饰符
- on，off和
- 过滤`filter`
- 内联模板 `attribute`
- `$destroy` 实例方法。用户不应再手动管理单个`Vue` 组件的生命周期。

## **参考文献**

- https://vue3js.cn/docs/zh
- https://cloud.tencent.com/developer/article/1794328
