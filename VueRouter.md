# Vue Router

vue-router是基于单页面应用Vue设计的一套路由框架，提供Hash和History两种模式。

单页面应用实质是改变页面呈现的内容，不向服务器请求页面，也不做URL跳转。因此，单页面应用需要一套独特的路由框架响应页面的跳转请求，更改页面展示的内容。

一般在Vue中使用vue-router是先用Vue.use引入，再创建vue-router的实例定义路由配置和列表，最后注入到Vue实例中。

```
import Vue from 'vue'
import VueRouter from 'vue-router'

// 引入Index页面（Vue实例）
import Index from './src/index.vue'
// 引入入口APP页面
import App from './App.vue'

// 在Vue中引入vue-router
Vue.use(VueRouter)

// 创建vue-router实例
const router = new VueRouter({
    mode: 'hash',   // hash或history
    routes: [
        {
            path: '/',
            component: Index
        }
    ]
})

// 将vue-router实例注入到Vue实例中
new Vue({
    el: '#app',
    router,
    components: {
        App
    },
    template: '<App />
})
```

Vue.use引入vue-router实际上是调用vue-router中install方法。

```
import View from './components/view'
import Link from './compoents/link'

VueRouter.install = function(Vue) {
    
    Object.defineProperty(Vue.prototype, '$router', {
        get() {
            return this.$root._router
        }
    })

    Object.definProperty(Vue.prototype, '$route', {
        get() {
            return this.$root._route
        }
    })

    Vue.mixin({
        beforeCreate() {
            if (this.$options.router) {
                this._routerRoot = this
                this._router = this.$options.router
                this._router.init(this)
                Vue.util.defineReactive(this, '_route', this._router.history.current)
            } else {
                this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
            }
        }
    })

    Vue.component('router-view', View)
    Vue.component('router-link', Link)
}
```

在install方法中主要做了三件事情:  
1. 在Vue.prototype对象上注入两个属性\$router和\$route，为所有Vue实例提供根路由（$root）上这两个属性的访问。
2. 在Vue的mixin注入钩子函数beforeCreate，所有页面在实例化前都会执行这个mixin的钩子函数。
   * 根路由会使用defineReactive挟持当前实例的_route属性，有修改时触发组件渲染。
   * 子组件会通过父组件访问_routerRoot属性，将当前实例通过挟持加入到依赖中。  
   （_route属性挟持是defineReactive方法中的Object.definProperty作用，原理也是通过get、set方法重写。）

    ```
    // Vue.util.defineReactive
    function defineReactive (obj, key) {

        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: function () {
                // ...
            },
            set: function (newVal) {
                // ...
                dep.notify()    // 通知依赖更新
            }
        })
    }

    ```

3. 在Vue的component上注册两个组件，所有Vue实例都能直接使用这两个组件。

    ```
    // App.vue
    <template>
        <router-view></router-view>
    </template>

    // index.vue
    <template>
        <router-link to='/'>index</router-link>
    </template>
    ```

（7.2更新）

Vue通过install安装vue-router插件之后，需要将vue-router的实例挂载在Vue.options属性中。上面的例子就是设置vue-router的模式和路由配置。

```
const router = new VueRouter({
    mode: 'history',
    routes: [
        {
            path: '/',
            component: Index
        }
    ]
})
```

实例化VueRouter的时候，会根据传入的配置mode创建一个对应类的实例（hash对应HashHistory，history对应HTML5History，abstract对应AbstractHistory）。

```
class VueRouter {
    constructor(options) {
        this.options = options
        this.matcher = createMatcher(options.routes || [], this)
        this.mode = options.mode

        switch (mode) {
            case 'history':
                this.history = new HTML5History(this, options.base)
                break
            case 'hash':
                this.history = new HashHistory(this, options.base, this.fallback)
                break
            case 'abstract':
                this.history = new AbstractHistory(this, options.base)
                break
            default: 
                
        }
    }
}
```

VueRouter构造函数最重要的是createMatcher和创建一个mode配置对应的实例。

## createMatcher

createMatcher是通过crateRouteMap方法利用路由的path和name创建两个映射对象，并且创建一个路由列表。但这两个映射和路由列表都是为内部方法服务的，最终createMatcher向外暴露了addRoutes和match两个方法。

