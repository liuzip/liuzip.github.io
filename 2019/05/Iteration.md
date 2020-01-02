# 尾调优化
@2019-05-28

---

问：迭代计算阶乘函数怎么写？
答：
```javascript
function m(i, v = 1) {
  if(i > 0) {
    v *= i
    return m(-- i, v)
  } else {
    return v
  }
}

m(10)
```
嗯，计算<b>m(10)</b>输出3628800，正确
计算<b>m(10000000)</b>傻眼

调用m，入栈，执行，哎？<b>i > 0</b>，再调用m，又入栈，执行，哎？<b>i > 0</b>，还调用m...
然后栈满了...

抛开严格模式下，尾调优化，对于尾调函数的处理，我们如果想要阻止迭代函数疯狂占用内存，就需要改变函数结构，阻止每次调用自身：

---

## 方案1：
每次不要在函数内部执行m，对于m的调用放在外部：
```javascript
function trans(f) {
  return function(...args) {
    let func = f
    func = func(...args)
    while(func instanceof Function) {
      func = func()
    }
    return func
  }
}


let m = trans(function foo(i, v = 1) {
  if(i > 0) {
    v *= i
    return foo.bind(null, -- i, v)
  } else {
    return v
  }
})

m(10)
```

迭代函数每次返回的是一个bind函数，这样保证了所谓的迭代函数每次都不会自身调用，循环部分放在外部，这样就可以使得每次函数入栈后能够及时出栈，避免了内存的溢出。

---

## 方案2：
方案1的搞法，实际上是改变了迭代函数的返回值，从某种程度上来讲，已经不再是真正的迭代函数了，所以我们来看一种别的办法来实现尾调优化。
```javascript
function trans(f) {
  let iterating = false
  let args = []
  let res = null

  return function(...arg) {
    args.push(arg)
    if(!iterating) {
      iterating = true
      while(args.length) {
        res = f(...(args.shift()))
      }
      iterating = false
      args = []
    }
    return res
  }
}

let m = trans(function(i, v = 1) {
  if(i > 0) {
    v *= i
    return m(-- i, v)
  } else {
    return v
  }
})

m(10)
```

这个方案，比较类似于python里面的装饰器，把阶乘函数封装了一下：
迭代函数是循环执行的，这个没错，只是每次传入的参数不一样，那理论上我们只需要把传入的参数存起来，循环的传入迭代函数就好啦。所以装饰器修改过后的函数（上述代码中的m），所干的事，就是首次运行的时候，把处理函数（也就是主要逻辑运算的迭代函数，trans所传入的匿名函数）存起来，然后函数m每次执行的时候，就是把形参保存，让迭代函数循环处理，然后迭代函数再调用m，这么一次次循环，直到迭代函数不再调用m，返回了结果，则把这个结果输出即可。

---

以上代码，仅限迭代函数尾调优化，如果迭代完了又处理了下结果再返回，心有余而力不足...
