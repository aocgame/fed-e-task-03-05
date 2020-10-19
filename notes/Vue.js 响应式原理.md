# Vue.js 响应式回顾

- Proxy 对象实现属性监听
- 多层属性嵌套，在访问属性过程中处理下一级属性
- 默认监听动态添加的属性
- 默认监听属性的删除操作
- 默认监听数组索引和 length 属性
- 可以作为单独的模块使用



## 核心方法

- `reactive`/ `ref` / `toRefs` / `computed`
- `effect`
- `track`
- `trigger`



### Proxy 对象回顾

```javascript
//  set 和 deleteProperty 中需要返回布尔类型的值
// 在严格模式下，如果返回 false 的话会出现 Type Error 的异常
const target = {
  foo: 'xxx',
  bar: 'yyy'
}
// Reflect.getPropertyOf()
// Object.getPropertyOf()
const proxy = new Proxy(target, {
  get (target, key, receiver) {
    // return target[key]
    return Reflect.get(target, key, receiver)
  },
  set (target, key, value, receiver) {
    // target[key] = value
    return Reflect.set(target, key, value, receiver)
  },
  deleteProperty (target, key) {
    // delete target[key]
    return Reflect.deleteProperty(target, key)
  }
})

proxy.foo = 'zzz'

// Proxy 和 Reflect 中使用的 receiver

// Proxy 中 receiver: Proxy 或者继承 Proxy 的对象
// Reflect 中 receiver: 如果 target 对象中设置了 getter，getter 中的 this 指向 receiver
const obj = {
  get foo() {
    console.log(this)
    return this.bar
  }
}

const proxy = new Proxy(obj, {
  get (target, key, receiver) {
    if (key === 'bar') {
      return 'value - bar'
    }
    return Reflect.get(target, key, receiver)
  }
})
console.log(proxy.foo)
```



### reactive

- 接收一个参数，判断这参数是否是对象
- 创建拦截器对象 `handler`，设置 `get` / `set` / `deleteProperty`
- 返回 `Proxy` 对象

```javascript
const isObject = val => val !== null && typeof val === 'object'
const convert = target => isObject(target) ? reactive(target) : target
const hasOwnProperty = Object.prototype.hasOwnProperty
const hasOwn = (target, key) => hasOwnProperty.call(target, key)

export function reactive(target) {
  if (!isObject(target)) return target
  
  const handler = {
    get (target, key, receiver) {
      // 收集依赖
      console.log('get', key)
      const result = Reflect.get(target, key, receiver)
      return convert(result)
    },
    set (target, key, value, receiver) {
      const oldValue = Reflect.get(target, key, receiver)
      let result = true
      if (oldValue !== value) {
        result = Reflect.set(target, key, value, receiver)
        // 触发更新
        console.log('set', key, value)
      }
      return result
    },
    deleteProperty (target, key) {
      const hadKey = hasOwn(target, key)
      const result = Reflect.deleteProperty(target, key)
      if (hadKey && result) {
        // 触发更新
        console.log('delete', key)
      }
      return result
    }
  }
  
  return new Proxy(target, handler)
}

```



### 收集依赖

| targetMap       | depsMap            | Dep           |
| --------------- | ------------------ | ------------- |
| `new WeakMap()` | `new Map()`        | `new Set()`   |
| 目标对象        | 目标对象的属性名称 | effect 的函数 |

#### `effect` && `track`

```js
let activeEffect = null
export function effect (callback) {
  activeEffect = callback
  callback() // 访问响应式对象属性，去收集依赖
  activeEffect = null
}

let targetMap = new WeakMap()

export function track (target, key) {
  if (!activeEffect) return
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  dep.add(activeEffect)
}


// 在 reactive 中
const handle = {
  get (target, key, receiver) {
    // 收集依赖
    track(target, key)
  }
}
```



### 触发更新

`trigger`

```js
export function trigger (target, key) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return
  const dep = depsMap.get(key)
  if (dep) {
    dep.forEach(effect => {
      effect()
    })
  }
}

// 在 reactive 中触发更新
// set 以及 deleteProperty 中
trigger(target, key)
```



### `ref`

```js
export function ref (raw) {
  // 判断 raw 是否是 ref 创建的对象，如果是的话直接返回
  if (isObject(raw) && raw.__v_isRef) return
  let value = convert(raw)
  const r = {
    __v_isRef: true,
    get value() {
      // 收集依赖
      track(r, 'value')
      return value
    },
    set value(newValue) {
      if (newValue !== value) {
        raw = newValue
        value = convert(raw)
        // 触发更新
        trigger(r, 'value')
      }
    }
  }
  return r
}
```

> `reactive` VS `ref`
>
> - `ref` 可以把基本数据类型数据，转换成响应式对象
> - `ref` 返回的对象，重新赋值成对象也是响应式的
> - `reactive` 返回的对象，重新赋值丢失响应式
> - `reactive` 返回的对象不可以解构

- `reactive`

  ```js
  const product = reactive({
    name: 'iPhone',
    price: 5000,
    count: 3
  })
  ```

- `ref`

  ```js
  const price = ref(5000)
  const count = ref(3)
  ```

  

### `toRefs`

把 reactive 函数返回的对象的每一个属性转换成类似 ref 返回的对象

```js
export function toRefs (proxy) {
  const ret = proxy instanceof Array ? new Array(proxy.length) : {}
  
  for (const key in proxy) {
    ret[key] = toProxyRef(proxy, key)
  }
  
  return ret
}

function toProxyRef (proxy, key) {
  const r = {
    __v_isRef: true,
    get value () {
      return proxy[key]
    },
    set value (newValue) {
      proxy[key] = newValue
    }
  }
  return r
}
```



### `computed`

```js
export function computed (getter) {
  const result = ref()
  
  effect(() => (result.value = getter()))
  
  return result
}
```

