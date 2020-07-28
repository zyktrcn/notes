# 作用域

作用域是指变量与函数可以访问的范围，在JavaScript中分为全局作用域和局部作用域。

顾名思义，全局作用域是作用在整个运行生命周期中，所有变量和函数都可以访问得到，用栈的方式表示就是始终存在并处于栈的底端。
```
[Global]
```

任何直接定义的变量或函数都会挂在全局作用域下，可以被直接访问或调用。
```
const count = 1

const fn = function() {
    console.log(count)  // 1
}

fn()
```

局部作用域是指当前活动对象所创建出来的封闭范围，仅内部变量和函数可以访问，用栈的方式表示就是处于上一级局部作用域（或全局作用域）和下一级局部作用域之间。
```
[LocalA, LocalB, Global]
```

但局部作用域内的变量或函数则不可以被外界（比如Global）访问或调用。
```
const fn = function() {
    const count = 2
}

fn()
console.log(count)  // ReferenceError: count is not defined
```

上面的例子fn函数创建了一个局部作用域，内部定义了一个变量count。外面的console.log方法尝试访问这个count，会报错count未定义，即fn内的count属性不可被外界所访问。

```
const fn = function() {
    const count = 2

    const innerFn = function() {
        const innerCount = 3
        console.log(count)  // 2
    }

    innerFn()
    console.log(innerCount) // ReferenceError: innerCount is not defined
}

fn()
```

而在fn内部定义的innerFn函数就可以访问count属性。这个例子中innerFn同时也创建了一个局部作用域和一个属性innerCount，fn尝试访问innerCount属性也报错了。

用栈表示上面的作用域层级如下：

```
[innerFn, fn, Global]
```

在这个栈里，作用域从大到小应该是Global > fn > innerFn，因为它们是层层包裹的；但可以访问的访问的范围却是相反的：Global < fn < innerFn，因为Global无法访问fn内的属性，fn也无法访问innerFn内的属性，而innerFn可以访问fn和Global的。

这就说明，作用域可访问的方向是从栈顶到栈底（Global）且单向。

这个作用域栈的形成是通过将当前活动对象推入栈顶，待活动对象销毁后推出栈顶。

1. 首先是全局作用域Global。
   ```
    [Global]
   ```
2. 然后fn函数被执行，创建出fn作用域并推入栈顶。
   ```
    [fn, Global]
   ```
3. 再者innerFn函数被执行，在fn作用域基础上创建innerFn作用域并推入栈顶。
   ```
    [innerFn, fn, Global]
   ```
4. 然后innerFn和fn函数的作用域相继销毁，推出栈顶。
   ```
    [fn, Global]
    [Global]
   ```

如果想在Global作用域下访问fn作用域的变量或函数，有两种方法：
1. 赋值或保留引用
   ```
    const fn = function() {
        const count = 2
        return count
    }

    const outerCount = fn()
    console.log(outerCount) // 2
   ```
   ```
    const fn = function() {
        const count = 2
        return {
            count
        }
    }
    const outerCountObj = fn()
    console.log(outerCountObj.count)    // 2
   ```
   前面的例子是通过将值类型传递给Global下的属性outerCount，保留了fn作用域的count属性结果；后面的例子用Global下的outerCountObj保存了fn函数返回的对象的引用，对象中也是保留了fn作用域的count属性结果。

2. 闭包
   ```
    const fn = function() {
        const count = 2
        return function() {
            return count
        }
    }
    const outerFn = fn()
    console.log(outerFn())  // 2
   ```
   fn函数执行后返回一个匿名函数，在匿名函数中访问了fn作用域的count，最终在Global下执行匿名函数（通过outerFn保存匿名函数引用）成功访问到了fn作用域下的count属性。
