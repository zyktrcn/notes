# Promise原理

如果多个异步操作的需求重叠起来，一般会变成这样：

```
setTimeout(() => {
    // do something
    setTimeout(() => {
        // also do something
        setTimeout(() => {
            // also do something
        })
    }, 1000)
}, 1000)
```

上面的例子只是三层各一个异步函数（setTimeout），实际业务场景远比这个复杂，这样的代码不易理解、难以维护。一般这种情况较回调地狱，就是多层回调函数嵌套。

ES6就推出了Promise对象，它的使用方法是这样的：

```
new Promise(resolve => {
    setTimeout(() => {
        // do something
        resolve()
    }, 1000)
}).then(() => {
    setTimeout(() => {
        // also do something
        return
    }, 1000)
}).then(() => {
    setTimeout(() => {
        // also do something
        return
    }, 1000)
})
```

这个例子跟上面的效果是一样的，但三层回调嵌套被解开并平铺。

Promise实现的方法是通过定义三个状态值，通过判断状态值去触发对应的方法队列。初始为pending（等待执行），另外两个是fulfilled（执行完毕）、rejected（执行错误）。

```
function Promise() {
    let state = 'pending'   // 初始值
}
```

这三个状态值是由开发者手动触发的，由Promise对象内部定义的resolve和reject方法将状态值更改为对应的值（fulfilled和rejected），这两个方法作为参数传入开发者传入的回调函数。

```
function Promise(fn) {
    // fn是开发者传入的回调函数
    let state = 'pending

    function resolve() {
        state = 'fulfilled'
    }

    function reject() {
        state = 'rejected'
    }

    fn(resolve, reject) // 执行fn时传入resolve和reject方法
}
```

这里对应的就是第一个Promise例子里的回调函数和resolve()调用。

```
new Promise(resolve => {
    // 回调函数内部
    setTimeout(() => {
        // do something
        resolve()   // 调用resolve方法
    }, 1000)
})
```

resolve/reject方法的原理就是发布——订阅模式，开发者在调用这两个方法的时候就相当于通知订阅者执行自己的逻辑，因此才可以实现在异步函数（setTimeout）完成之后才会执行then里的逻辑。

发布——订阅模式首先需要做的就是建立订阅者列表，而then就是将回调函数加入到订阅者列表中。

```
function Promise(fn) {
    const callbacks = []    // 订阅者列表

    // 在Promise对象上创建一个then方法
    this.then = function(onFulfilled, onRejected) {
        return new Promise((resolve, reject) => {
            // 将回调函数加入到订阅者列表中
            callbacks.push({
                onFulfilled,
                onRejected,
                resolve,
                reject
            })
        })
    }
}
```

对应Promise的第一个例子就是then中的回调函数：

```
new Promise(resolve => {
    ...
}).then(res => {
    setTimeout(() => {
        // do something
    }, 1000)
})
```

then方法的参数其实有两个，第一个是onFulfilled，对应状态值fulfilled的回调；第二个是onRejected，对应状态值rejected的回调。

当Promise的回调函数中调用的是resolve方法就会执行前者，调用的是reject方法就会执行后者。

另外可以注意到，then本身返回了一个Promise对象，因此它也有then方法，可以在then后面继续调用then方法，形成链式调用。

对应Promise的第一个例子就是then().then()。

订阅者列表创建好之后，就需要resolve/reject实行通知。

```
function Promise(fn) {
    let state = 'pending'
    const callbacks = []

    function resolve(newValue) {
        setTimeout(() => {
            if (state !== 'pending') return

            state = 'fulfilled'
            // 通知订阅者列表
            while(callbacks.length) {
                const fn = callbacks.shift()
                const cb = fn.onFulfilled
                const next = fn.resolve
                const ret = cb(newValue)
                next(ret)
            }
        }, 0)
    }
}
```

在resolve里会创建一个0延时的setTimeout，因为Promise对象中的then会被安排在微任务列表中，当前宏任务（其实是Promise回调函数）执行完毕后会先执行微任务列表中的逻辑。而setTimeout会被安排在宏任务列表中，只有当上一个宏任务的微任务列表执行完毕并清空后才会执行。

