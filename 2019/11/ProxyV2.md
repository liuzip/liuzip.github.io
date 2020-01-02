# Proxy V2
@ 2019-11-18

---

升级啦，data参数支持多层得了：
```javascript
const isSymbol = (val) => typeof val === 'symbol'
const builtInSymbols = new Set(Object.getOwnPropertyNames(Symbol)
      .map(key => Symbol[key])
      .filter(isSymbol))

let proxied = {
  instance: null,
  computedStack: null,
  currentComputedKey: ''
}

export default function({ data, computed = {}, methods = {} }) {
  let assembled = Object.assign({},
              JSON.parse(JSON.stringify(data)), computed, methods)
  let mapStack = new Map() // 用于存储data数据和computed数据得映射关系

  proxied.computedStack = computed

  proxied.instance = proxify(assembled, mapStack, proxied) // 创建proxy对象

  // 设定computed和data数据之间得依赖关系
  Object.keys(computed).forEach(func => {
    proxied.currentComputedKey = func
    computed[func].call(proxied.instance)
  })
  delete proxied.currentComputedKey

  // 设定data数据，以更新comupted数据
  dataSet(data, proxied.instance)

  return proxied.instance
}

// helpers
function dataSet(data, p) {
  Object.keys(data).forEach(d => {
    typeof(data[d]) === 'object' ? dataSet(data[d], p[d]) : (p[d] = data[d])
  })
}

function proxify(data, mapStack, proxied) {
  return new Proxy(data, {
    get(target, property, receiver) {
      let res = Reflect.get(target, property, receiver)
      // 如果是proxy中得symbol，直接返回结果
      if(isSymbol(property) && builtInSymbols.has(property))
        return res

      if(typeof(res) === 'object')
        return proxify(res, mapStack, proxied) // 子层object也需要监控
      else {
        if(proxied.currentComputedKey) {
          // 如果是更新依赖环节，还需要更新依赖关系
          updateStack(target, property, mapStack, proxied.currentComputedKey)
        }
        return res
      }
    },
    set(target, property, value) {
      Reflect.set(target, property, value)
      let updateArr = getItemFromStack(target, property, mapStack) // 如果有依赖需要被更新
      if(updateArr !== void 0) {
        updateArr.forEach(key => { // 更新相应得computed数据
          Reflect.set(proxied.instance, key, proxied.computedStack[key].call(proxied.instance))
        })
      }

      return true
    }
  })
}

/*
* mapStack Map( 
*   porpertyStack Map( target 为 key
*     property Set( computed ), 设定的target的property为key，受到影响的computed为Set Value
*     property Set( computed )
*   )
* )
*/
function updateStack(target, property, mapStack, key) {
  let porpertyStack = mapStack.get(target)
  if(porpertyStack === void 0)
    mapStack.set(target, (porpertyStack = new Map()))
  let dep = porpertyStack.get(property)
  if(dep === void 0)
    porpertyStack.set(property, (dep = new Set()))
  if(key)
    dep.add(key)
}

function getItemFromStack(target, property, mapStack) {
  let porpertyStack = mapStack.get(target)
  if(porpertyStack === void 0) 
    return 
  return porpertyStack.get(property)
}

```

实际使用时候，对应得测试：
```javascript
import Vue from './core/proxy'

let instance = Vue({
  data: {
    att1: 1,
    att2: 2,
    attribute: {
      val: 2,
      arr: [ 1, 2, 3 ]
    }
  },
  computed: {
    special() { return this.attribute.val + this.attribute.arr[0] },
    add() { return this.att1 + this.att2 },
    minus() { return this.att1 - this.att2 },
    sum() { return this.add + this.minus }
  },
  methods: {
    multiple() {
      return this.att1 * this.att2
    }
  }
})


console.log('************************************')
console.log(instance) // { att1: 1, att2: 2, attribute: { val: 2, arr: [ 1, 2, 3 ] }, special: 3, add: 3, minus: -1, sum: 2, multiple: [Function: multiple] }
console.log(instance.multiple()) // 2

instance.att1 = 3
instance.att2 = 5
instance.attribute.val = 5
console.log('************************************')

console.log(instance.multiple()) // 15

console.log(instance) // { att1: 3, att2: 5, attribute: { val: 5, arr: [ 1, 2, 3 ] }, special: 6, add: 8, minus: -2, sum: 6, multiple: [Function: multiple] }
```

和之前[第一版](https://liuzip.github.io/2019/11/Proxy.md)得区别在于：
1. data和computed得映射关系，虽然还是使用map作为数据存储得容器，但是key指开始使用对应得对象
2. 迭代更新Proxy中得data数据，[之前写过一个版本](https://github.com/liuzip/proxy/commit/e578781a68dbbefdf9b14b25b9fafca539f653df)，是先转换成一层层得proxy对象，再统一使用new Proxy，但这种做法会使得生成很多新的对象，再查找Map得时候，会有问题

其实整体思路是一样得，先生成Proxy，再确定映射关系，更新data值来更新computed值，一旦更新数据，查找Map找到受到影响得数据，再更新对应得数据就好

