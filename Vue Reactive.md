# Vue响应式原理（2.5）

在JavaScript中对网页数据的修改要通过对DOM频繁操作实现动态修改，比如在页面上展示一个数字，每次对这个数字的修改（++）就需要调用doucment.getElementById('count').innerHTML = count一次进行页面同步。
```
<div id='count'></div>

<script>
var count = 1
count++
doucment.getElementById('count').innerHTML = count
</script>
```

Vue的响应式数据结构可以让开发者专注Model层的变化，由ViewModal层代替开发者对View层数据渲染的操作。在Vue3.0前，Vue的响应式数据是用Object.defineProperty实现的。

Object.defineProperty方法接受三个参数，分别是一个对象、对象一个属性的键名以及属性的定义。通过这个方法，我们可以对一个对象的属性的get方法和set方法进行重写。
```
const obj = {}
let val = 'value'
Object.defineProperty(obj, 'key', {
    get() {
        console.log('这是一个get方法')
        return val
    },
    set(newVal) {
        console.log('这是一个set方法')
        val = newVal
    }
})

obj.key = 10    // 这是一个set方法
console.log(obj.key)    // 这是一个get方法  10
```

借助这个特性，Vue可以对开发者定义的每个对象进行挟持，追踪其被读取的位置和被写入的时机。

## 读取的位置

首先，Vue需要对开发者定义的数据进行挟持。
```
function reactive(data, key) {
    let val = data[key]

    Object.defineProperty(data, key, {
        get() {
            // 记录读取的位置
            return val
        }
    })
}

// 这里data是所有数据的父对象
const data = {
    count: 0
}

for (let key in data) {
    reactive(data, key)
}
```

Modal层的数据最终要经过ViewModal层处理渲染到View上，这里用JavaScript设置代表数据被读取并渲染到页面上，这个在Vue中就代表了读取的位置。
```
document.getElementById('count').innerHTML = data.count
```

记录读取位置最终的目的就是要在数据更新的时候通知这个位置重新获取新的值，因此用一个Watch类的实例保存读取位置。
```
class Watcher {
    construcotr(getter) {
        this.getter = getter
    }

    get() {
        this.getter()
    }
}

new Watch(() => {
    document.getElementById('count').innerHTML = data.count
})
```
这里用一个函数getter代表读取的位置，可以复用多次。

因为读取的位置可能是多个，因此Vue借助发布——订阅的设计模式定义了一个订阅者（Watcher)的管理器（Dep），每个管理器对应一个数据。
```
class Dep {
    constructor() {
        this.deps = new Set()   // 防止添加重复的Watcher
    }

    depend() {
        // 把Watcher类实例添加到管理器的订阅者列表中
    }
}
```

因此，挟持数据reactive方法中记录读取的位置其实就是调用Dep实例中的depend，将包装这个位置的Watcher类实例加入到订阅者列表中。
```
function reactive(data, key) {
    let val = data[key]
    const dep = new Dep()

    Object.defineProperty(data, key {
        get() {
            dep.depend()
            return val
        }
    })
}
```
在Dep类中的depend方法将Watch类实例添加到管理器的订阅者列表是通过将自身挂在到Dep对象（类）下的一个属性实现的。
```
class Dep {
    constructor() {
        this.deps = new Set()
    }

    depend() {
        if (Dep.target) {
            this.deps.push(Dep.target)
        }
    }
}

class Watcher {
    constructor(getter) {
        this.getter = getter
        this.get()
    }

    get() {
        Dep.target = this
        this.getter()
        Dep.target = null
    }
}
```
因此，只要在创建一个Watch类实例（包装读取数据的位置）调用getter方法就可以触发数据挟持。

执行逻辑：  
1. 创建一个Watch类实例包装读取数据的位置
2. 将当前Watcher类挂载到Dep.target属性
3. 执行读取数据位置的包装函数
4. 触发数据get方法和get方法中Dep实例的depend
5. 将挂载在Dep.target属性的Watch实例加入当前Dep实例的deps订阅者列表

## 写入的时机

Dep类实例的deps属性已经记录了这个数据的订阅者列表，因此在数据写入的时候通知订阅者列表管理器（Dep类实例），再通知每个订阅者获取更新后的数据即可。

首先是在Watcher类中定义接收通知的方法
```
class Watcher {
    constructor(getter) {
        this.getter = getter
        this.get()
    }

    get() {
        Dep.target = this
        this.getter()
        Dep.target = null
    }

    update() {
        this.get()
    }
}
```
然后是在Dep类中定义接收通知的方法并通知所有订阅者更新
```
class Dep {
    constructor() {
        this.deps = new Set()
    }

    depend() {
        if (Dep.target) {
            this.deps.push(Dep.target)
        }
    }

    notify() {
        this.deps.forEach(wathcer => {
            watcher.update()
        })
    }
}
```
最后是对数据的set方法进行挟持，通知对应的Dep类实例（订阅者管理器）。
```
function reactive(data, key) {
    let val = data[key]
    const dep = new Dep()

    Object.defineProperty(data, key {
        get() {
            dep.depend()
            return val
        },
        set(newVal) {
            val = newVal
            dep.notify()
        }
    })
}
```

