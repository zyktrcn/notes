# 闭包

闭包是指可以访问另一个函数作用域的函数。

我们知道作用域链的访问是单向的，即从当前活动对象（栈顶）开始，到全局作用域（栈底）结束，逆向访问作用域则会报错（比如全局变量访问函数作用域）。

闭包的形成就必然是在一个函数（外在）中创建另一个函数（内在），后者访问处于前者作用域中的变量。但是函数的特性就是执行完毕立即销毁，如果内在函数的引用没有被保存下来就会被内存回收。

因此，最常见的闭包就是一个函数返回另一个函数。
```
const outer = function() {
    const a = 1

    conset inner = function() {
        retun a + 1
    }

    return inner
}

const closure = outer()
closure()   // 2
```
在上面的例子中，当outer函数执行后就会创建一个变量a并返回一个函数inner，inner函数内部访问了outer函数作用域下的a，因此就形成了闭包。而这个闭包的表现就是a变量在outer函数销毁后也被保存了下来，最终返回结果2（a + 1）。

首先outer函数执行返回了inner函数的引用，这个引用被赋值给closure，也就是说这个函数（inner）仍在活动对象中，内存不会回收。而这个函数（inner）内部访问了outer函数的变量a，因此这个变量也被保存下来了，所以最后输出结果2。

事实上，只要内部的函数访问了外部函数的作用域并且函数的引用被保留下来，闭包就会形成，不限定在返回函数这个操作。
```
let closure
const outer = function() {
    const a = 1

    const inner = function() {
        return a + 1
    }

    closure = inner
}

outer()
closure()   // 2
```

此外，闭包保留的是外部函数作用域的结果，并不是执行过程的某个状态值。
```
let arr = []
const outer = function() {
    for (var i = 0; i < 10; i++) {
        arr.push(function() {
            console.log(i)
        })
    }

    console.log(i)  // 10
}

outer()
arr.forEach(fn => {
    fn()
})
```
这个例子最后输出的是10个10，而并非0-9。

首先push方法往arr数组添加一个函数引用形成闭包，闭包内部访问了外部函数（outer）的变量i。而闭包只会保留外部函数（outer）作用域中的结果，即变量i最后的值，从outer函数执行时打印的10可以看到。

如果想要输出0-9可以借助立即执行方法。
```
let arr = []
const outer = function() {
    for (var i = 0; i < 10; i++) {
        arr.push((function(i) {
            return function() {
                console.log(i)
            }
        })(i))
    }

    console.log(i)  // 10
}

outer()
arr.forEach(fn => {
    fn()
})
```
在这个例子我们就可以看到打印出0-9.

这里是借助立即执行方法，将i作为函数的参数传入，立即执行函数会创建一个作用域，然后作用域内部会保留值类型参数的副本，最后这个参数再被返回的函数访问行程闭包保留这个作用域。

这里的闭包不再是内部匿名函数访问outer函数的作用域行程的，而是内部匿名函数访问立即执行函数的作用域。

要想输出0-9还可以用ES6推出的let声明变量i。
```
let arr = []
const outer = function() {
    for (let i = 0; i < 10; i++) {
        arr.push(function() {
            console.log(i)
        })
    }

    // console.log(i)  // i is not defined
}

outer()
arr.forEach(fn => {
    fn()
})
```
可以看到打印台输出的结果是0-9，而改动就是将for循环里的var改为了let。

在ES6之前没有块级作用域这个概念，因此var声明的变量都会并入当前的作用域下（即outer），即便是在for循环的初始化条件中声明的i也可以被outer打印出来。

ES6引入了const和let，这两个关键词会创建一个块级作用域，简单来说就是并入当前块中（如for），for每循环一次都会创建一个块级作用域，因此每一个i都是独立的，而闭包只是将这个独立的i值保留了下来。而const和let定义的变量活动范围仅限于归属的块，可以从outer函数打印i会报错看出。

如果将这个i从for循环中提出来，结果就又跟var是接近的。
```
let arr = []
const outer = function() {
    let i = 0
    for (; i < 10; i++) {
        arr.push(function() {
            console.log(i)
        })
    }

    console.log(i)  // 10
}

outer()
arr.forEach(fn => {
    fn()
})
```
这个例子最终也会输出10个10（除开outer函数输出的10），因为i变量被并入了函数块中，由始至终都只有这一个变量i，for循环只是改变了它的赋值，最终闭包保留的也是它的结果。

## 闭包的用处

闭包会将外部函数的作用域保留下来，但外部也没有办法访问到外部函数的作用域，因此可以借助闭包创建私有属性和方法。
```
const outer = function() {
    let a = 1

    return {
        get() {
            return a
        },
        set(val) {
            a = val
        }
    }
}

const closure = outer()
closure.get()   // 1
closure.set(2)  // a = 2
```

## 闭包的弊端

由于闭包会保留另一个函数的作用域，内存就无法对这些变量进行回收，闭包的滥用和循环引用就会导致内存溢出的问题。
```
const outer = function() {
    let innerA

    const innerB = function() {
        innerA()
    }

    innerA = function() {
        innerB()
    }

    return {
        innerA,
        innerB
    }
}

const closures = outer()
closure.innerA()    // Maximum call stack size exceeded
``