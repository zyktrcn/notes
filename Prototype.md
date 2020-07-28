# 原型

JavaScript的继承是通过原型实现的，继承与原型是息息相关的，要理解其中一个就需要涉及到另一个的内容。

```
const sub = {}
sub.toString()  // [object Object]
```

上面的例子定义了一个空对象sub，按照表面的信息来看sub.toString这个方法的调用应该是报错的，但最终可以输出[object Object]这个字符串。在sub对象上的toString方法就是通过原型继承而来的，并不是在sub对象内部定义的。

sub对象是使用字面量{}创建的，而作用跟new Object()是一致的，都是通过构造函数创建（这里的构造函数是Object）一个对象。toString方法可能是从Object这里继承下来的，研究的目标就转移到构造函数Object身上了。

```
const sub1 = {} // {}
const sub2 = new Object()   // {}
```

对Object的研究可以先从它的属性着手，下面先尝试使用for..in...去寻找Object的可枚举属性，发现是空的。

```
for (let key in Object) {
    console.log(key)
}
//  
```

然后再通过getOwnPropertyNames获取Object的不可枚举属性，在里面就可以发现所用到的getOwnPropertyNames方法。但是在这个列表中也是没有toString方法可以被继承。

```
Object.getOwnPropertyNames(Object)
/* 
[ 'length',
  'name',
  'prototype',
  'assign',
  'getOwnPropertyDescriptor',
  'getOwnPropertyDescriptors',
  'getOwnPropertyNames',
  'getOwnPropertySymbols',
  'is',
  'preventExtensions',
  'seal',
  'create',
  'defineProperties',
  'defineProperty',
  'freeze',
  'getPrototypeOf',
  'setPrototypeOf',
  'isExtensible',
  'isFrozen',
  'isSealed',
  'keys',
  'entries',
  'values' ]
 */
```

在上面的列表里可以发现有一个prototype（原型）的属性，使用同样的方式去查找Object.prototype里的属性可以发现sub对象继承下来的toString这个方法。

```
for (let key in Object.prototype) {
    console.log(key)
}
//  

Object.getOwnPropertyNames(Object.prototype)
/*
[ 'constructor',
  '__defineGetter__',
  '__defineSetter__',
  'hasOwnProperty',
  '__lookupGetter__',
  '__lookupSetter__',
  'isPrototypeOf',
  'propertyIsEnumerable',
  'toString',
  'valueOf',
  '__proto__',
  'toLocaleString' ]
  */
```

说明sub对象和Object函数的toString方法都是继承自Object.prototype这个对象的。另外，可以通过一个简单的例子来查看sub对象与Object.prototype对象之间的继承关系是否有Object的插入。

```
Object.protoFn = function() {
    console.log('protoFn comes from Object')
}

const sub = {}
sub.protoFn()   // sub.protoFn is not a function
```

在Object构造函数（同时也是一个对象）上定义一个protoFn函数，再创建sub对象并通过sub对象调用protoFn方法，发现并不存在，也就是说sub对象是直接继承Object.prototype对象上的方法（以及属性），这个就是简单的原型继承。

使用Object.prototype下的isPrototypeOf方法也可以验证。

```
const sub = {}
Object.isPrototypeOf(sub)   // false
Object.prototype.isPrototypeOf(sub) // true
Object.prototype.isPrototypeOf(Object)  // true
```

另外，还可以看到Object.prototype对象上有一个属性__proto__，这个属性其实是指向对象继承的原型。（Object.getPrototypeOf也可以找到对象的原型，MDN推荐这种方法。）

而Object.prototype的__proto__属性就是指向null，因为它是没有原型，也没有从其他地方继承属性和方法。

```
const sub = {}
sub.__proto__   // {}
sub.__proto__ === Object.prototype  // true

Object.getPrototypeOf(sub)  // {}
Object.getPrototypeOf(sub) === Object.prototype // true

Object.prototype.__proto__  // null
Object.getPrototypeOf(Object.prototype) // null
```

总结：所有字面量或new Object()创建的对象都会继承Object.prototype上的方法和属性。

上面的Object是一个构造方法，这个构造方法可以换成自定义的函数，但继承的路线就会变更。

```
// ES6以后class Person与function Person是类似的，但前者不可以直接执行。
class Person {
    constructor() {
        this.name = 'xiaohong'
        this.age = 8
    }
}

const xh = new Person() // { name: 'xiaohong', age: 8 }
```

上面的例子创建一个Person类（构造函数），通过new关键词创建一个新的对象，包含name和age属性。

