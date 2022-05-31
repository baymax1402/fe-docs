# ES6新特性

- 解决原有语法上的不足  let const的块级作用域
- 对原有语法进行增强   比如解构、展开、参数默认值、模板字符串
- 全新对象、方法、功能  promise、proxy、object的assign、is
- 全新的数据类型和数据结构   symbol、map、set



## 1.解决原有语法不足

### 1.1let、const、var的区别

- 和var的区别

| 声明方式 | 变量提升 | 作用域 | 初始值 | 重复定义 |
| -------- | -------- | ------ | ------ | -------- |
| Var      | 是       | 函数级 | 不需要 | 允许     |
| Let      | 否       | 块级   | 不需要 | 不允许   |
| Const    | 否       | 块级   | 必需   | 不允许   |

- Let、const声明的变量，在for、if中会形成块级作用域，块级作用域内的变量不能被作用域外部使用
- Let、const声明变量不会有声明提升，在变量声明之前使用运行时会报错
- 块级作用域声明变量，会出现”暂时性死区“, 声明前使用变量，将会报错
- const声明的是一个常量，声明必需初始化
- 如果const声明的是基本类型常量，初始化之后不能修改；引用类型的常量，可以修改其成员变量



**临时性死区:** 在当前的执行上下文中，会进行变量提升，但未初始化，所以在上下文执行阶段，执行代码如果还没有执行到变量赋值，引用此变量就会报错，此变量并未初始化





## 2.对原有语法增强

### 2.1rest参数&函数形参默认值

用途：获取函数的多余参数，不需要使用arguments对象了

用法：1）搭配的变量是一个数组，该变量将多余的参数放入数组中

​			2）rest参数之后不能有其他参数

​            3）函数的length属性，不包括rest参数

​            4）箭头函数不存在arguments, 可以用rest参数代替



浅拷贝：不复制继承的属性或类的属性，但是它会复制ES6的Symbols属性

### 2.2字符串扩展方法

Includes---是否包含   startsWith---是否以。。。开始  endsWith ---是否以。。。结束 



### 2.3解构赋值-快速提取数组/对象中的元素

#### 数组解构

- 根据索引单独解构
- 解构时可以给变量设置默认值
- ...变量名 解构剩余参数到新数组，只能用一次

#### 对象解构

- 根据key单独或多个解构
- 给解构出来的变量重命名
- 给解构变量设置默认值



### 2.4模板字符串

用法：用``将字符串包裹起来

功能：可以换行、插值、使用标签函数进行字符串操作



### 2.5数组扩展方法

#### 2.5.1  ...arr  、扩展运算符的应用

#### 2.5.2  构造函数新增的方法

Array.from() 将两类对象转为真正的数组：





### 2.6箭头函数

#### 什么是箭头函数

ES6中允许使用箭头=>来定义箭头函数

相较普通函数语法简洁，省去了function关键字

#### 与普通函数的区别

- 语法更加简洁、清晰

- 箭头函数不会创建自己的this

  箭头函数没有自己的this, 它会捕获自己在定义时所处的外层执行环境的this, 并继承这个this值。所以箭头函数中this的指向在它被定义时已经确定了，之后永远不会改变

- 箭头函数继承而来的this指向永远不变

- call/apply/bind无法改变箭头函数中this的指向

- 箭头函数不能作为构造函数使用

- 箭头函数没有自己的arguments

  在箭头函数中访问arguments实际上获得的是外层局部（函数）执行环境中的值

  箭头函数没有原型prototype

- 箭头函数不能用作generator函数，不能使用yield关键字

#### 好处和优势

- 简化了函数的写法
- 没有this机制，this继承自上一个函数的上下文，如果上一层没有函数，则指向window
- 作为异步回调函数时，可解决this指向问题



### 2.7对象字面量增强

- 同名属性可以省略key:value形式，直接key
- 函数可以省略key:value形式
- 可以直接func()
- 可以使用计算属性 {[Math.random()]: value}



## 3.全新的对象、方法、功能

### 3.1 Object.assign(target1, target2, ...) 

作用: 复制合并对象   浅拷贝

### 3.2 Object.is(value1, value2) 

作用：比较两个值是否相等

特性:

- 没有隐式转换
- 可以比较+0, -0、NaN

### 3.3 Proxy(object, handler)

作用：代理一个对象的所有，包括读写操作和各种操作的监听

#### 与Object.defineProperty的区别：

Object.defineProperty产生三个主要问题:

- 不能监听数组的变化
- 必须遍历对象的每个属性
- 必须深层遍历嵌套的对象