```
function createMatcher(routes, router) {
    const { pathList, pathMap, nameMap } = createRouteMap(routes)

    funtcion addRoutes(routes) {
        createRouteMap(routes, pathList, pathMap, nameMap)
    }

    function match(raw, currentRoute, redirectedFrom) {
        // ...
    }

    return {
        addRoutes,
        match
    }
}

function createRouteMap(routes, oldPathList, oldPathMap, oldNameMap) {
    const pathList = oldPathList || []
    const pathMap = oldPathMap || Object.create(null)
    const nameMap = oldPathMap || Object.create(null)

    routes.forEach(route => {
        addRouteRecord(pathList, pathMap, nameMap, route)
    })

    return {
        pathList,
        pathMap,
        nameMap
    }
}

function addRouteRecord(pathList, pathMap, nameMap, route, parent, matchAs) {
    const { path, name } = route
    const record = {
        path,
        components: route.compoents || { default: route.component },
        name,
        parent,
        matchAs,
        redirect: route.redirect,
        beforeEnter: route.beforeEnter,
        meta: route.meta || {},
        props: route.props === null ? {} : (route.compoents ? route.props : { default: route.props })
    }

    if (route.children) {
        route.children.forEach(child => {
            cconst childMatchAs = matchAs ? cleanPath(`${matchAs}/${child.path}`) : undefined
            addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
        })
    }

    if (!pathMap[record.path]) {
        pathList.push(record.path)
        pathMap[record.path] = record
    }

    if (name) {
        if (!nameMap[name]) {
            nameMap[name] = record
        }
    }
}
```

总体思路就是沿着路由的子路由用递归的方式以深度优先遍历，组装两个路由映射和一个路由列表，并提供给createMatcher内部的方法使用，并且挂载到VueRouter实例上的matcher属性。

## mode

1. history模式是基于HTML5的history，利用history对象保存浏览记录以及处理URL变化。在客户端上的表现就是正常的URL请求，这意味着服务器要对应vue-router在history模式下单页面应用的配置，否则会报错。
2. hash模式是基于锚点（#），即在页面的不同位置之间跳转。在客户端上的表现就是域名 + # + 路由，因为浏览器请求页面资源的时候不会携带#后的字符，在hash模式下页面的跳转是更改#后的路由，对客户端和服务器而言都是同一个页面资源。

### HTML5History

```
class HTML5History extends History {
    constructor(router, base) {
        super(router, base)
        this._startLocation = getLocation(this.base)
    }
}

function getLocation(base) {
    let path = decodeURI(window.location.pathname)
    if (base && path.toLowerCase().index(base.toLoweCase()) === 0) {
        path = path.slice(base.length)
    }
    return (path || '/') + window.location.search + window.location.hash
}
```

HTML5History构造函数主要是继承通用类History的属性，结合配置里的base前缀缓存初始URL。

```
class History {
    constructor(router, base) {
        this.router = router
        this.base = normalizeBase(base),
        this.current = createRoute(null, { path: '/' })
        this.pending = null
        this.ready = false
        this.readyCbs = []
        this.readyErrorCbs = []
        this.errorCbs = []
        this.listeners = []
    }
}

function normalizeBase(base) {
    if (!base) {
        base = '/'
    }

    if (base.charAt(0) !== '/) {
        base = '/' + base
    }

    return base.replace(/\/$/, '')
}

function createRoute(record, location, redirectedFrom, router) {
    const stringifyQuery = router && router.options.stringifyQuery
    let query = location.query || {}

    try {
        query = clone(query)
    } catch(e) {}

    const route = {
        name: location.name || (record && record.name),
        meta: (record && record.meta) || {},
        path: location.path || '/',
        hash: location.hash || '',
        query,
        params: location.params |\ {},
        fullPath: getFullPath(location, stringifyQuery),
        matched: record ? formatMatch(record) : []
    }

    if (redirectedFrom) {
        route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
    }

    return Object.freeze(route)
}
```

通用类History的构造函数主要是初始化实例上的属性，值得注意的是base和current属性。前者是基于配置中base属性进行修正，后者是初始化一个不可更改的当前路由对象。

### HashHistroy

