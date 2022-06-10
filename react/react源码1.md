# 1.开篇介绍以及面试题

## 课程特色

本课程遵循react 源码的核心思想，目的是打造一门通俗易懂的react源码解析系列文章。课程从源码的各个模块开始，结合大量demo、示例图解和视频教程，带着大家一步一步调试react源码，课程也会完全遵循react手写hook和精简版的react方便大家的理解，随着react大版本的更新，此课程也会一直更新。

## 课程收获

为什么要学习react源码呢，你是使用了api很久，停留在框架使用层面还是会主动去了解框架的逻辑和运行方式呢。跟着课程走，理解起来不费力，你将收获：

- 面试加分：大厂前端岗都要求熟悉框架底层原理，也是面试必问环节，熟悉react源码会为你的面试加分，也会为你的谈薪流程增加不少筹码
- 巩固基础知识：在源码的scheduler中使用了***小顶堆*** 这种数据结构，调度的实现则使用了***messageChannel***，在render阶段的reconciler中则使用了***fiber、update、链表*** 这些结构，lane模型使用了二进制掩码，我们也会介绍diff算法怎样降低对比复杂度。学习本课程也顺便巩固了数据结构和算法、事件循环。
- **日常开发提升效率**：熟悉react源码之后，你对react的运行流程有了新的认识，在日常的开发中，相信你对组件的性能优化、react使用技巧和解决bug会更加得心应手。



## 常见面试题（带上问题学习吧）

以下这些问题可能你已经有答案了，但是你能从源码角度回答出来吗。

1. jsx和Fiber有什么关系
2. react17之前jsx文件为什么要声明import React from 'react'，之后为什么不需要了
3. Fiber是什么，它为什么能提高性能

**hooks**

1. 为什么hooks不能写在条件判断中

**状态/生命周期**

1. setState是同步的还是异步的
2. componentWillMount、componentWillMount、componentWillUpdate为什么标记UNSAFE

**组件**

1. react元素$$typeof属性什么
2. react怎么区分Class组件和Function组件
3. 函数组件和类组件的相同点和不同点

**开放性问题**

1. 说说你对react的理解/请说一下react的渲染过程
2. 聊聊react生命周期
3. 简述diff算法
4. react有哪些优化手段
5. react为什么引入jsx
6. 说说virtual Dom的理解
7. 你对合成事件的理解
   1. 我们写的事件是绑定在`dom`上么，如果不是绑定在哪里？
   2. 为什么我们的事件手动绑定`this`(不是箭头函数的情况)
   3. 为什么不能用 `return false `来阻止事件的默认行为？
   4. `react`怎么通过`dom`元素，找到与之对应的 `fiber`对象的？

**解释结果和现象**

1. 点击Father组件的div，Child会打印Child吗

   ```js
   function Child() {
     console.log('Child');
     return <div>Child</div>;
   }
       
       
   function Father(props) {
     const [num, setNum] = React.useState(0);
     return (
       <div onClick={() => {setNum(num + 1)}}>
         {num}
         {props.children}
       </div>
     );
   }
       
       
   function App() {
     return (
       <Father>
         <Child/>
       </Father>
     );
   }
       
   const rootEl = document.querySelector("#root");
   ReactDOM.render(<App/>, rootEl);
   ```

2. 打印顺序是什么

   ```js
   function Child() {
     useEffect(() => {
       console.log('Child');
     }, [])
     return <h1>child</h1>;
   }
       
   function Father() {
     useEffect(() => {
       console.log('Father');
     }, [])
         
     return <Child/>;
   }
       
   function App() {
     useEffect(() => {
       console.log('App');
     }, [])
       
     return <Father/>;
   }
   ```

3. useLayoutEffect/componentDidMount和useEffect的区别是什么

   ```js
   class App extends React.Component {
     componentDidMount() {
       console.log('mount');
     }
   }
       
   useEffect(() => {
     console.log('useEffect');
   }, [])
   ```

