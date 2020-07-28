# Vue响应式原理（3.0）

Vue 3.0的响应式原理依旧沿用2.5的思路，从数据读取（get）处添加依赖，在数据写入（set）时通知依赖更新。Vue 3.0主要是用Proxy代替了Object.defineProperty对数据进行挟持。

相比Object.defineProperty，Proxy的好处是针对整个对象进行挟持，也能避免了无法监控对象属性增加或删除的问题。
```
let propertyObj = {}
let count = 1
Object.defineProperty(propertyObj, 'count', {
    get() {
        // 挟持count属性
        return count
    },
    set(value) {
        count = value
        // 通知依赖更新
    }
})

document.getElementById('count').innerHTML = propertyObj.count  // 挟持成功
propertyObj.count = 2   // 通知依赖更新成功
document.getElementById('newCount').innerHTML = propertyObj.newCount    // 无挟持
propertyObj.newCount = 3    // 无更新
```
```
let proxyObj = {}
proxyObj = new Proxy(proxyObj, {
    get(target, key, receiver) {
        // 挟持proxyObj对象
        return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
        let ret = Reflect.set(target, key, value, receiver)
        // 通知依赖更新
        return ret
    }
})

document.getElementById('count').innerHTML = proxyObj.count // 挟持成功
proxyObj.count = 2  // 通知依赖更新成功
document.getElementById('newCount').innerHTML = proxyObj.newCount   // 挟持成功
proxyObj.newCount = 3   // 通知依赖更新成功
```

从上面两个例子可以看出，Proxy挟持的目标是整个对象，因此在对象中新添加的newCount属性也能挟持并响应变化；而Object.defineProperty挟持的目标是对象中的某个属性，因此必须要在属性已被挟持情况下才能响应变化。

同样地，Vue 3.0将数据读取的位置包装为一个函数放在依赖中以复用。
```
let fn = () => {
    document.getElementById('count').innHTML = proxyObj.count
}
```

首先是在全局定义一个数组作为栈，然后用effect函数将上面的fn置于栈顶。
```
cont effectStack = []

function effect(fn) {
    const effectFn = activeEffect(fn)
    effectFn()
}

function activeEffect(fn) {
    const effect = function() {
        effectStack.push(fn)
        fn()
        effectStack.pop()
    }
    return effect
}
```

fn被置于栈顶后立刻执行并触发对应数据的get方法，将置于栈顶的当前位置（fn）加入到依赖（订阅者）列表中。完成以上的流程后就会将当前位置（fn）推出栈顶。

```
let proxyObj = {}
proxyObj = new Porxy(proxyObj, {
    get(target, key, receiver) {
        track(target, key)
        return Reflect.get(target, key, receiver)
    }
})

let targetMap = new Map()

function track(target, key) {
    let effect = effectStack[effectStack.length - 1]
    let depMap = targetMap.get(target)

    if (!depMap) {
        depMap = new Map()
        targetMap.set(target, depMap)
    }

    let dep = depMap.get(key)
    if (!dep) {
        dep = new Set()
        depMap.set(key, dep)
    }

    if (!dep.has(effect)) {
        dep.add(effect)
    }
}
```

在track中使用到的targetMap、depMap是使用Map类型，而dep使用的是Set类型，其对应的是Object和Array，但Map支持Object作为键值，Set则是无重复元素的数组。借助这两种类型的特性去避免重复添加同一个位置的依赖。

这几个属性之间的关系如下:

```
let proxyObj = {
    count: 1
}

targetMap = {
    { count: 1 }: depMap
}

depMap = {
    count: dep
}

dep = [...]
```

依靠这种关系，最终将当前位置（fn）加入到依赖（订阅者）列表中。当对象数据发生变化时，也可以根据这个关系找到依赖（订阅者）列表逐个通知。

```
let proxyObj = {}
proxyObj = new Proxy(proxyObj, {
    set(target, key, value, receiver) {
        let ret = Reflect.set(target, key, value, receiver)
        trigger(target, key)
    }
})

function trigger(target, key) {
    const depMap = targetMap.get(target)
    if (depMap) {
        const dep = depMap.get(key)
        if (dep) {
            dep.forEach(fn => {
                fn()
            })
        }
    }
}
```

另外，由于Object.defineProperty的缺陷，Vue 2.5对数组方法带来的数据变化是通过重新数组方法而实现的，但在Vue 3.0中Proxy本身就能够监测到数组方法带来的数据变化，这是通过数组的length实现的。

```
let proxyArr = [1, 2]
proxyArr = new Proxy(proxyArr, {
    get(target, key, receiver) {
        console.log(`target: ${target}, key: ${key}.`)
        return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
        console.log(`target: ${target}, key: ${key}, value: ${value}`)
        return Reflect.set(target, key, value, receiver)
    }
})

console.log(proxyArr[0])
//  target: 1,2, key: 0.
//  1
proxyArr.push(4)
//  target: 1,2, key: push.
//  target: 1,2, key: length.
//  target: 1,2, key: 2, value: 4.
//  target: 1,2,4, key: length, value: 3.
console.log(proxyArr[2])
//  target: 1,2,4, key: 2.
//  4
```

从上面的例子可以看到，在调用数组的push方法对挟持数组（对象）数据进行修改时，Proxy可以挟持到当时调用的push方法，以此读、写一次数组的length属性，因为push会对数组长度造成影响。