```
class HashHistory extends History {
    constructor(router, base, fallback) {
        super(router, base)

        if (fallback && checkFallback(this.base)) return

        ensureSlash()
    }

    function checkFallBack(base) {
        const location = getLocation(base)
        if (!/^\/#/.test(location)) {
            window.location.replace(cleanPath(base + '/#' + location))
            return true
        }
    }

    function ensureSlash() {
        const path = getHash()
        if (path.charAt(0) === '/') {
            return true
        }
        replaceHash('/' + path)
        return false
    }

    function getHash() {
        let href = window.location.href
        const index = href.indexOf('#')

        if (index < 0) return ''

        href = href.slice(index + 1)
        const searchIndex = href.indexOf('?')
        if (searchIndex < 0) {
            const hashIndex = href.indexOf('#')
            
            if (hashIndex > -1) {
                href = decodeURI(href.slice(0, hashIndex)) + href.slice(hashIndex)
            } else {
                href = decodeURI(href)
            }
        } else {
            href = decodeURI(href.slice(0, searchIndex)) + href.slice(searchIndex)
        }

        return href
    }

    function replaceHash(path) {
        if (supportsPushState) {
            replaceState(getUrl(path))
        } else {
            window.location.replace(getUrl(path))
        }
    }
}
```

hash模式对应的HashHistory类也是继承自通用类History，其构造函数除了初始化属性以外，还对当前URL基于#和?进行修正，并且根据浏览器对HTML5的history支持选择不同的方式响应路由的变化。也就是说，在支持HTML5 history对象的浏览器中，hash和history模式除了对URL的处理以外，其他原理都是相同的。

（7.3更新）

## 路由组件实例化

当路由配置列表中的组件实例化时，会先触发vue-router在Vue实例中注册的全局mixin。

```
const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

Vue.mixin({
    beforeCreate() {
        if (this.$options.router) {
            this._routerRoot = this
            this._router = this.$options.router
            this._router.init(this)
            Vue.util.defineReactive(this, '_route', this._router.history.current)
        } else {
            this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
        }
        registerInstance(this, this)
    }
})
```

先判断是否根组件，因为VueRouter实例router是挂载到根组件的options属性上，把这个router通过根组件暴露给下层组件，使用defineReactive方法定义并挟持_route属性。

如果当前组件不是根组件，就会沿着父组件寻找根组件上的_routerRoot属性，从而访问_router属性。

VueRouter类中的init方法设置了一个历史路由监听器，当路由更新触发updateRoute，监听器设置的回调函数cb就遍历apps并修改每一个实例里的_route属性为需更新的路由，触发在根组件上定义的_route属性挟持。

```
class VueRouter {
    // ...
    function init(app) {
        this.apps.push(app)

        if (this.app) return

        this.app = app
        const history = this.history
        // ...
        history.listen(route => {
            this.apps.forEach(app => {
                app._route = route
            })
        })
    }
}

class History {
    // ...
    listen(cb) {
        this.cb = cb
    }

    updateRoute(route) {
        // ...
        this.cb && this.cb(route)
    }
}
```

接着就是调用当前组件实例options._parentVnode.data上的registerRouteInstance方法，这个方法是在Vue实例中引入的全局组件RouterView（<router-view>）中定义的。

```
var name = props.name
var route = parent.$route
var depth = 0
// ...
var matched = route.matched[depth]

data.registerRouteInstance = function (vm, val) {
    var current = matched.instances[name];
    if (
        (val && current !== vm) ||
        (!val && current === vm)
    ) {
        matched.instances[name] = val;
    }
}
```

这个方法主要是根据当前组件的层级记录实例，最后在根组件上的matched组装一个以组件层级为单位的路由实例列表。

当页面调用this.$router.push时，实际是调用history.push方法，以hash模式为例。

