# Vuex

Vuex是Vue的一个全局状态管理器，可以实现跨组件通信、管理状态以及统一状态入口等功能。

## 基本用法

```
import Vue from 'vue'
import Vuex from 'vuex'
import App from './App'

Vue.use(Vuex)

const store = new Vuex.Store({
    state: {
        num: 0,
    },
    getters: {
        outputNum: state => '0' + state.num
    },
    actions: {
        addNum({ commit }) {
            commit('add', {
                add: 1
            })
        }
    },
    mutations: {
        add(state, { add }) {
            state.num += add
        }
    }
})

new Vue({
    el: '#app',
    store,
    components: {
        App
    },
    template: '<App />
})
```

首先是用Vue.use注册Vuex，然后将Vuex中Store类的实例注册到Vue实例中。

Store类的实例由四部分组成，分别是state、getters、actions、mutations。
1. state存放原始状态（数据）。
2. getters是向外暴露的状态（数据）。
3. actions是操作行为，一般是异步，用于获取或提交数据。
4. mutations是操作状态，必须是同步，用于修改状态（state）。

借助这四个模块可以在Vue任意一个页面或组件引入状态和行为。

```
// index.vue
<template>
    <div>{{ outputNum }}</div>
    <button @click="addNum(1)">add</button>
<template>

<script>
import { mapGetters, mapActions } from 'vuex'

export default {
    computed: mapGetters([
        'outputNum'
    ]),
    methods: mapActions([
        'addNum'
    ])
}
</script>
```

上面的例子通过mapGetters将Store类中向外暴露的状态outputNum引入到页面上并作为一个computed的属性，页面可以直接访问，但不可以写入；还通过mapActions将Store实例中actions的方法addNum引入到methods中。

通过button触发就可以完成Store实例中状态num的修改，从而影响getters向外暴露的属性outputNum，进而呈现到页面上。

## 原理

使用Vue.use注册Vuex，实际上是调用了Vuex对象中install方法。

```
let Vue

function install(_Vue) {
    if (Vue && _Vue === Vue) return

    Vue = _Vue
    applyMixin(Vue)
}

function applyMixin(Vue) {
    Vue.mixin({
        beforeCreate: vuexInit
    })

    function vuexInit() {
        const options = this.$options

        if (options.store) {
            this.$store = typeof options.store === 'function' ? options.store() : options.store
        } else if (options.parent && options.parent.$store) {
            this.$store = options.parent.$store
        }
    }
}
```

install方法通过判断当前Vue实例是否已注册过，未注册就会将通过Vue.mixin方法在所有Vue实例中挂载一个beforeCreate方法，使得所有页面或组件被创建前都会将自身或父页面上的store对象挂载到this.$store供页面访问，这个store对象其实就是Store实例。

```
class Store {
    constructor(options = {}) {
        this._modules = new ModuleCollection(options)
    }
}

class ModuleCollection {
    constructor(rawRootModule) {
        this.register([], rawRootModule, false)
    }

    register(path, rawModule, runtime = true) {
        const newModule = new Module(rawModule, runtime)
        this.root = newModule
    }
}

class Module {
    constructor(rawModule, runtime) {
        this.runtime = runtime
        this._children = Object.create(null)
        this._rawModule = rawModule
        const rawState = rawModule.state
        this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
    }
}
```

Store类构造函数给实例赋值一个_modules对象，这个对象是ModuleCollection类的实例，而ModuleCollection类的构造函数又创建了一个Module类的实例并挂载到自己实例的root对象上，因此就会有以下结构。

```
store = {
    _modules: {
        root: {
            runtime: false,
            _children: {},
            _rawModule: {
                // options
            },
            state: options.state
        }
    }
}
```

（7.10更新）

除了_modules属性，Store类的构造函数还初始化了其他属性挂载到实例上。

