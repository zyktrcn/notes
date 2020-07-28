# 继承

在JavaScript中没有类的概念（class 本质是function），因此父类和子类之间的继承关系是通过原型链实现的。

## 原型继承

```
function Parent(name) {
    this.name = name
}

Parent.prototype.getName = function() {
    return this.name
}

function Child(name) {
    this.name = name
}

Child.prototype = new Parent('father')
Child.prototype.constructor = Child

const child = new Child('son')
child.getName() // son
```

上面的例子创建了一个父类（Parent）和一个子类（Child），将子类的prototype属性设置为父类的一个实例，子类实例（child）通过__proto__属性（父类实例）的__proto__属性访问父类的prototype对象中getName方法。

子类（Child）的prototype属性直接设置为父类（Parent）的实例，这时候prototype属性的constructor指向的还是Parent，因此需要改变这个指向为Child。

通过原型可以让子类继承父类的一些方法和属性，但同时也会污染子类的prototype属性。在将父类的实例赋值给子类的prototype属性时也会创建一个name属性，子类及所有子类的实例都能访问到该属性，只是被子类实例的name属性给掩盖了。

此外，子类实例无法在创建的时候给父类传递参数。

```
Child.prototype.name    // father
child.__proto__.name    // father
child.name  // son
```

## 构造函数继承

```
function Parent(name) {
    this.fatherName = name
}

Parent.prototype.getFatherName = function() {
    return this.fatherName
}

function Child(childName, fatherName) {
    Parent.call(this, fatherName)
    this.name = childName
}

Child.prototype.getName = function() {
    return this.name
}

const child = new Child('child', 'father')
child.name   // child
child.fatherName    // father
child.getName() // child
child.getFatherName()   // child.getFatherName is not a function
```

在子类（Child）构造函数中用call调用父类（Parent）并指向子类，可以实现子类实例（child）向父类构造函数传参，同时可以继承父类的属性。

但在父类prototype对象中定义的方法没有被子类实例继承，也就是子类没有继承父类的原型链，这样就没有办法继承父类的公用方法。

## 组合式继承

```
function Parent(name) {
    this.fatherName = name
}

Parent.prototype.getFatherName = function() {
    return this.fatherName
}

function Child(childName, fatherName) {
    Parent.call(this, fatherName)
    this.name = childName
}

Child.prototype = new Parent('proto father')
Child.prototype.constructor = Child

Child.prototype.getName = function() {
    return this.name
}

const child = new Child('child', 'father')

child.name  // child
child.fatherName    // father
child.getName() // child
child.getFatherName()   // father
```

这种方法其实就是前面两种的结合，既在子类构造函数中调用父类构造函数传参，又会将子类的prototype属性设置为父类的实例，通过原型链传递公用方法。

但这里可以发现，父类（Parent）其实是执行了两次，第一次在子类构造函数中将自身的属性指向子类，第二次创建实例并赋值给子类的prototype。因此，在子类实例（child）中有fatherName属性，在子类实例的__proto__属性指向的对象中也有fatherName属性。

```
child.__proto__.fatherName  // proto father
```

## 寄生式组合继承

```
function Parent(name) {
    this.fatherName = name
}

Parent.prototype.getFatherName = function() {
    return this.fatherName
}

function Child(childName, fatherName) {
    Parent.call(this, fatherName)
    this.name = childName
}

function inheritProto(parent) {
    const F = function() {}
    F.prototype = parent.prototype
    F.prototype.constructor = F
    return new F()
}

Child.prototype = inheritProto(Parent)

Child.prototype.getName = function() {
    return this.name
}

const child = new Child('child', 'father')

child.name  // child
child.fatherName    // father
child.getName() // child
child.getFatherName()   // father
child.__proto__.fatherName  // undefined
```

寄生式组合继承的原理是创建一个新的函数F，把F的prototype属性指向父类的prototype属性，然后再把F的实例赋值给子类的prototype属性，与创建父类实例并赋值给子类prototype属性思想是一致的。但是新创建的F是空函数，不存在再执行一次父类构造函数的缺陷，也避免了给子类prototype属性新增父类构造函数创建的属性导致原型链污染问题。

## ES6继承

```
class Parent {
    constructor(name) {
        this.fatherName = name
    }

    getFatherName() {
        return this.fatherName
    }
}

class Child extends Parent {
    constructor(name, fatherName) {
        super(fatherName)
        this.name = name
    }

    getName() {
        return this.name
    }
}

const child = new Child('child', 'father')

child.name  // child
child.fatherName    // father
child.getName() // child
child.getFatherName()   // father
child.__proto__.fatherName  // undefined
```

ES6新增了类声明（class）和继承（extends）关键词。类声明（class）本质还是function，但没有办法直接执行，只能通过new调用创建一个对象。而继承（extends）需搭配class使用，子类通过super传递参数并调用父类构造函数继承属性，而子类的prototype中的__proto__属性也会指向父类的prototype属性。

最终的效果与寄生式组合继承是一样的，但ES6的类声明和继承更简洁明了。