最终达到的效果是，在Promise的回调函数中执行resolve，将这个0延时的setTimeout注册到宏任务列表中，然后先执行then方法，将then方法中的回调函数注册到订阅者列表（callbacks）中。当then执行完毕后再回到下一个宏任务（setTimeout）并通知订阅者列表的回调函数执行。

订阅者列表（callbacks）中的回调函数应该是下面的格式：

```
callbacks: [
    {
        onFulfilled,
        onRejected,
        resolve,
        reject
    }
]
```

其实就是两个不同状态值对应的回调函数，还有Promise对象（then也是创建并返回了一个Promise对象）的resolve和reject方法。

在Promise对象中的resolve方法首先是将状态值改为fulfilled，表示已经完成；而reject方法是将状态值改为rejected，表示执行错误。cb和next变量根据状态值赋值onFulfilled（onRejected）和resolve（reject）。

另外，resolve/reject方法在开发者手动调用时还可以给then的回调函数传参数(newValue)。因此在then中将resolve/reject传入的参数传到cb（onFulfilled/onRejected）中并执行获取结果，最后将执行的结果再通过next（resolve/reject）往下一个then传递。

最终就形成了异步操作的平铺和链式调用：

```
new Promise((resolve, reject) => {
    setTimeout(() => {
        console.log(1)
        resolve(2)
    }, 1000)
}).then(res => {
    setTimeout(() => {
        console.log(res)
        return 3
    }, 1000)
}).then(res => {
    setTimeout(() => {
        console.log(res)
    }, 1000)
})

//  1
//  2
//  undefined
```

（续）

在这个例子中可以看到then的第一个参数（回调函数）接收到了Promise用resolve方法传来的值（2），但第二个then并没有接收到前一个then通过return返回的值（3），而是undefined。

可以尝试将第一个then换成同步操作，再返回一个值给第二个then打印。

```
new Promise((resolve, reject) => {
    setTimeout(() => {
        console.log(1)
        resolve(2)
    }, 1000)
}).then(res => {
    console.log(res)
    return 3
}).then(res => {
    setTimeout(() => {
        console.log(res)
    }, 1000)
})

//  1
//  2
//  3
```

通过这个例子可以看到then的链式调用也是可以通过return传参，上一个例子的问题就出在异步和同步之间的差异。

理论上，then是一个Promise对象，是支持异步操作和发布——订阅模式，像第一个Promise一样完成异步逻辑再通知下一个then的回调函数执行。

事实上then里的回调函数是由上一个Promise执行获取结果的，而then返回的Promise对象中的resolve方法也是在上一个Promise中执行并通知下一个then。

```
function Promise(fn) {
    let state = 'pending'
    const callbacks = []

    function resolve(newValue) {
        setTimeout(() => {
            if (state !== 'pending') return

            state = 'fulfilled'
            // 通知订阅者列表
            while(callbacks.length) {
                const fn = callbacks.shift()
                // 获取then的回调函数
                const cb = fn.onFulfilled
                // 获取then里的resolve
                const next = fn.resolve
                // 执行then回调函数获取结果
                const ret = cb(newValue)
                // 用resolve给下一个then传参
                next(ret)
            }
        }, 0)
    }
}
```

因此在then回调函数中的setTimeout中返回值（3）是没有意义的，源代码中的cb其实就是setTimeout所在的回调函数，而cb执行只是将setTimeout注册到宏任务队列中，而这个回调函数没有使用return返回一个值（默认undefined），因此下一个then才接收到undefiened。

在then中要想实现异步操作并完成后再传参给下一个then，只要return一个Promise对象，用这个Promise执行异步函数就可以了。

```
new Promise((resolve, reject) => {
    setTimeout(() => {
        console.log(1)
        resolve(2)
    }, 1000)
}).then(res => {
    return new Promise(resolve => {
        setTimeout(() => {
            console.log(2)
            resolve(3)
        }, 1000)
    })
}).then(res => {
    setTimeout(() => {
        console.log(res)
    }, 1000)
})

// 1
// 2
// 3
```

而之前在resolve中直接将newValue传给下一个then回调函数作为参数执行是不足够的，因为现在newValue已经变成一个Promise，因此需要获取Promise中resolve的传参作为新的newValue。