Proxy

- 针对的整个对象，Object.defineProperty针对单个属性，这就解决了需要对对象进行深度递归实现对每个属性劫持的问题
- 解决了Object.defineProperty无法劫持数组的问题
- 比Object.defineProperty有更多的拦截方法，对比一些新的浏览器，可能会对Proxy进行新的优化，有助于性能提升

### 3.4 Reflect

不是构造函数，创建时不是用new来进行创建

#### 增加该对象的目的：

- 将Object对象的一些明显属于语言层面的方法放到Reflect对象上

  Object.defineProperty

- 修改某些Object方法的返回结果，使其更加合理

  Object.defineProperty无法定义属性时会抛出一个错误，而 Reflect.defineProperty返回false

- 让对象操作都变成函数行为

​       name in obj   Reflect.has(obj, name)

- Reflect对象的方法和Proxy对象的方法一一对应，只要是Proxy对象的方法，都能在Reflect对象上找到对应的方法。这就让Proxy对象可以方便地调用对应的Reflect方法完成默认行为

#### Reflect提供了以下的静态方法：

**1、Reflect.apply ( target, thisArgument, argumentsList )**

等同于Function.prototype.apply.call( target, thisArgument, argumentsList )。一般来说，如果要绑定一个函数的this对象，可以写成fn.apply(obj, args)。但如果函数自己定义了apply方法，就只能写成Function.prototype.apply.call(fn, obj, args)。而采用Reflect对象可以简化这种操作。

```js
var obj = { a: 1 };
function fun(b) {
    console.log(this.a + b)
}
Reflect.apply(fun, obj, [2]); // 3

// 等价于
fun.apply(obj, [2]);
```

如果函数没有参数，那么Reflect.apply第三个参数应传入空数组。



**2、Reflect.construct ( target, argumentsList [ , newTarget ] )**

这里提供了一种不使用new来调用构造函数的方法。

```js
new Array(1,2,3); // [1,2,3] 
// 等价于
Reflect.construct(Array, [1,2,3]); // [1,2,3]
```



**3、Reflect.defineProperty ( target, propertyKey, attributes )**

用于定义或修改对象属性，返回一个布尔值表示是否操作成功。其对应的Object方法如果操作失败的话，会抛出异常。

```js
var obj = { a: 1 };
Object.getOwnPropertyDescriptor(obj, 'a'); 
// {value: 1, writable: true, enumerable: true, configurable: true}
// 现在将其修改为不可配置
Object.defineProperty(obj, 'a', {configurable: false});
// 再将其改回可配置
Object.defineProperty(obj, 'a', {configurable: true});
// TypeError: Cannot redefine property
// 如果使用Reflect.defineProperty将返回一个布尔值，而不是抛出异常。
Reflect.defineProperty(obj, 'a', {configurable: true}); // false
```



**4、Reflect.deleteProperty ( target, propertyKey )**

该方法主要是将Object操作变成函数行为。等同于delete obj[name];

```js
var obj = { a: 1 };
Reflect.deleteProperty(obj, 'a'); // true
obj.a; // undefined
```



**5、Reflect.get ( target, propertyKey [ , receiver ] )**

查找并返回target对象的propertyKey属性。如果没有该属性，则返回undefined。如果propertyKey属性部署了读取函数，则读取函数的this绑定到receiver。

```js
var obj = {
    get foo() { return this.bar(); },
    bar() { return 1; }
};

Reflect.get(obj, 'foo'); // 1

var wrapper = { bar() { return 2; }};
Reflect.get(obj, 'foo', wrapper); // 2
```



**6、Reflect.getOwnPropertyDescriptor ( target, propertyKey )**

等同于Object.getownPropertyDescriptor(target, propertyKey);

```text
var obj = { a: 1 };
Object.getOwnPropertyDescriptor(obj, 'a');
// {value: 1, writable: true, enumerable: true, configurable: true}
Reflect.getOwnPropertyDescriptor(obj, 'a');
// {value: 1, writable: true, enumerable: true, configurable: true}
```



**7、Reflect.getPrototypeOf ( target )**

获取对象的原型。相当于Object.getPrototypeOf(target);

```js
var obj = {};
Object.getPrototypeOf(obj) === Object.prototype;  // true
Reflect.getPrototypeOf(obj) === Object.prototype; // true
```



**8、Reflect.has ( target, propertyKey )**

相当于propertyKey in target，将该操作变成函数行为。

```js
var obj = { a:1 };
'a' in obj; // true
Reflect.has(obj, 'a'); // true
```



