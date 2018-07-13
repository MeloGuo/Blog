# new 操作符到底干了什么。
前几天无意间发现了刚开始学JavaScript时在知乎写的一些回答，有一个就是讲new操作符到底干了什么。从现在的视角看我当时的回答虽然是正确的，但是在对原理的剖析和细节的理解上还相去甚远。所以借此机会，就想重新梳理一下这一年多来对new操作符理解的变化与成长。

第一次去了解new操作符，是在看《JS高级程序设计（第三版）》这本书时，P145是这样写到的。
> 要创建Person的新实例，必须使用new操作符。以这种方式调用构造函数实际上会经历以下4个步骤：
1. 创建一个新对象；
2. 将构造函数的作用域赋给新对象（因此this就指向了这个新对象）；
3. 执行构造函数中的代码（为这个新对象添加属性）；
4. 返回新对象；

尽管书中是这样描述，但是并没有给出实践的代码，所以我就按照这个步骤自己去实践模拟一个new操作符。代码如下：

```javascript
var mockNew = function (constructor) {
  var o = {} // 创建一个新对象
  constructor.apply(o, Array.prototype.slice.call(arguments, 1)) // 赋作用域 执行代码
  return o // 返回新对象
}
```
然后我们用这个模拟new操作符的函数来创造一个对象试试。

```javascript
var Person = function (name, age) {
  this.name = name
  this.age = age
}

Person.prototype.sayName = function () {
  console.log(this.name)
}

var person1 = mockNew(Person, 'MeloGuo', 22)

console.log(person1.name) // 'MeloGuo'
console.log(person1.age) // 22
person1.sayName() // Uncaught TypeError: person1.sayName is not a function
```
看起来我的模拟函数虽然能访问实例中的属性，但是却不能访问sayName方法，而且当我使用`instanceof`操作符检测时却得到了这样的结果。

```javascript
console.log(person1 instanceof Person) // false
console.log(person1 instanceof Object) // true
```
可见person1并不是Person的实例。当时的我还不知道问题出在哪里，直到学习到原型链时我才直到mockNew函数缺少了一个步骤，即绑定构造函数原型。所以person1实例是无法访问到Person原型中的sayName方法，同时`instanceof`操作符的结果也为`false`。因为`instanceof`操作符是用来检测一个对象在其原型链中是否存在一个构造函数的prototype属性的，而person1的原型链中并不存在Person.prototype，所以返回值为`false`。因此，我们改造mockNew函数如下：

```javascript
var mockNew = function (constructor) {
  var o = {}
  o.__proto__ = constructor.prototype // 绑定构造函数原型，但是生产代码中千万别用.__proto__
  constructor.apply(o, Array.prototype.slice.call(arguments, 1))
  return o
}
```
这时我们再创建实例，然后使用`instanceof`操作符检测一下，同时调用下sayName方法。

```javascript
var person1 = mockNew(Person, 'MeloGuo', 22)

console.log(person1 instanceof Person) // true
console.log(person1 instanceof Object) // true
person1.sayName() // 'MeloGuo'
```

这时看似mockNew函数已经完全模拟了new操作符了，但是当我们尝试下面这种情况时，又出现了问题。

```javascript
function Person (name) {
  this.name = name
  return { age: 22 }
}

var person1 = new Person('MeloGuo')
var person2 = mockNew(Person, 'MeloGuo')

console.log(person1) // {age: 22}
console.log(person2) // Person {name: "MeloGuo"}
console.log(person1 instanceof Person) // false
console.log(person2 instanceof Person) // true
```
什么！！！用new操作符调用的Person构造函数并没有按照预期返回带有name属性并且在Person.prototype上的对象，而是返回了我们手动return的带有age属性的对象，但是我们的mockNew函数是正常返回了。所以我们的mockNew函数中肯定又丢失了一些细节，为了弄清楚，只好硬着头皮去读[ECMA-262](https://www.ecma-international.org/ecma-262/6.0/#sec-new-operator)规范了。看到规范中的steps后才恍然大悟了new操作符的整个执行流程。
![new-oprato](media/15311280174159/new-oprator.png)
简单来说我们在返回对象前缺失了判断返回值类型的步骤。
* 如果构造函数的返回值是值类型，那么就丢弃掉，依然返回构造函数创建的实例。
* 如果构造函数的返回值是引用类型，那么就返回这个引用类型，丢弃构造函数创建的实例。

> 注：如果没有显示return，那么相当于隐式返回了`undefined`，则丢弃它。

吸取了规范中的内容，并且加入ES6语法后，我们新的mockNew函数如下:

```javascript
function mockNew (constructor, ...args) {
  const isPrimitive = result => {
    // 这里就是判断如果是
  }

  const o = Object.create(constructor.prototype)
  const result = constructor.apply(o, args)
  
  return isPrimitive(result) ? o : result
}
```
这时的mockNew函数可以说是较好的模拟了new操作符的功能。

其实new操作符就是一个语法糖。在传统的面向类的语言中，“构造函数”是类中的一些特殊方法，使用new初始化类时会调用类中的构造函数。而JavaScript中的new其实是用来告诉函数，我要以“构造”的方式调用你了，从而向mockNew函数中的流程一样，得到我们的实例。