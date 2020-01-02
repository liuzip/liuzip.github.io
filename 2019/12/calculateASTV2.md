# 四则运算AST完成版本
@ 2019-12-15

---

一共两大块部分，tokenizer来解析整个四则运算的表达式，变成tokens，walker来解析并生成ast树，最后计算嘛，很随意

使用eval是为了适配另外一个项目，写的一个类似Vue的小东西，其中需要计算模板文件中的文本内容（双大括号），因此需要使用eval来进行四则运算

```javascript
export function calculation(expression) {
  let tokens = tokenizer(expression)
  let ast = walker(tokens)()
  return function calculateExpression(target, node = ast) {
    if(node.type === 'Parameter')
      return get(target, node.value, '')
    else {
      let left = calculateExpression(target, node.children[0])
      let right = calculateExpression(target, node.children[1])
      let result
      eval(`result = ${ left } ${ node.value } ${ right }`)
      return result
    }
  }
}

// helpers
// 拆分操作符
function tokenizer(expression) {
  return expression.split(/(\+|-|\*|\/|\(|\))/g)
      .map(w => w.trim()).filter(Boolean)
      .map(w => {
        if(w === '(' || w === ')')
          return { type: 'parenthesis', value: w }
        else if(w === '+' || w === '-' || w === '*' || w === '/')
          return { type: 'operator', value: w }
        else
          return { type: 'parameter', value: w }
      })
}

// 建立AST tree
// 四则运算的AST树：
// 1+2*3+(4-5*6)*7
//   +
//  / \
// 1    +
//    /   \
//   *     *
//  / \   / \
// 2   3 -   7
//      / \
//     4   *
//        / \
//       5   6

// 生成的数据结构应该能够表明四则运算的操作顺序
// {
//   type: '', // Operator - 操作符 Parameter - 操作对象
//   value: '', // 数值
//   children: [Object, Object] // 如果是操作符，则具有俩子节点
// }
function walker(tokens) {
  function isThereParenthesis(list) {
    return [
      list.findIndex(t => t.type === 'parenthesis' && t.value === '('),
      findLastIndex.call(list, t => t.type === 'parenthesis' && t.value === ')')
    ]
  }
  
  function findLastIndex(cb) {
    let index = this.map(t => t).reverse().findIndex(cb)
    return index === -1 ? index : this.length - index - 1
  }

  function isInvalidExpression() { throw Error('invalid expression') } // 异常表达式
  function findOperator(operator) { return this.findIndex(t => t.type === 'operator' && operator.some(o => o === t.value))}

  return function parser(start = 0, end = tokens.length) {
    if(tokens[start] && tokens[start].value === '(' && tokens[end - 1] && tokens[end - 1].value === ')') {
      start ++
      end --
    }

    if(start >= end) // 取完了，或者传入得是类似()这般得表达式
      return null
  
    let list = tokens.slice(start, end) // 待处理得字段
    let [ sp, ep ] = isThereParenthesis(list)

    if(list.length === 1) // 只有一个点，只能是数字
        return list[0].type === 'parameter' ? { type: 'Parameter', value: list[0].value } : isInvalidExpression()
    else if(sp === -1 && ep === -1) {
      let lowerOperator = findOperator.call(list, ['+', '-'] ) // 首个低级操作符
      let operator = (lowerOperator === -1 ?
                      findOperator.call(list, ['*', '/'] ) :
                      lowerOperator) + start // 待分类的操作符
      if(operator < start)
        isInvalidExpression() // 没找到操作符
      else
        return {
          type: 'Operator',
          value: tokens[operator].value,
          children: [
            parser(start, operator),
            parser(operator + 1, end)
          ]
        }
    }
    else if(sp !== -1 && ep !== -1) {
      let operator = findOperator.call(list.slice(0, sp), ['+', '-'] ) // 括号左半部分低级操作符
      let offset = 0
      if(operator === -1) { // 括号右半部分低级操作符
        operator = findOperator.call(list.slice(ep, list.length), ['+', '-'] )
        offset = ep
      }
        
      if(operator === -1) { // 括号左半部分高级操作符
        operator = findOperator.call(list.slice(0, sp), ['*', '/'] )
        offset = 0
      }

      if(operator === -1) { // 括号右半部分高级操作符
        operator = findOperator.call(list.slice(ep, list.length), ['*', '/'] )
        offset = ep
      }
        
      if(operator !== -1) {
        operator += (start + offset)
        return {
          type: 'Operator',
          value: tokens[operator].value,
          children: [
            parser(start, operator),
            parser(operator + 1, end)
          ]
        }
      } else
        isInvalidExpression() // 括号两侧没有操作符，直接接了数字
    }
    else
      isInvalidExpression()
  }
}

// 获取指定路径上的结果
function get(target, path, notFoundValue = undefined) {
  if(/^\d+(\.\d+)?$/.test(path)) // 纯数字
    return path
  else {
    let keyPath = path.split('.')

    for(let i = 0; i < keyPath.length; i++) {
      if (target && target.hasOwnProperty(keyPath[i]))
        target = target[keyPath[i]];
      else
        return notFoundValue
    }
  
    return target
  }
}
```

Oh, yeah~