```
function Promise(fn) {
    let state = 'pending'
    const callbacks = []

    function resolve(newValue) {
        setTimeout(() => {
            if (state !== 'pending') return

            if (newValue && (typeof newValue === 'objcet' || typeof newValue === 'function')) {
                const { then } = newValue

                if (typeof then === 'function') {
                    then.call(newValue, resolve)
                    
                    return
                }
            }
        }, 0)
    }
}
```

其实是先对newValue进行判断是否满足Promise的特性（object或者function并有一个then方法），如果newValue是一个Promise对象，则用call方法执行newValue的then方法并用当前的resolve作为then的回调函数。

then执行之后回调函数（resolve）就会注册到newValue（Promise）的订阅者列表中，当newValue中的异步操作完成之后就会用自身的resolve通知并传参给then的回调函数（resolve），最后当前的resolve方法接受到这个参数再次执行往下一个then传参。

执行的顺序是这样的：
1. Promise/then通过resolve或者return传入一个Promise对象，即newValue是一个Promise对象。
2. newValue的then中回调函数注册到订阅者列表中，newValue执行完毕调用自己的resolve传参。
3. then中的回调函数（resolve）收到通知并接受参数，重新执行resolve(newValue)。
4. 下一个then接收到新的newValue（值）。

相当于在一个Promise对象中再创建了一个Promise对象并通过then将异步函数的结果返回到外面这个Promise对象。

## Promise.all和Promise.race

Promise类上还挂载了几个可以直接使用的方法，比如all和race。两者有点相似，all是执行一个或多个Promise对象（数组），全部Promise中状态都为fulfilled时，all中的状态就为fulfilled，否则为rejected；race则是执行一个或多个Promise对象（数组），但只要有一个状态更改为fulfilled，race中的状态就为fulfilled。

```
let promiseArr = []
for (let i = 0; i < 5; i++) {
    promiseArr.push(new Promise(resolve => {
        setTimeout(() => {
            return i
        }, (i + 1) * 1000)
    }))
}

Promise.all(promiseArr).then(res => {
    console.log(res)    // [0, 1, 2, 3, 4]
})

Promise.race(promiseArr).then(res => {
    console.log(res)    // 0
})
```

原理就是创建一个新的Promise对象，然后遍历数组获取每个Promise对象并执行，收集执行后的结果以对应的顺序返回一个数组。

```
Promise.all = function(arr) {
    return new Promise(resolve => {
        // 空数组返回空数组结果
        if (arr.length === 0) return resolve([])
        
        let remaining = arr.length

        for (let i = 0; i < arr.length; i++) {
            res(i, arr[i])
        }

        function res(i, val) {
            if (newValue && (typeof newValue === 'objcet' || typeof newValue === 'function')) {
                const { then } = newValue

                if (typeof then === 'function') {
                    then.call(val, val => {
                        res(i, val)
                    })
                    
                    return
                }
            }

            arr[i] = val
            if (--remaining === 0) {
                resolve(arr)
            }
        }
    })
}
```

在res方法中的逻辑就是Promise对象中resolve方法对newValue为Promise对象时的处理，通过then方法获取并返回Promise对象异步操作的resolve传参。

最后数组都遍历完了就调用Promise对象的resolve方法返回结果（数组），然后Promise.all返回的也是一个Promise对象，因此在后面的then就可以接收到这个数组结果。

Promise.race更为简单，就是赛跑，返回最快执行完毕的Promise对象的结果。

```
Promise.race = function(arr) {
    return new Promise(resolve => {
        for (let i = 0; i < arr.length; i++) {
            arr[i].then(resolve)
        }
    })
}
```

也是通过将resolve方法作为then的回调函数接收每个Promise对象异步操作的结果，同时Promise.race返回一个Promise对象。

当最快执行完毕的异步操作调用resolve将race返回的Promise对象状态更改为fulfilled后，其他异步操作虽然也会调用resolve，但状态值已不是fulfilled，所以不会有影响。

在resolve方法中是这样的：

````
function Promise(fn) {
    let state = 'pending'
    const callbacks = []

    function resolve(newValue) {
        setTimeout(() => {
            // 状态值不为pending时return
            if (state !== 'pending') return
        }, 0)
    }
}
```