```
class VueRouter {
    // ...
    push(location, onComplete, onAbort) {
        if (!onComplete && !onAbort && typeof Promise !== 'undefined') {
            return new Promise((resolve, reject) => {
                this.history.push(location, resolve, reject)
            })
        } else {
            this.history.push(location, onComplete, onAbort)
        }
    }
}

class HashHistory extends History {
    // ...
    push(location, onComplete, onAbort) {
        this.transitionTo(
            location,
            route => {
                pushHash(route.fullPath)
                onComplete && onComplete(route)
            },
            onAbort
        )
    }

    ensureURL (push) {
        const current = this.current.fullPath
        if (getHash() !== current) {
            push ? pushHash(current) : replaceHash(current)
        }
    }
}

class History {
    // ...
    transitionTo(location, onComplete, onAbort) {
        const route = this.router.match(location, this.current)
        this.confirmTransition(
            route,
            () => {
                this.updateRoute(route)
                onComplete && onComplete(route)
                // ...
            },
            err => {
                if (onAbort) {
                    onAbort(err)
                }
                // ...
            }
        )
    }

    confirmTransition(route, onComplete, onAbort) {
        const current = this.current
        const lastRouteIndex = route.matched.length - 1
        const lastCurrentIndex = route.matched.length - 1
        if (
            isSameRoute(route, current) &&
            lastRouteIndex === lastCurrentIndex &&
            route.matched[lastRouteIndex] === current.matched[lastCurrentIndex]
        ) {
            this.ensureURL()
            return onAbort && onAbort()
        }

        // ...
        onComplete(route)
    }
}
```

在调用this.$router.push('/')之后会经历几个步骤：
1. 判断是否有传入回调函数。
   * 有回调函数（onComplete / onAbort）就会直接传入回调，即便其中一个是undefined。
   * 如果没有回调函数就会用Promise封装，调用push的时候可以用then执行回调。
2. 用match方法在路由表中查找传入路由（'/' / { name: 'index }）转化为路由对象，match方法就是创建VueRouter实例时用路由配置获取createMatcher向外暴露的方法之一。简化之后就是根据需要跳转的路由查找对应的路由对象（包含路由配置和对应的组件），并且对params属性进行适配。

   ```
    function match(raw, currentRoute, redirectedFrom) {
        const location = normalizeLocation(raw, currentRoute, false, router)
        const { name } = location

        if (name) {
            const record = nameMap[name]
            if (!record) return _createRoute(null, location)
            // ...
            return _createRoute(record, location, redirectedFrom)
        } else if (location.path) {
            // ...
            return _createRoute(record, location, redirectedFrom)
        }

        return _createRoute(null, location)
    }
   ```

3. 在调用updateRoute更新路由之前对跳转路由和当前路由进行确认，主要是通过两者的路由对象和层级关系判断跳转路由与当前页路由是否一致。
   
   ```
    isSameRoute(a, b) {
        if (b === START) {
            return a === b
        } else if (!b) {
            return false
        } else if (a.path && b.path) {
            return (
            a.path.replace(trailingSlashRE, '') === b.path.replace(trailingSlashRE, '') &&
            a.hash === b.hash &&
            isObjectEqual(a.query, b.query)
            )
        } else if (a.name && b.name) {
            return (
            a.name === b.name &&
            a.hash === b.hash &&
            isObjectEqual(a.query, b.query) &&
            isObjectEqual(a.params, b.params)
            )
        } else {
            return false
        }
    }
   ```

4. updateRoute方法就会更新VueRouter实例中apps属性每一个对象（根组件）的_route，然后触发_route属性的挟持。_route属性被挂载到this.$route上供其他组件调用。
   
   ```
    class VueRouter {
        // ...
        function init(app) {
            this.apps.push(app)

            if (this.app) return

            this.app = app
            const history = this.history
            // ...
            history.listen(route => {
                this.apps.forEach(app => {
                    app._route = route
                })
            })
        }
    }

    class History {
        // ...
        listen(cb) {
            this.cb = cb
        }

        updateRoute(route) {
            // ...
            this.cb && this.cb(route)
        }
    }

    function install() {
        // ...
        Vue.mixin({
            beforeCreate() {
                Vue.util.defineReactive(this, '_route', this._router.history.current)
            }
        })

        Object.defineProperty(Vue.prototype, '$route', {
            get () { return this._routerRoot._route }
        })
    }
    ```

5. 当$route属性变化时，RouterView中渲染的component就会随之改变，实现页面内容的变更。

    ```
    render(_, { props, children, parent, data }) {
        const h = parent.$createElement
        const route = parent.$route
        const depth = 0
        // ...
        const matched = route.matched[depth]

        return h(component, data, children)
    }
    ```