# Proxy代理
@ 2019-11-08

---

Vue都3了...

除了一些七七八八的特性，其中新一代的Vue对于其响应式的数据实现方式，换了搞法，从getter、setter变成了ES6的Proxy

我个人理解，Proxy这种代理，不仅仅能够更加方便的处理响应式数据，还提供了针对函数的执行、new操作等一系列新的处理方式，在此我依照我个人的理解，尝试去实现了下这种新的响应式数据的处理方式。

这部分demo代码有以下限制：

1. computed里面所涉及到的所有数据，均从data中获取，没有进行循环检测
2. data里面的数据只支持一层，所有参数不能是数组、结构体

```javascript
let Vue = function(opts) {
  let { data, computed } = opts
  let computedMap = new Map() // 用以存储，每个data属性会影响哪些computed值

  let addComputedMap = function(dataKey, computedKey) {
    // 每个data里的参数变动，会影响哪些computed参数的值
    let arr = computedMap.get(dataKey)
    if(!arr) {
      computedMap.set(dataKey, [ computedKey ])
    } else {
      computedMap.set(dataKey, [...arr, computedKey])
    }
  }

  // 专门用于设定computedMap参数的proxy
  let computedProxy = new Proxy(Object.assign({}, data, computed), {
    get(target, attr) {
      if(setComputedKey) {
        // 设定特定computedKey，需要读取哪个attr
        addComputedMap(attr, setComputedKey)
      }
      return Reflect.get(target, attr)
    },
    set(_1, attr, _2, receiver) {
      // 找到指定的computed，调用对应的函数，从而确定计算指定的computed需要哪些data数据
      computed[attr].call(receiver)
    }
  })

  // 更新computedMap
  let setComputedKey = ''

  Object.keys(computed).forEach(func => {
    setComputedKey = func
    computedProxy[func] = ''
    setComputedKey = ''
  })

  // 返回真正提供给用户的Proxy
  let proxied = new Proxy(Object.assign({}, data, computed), {
      set(target, attr, value, receiver) {
        // 设定值
        Reflect.set(target, attr, value)

        // 找到收到影响的computed
        let arr = computedMap.get(attr)
        if(arr) {
          arr.forEach(computedAttr => {
            // 将所有待更新的computed数据重新计算一遍
            computed[computedAttr] && (receiver[computedAttr] = computed[computedAttr].call(receiver))
          })
        }
      }
    })
  
  // 更新所有默认数据，从而更新对应的computed值
  Object.keys(data).forEach(d => proxied[d] = data[d])

  return proxied
}

let instance = Vue({
  data: {
    att1: 1,
    att2: 2
  },
  computed: {
    add() { return this.att1 + this.att2 },
    minus() { return this.att1 - this.att2 },
    sum() { return this.add + this.minus }
  }
})

console.log(instance) // { att1: 1, att2: 2, sum: 3, minus: -1 }
instance.att1 = 3
instance.att2 = 1
console.log(instance) // { att1: 3, att2: 1, sum: 4, minus: 2 }
```

上述代码，构建了一个Proxy代理，真实的所数据值都存储在Proxy对应的原始target上，每当有设定了data值，则会根据预先存储的Map找到受到影响的computed值，从而在设定时就更新了computed值。

不过依然有些问题，譬如：
1. 如果data是多层的结构，如何将data作为一个多层的Proxy值，每次修改都能触发修改最外层的computed值
2. 没有校验computed彼此之间的循环调用（譬如计算A需要B，计算B又需要A），用户去设定值的时候没有去校验是否设定的为computed值