```
class Store {
    constructor(options = {}) {
        this._actions = Object.create(null)
        this._mutations = Object.create(null)

        const state = this._modules.root.state
        installModule(this, state, [], this._modules.root)
    }
}

function installModule(store, rootState, path, module, hot) {
    const isRoot = !path.length
    const namespace = store._modules.getNamespace(path)
    const local = module.context = makeLocalContent(store, namespace, path)

    module.forEachMutation((mutation, key) => {
        const namespaceType = namespace + key
        registerMutation(store, namespaceType, mutation, local)
    })

    module.forEachAction((action, key) => {
        const type = action.root ? key : namespace + key
        const handler = action.handler || action
        registerAction(store, type, handler, local)
    })

    module.forEachGetter((getter, key) => {
        const namespaceType = namespace + key
        registerGetter(store, namespaceType, getter, local)
    })
}
```

installModule将options配置里定义的Mutations、Actions和Getters逐一挂载到Store类实例上。

通过getNamespace方法获取子模块并根据子模块的层次路径拼接出命名空间（namespace），使用命名空间避免子模块之间的命名冲突。但对于根模块，namespace为''。

```
getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
        module = module.getChild(key)
        return namespace + (module.namespaced ? key + '/' : '')
    }, '')
}
```

makeLocalContent会基于当前store对象创建一个上下文对象，这个对象包含了state和getters属性，这两个属性的get方法是指向Store类实例中的getters和state属性，state还基于路径做了处理。另外还会挂载两个调用方法dispatch和commit。

```
function makeLocalContext(store, namespace, path) {
    const local = {
        dispatch: store.dispatch,
        commit: store.commit
    }

    Object.defineProperties(local, {
        getters: {
            get: () => store.getters
        },
        state: {
            get: () => getNestedState(store.state, path)
        }
    })

    return local
}

function getNestedState(state, path) {
    return path.reduce((state, key) => state[key], state)
}
```

在挂载Mutations、Actions和Getters属性时，是通过遍历实例上的根模块属性（_modules.root）对应的options配置，然后基于命名空间（根模块命名空间为''）和上下文对象做处理，以Mutations为例。

```
module.forEachMutation((mutation, key) => {
    const namespaceType = namespace + key
    registerMutation(store, namespaceType, mutation, local)
})

class Module {
    // ...

    forEachMutation (fn) {
        if (this._rawModule.mutations) {
            forEachValue(this._rawModule.mutations, fn)
        }
    }
}

function forEachValue (obj, fn) {
    Object.keys(obj).forEach(key => fn(obj[key], key))
}

function registerMutation(store, type, handler, local) {
    const entry = store._mutations[type] || (store._mutations[type] = [])
    entry.push(function (payload) {
        handler.call(store, local.state, payload)
    })
}
```

回顾一下：

在创建Store类实例store时，实际上是创建了ModuleCollection类实例和Model类实例，最后基于Store类构造函数传入的配置options生成以下的结构。

```
store = {
    _modules: {
        root: {
            runtime: false,
            _children: {},
            _rawModule: {
                // options
            },
            state: options.state
        }
    }
}
```

因此，在options中定义的state、getters、mutations、actions都会被存储中_rawModule属性上。

forEachMutation方法就是遍历options上定义的mutations属性，并且根据当前模块的命名空间（namespace）与mutations上的key值组合作为存储的key值，最后用call方法将Store类实例store设为mutations中定义方法的this指向并传入两个参数（local.state和payload）。

payload是mutation在被调用时传入的参数。

```
class Store {
    constructor(options = {}) {
        get state() {
            return this._vm._data.$$store
        }

        set state(v) {

        }
    }
}
```

local.state是基于子模块层次路径处理过的store.state，store.state指向的则是Store类实例store中的_vm._data.$$state属性，并且重写了set方法，阻止直接写入state。而_vm则是resetStoreVM方法创建的（根模块）。