**9、Reflect.isExtensible ( target )**

等同于Object.isExtensible ( target ).

```text
var obj = {};
Object.isExtensible(obj);  // true
Reflect.isExtensible(obj); // true
```



**10、Reflect.ownKeys ( target )**

等同于Object.getOwnPropertyNames(target)和Object.getOwnPropertySymbols(target)的返回值组合。

```js
var obj = { a: 1 };
obj[Symbol('b')] = 2;
Object.getOwnPropertyNames(obj); // ['a'];
Object.getOwnPropertySymbols(obj); // [Symbol(b)]
Reflect.ownKeys(obj); // ["a", Symbol(b)]
```



**11、Reflect.preventExtensions ( target )**

禁止对象扩展，相当于Object.preventExtensions ( target )

```js
var obj = { a: 1 };
Reflect.preventExtensions ( obj );
obj.b = 2;
obj.b; // undefined
```



**12、Reflect.set ( target, propertyKey, V [ , receiver ] )**

设置target对象propertyKey属性的值等于V。如果propertyKey属性设置了赋值函数，则赋值函数的this绑定到receiver上。

```js
var obj = { a: 1 };
Reflect.set(obj, 'a', 2);
obj.a; // 2

var obj2 = {
    set foo(a) {
        this.a = a;
    }
};
var wrapper = { a: 3 };
Reflect.set(target, 'foo', 4, wrapper);
wrapper.a; // 4
```



**13、Reflect.setPrototypeOf ( target, proto )**

将target的原型对象设置为proto，等同于Object.setPropertyOf方法。

```js
var array = {};
Reflect.setPrototypeOf(array, Array.prototype);
array.length; // 0
```

### 3.5 Promise

定义：

> Promise对象是一个代理对象，被代理的值在Promise对象创建时可能是未知的。它允许你为异步操作的成功和失败分别绑定相应的处理方法（handlers）。这让异步方法可以像同步方法那样返回值， 但不是立即返回最终执行结果，而是一个能代表未来出现的结果的promise对象

作用：解决异步编程中回调嵌套过深问题

#### Async/await

说明：是generator + promise的语法糖

关系：await必须在async内使用，并修饰一个Promise对象，async返回的也是一个promise对象

​          Async/await中的return/throw会代理自己返回的promise的resolve/reject, 而一个Promise的resolve/reject会使得await得到返回值或抛出异常

- 方法内无await节点
  - return 一个字面量则会得到一个{PromiseStatus: resolved}的Promise
  - Throw 一个Error则会得到一个{PromiseStatus: rejected}的Promise
- 方法内有await节点
  - async return 一个{PromiseStatus: pending}的Promise (发生切换，异步等待Promise的执行结果)
  - Promise的resolve会使得await代码节点获得相应返回结果，并继续向下执行
  - Promise的reject会使得await代码自动抛出相应异常，终止向下继续执行



### 3.6  class&静态方法&继承

#### 定义 

使用class关键字定义类

#### 方法

- 实例方法：需要实例化之后才可以调用，this指向实例
- 静态方法：用static修饰符修饰，可以直接通过类名调用, 不需要实例化，this不指向实例, 而是指向当前类

#### 继承

子类使用extends关键字实现继承，可以继承父类所有属性（包括静态属性和静态方法）

#### 与ES5区别

es5中主要是通过构造函数方式和原型方式来定义一个类，es6中可以使用class定义

- Class类必须new调用，不能直接执行
- class类没有变量提升
- class类无法遍历它实例原型链上的属性和方法
- es6为new命令引入了一个new.target属性, 他会返回new命令作用于的那个构造函数。如果不是通过new调用或Reflect.construct()调用的，new.target()会返回undefined
- class类有static静态方法， 类调用时this指向类

### 3.7 模块化

#### 3.7.1概念

模块：能单独命名并独立完成一定功能的程序语句的集合（即程序代码和数据结构的集合体）

特征：

- 外部特征：模块和外部环境联系的接口 （调用模块的方式：输入输出参数、引用的全局变量）和模块的功能
- 内部特征：模块内部环境具有的特点 （该模块的局部数据和程序代码）

#### 3.7.2 发展背景





#### common.js 和 es6中模块引入的区别

commonJS是一种模块规范，最初被应用于Nodejs, 成为Nodejs的模块规范

浏览器端的js也实现了一套相同的模块规范(例如：AMD), 用于对前端模块进行管理

自ES6起，引入了一套新的ES6 Module规范，在语言标准层面上实现了模块功能，而且实现得相当简单，有望成为浏览器和服务器通用的模块解决方案