4. 如何解释demo_4、demo_8、demo_9出现的现象

# 2.react的设计理念

#### 异步可中断

- **React15慢在哪里**

在讲这部分之前，需要讲是那些因素导致了react变慢，并且需要重构呢。

React15之前的协调过程是同步的，也叫stack reconciler，又因为js的执行是单线程的，这就导致了在更新比较耗时的任务时，不能及时响应一些高优先级的任务，比如用户的输入，所以页面就会卡顿，这就是cpu的限制。

- **解决方案**

如何解决这个问题呢，试想一下，如果我们在日常的开发中，在单线程的环境中，遇到了比较耗时的代码计算会怎么做呢，首先我们可能会将任务分割，让它能够被中断，在其他任务到来的时候让出执行权，当其他任务执行后，再从之前中断的部分开始异步执行剩下的计算。所以关键是实现一套异步可中断的方案。

- **实现**

在刚才的解决方案中提到了任务分割，和异步执行，并且能让出执行权，由此可以带出react中的三个概念

1. Fiber：react15的更新是同步的，因为它不能将任务分割，所以需要一套数据结构让它既能对应真实的dom又能作为分隔的单元，这就是Fiber。

   ```js
   let firstFiber
   let nextFiber = firstFiber
   let shouldYield = false
   //firstFiber->firstChild->sibling
   function performUnitOfWork(nextFiber){
     //...
     return nextFiber.next
   }
      
   function workLoop(deadline){
     while(nextFiber && !shouldYield){
             nextFiber = performUnitOfWork(nextFiber)
             shouldYield = deadline.timeReaming < 1
           }
     requestIdleCallback(workLoop)
   }
      
   requestIdleCallback(workLoop)
   ```

2. Scheduler：有了Fiber，我们就需要用浏览器的时间片异步执行这些Fiber的工作单元，我们知道浏览器有一个api叫做**requestIdleCallback**，它可以在浏览器空闲的时候执行一些任务，我们用这个api执行react的更新，让高优先级的任务优先响应不就可以了吗，但事实是requestIdleCallback存在着浏览器的兼容性和触发不稳定的问题，**所以我们需要用js实现一套时间片运行的机制，在react中这部分叫做scheduler。**

3. Lane：有了异步调度，我们还需要细粒度的管理各个任务的优先级，让高优先级的任务优先执行，各个Fiber工作单元还能比较优先级，相同优先级的任务可以一起更新，想想是不是更cool呢。

- **产生出来的上层实现**

  由于有了这一套**异步可中断的机制，我们就能实现batchedUpdates批量更新和Suspense**

下面这两张图就是使用异步可中断更新前后的区别，可以体会一下

