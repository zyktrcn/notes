# KOA

```
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

上面是一个简单的Koa搭建HTTP服务器应用例子，通过use在Koa中注册一个中间件处理请求，通过listen创建一个端口监听。

## Compose

Koa的中间件是通过Promise构造出一个将多个中间件包装起来的**洋葱模型**，在源码中实际作用的是koa-compose。

```
app.use(async (ctx, next) => {
    ctx.body = 'Hello Wrold'
})
```

当调用use注册中间件（函数）时，实际是向Koa实例的middleware属性添加一个回调函数，middleware数组是用来收集多个中间件的。

```
class Application extends Emitter {
    constructor(options) {
        // ...
        this.middleward = []
    }

    use(fn) {
        this.middleward.push(fn)
        return this
    }
}
```

当调用listen方法创建监听时，callback方法会对middleware数组进行Promise封装并串联，这个就是Koa中间件处理的核心思路。

```
const http = require('http')
class Application extends Emitter {
    // ...

    listen(...args) {
        const server = http.createServer(this.callback())
        return server.listen(...args)
    }

    callback() {
        const fn = compose(this.middleware)

        // ...
    }
}

function compose(middleware) {
    return function(context, next) {
        let index = -1
        return dispatch(0)

        function dispatch(i) {
            if (i <= index) return Promise.reject(new Error('next() called multiple times'))

            index = i
            let fn = middleware[i]

            if (i === middleware.length) fn = next

            if (!fn) return Promise.resolve()

            try {
                return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```

compose方法通过递归的方式，从middleware数组第一项起通过调用自己（dispatch）**往后遍历并向前抛出一个Promise封装**，直到最后一个中间件获取到的是一个纯粹的Promise实例（无回调执行）。

中间件每一次向前抛出都调用Promise.resolve()方法封装并执行当前中间件的回调函数（fn = middleware[i]），因此前一个中间件都能接收到下一个中间件经过Promise封装的实例（next），通过这个实例可以将当前线程先交付给下一个中间件（await特性），待下一中间件完成时再回到暂停中间件（即交出线程的中间件）。

由于每一个中间件都存在上述逻辑，即每个中间件都可以实现暂停并等待下一个中间件返回，最后形成**洋葱模型**（执行逻辑从最表层到最里层再回到最表层）。

```
callback() {
    const fn = compose(this.middleware)

    const handleRequest = (req, res) => {
        const ctx = this.createContext(req, res)
        return this.handleRequest(ctx, fn)
    }
}

handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
}
```

此外，compose方法仅仅是返回多个嵌套中间件的外层（即第一个中间件）的方法，实际运行的时候是由handleRequest方法发起的。在handleRequest方法中，通过在多个嵌套中间的外层中注册一个then和catch方法实现垫底处理和错误收集。

then方法中注册的回调函数会在中间件完成调用之后触发，中间件的执行要依赖开发者通过await next()层层向**洋葱模型**的最里层中间件执行（除第一个中间件）。

catch方法用于在顶层捕获中间件抛出的错误，而每一次的中间件也可以通过try...catch...对里面层次的中间件错误进行捕获，开发者可以自由处理错误捕获的逻辑。

可以看出，Koa只负责中间件的封装，如何调用由开发者自由选择。

***
（8.5更新）

## Delegate

```
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

在这个例子中，中间件（函数）参数ctx指的是当前上下文，实质上就是res和req的集合，并且用context做了一层代理（Delegate）。

这个context上下文是在callback方法返回的函数（handleRequest）执行时，通过createContext创建的。

```
const context = require('./context');

class Application extends Emitter {
    constructor(options) {
        this.context = Object.create(context)
    }

    callback() {
        // ...

        const handleRequest = (req, res) => {
            const ctx = this.createContext(req, res)
            return this.handlerRequest(ctx, fn)
        }
        return handleRequest
    }

    createContext(req, res) {
        const context = Object.create(this.context)
        // ...
        return context
    }
}
```

constructor以及createContext都只是在引入的context基础上用Object.create方法创建了空对象，但这两个空对象的原型不一样，constructor创建的context是以引入的context为原型的对象，createContext创建的context是以constructor创建的context对象为原型的对象。

```
// context.js
const delegate = require('delegates')

const proto = module.exports = {
    // ...
}

delegate(proto, 'response')
  .method('attachment')
  // ...

// delegates.js
module.exports = Delegator

function Delegator(proto, target) {
    if (!(this instanceof Delegator)) return new Delegator(proto, target)

    this.proto = proto
    this.target = target
    this.methods = []
}

Delegator.prototype.method = function(name) {
    var proto = this.proto
    var target = this.target
    this.methods.push(name)

    proto[name] = function() {
        return this[target][name].apply(this[target], arguments)
    }
}
```

context的主体proto做了错误处理和cookie的包装，主要的内容是引入delegate做代理。

在上面的例子中，delegate首先把proto和指定的target（response）作为实例的属性缓存。然后调用原型中的methods方法将proto.response上的attachment方法挂载到proto上，即proto.response.attachment === proto.attachment。

这样形成的效果就是对proto.attachment调用，实际上是对proto.response.attachment的调用。

除了methods，context还通过access和getter对读写属性和只读属性进行代理。

```
function Delegator(proto, target) {
    // ...

    this.getters = []
    this.setters = []
}

Delegator.prototype.access = function(name) {
    return this.getter(name).setter(name)
}

Delegator.prototype.getter = function(name) {
    var proto = this.proto
    var target = this.target
    this.getters.push(name)

    proto.__defineGetter__(name, function() {
        return this[target][name]
    })

    return this
}

Delegator.prototype.setter = function(name) {
    var proto = this.proto
    var target = this.target
    this.setters.push(name)

    proto.__defineSetter__(name, function(val) {
        return this[target][name] = val
    })

    return this
}
```

从源码可以看出，access和getter的思路与methods一样，都是通过在proto（context）上定义一个同名属性再指向proto.response中的属性，最终实现可以通过context就访问context.response下的属性。

此外，对于可写属性的代理（access），proto（context）上的属性set方法实际是对proto.response上对应属性的重写。

基于以上Delegate的思路，context上下文实际上是对req和res的属性、方法等进行代理。

```
const context = require('./context')
const request = require('./request')
const response = require('./response')

class Application extends Emitter {
    constructor(options) {
        // ...

        this.context = Object.create(context)
        this.request = Object.create(request)
        this.response = Object.create(response)
    }

    createContext(req, res) {
        const context = Object.create(this.context);
        const request = context.request = Object.create(this.request);
        const response = context.response = Object.create(this.response);
        context.app = request.app = response.app = this;
        context.req = request.req = response.req = req;
        context.res = request.res = response.res = res;
        request.ctx = response.ctx = context;
        request.response = response;
        response.request = request;
        context.originalUrl = request.originalUrl = req.url;
        context.state = {};
        return context;
    }
}
```

可以理解为中间件（函数）的第一个参数context上下文是request和response中属性和方法的总和（代理）。

而request是对请求对象req的属性进行一些封装，比如header、url等。response同理。

```
// request.js
module.exports = {
    get header() {
        return this.req.headers
    }

    set header(val) {
        this.req.headers = val
    }

    get url() {
        return this.req.url
    }

    set url(val) {
        return this.req.url = val
    }
}
```

## Cookie

context上下文除了对request和response属性和方法进行代理以外，还封装了对cookie的处理。

```
const Cookies = require('cookies')
const COOKIES = Symbol('context#cookies')

var proto = module.exports = {
    get cookies() {
        if (!this[COOKIES]) {
            this[COOKIES] = new Cookies(this.req, this.res, {
                keys: this.app.keys,
                secure: this.request.secure
            })
        }

        retrun this[COOKIES]
    }

    set cookies(_cookies){
        this[COOKIES] = _cookies
    }
}
```

首先通过Symbol创建唯一标识符，然后在proto（context）上用标识符作为键值定义一个cookies的实例，通过该实例进行cookies的读与写。

***
（8.6更新）

## Response

```
handleRequest(ctx, fnMiddleware) {
    // ...
    const handleResponse = () => respond(ctx);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
}
```

在fnMiddleware方法执行完嵌套的中间件后，需要向请求发出一个响应（response）。由于fnMiddleware本质上是一个Promise实例，可以通过then方法将handleResponse作为回调函数，保证所有**同步**的中间件执行完毕后调用handleResponse方法处理响应。

这里的**同步**是指同步逻辑（包含用async和await处理异步逻辑）处理完之后，就会触发then中注册的回调函数。而异步逻辑如request、setTimeout等会插入对应的任务行列等待执行。

而response方法就是对上下文（context）的响应内容进行处理，包括header、body等。

```
function respond(ctx) {
  let body = ctx.body;

  // ...

  body = JSON.stringify(body);

  res.end(body);
}
```

## Error

```
handleRequest(ctx, fnMiddleware) {
    // ...
    const onerror = err => ctx.onerror(err);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
}
```

Koa还基于Promise的catch方法做了顶层错误的封装。

```
app.use((ctx, next) => {
    throw Error('this is an error')
})
```

比如上面的中间件会被fnMiddleware方法用Promise.resolve()包装为Promise实例，在执行过程中抛出的错误会由Promise实例的catch方法捕捉到，从而执行onerror回调函数。

```
app.use((ctx.next) => {
    console.log('first middleware')
    next()
})

app.use((ctx, next) => {
    throw Error('second middleware')
})
```

在上面的源码中可以了解到，多个中间件其实是通过封装形成一个**洋葱模型**，fnMiddleware执行的时候会根据中间件是否调用next方法决定是否继续往内层执行下一中间件。而每个中间抛出的错误会像事件冒泡一样，逐层向上传递Error，一直到最顶层的catch或是有中间件自行处理错误（try...catch...）。

上面的例子可以看到，即便是在第二个中间件抛出的错误，最后也会被顶层的catch捕捉错误并处理错误响应。

```
app.use((ctx.next) => {
    console.log('first middleware')
    try {
        next()
    } catch(e) {
        console.log(e)
    }
})

app.use((ctx, next) => {
    throw Error('second middleware')
})
```

而即便在第一层中间件上增加处理错误的逻辑（try...catch...），错误也会冒泡到最顶层的catch错误捕捉，因为try...catch...错误捕捉无法处理Promise.reject抛出的错误，而在上面的源码学习中可以发现，每个中间件都是被Promise.resolve和Promise.reject封装为Promise实例抛出的。

```
function compose(middleware) {
    return function(context, next) {
        let index = -1
        return dispatch(0)

        function dispatch(i) {
            if (i <= index) return Promise.reject(new Error('next() called multiple times'))

            index = i
            let fn = middleware[i]

            if (i === middleware.length) fn = next

            if (!fn) return Promise.resolve()

            try {
                return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```

但在另一种情况下是可以在每一层中间中捕捉到下一层中间件抛出的错误，就是通过async和await执行next（Promise实例）。

```
app.use(async (ctx.next) => {
    console.log('first middleware')
    try {
       await next()
    } catch(e) {
        console.log('err')
    }
})

app.use((ctx, next) => {
    throw Error('second middleware')
})
```

在这种情况下，第二层中间件抛出的错误经过await执行后不再是Promise实例，而是Error实例，会被第一层中间件的try...catch...捕捉并处理。最终的效果就是Koa返回正常的响应，而错误被捕捉到时打印出err。

这种巧妙的错误冒泡与顶层错误捕捉都是基于Promise封装的**洋葱模型**，每一个中间件都用Promise.resolve包装为Promise，而中间件的next参数又是下一个中间件的Promise，层层封装后通过Promise.reject向上层抛出执行过程的错误。如果没有对错误进行捕捉和处理就会冒泡到顶层的catch中做错误响应处理。