使用上的差别主要有：

- CommonJS模块输出的是一个值的拷贝， ES6输出的是值的引用；CommonJS模块是运行时加载，ES6模块是编译时输出接口
- CommonJs是单个值导出，ES6 可以导出多个
- CommonJs是动态语法可以写在判断里, ES6 静态语法只能写在顶层; CommonJs的this是当前模块, ES6 Module的this是undefined



## 4.全新的数据结构和数据类型

### 4.1Map和Set

#### 1）区别

- 两种方法都具有极快的查找速度  Set执行时间最短，查找速度最快
- 初始化需要的值不一样 Map需要一个二维数组，Set需要一个一维Array数组
- 两者都不允许键重复
- Map的键是不能修改，但键对应的值是可以修改的；Set不能通过迭代器来改变值，因为Set的值就是键
- Map是键值对的存在，值也不作为键；而Set没有value只有key, value就是key

#### 2）Set

说明：是一种类似于数组的数据结构

特性：

- 元素唯一性，不允许重复元素
- 使用add添加重复元素，将会被忽略

用途：

- 数组去重
- 数据存储

#### 3）Map

说明：类似于Object, 以key、value形式存储数据

区别：Map键不会隐式转换成字符串，而是保持原有类型

### 4.2 Symbol

背景：对象属性名是字符串，容易造成属性名的冲突，引入Symbol 使属性独一无二，防止冲突

前六种：undefined、null、Boolean、String、Number、Object

说明：第七种原始数据类型，用来定义一个唯一的变量

作用：

- 创建惟一的变量，解决对象键名重复问题
- 为对象、类、函数等创建私有属性
- 修改对象的toString标签
- 为对象添加迭代器属性

如何获取对象的symbol属性

- Object.getOwnPropertySymbols(object)

### 4.3 for ... of  

说明：以统一的方式，遍历所有引用数据类型

特性：可以随时使用break终止遍历，而forEach不行

### 4.4 迭代器模式

作用：通过Symbol.iterator 对外提供统一的接口，获取内部的数据

### 4.5 Generator生成器

- 函数前加* 生成一个生成器
- 一般配合yield关键字使用
- 惰性执行，调next才会往下执行
- 主要用来解决异步回调过深的问题

### 4.6 装饰器Decorator

#### 4.6.1 介绍

装饰器模式：就是一种在不改变原类和使用继承的情况下，动态地扩展对象功能的设计理论

ES6中的Decorator就是一个普通的函数，用于扩展类属性和类方法

```
@testable
class MyTestableClass {
  // ...
}

function testable(target) {
  target.isTestable = true;
}

MyTestableClass.isTestable // true
```

两大优点：

- 代码可读性变强了，命名相当于一个注释
- 在不改变原有代码情况下，对原来功能进行扩展

#### 4.6.2 用法

分为两种：类的装饰和类属性的装饰

##### 类的装饰

装饰器可以用来装饰整个类。

```javascript
@testable
class MyTestableClass {
  // ...
}

function testable(target) {
  target.isTestable = true;
}

MyTestableClass.isTestable // true
```

上面代码中，`@testable`就是一个装饰器。它修改了`MyTestableClass`这个类的行为，为它加上了静态属性`isTestable`。`testable`函数的参数`target`是`MyTestableClass`类本身。

基本上，装饰器的行为就是下面这样。

```javascript
@decorator
class A {}

// 等同于

class A {}
A = decorator(A) || A;
```

也就是说，装饰器是一个对类进行处理的函数。装饰器函数的第一个参数，就是所要装饰的目标类。

```javascript
function testable(target) {
  // ...
}
```

上面代码中，`testable`函数的参数`target`，就是会被装饰的类。

如果觉得一个参数不够用，可以在装饰器外面再封装一层函数。

```javascript
function testable(isTestable) {
  return function(target) {
    target.isTestable = isTestable;
  }
}

@testable(true)
class MyTestableClass {}
MyTestableClass.isTestable // true

@testable(false)
class MyClass {}
MyClass.isTestable // false
```

上面代码中，装饰器`testable`可以接受参数，这就等于可以修改装饰器的行为。

注意，装饰器对类的行为的改变，是代码编译时发生的，而不是在运行时。这意味着，装饰器能在编译阶段运行代码。也就是说，装饰器本质就是编译时执行的函数。

前面的例子是为类添加一个静态属性，如果想添加实例属性，可以通过目标类的`prototype`对象操作。