首先从Person着手，通过isPrototypeOf方法找出它的原型。

```
Object.prototype.isPrototypeOf(Person)  // true
Object.isPrototypeOf(Person)    // false
```

这个结果说明了：
1. Person和Object都是继承自Object.prototype，可以使用它的属性和方法。
2. Object和Person不是继承关系，而是同级关系。

由此可以推断所有类（构造函数）都是继承自Object.prototype，可以使用它的属性和方法，以及这些类（构造函数）都是同级关系（不使用extends继承其他类的情况下）。

Object和Person两个类本质上是函数（function Object和function Person），而在JavaScript中函数也是对象，因此满足上面的结论。但如果使用getPrototypeOf方法或者__proto__属性直接获取Object/Person的原型，输出的不是Object.prototype这个对象，而是[Function]。

```
Object.getPrototypeOf(Person)   // [Function]
Object.getPrototypeOf(Person) === Object.prototype  // false
```

这就说明在Person/Object与Object.prototype的继承关系之间还有一个对象。

```
const objectProto = Object.getPrototypeOf(Object)

for (let key in objectProto) {
    console.log(key)
}
//  

Object.getOwnPropertyNames(objectProto)
/*
[ 'length',
  'name',
  'arguments',
  'caller',
  'constructor',
  'apply',
  'bind',
  'call',
  'toString' ]
  */

Object.prototype.isPrototypeOf(objectProto) // true
Object.prototype === Object.getPrototypeOf(objectProto)   // true
```

使用之前的方法获取Object构造函数原型对象的不可枚举属性，可以发现函数专属的属性和方法（arguments和apply等）。可以猜测函数都是继承自这个原型对象，并且这个原型对象的原型就是Object.prototype。

找到这个原型对象前提是了解函数的创建，因为上面的猜测是函数都继承自这个对象。函数的创建分为三种：函数声明、函数字面量和构造函数。

```
// 函数声明
function fn() {
    ...
}

// 字面量
const fn = function() {
    ...
}

// 构造函数
const fn = new Function('...')
```

第三种使用构建函数创建函数就与对象的创建很类似，因此可以通过判断Function这个构造函数的原型对象是否Object/Person等构造函数的原型对象来判断它们之间的关系。

```
Object.getPrototypeOf(Object) === Object.getPrototypeOf(Function)   // true
Object.prototype === Object.getPrototypeOf(Object.getPrototypeOf(Function)) // true
```

总结：所有函数（普通函数和构造函数）都是继承自Function构造函数的原型对象，而这个原型对象又继承自Object.prototype。

## 关于prototype与__proto__（Object.getPrototypeOf）

其实上面prototype与__proto__的称呼是有点混乱的。简单地说，对象只有__proto__，而函数既有prototype也有__proto__（函数也是对象）。

```
const obj = {}
const fn = function() {}

obj.prototype   // undefined
obj.__proto__   // {}

fn.prototype    //  {}
fn.__proto__    //  [Function]
```

按照上面的逻辑：
1. 对象的构造函数是Object，对象的__proto__是Object.prototype。（字面量{}或new Object创建的对象）
2. 函数的构造函数是Function，函数的__proto__是Function.prototype。（没有使用extends）
3. Function.prototype的__proto__是Object.prototype

由上面的继承关系连接起来的就是原型链，而使用自定义构造函数创建的对象或者使用extends继承其他类的类只是在原来的链上多加一层或多层继承（原型对象），扩展这条原型链。

## 原型的应用

基于原型的继承可以为多个实例（对象或函数）增加通用的属性和方法。

```
Object.prototype.protoFn = function() {
    console.log('protoFn')
}

const obj = {}
obj.protoFn()   // protoFn

class Obj {
    
}

const newObj = new Obj()
newObj.protoFn()    // protoFn
```

这是因为JavaScript会先在实例上尝试获取该属性/方法，如果找不到就沿着原型链向上查找，直到原型链的末端（Object.prototype），如果都找不到就会返回underfined（函数调用会报错）。

原型对象的属性/方法会被实例上的同名属性/方法遮蔽，但不会被覆盖。

```
Object.prototype.protoFn = function() {
    console.log('protoFn')
}

const obj = {
    protoFn: function() {
        console.log('objFn')
    }
}

obj.protoFn()   // objFn
Object.prototype.protoFn()  // protoFn
```

而原型对象上的属性/方法滥用则会导致对象被污染。

```
Object.prototype.name = 'obj'

const obj = {}
const otherObj = {}

otherObj.name = 'otherObj'
obj.name    //  otherObj
```