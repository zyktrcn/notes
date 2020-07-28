# XMLHttpRequest

XMLHttpRequest对象的引入实现了页面的无刷新请求数据，前端借助该对象直接向服务器请求或提交数据，而无需借助内嵌框架等操作。XMLHttpRequest对象也成为了Ajax技术的核心。

```
const xhr = new XMLHttpRequest()
xhr.onreadystatechange = function() {
    if (xhr.readyState === 4) {
        if (xhr.status === 200) {
            console.log(xhr.responseText)
        } else {
            alert(xhr.status)
        }
    }
}
xhr.open('get', 'localhost:3300')
xhr.send(null)
```

上面的例子模拟了向本地服务器（localhost:3300）发送一个get请求，该请求不携带其他参数。当服务器返回响应时，若HTTP状态码（status）为200则打印响应内容（responseText），否则弹窗提示HTTP状态码。

在例子中首先创建一个XMLHttpRequest实例；然后设置onreadystatechange的回调方法，每当readyState改变都会触发一次该方法，因此可以通过readyState判断页面与服务器的连接是否初始化（0）、连接是否启动（1）、请求是否发送（2）、页面是否接收到数据（3）以及页面是否完成数据接收（4）；接着就是初始化页面与服务器的连接，传入请求方法（get）、请求url（localhost:3300）；最后就是通过调用send方法启动该连接把请求发送到服务器，等待服务器的响应。

XMLHttpRequest对象还可以发送POST请求。POST请求与GET请求不一样，GET请求的参数是在url上拼接，因此请求体为null；而POST请求需要补充send方法的参数（即请求体）作为请求参数。

```
const xhr = new XMLHttpRequest()
xhr.onreadystatechange = function() {
    if (xhr.readyState === 4) {
        if (xhr.status === 200) {
            console.log(xhr.responseText)
        } else {
            alert(xhr.status)
        }
    }
}
xhr.open('post', 'localhost:3300')
xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')
xhr.send('POST request')
```

上面的例子还调用了setRequestHeader方法修改了HTTP请求的头部属性Content-Type为application/x-www-form-urlencoded，让服务器将请求体中的数据视为表单的内容。

## GET

GET请求通常是作为一个查询数据的请求被发送到服务器端，参数是通过拼接的形式组装到请求url上的。

```
'localhost:3300?a=1&b=2'
```

上面的url就包含了a=1和b=2两个参数，?为参数标识符，&为参数拼接符。

正因为参数是在url上拼接的，因此敏感信息会被url暴露，比如用户名和密码，这样就很容易导致敏感信息的泄露，因此GET请求也被视作为不安全。

```
'localhost:3300?username=123&password=321321'
```

## POST

POST请求通常是作为一个修改数据的请求被发送到服务器端，参数是通过请求体发送到服务器端的。

```
const data = {
    a: 1,
    b: 2
}

xhr.send(JSON.stringify(data))
```

上面的例子就是使用JSON.stringify方法将data对象序列化为字符串，通过xhr.send方法传输给服务器。

与GET请求相比，POST请求会更安全，因为参数没有被url暴露，传输敏感信息如用户名和密码等也不容易泄露。

GET请求的不安全和POST请求的安全都是相对的，因为HTTP请求会被恶意攻击者拦截，在请求中的内容都会泄露。

## Abort、Timeout、Progress

XMLHttpRequest对象还支持定义超时和取消请求的方法。

```
const xhr = new XMLHttpRequest()

xhr.onreadystatechange = function() {
    if (xhr.readyState === 4) {
        console.log(xhr.status, xhr.responseText)
    }
}

xhr.onprogress = function(event) {
    if (event.lengthComputable) {
        console.log(evet.position / event.totalSize * 100 + '%')
    }
}

xhr.open('get', 'localhost:3300')

xhr.timeout = 10000
xhr.ontimeout = function() {
    alert('Timeout')
}

xhr.send(null)

setTimeout(() => {
    xhr.abort()
}, 1000)
```

上面的例子定义了一个GET请求，这个请求超时时间设置为10秒，如果10秒后服务器没有应答则视为连接超时，会触发ontimeout回调函数；如果服务器在超时前应答并回传数据后，就会触发onprogress回调函数，打印数据传输的进度（%）；所有数据传输完毕后会打印服务器响应的HTTTP状态码以及响应内容。

此外，例子在最后设置了一个计时器，一秒后通过abort方法取消该次请求连接，因此服务器在1秒内无法完成响应该连接都无法实现上面的步骤。

## Axios

前端和基于node.js的服务端常用的axios组件就是基于XMLHttpRequest做的封装，使用Promise对象封装XMLHttpRequest，并通过其then方法组装一个请求拦截和响应拦截的队列。下面是一个简单的Promise封装XMLHttpRequest例子，使其支持then、await等特性。

```
function axios() {
    // ...
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest()
        xhr.readystatechange = function() {
            resolve({
                status: xhr.status,
                data: xhr.responseText
            })
        }

        xhr.open('get', 'localhost:3300')
        xhr.send(null)
    })
}

axios().then(response => {
    console.log(response.status, response.data)
})

async function get() {
    const response = await axios()
    console.log(response.status, response.data)
}
```

## 跨域

虽然Ajax异步请求不会导致页面的刷新，但它是通过JavaScript脚本去提交HTTP请求的，会受到浏览器的同源策略影响。

同源策略是指JavaScript脚本提交的HTTP请求目标必须是与页面同协议、同域名、同端口才能对服务器的响应进行操作，也就是说：

* HTTP请求被提交到非同源服务器后，JavaScript脚本无法对服务器的响应进行操作，浏览器会报错，但请求是到达了服务器的。
* 只有JavaScript脚本提交的请求才受同源策略限制，script、img等标签提交的HTTP资源请求不受限制。

解决同源策略问题的办法也有很多，比如：

* JSONP，通过createElement方法创建一个script标签，将请求参数和回调函数方法名拼接到url上，通过script向服务器发起一次GET请求。
* CORS，由服务器设定头部属性Access-Control-Allow-Origin为当前页面域名，向浏览器表示接受该域名的跨域请求。
* 正向代理，设置一个同域名的代理服务器，代理服务器转发页面的请求和目标服务器的响应，绕开浏览器的同源策略。