```javascript
function testable(target) {
  target.prototype.isTestable = true;
}

@testable
class MyTestableClass {}

let obj = new MyTestableClass();
obj.isTestable // true
```

上面代码中，装饰器函数`testable`是在目标类的`prototype`对象上添加属性，因此就可以在实例上调用。

下面是另外一个例子。

```javascript
// mixins.js
export function mixins(...list) {
  return function (target) {
    Object.assign(target.prototype, ...list)
  }
}

// main.js
import { mixins } from './mixins.js'

const Foo = {
  foo() { console.log('foo') }
};

@mixins(Foo)
class MyClass {}

let obj = new MyClass();
obj.foo() // 'foo'
```

上面代码通过装饰器`mixins`，把`Foo`对象的方法添加到了`MyClass`的实例上面。可以用`Object.assign()`模拟这个功能。

```javascript
const Foo = {
  foo() { console.log('foo') }
};

class MyClass {}

Object.assign(MyClass.prototype, Foo);

let obj = new MyClass();
obj.foo() // 'foo'
```

实际开发中，React 与 Redux 库结合使用时，常常需要写成下面这样。

```javascript
class MyReactComponent extends React.Component {}

export default connect(mapStateToProps, mapDispatchToProps)(MyReactComponent);
```

有了装饰器，就可以改写上面的代码。

```javascript
@connect(mapStateToProps, mapDispatchToProps)
export default class MyReactComponent extends React.Component {}
```

相对来说，后一种写法看上去更容易理解。

##### 类属性的装饰

装饰器不仅可以装饰类，还可以装饰类的属性。

```javascript
class Person {
  @readonly
  name() { return `${this.first} ${this.last}` }
}
```

上面代码中，装饰器`readonly`用来装饰“类”的`name`方法。

装饰器函数`readonly`一共可以接受三个参数。

```javascript
function readonly(target, name, descriptor){
  // descriptor对象原来的值如下
  // {
  //   value: specifiedFunction,
  //   enumerable: false,
  //   configurable: true,
  //   writable: true
  // };
  descriptor.writable = false;
  return descriptor;
}

readonly(Person.prototype, 'name', descriptor);
// 类似于
Object.defineProperty(Person.prototype, 'name', descriptor);
```

**装饰器第一个参数是类的原型对象**，上例是`Person.prototype`，装饰器的本意是要“装饰”类的实例，但是这个时候实例还没生成，所以只能去装饰原型（这不同于类的装饰，那种情况时`target`参数指的是类本身）；**第二个参数是所要装饰的属性名，第三个参数是该属性的描述对象**。

装饰器有注释的作用。如果同一个方法有多个装饰器，会像剥洋葱一样，先从外到内进入，然后由内向外执行。

```javascript
function dec(id){
  console.log('evaluated', id);
  return (target, property, descriptor) => console.log('executed', id);
}

class Example {
    @dec(1)
    @dec(2)
    method(){}
}
// evaluated 1
// evaluated 2
// executed 2
// executed 1
```

上面代码中，外层装饰器`@dec(1)`先进入，但是内层装饰器`@dec(2)`先执行。

除了注释，装饰器还能用来类型检查。所以，对于类来说，这项功能相当有用。从长期来看，它将是 JavaScript 代码静态分析的重要工具。

##### 注意

不能用于修饰函数，因为函数存在变量提升情况

#### 4.6.3 使用场景

##### core-decorators

**@readonly**

`readonly`装饰器使得属性或方法不可写。

```javascript
import { readonly } from 'core-decorators';

class Meal {
  @readonly
  entree = 'steak';
}

var dinner = new Meal();
dinner.entree = 'salmon';
// Cannot assign to read only property 'entree' of [object Obje
```

##### React-redux  @connect 

##### Mixins

```javascript
import { mixins } from './mixins.js';

const Foo = {
  foo() { console.log('foo') }
};

@mixins(Foo)
class MyClass {}

let obj = new MyClass();
obj.foo() // "foo"
```

## 5.ES2016与ES2017

### 5.1 includes() -2016

判断数组是否包含某个元素

### 5.2 运算符-2016

Math.pow(2, 10)

### 5.3  values() - 2017

Object.values()  对象的值以数组的形式返回

### 5.4 entries() - 2017

将对象以键值对二维数组返回  使之可以使用for...of遍历

### 5.5 Object.getOwnPropertyDesriptor - 2017

### 5.6 padStart, padEnd - 2017

在字符串前/后追加指定字符串

参数：(targetLength, padString)  填充后的目标长度，填充的字符串

用途：一般用来对齐字符串输出