```
class Store {
    constructor(options = {}) {
        this._wrappedGetters = Object.create(null)
        const state = this._modules.root.state
        resetStoreVM(this, state)
    }
}

function resetStoreVM(store, state, hot) {
    const silent = Vue.config.silent
    Vue.config.silent = true
    store._vm = new Vue({
        data: {
            $$state: state
        }
    })
    Vue.config.silent = silent
}
```

resetStoreVM方法实际上创建了一个Vue实例并挂载到Store类实例store的_vm属性上，把store._modules.root.state放入到Vue实例上的data，这样Vue就会对store._modules.root.state做一个数据挟持。

在Vue中任何地方读取state上的属性都会被添加到对应的订阅者列表中（包括getters），当mutations中定义的方法对state中定义的属性进行修改，其实是通过local.state的指向对store._modules.root.state进行修改，然后通过订阅者列表通知所有读取state属性的位置更新。（Vue响应式原理）

```
class Store {
    constructor(options = {}) {
        const state = this._module.root.state
        installModule(this)
        resetStoreVM(this, state)
    }

    get state() {
        return this._vm._data.$$state
    }
}

function installModule (store) {
    const local = makeLocalContext(store)
}

function makeLocalContext(store) {
    const local = {}
    Object.defineProperties(local, {
        state: {
            get: () => getNestedState(store.state, path)
        }
    })
}

function resetStoreVM(store, state) {
    store._vm = new Vue({
        data: {
            $$state: state
        }
    })
}
```

形成的访问路径如下：

```
local.state -> store.state -> store._vm._data.$$state -> store._module.root.state
```

最终访问和操作的对象都是指向store._module.root.state上的属性，即在创建Store类实例时传入的options中state对象。

在页面中直接获取state一般会使用this.$store.state或者mapState方法。

```
// index.vue
<template>
    <div>{{ num }}</div>
<template>

<script>
import { mapState } from 'vuex'

export default {
    computed: mapState([
        'num'
    ])
}
</script>
```

mapState可以接收数组或对象作为参数，数组里的元素（对象的key）作为key值提取state中对应的属性并返回（参数为对象则重命名这个属性）。

```
mapState = normalizeNamespace((namespace, states) => {
    const res = {}
    
    normalizeMap(states).forEach(({key, val}) => {
        res[key] = function() {
            let state = this.$store.state
            return typeof val === 'function' ? val.call(this, state) : state[val]
        }
    })

    return res
})

function normalizeNamespace(fn) {
    return (namespace, map) => {
        if (typeof namespace !== 'string') {
            map = namespace
            namspace = ''
        } else if (namespace.charAt(namespace.length - 1) !== '/') {
            namespace += '/'
        }

        return fn(namespace, map)
    }
}

function normalizeMap(map) {
    return Array.isArray(map) ? map.map(key => ({ key, val: key })) : Object.keys(map).map(key => ({ key, val: map[key] }))
}
```

mapState会对参数进行处理，如果第一个参数不是命名空间就会将其延后作为第二个参数（states）。然后对第二个参数进行遍历获取key值，最终在Vue实例上的\$store.state获取对应的属性。

Vuex创建全局状态实现组件间通信是通过创建一个Vue实例挂载state并在页面实例上定义一个$store属性访问实现的。

（7.13更新）
Vuex设计的理念就是将状态修改路径设置为单向，实际表现就是只可以使用mutation函数更改state中的属性。

比如，直接对this.$store.state上的属性直接赋值，是无法更改\$store上的state状态的，因为set方法被重写了。

```
class Store {
    get state() {
        return this._vm._data.$$state
    }

    set state() {

    }
}
```

而mutation方法是通过makeLocalContext方法创建一个Store类实例store上_module.root.state属性指引，将其作为参数传入每一个mutation方法中，通过这个指引直接修改原始state属性。

```
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function (payload) {
    handler.call(store, local.state, payload)
  })
}
```

mutation方法的调用有三种：
1. mapMutation挂载到当前Vue实例。
2. this.$store.commit方法直接调用。
3. action方法通过commit调用。

mapMutations实际上也是调用commit方法实现。