![react源码2.2](https://xiaochen1024.com/react%E6%BA%90%E7%A0%812.2.png)

![react源码2.3](https://xiaochen1024.com/react%E6%BA%90%E7%A0%812.3.png)

#### 代数效应（Algebraic Effects）

除了cpu的瓶颈问题，还有一类问题是和副作用相关的问题，比如获取数据、文件操作等。不同设备性能和网络状况都不一样，react怎样去处理这些副作用，让我们在编码时最佳实践，运行应用时表现一致呢，这就需要react有分离副作用的能力，为什么要分离副作用呢，因为要解耦，这就是代数效应。

提问：我们都写过获取数据的代码，在获取数据前展示loading，数据获取之后取消loading，假设我们的设备性能和网络状况都很好，数据很快就获取到了，那我们还有必要在一开始的时候展示loading吗？如何才能有更好的用户体验呢？

看下下面这个例子

```js
function getPrice(id) {
  return fetch(`xxx.com?id=${productId}`).then((res)=>{
    return res.price
  })
}

async function getTotalPirce(id1, id2) {
  const p1 = await getPrice(id1);
  const p2 = await getPrice(id2);

  return p1 + p2;
}

async function run(){
	await getTotalPrice('001', '002');  
}
```

getPrice是一个异步获取数据的方法，我们可以用async+await的方式获取数据，但是这会导致调用getTotalPrice的run方法也会变成异步函数，这就是async的传染性，所以没法分离副作用。

```js
function getPrice(id) {
  const price = perform id;
  return price;
}

function getTotalPirce(id1, id2) {
  const p1 = getPrice(id1);
  const p2 = getPrice(id2);

  return p1 + p2;
}

try {
  getTotalPrice('001', '002');
} handle (productId) {
  fetch(`xxx.com?id=${productId}`).then((res)=>{
    resume with res.price
  })
}
```

现在改成下面这段代码，其中perform和handle是虚构的语法，当代码执行到perform的时候会暂停当前函数的执行，并且被handle捕获，handle函数体内会拿到productId参数获取数据之后resume价格price，resume会回到之前perform暂停的地方并且返回price，这就完全把副作用分离到了getTotalPirce和getPrice之外。

这里的关键流程是perform暂停函数的执行，handle获取函数执行权，resume交出函数执行权。

但是这些语法毕竟是虚构的，但是请看下下面的代码

```js
function usePrice(id) {
  useEffect((id)=>{
      fetch(`xxx.com?id=${productId}`).then((res)=>{
        return res.price
  	})
  }, [])
}

function TotalPirce({id1, id2}) {
  const p1 = usePrice(id1);
  const p2 = usePrice(id2);

  return <TotalPirce props={...}>
}
```

如果把getPrice换成usePrice，getTotalPirce换成TotalPirce组件，是不是有点熟悉呢，这就是hook分离副作用的能力。

我们知道generator也可以做到程序的暂停和恢复啊，那用generator不行就行了吗，但是generator暂停之后的恢复执行，还是得把执行权交换给直接调用者，调用者会沿着调用栈继续上交，所以也是有传染性的，并且generator不能计算优先级，排序优先级。

```js
function getPrice(id) {
  return fetch(`xxx.com?id=${productId}`).then((res)=>{
    return res.price
  })
}

function* getTotalPirce(id1, id2) {
  const p1 = yield getPrice(id1);
  const p2 = yield getPrice(id2);

  return p1 + p2;
}

function* run(){
	yield getTotalPrice('001', '002');  
}
```

解耦副作用在函数式编程的实践中非常常见，例如redux-saga，将副作用从saga中分离，自己不处理副作用，只负责发起请求

```js
function* fetchUser(action) {
   try {
      const user = yield call(Api.fetchUser, action.payload.userId);
      yield put({type: "USER_FETCH_SUCCEEDED", user: user});
   } catch (e) {
      yield put({type: "USER_FETCH_FAILED", message: e.message});
   }
}
```

严格意义上讲react是不支持Algebraic Effects的，但是react有Fiber啊，执行完这个Fiber的更新之后交还执行权给浏览器，让浏览器决定后面怎么调度，由此可见Fiber得是一个链表结构才能达到这样的效果，

Suspense也是这种概念的延伸，后面看到了具体的Suspense的源码就有些感觉了。先看个例子

```js
const ProductResource = createResource(fetchProduct);

const Proeuct = (props) => {
    const p = ProductResource.read( // 用同步的方式来编写异步代码!
          props.id
    );
  return <h3>{p.price}</h3>;
}

function App() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <Proeuct id={123} />
      </Suspense>
    </div>
  );
}
```

可以看到ProductResource.read完全是同步的写法，把获取数据的部分完全分离出了Proeuct组件之外，在源码中，ProductResource.read会在获取数据之前会throw一个特殊的Promise，由于scheduler的存在，scheduler可以捕获这个promise，暂停更新，等数据获取之后交还执行权。ProductResource可以是localStorage甚至是redis、mysql等数据库，也就是组件即服务，可能以后会有server Component的出现。