执行的逻辑：
1. data.count = 1触发数据的set方法
2. set方法通知对应的订阅者管理器（Dep类实例）
3. Dep类实例notify方法通知deps中的订阅者获取更新后的数据
4. Watcher类实例（订阅者）调用getter（读取数据的位置）将新数据渲染到页面上

## PS

Object.defineProperty方法是重新定义对象下的一个属性的特性（比如get和set方法），因此在对象本身（即上面父对象data）添加新的属性，原来的挟持是不起作用的。  

此外，虽然也能对数组的子元素进行挟持，数组原型链上的方法（如push等）无法触发数据中已挟持的set方法。  

对于这两个问题，Vue2.5是通过其他方式解决的，而Vue3.0是用Proxy代替Object.defineProperty。

## 无法挟持的数据

Object.defineProperty的特性限制了对某些场景下数据操作的挟持，比如给对象新增属性、调用数组原生方法对数组进行操作等。
```
// 对象
let val = 1
const obj = {}
Object.defineProperty(obj, 'count', {
    get() {
        console.log('get方法')
        return val
    },
    set(newVal) {
        console.log('set方法')
        val = newVal
    }
})

obj.count = 2   // set方法
console.log(obj.count)  // get方法  2
obj.newCount = 3    //
console.log(obj.newCount)   //

// 数组
let val = 1
const arr = []
Object.defineProperty(arr, 0, {
    get() {
        console.log('get方法')
        return val
    },
    set(newVal) {
        console.log('set方法')
        val = newVal
    }
})

arr[0] = 5  // set方法
console.log(arr[0]) // get方法 5
arr.push(2) //
console.log(arr[1]) //
```
因此，Vue需要增加额外的操作实现这些场景下的数据挟持。

首先是提供一个Api（set）把新设定的数据增加挟持。
```
function set(obj, key, val) {
    obj[key] = val
    reactive(obj, key)
}
```
然后对数组的原生方法重写，手动触发更新。
```
const arrProto = Array.prototype
const arrMethod = Object.create(arrProto)

['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse'].forEach(method => {
    const originalMethod = arrProto[method]
    arrMethod[method] = function(...args) {
        const result = originalMethod.apply(this, args)
        // 手动触发更新
        return result
    }
})
```
手动触发更新的逻辑是将一个Observer类的实例挂载到对象/数组中不可遍历的属性__ob__，然后Dep类的实例dep挂到__ob__属性上。当这个对象/数组进行属性增减需要手动触发更新方法（dep.notify）时就通过这个不可遍历的属性__ob__。


## Computed
coumputed可以定义不可直接写入的属性，原意是初始数据经过computed逻辑加工后变成可用数据，是通过监测调用的属性然后输出结果。比如下面的editedCount就相当于调用者与count属性之间加价的中间商了。
```
computed: {
    editedCount() {
        return this.count + 5
    }
}
```
computed实现的原理是基于Vue的响应式数据。上面提到数据读取的位置被包装为一个getter，当数据被写入触发set方法时重新调用这个getter获取属性新值并渲染到页面上，而这个computed的属性就相当于一个挂在到属性的订阅者列表中的getter。
```
// getter
new Watch(() => {
    document.getElementById('count').innerHTML = data.count
})

// computed
new Watch(() => {
    return this.count + 5
})
```
不同的是，这个computed自己也有一个订阅者列表（Dep类实例），缓存读取computed中数据被读取的位置。
```
class Watcher {
    constructor(getter, computed) {
        this.getter = getter
        this.computed = computed
        
        if (computed) {
            this.dep = new Dep()
        } else {
            this.get()
        }
    }

    get() {
        Dep.target = this
        this.getter()
        Dep.target = null
    }

    update() {
        if (this.computed) {
            this.get()
            this.dep.notify()
        } else {
            this.get()
        }
    }

    depend() {
        this.dep.depend()
    }
}
```
当computed定义的属性被读取时，会在Watcher类实例中的Dep类实例保存读取的位置，逻辑跟普通数据是一致的。

当computed订阅的属性更新时，会通知这个Watcher类实例，然后通过自身的Dep类实例通知自身的订阅者，实现更新。

执行逻辑：
1. 定义一个computed属性作为getter并创建对应的Watcher类实例和Dep类实例
2. 当coumputed属性被读取时会把当前读取位置记录在Dep类实例中
3. computed属性对应的Watcher类实例也会触发订阅属性的get方法
4. 当订阅属性的set方法触发时会通知computed属性对应的Watcher类实例
5. Watcher类实例获取更新后的属性并通知订阅者获取自身更新后的值

## Watch
watch是监测一个属性更改并执行callback逻辑，实质就是将callback传到Watch类中，在update方法被触发时执行这个callback。
```
class Watcher {
    constructor(getter, options = {}) {
        const {computed, watch, callback} = options
        this.getter = getter
        this.computed = computed
        this.watch = watch
        this.callback = callback
        this.value = null
        
        if (computed) {
            this.dep = new Dep()
        } else {
            this.get()
        }
    }

    get() {
        Dep.target = this
        this.getter()
        Dep.target = null
    }

    update() {
        if (this.computed) {
            this.get()
            this.dep.notify()
        } else if (this.watch) {
            const oldVal = this.value
            this.get()
            callback(this.value, oldVal)
        } else {
            this.get()
        }
    }

    depend() {
        this.dep.depend()
    }
}
```
this.value是在Watcher类中缓存当前值，然后在callback中传入新值与旧值。