```
const mapMutations = normalizeNamespace((namespace, mutations) => {
    const res = {}

    normalizeMap(mutations).forEach(({ key, val }) => {
        res[key] = function (...args) {
            let commit = this.$store.commit
            return typeof val === 'function' ? val.apply(this, [commit].concat(args)) : commit.apply(this.$store, [val].concat(args))
        }
    })

    return res
})
```

通过遍历传入的参数（数组或对象），将参数中需要提取的mutation封装一次，用$store上的commit方法调用。

```
class Store {
    commit(_type, _payload, _options) {
        const { type, payload, options } = unifyObjectStyle(_type, _payload, _options)

        const mutation = { type, payload }
        const entry = this._mutations[type]
        
        this._withCommit(() => {
            entry.forEach(function (handler) {
                handler(payload)
            })
        })
    }

    _withCommit(fn) {
        const committing = this._committing
        this._committing = true
        fn()
        this._com,itting = committing
    }
}

function unifyObjectStyle(type, payload, options) {
    if (isObject(type) && type.type) {
        options = payload
        payload = type
        type = type.type
    }

    return { type, payload, options }
}

```

commit方法是从Store实例中的_mutations属性提取对应的mutation方法，然后执行。在执行的过程中通过_committing方法加一个锁，防止mutation对state属性修改时其他方法副作用影响到state，导致状态值出现无法预料的变化。（即非单向改变）

这个_mutations属性中存放的元素就是registerMutation将mutation注入到实例时push进去的。

```
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function (payload) {
    handler.call(store, local.state, payload)
  })
}
```

对于action通过commit方法调用mutation更新state，也是通过Store实例上的commit方法。

```
function registerAction(store, type, handler, local) {
    const entry = store._actions[type] || (store._actions[type] = [])
    entry.push(function (payload) {
        let res = handler.call(store, {
            dispatch: local.dispatch,
            commit: local.commit,
            state: local.state,
            rootGetters: store.getters,
            rootState: store.state
        }, payload)

        if (!isPromise(res)) {
            res = Promise.resolve(res)
        }

        return res
    })
}
```

首先可以从action的注册过程可以看到传入的参数包含dispatch、commit等方法。跟mutation的过程比较像，都是在Store实例上推入一个列表，但action多做了Promise的判断。

另外，action本身是可以对state进行修改的，但一般mutation会对数据进行处理，所以通过commit方法调用mutation修改state属性会更符合修改路径单向的原则。

此外，action方法还接收rootState和rootGetters属性，可以通过这个属性跨模块访问state属性。

与mutation相似，action方法也有3个调用方式：
1. mapActions挂载到当前Vue实例。
2. this.$route.dispatch方法直接调用。
3. action方法之间的调用。

```
const mapActions = normalizeNamespace((name, actions) => {
    const res = {}
    normalizeMap(actions).forEach(({ key, val }) => {
        res[key] = function(..args) {
            let dispatch = this.$store.dispatch
            return typeof val === 'function' ? val.apply(this, [dispatch].concat(args)) : dispatch.apply(this.$store, [val].concat(args))
        }
    })

    return res
})
```

从上面两个例子可以看出，最终调用action方法的是Store实例上的dispatch方法。

```
class Store {
    dispatch (_type, _payload) {
        const { type, payload } = unifyObjectStyle(_type, _payload)
        const action = { type, payload }
        const entry = this._actions[type]

        const result = entry.length > 1 ? Promise.all(entry.map(handler => handler(payload))) : entry[0](payload)

        return result
    }
}
```

dispatch与commit也比较类似，只是加了Promise对异步action的处理。

最后，在Store类生成实例时，会将commit和dispatch方法的this指向实例store。

```
class Store {
    constructor (options = {}) {
        const store = this
        const { dispatch, commit } = this

        this.dispatch = function(type, payload) {
            return dispatch.call(store, type, payload)
        }

        this.commit = function(type, payload, options) {
            return commit.call(store, type, payload, options)
        }
    }
}
```