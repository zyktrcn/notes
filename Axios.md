# Axios

Axios是前端异步请求经常使用的一个组件库，本质是基于Promise和XMLHttpRequest进行封装，可以实现请求拦截、取消请求等功能。

## 使用方法

使用npm安装axios包之后就可以引入直接使用。

```
import axios from 'axios';

const getRequest = axios.get('/test', {
    params: {
        user: 'xiaoming'
    }
})

const postRequest = axios.post('/test'. {
    user: 'xiaoming'
})

getRequest.then(response => {
    console.log(response)
})

postRequest.then(response => {
    console.log(response)
})
```

上面的例子创建了get请求和post请求各一个，然后可以在axios请求后调用then处理服务端的响应。

## 原理

```
// axios.js
function createInstance(defaultConfig) {
    var context = new Axios(defaultConfig)
    var instance = bind(Axios.prototype.request, context)

    utils.extend(instance, Axios.prototype, context)
    utils.extend(instance, context)

    return insance
}

var axios = createInstance(defaults)
var axios.Axios = Axios

module.exports = axios
module.exports.default = axios


function Axios(instanceConfig) {
    this.defaults = instanceConfig
}
```

首先是import导入axios包实际上是引入了Axios的一个实例。Axios构造函数在创建实例时会将配置赋值到实例上的defaults属性。

其次是将Axios.prototpe上的request方法与Axios实例整合，再把Axios.prototype上的属性和方法复制到整合后的实例对象上，因此实例本身是一个接收参数并调用request函数的方法，也是一个可以拥有Axios.prototype上属性的对象。

```
function bind(fn, thisArg) {
    return function() {
        var args = new Array(arguments.length)
        for (var i=0; i < args.length; i++) {
            args[i] = arguments[i]
        }

        return fn.apply(thisArgs, args)
    }
}

function extend(a, b, thisArg) {
    forEach(b, function(val, key) {
        if (thisArg && typeof val === 'function') {
            a[key] = bind(val, thisArg)
        } else {
            a[key] = val
        }
    })

    return a
}
```

而在axios上对get、post等方法的调用实际上也是围绕request方法去展开的。

```
['delete', 'get', 'head', 'options'].forEach(function(method) {
    Axios.prototype[method] = function(url, config) {
        return this.request(mergeConfig(config || {}, {
            method,
            url
        }))
    }
})

['post', 'put', 'patch'].forEach(function(method) {
    Axios.prototype[method] = function(url, data, config) {
        return this.request(mergeConfig(config || {}, {
            method,
            url,
            data
        }))
    }
})
```

这里对会用到的http请求方法进行遍历，并且注册到Axios.prototype上，返回的是对request方法的调用并针对不同类型请求传入不同配置。结合上面Axios实例的创建会将Axios.prototype上的属性和方法赋值到Axios实例对象上，因此可以通过axios.get等方式进行http请求。

```
Axios.prototype.request = function(config) {
    config = mergeConfig(this.defaults, config)
    var chain = [disptachRequest, undefined]
    var promise = Promise.resolve(config)

    while(chain.length) {
        promise = promise.then(chain.shift(), chain.shift())
    }

    return promise
}
```

简单地说，request方法其实就是用mergeConfig方法对配置进行标准化，然后使用Promise.resolve方法对配置进行包装，这个配置会通过promise对象传递给then方法的第一个参数（resolve处理），因此在chain数组中第一个方法其实就是接收配置并执行请求，第二个方法是对应promise.then的reject回调函数。

```
function dispatchRequest(config) {
    // ...
    
    var adapter = config.adapter || defaults.adapter

    return adapter(config).then(function(response) {
        response.data = transformData(
            response.data,
            response.headers,
            config.transformResponse
        )

        return response
    }, function(reason) {
        // ...

        return Promise.reject(reason)
    })
}
```

其实dispatchRequest还是一个封装，除了对配置config的一些处理，最终执行的还是config或defaults中adapater方法。

```
function getDefaultAdapter() {
    var adapter;
    if (typeof XMLHttpRequest !== 'undefined') {
        adapter = require('./adpaters/xhr')
    } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
        adapter = require('./adapters/http');
    }

    return adapter
}

var defaults = {
    adapter: getDefaultAdapter(),
    // ...
}

module.exports = defaults

```

```
// xhr.js
module.exports = function xhrAdapter(config) {
    return new Promise(function(resolve, reject) {
        var request = new XMLHttpRequest()

        request.open(config.method.toUpperCase(), config.url, true);

        request.onreadystatechange = function() {
            if (!request || reuqest.readyState !== 4) return

            var responseData = !config.responseType || config.responseType === 'text' ? request.responseText : request.response;
            var response = {
                data: responseData,
                status: request.status,
                statusText: request.statusText,
                config,
                request
            }

            settle(resolve, reject, response)
        }

        var requestData = config.data || null

        request.send(requestData)
    })
}
```

从上面的代码可以看到defaults中的adapter方法是指向xhrAdapter的，实际上就是用Promise对象封装XMLHttpRequest的异步请求，在onreadystatechange中调用resolve或reject通知处理响应的回调函数。

Axios对请求和响应的拦截也是在request方法中操作。

```
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}

Axios.prototype.request = function(config) {
    config = mergeConfig(this.defaults, config)
    var chain = [disptachRequest, undefined]
    var promise = Promise.resolve(config)

    this.interceptors.request.forEach(function(interceptor) {
        chain.unshift(interceptor.fulfilled, interceptor.rejected);
    });

    this.interceptors.response.forEach(function(interceptor) {
        chain.push(interceptor.fulfilled, interceptor.rejected);
    });

    while(chain.length) {
        promise = promise.then(chain.shift(), chain.shift())
    }

    return promise
}

function InterceptorManager() {
    this.handlers = []
}
```

拦截器的形成其实是通过队列实现的，promise对象在遍历chain数组时会将元素作为回调函数在下一层then方法中注册到当前Promise对象，chain数组一开始只有disptachRequest方法（即提交请求），在该方法前插入请求拦截，在该方法后插入响应拦截。

而拦截器是通过InerceptorManager.prototype.use方法插入到Axios实例的interceptors方法中的。

```
InterceptorManager.prototype.use = function(fulfilled, rejected) {
    this.handlers.push({
        fulfilled,
        rejected
    })

    return this.handlers.length - 1
}
```

拦截器遍历的对象实际上是对handlers属性，而并非是InterceptorManager对象本身。

```
InterceptorManager.prototype.forEach = function(fn) {
    this.handlers.forEach(fn)
}
```