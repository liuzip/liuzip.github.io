# 四则运算AST
@ 2019-04-27

---

重学前端里，有一个小实验，讲实现一个四则运算的AST，现在在看winter的说明之前，先自己写一个试试看，主要有三个模块，tokenizer解析字符串，parser转换为AST，executor执行AST：

```javascript
function tokenizer(input) {
  let pointer = 0 // 指向当前解析的位置
  let tokens = [] // 解析的结果
  let WHITE_SPACE = /\s/ // 空格
  let NUMBERS = /[0-9]/ // 数字
  let number = '' // 用以支持小数

  while(pointer < input.length) {
    let char = input[pointer]

    // 数字或小数点
    if(NUMBERS.test(char) || char === '.') {
      number += char
      pointer ++
      continue
    } else if(number.length !== 0) {
      // 如果之前连续读的数字，则存下来
      tokens.push({
        type: 'number',
        value: Number(number)
      })
      number = ''
    }

    if(char === '(' || char === ')') {
      // 判断括号
      tokens.push({
        type: 'parenthesis',
        value: char
      })
    } else if(char === '+' || char === '-' ||
      char === '*' || char === '/') {
      // 操作符
      tokens.push({
        type: 'operator',
        value: char
      })
    } else if(WHITE_SPACE.test(char)) {
      // 是空格
    } else {
      console.error('unsupport character!')
    }

    pointer ++
  }

  // 如果是以数字结尾的
  if(number.length !== 0) {
    // 如果之前连续读的数字，则存下来
    tokens.push({
      type: 'number',
      value: Number(number)
    })
  }

  return tokens
}

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
// 是以操作符作为根节点来进行计算的
// 生成的数据结构应该能够表明四则运算的操作顺序
// {
//   type: '', // Operator - 操作符 Number - 数字
//   value: '', // 数值
//   children: [Object, Object] // 如果是操作符，则具有俩子节点
// }
// 这个AST树的特点，就是优先级较高的，放在树杈上，优先级较低的，放在树根上
// 因此获得一个多项式的时候，首先找优先级最低的操作（+、-），再将其分为左右两块
// 分别对左右两块进行迭代操作，直至其中一边只是两个数字为止
function parser(tokens) {
  let ast = {
    type: 'Caculator',
    body: null,
  } // 即将转换成为的AST
  
  // 用来处理一整个ast树
  let astWalker = function(tokens) {
    let node = {
      type: 'Number',
      value: '',
      children: []
    }

    // 只剩一个的时候，一定是个数字
    if(tokens.length === 1) {
      node.node = 'Number'
      node.value = tokens[0].value
    } else {
      // 当前多项式还有操作符，则先找到优先级最低的操作符（括号外的加减号）
      let pointer = 0
      let inParenthesisFlag = 0 // 表明当前指针是否指在括号内，数字每加1，则表示在一级括号内
      let parenthesisPlusPos = {
        level: -1, // 处于的括号深度，0是不在括号内
        position: -1 // 位置
      } // 位于括号内的加减号的位置
      let parenthesisMultiPos = {
        level: -1, // 处于的括号深度，0是不在括号内
        position: -1 // 位置
      } // 位于括号内的乘除号的位置

      while(pointer < tokens.length) {
        let token = tokens[pointer]
        // 过滤括号
        if(token.type === 'parenthesis') {
          if(token.value === '(') {
            inParenthesisFlag ++
          } else if(token.value === ')') {
            inParenthesisFlag --
          }
          pointer ++
          continue
        }
        // 找到操作符
        if(token.type === 'operator') {
          // 如果是加减号
          if(token.value === '+' || token.value === '-') {
            // 如果加减号不在括号内，那么他的优先级最低了，以他拆为两半
            if(inParenthesisFlag === 0){
              node.node = 'Operator'
              node.value = token.value
              node.children.push(
                astWalker(tokens.slice(0, pointer)),
                astWalker(tokens.slice(pointer + 1, tokens.length))
              )
              break
            } else {
              // 如果是在括号内，记住位置
              if(parenthesisPlusPos.level === -1 || parenthesisPlusPos.level > inParenthesisFlag) {
                // 如果未更新，或者已经记录的位置不是最浅的，更新之
                parenthesisPlusPos.level = inParenthesisFlag
                parenthesisPlusPos.position = pointer
              }

              pointer ++
              continue
            }
          } else {
            // 如果遇到的是乘除号
            if(parenthesisMultiPos.level === -1 || parenthesisMultiPos.level > inParenthesisFlag) {
              // 如果未更新，或者已经记录的位置不是最浅的，更新之
              parenthesisMultiPos.level = inParenthesisFlag
              parenthesisMultiPos.position = pointer
            }
            pointer ++
            continue
          }
        }
      }
    }

    if(node.children.length === 0) {
      // 说明没有更新过，试着看看parenthesisPlusPos和parenthesisMultiPos的值
      let { level: mLevel, position: mPos } = parenthesisMultiPos
      let { level: pLevel, position: pPos } = parenthesisPlusPos
      let pos = 0

      if(mLevel !== -1 && pLevel !== -1) {
        // 都找到了
        if(pLevel > mLevel) {
          // 只有乘法在更浅的括号内，才会以他为准
          pos = mPos
        } else {
          // 同级或者，加减法更前，则用加减法的
          pos = pPos
        }
      } else if(mLevel !== -1) {
        pos = mPos
      } else if(pLevel !== -1) {
        pos = mPos
      } else {
        return node
      }

      node.node = 'Operator'
      node.value = tokens[pos].value
      node.children.push(
        astWalker(tokens.slice(0, pos)),
        astWalker(tokens.slice(pos + 1, tokens.length))
      )
    }

    return node
  }
}

```

<s>to be continued...</s>

[完成版本](https://liuzip.github.io/2019/12/calculateASTV2.md)
