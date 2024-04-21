---
title: Go抽象语法树（AST）实用指南 （二）
seo_title: Go抽象语法树（AST）实用指南 （二）
categories:
  - Go语言实战系列
book: 柠芒技术博客
book_title: Go抽象语法树（AST）实用指南 （二）
date: 2021-12-26 23:32:00
description:
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/AST/Go%E6%8A%BD%E8%B1%A1%E8%AF%AD%E6%B3%95%E6%A0%91%EF%BC%88AST%EF%BC%89%E5%AE%9E%E7%94%A8%E6%8C%87%E5%8D%97%20%EF%BC%88%E4%BA%8C%EF%BC%89.md
id: post-2112262332
---

> grade == "四(3)班" && age > 10

AST 抽象语法树在表达式解析上面也有不错的表现。通常在规则引擎场景可以得到充分的发挥。
比如一段表达式
```golang
grade == 四(3)班 && age > 10
```

这个表达式，对于人来说是非常容易理解的。年级是四(3)班，年纪大于10。

但是计算机来说，传入一段字符串表达式，理解起来可能并不是这么容易。

好在Golang提供了表达式解析的能力，我们可以通过查看解析后的数据，理解ast如果解析这串表达式。

### 解析表达式

```golang
 0  *ast.BinaryExpr {
 1  .  X: *ast.BinaryExpr {
 2  .  .  X: *ast.Ident {
 3  .  .  .  NamePos: -
 4  .  .  .  Name: "grade"
 5  .  .  .  Obj: *ast.Object {
 6  .  .  .  .  Kind: bad
 7  .  .  .  .  Name: ""
 8  .  .  .  }
 9  .  .  }
10  .  .  OpPos: -
11  .  .  Op: ==
12  .  .  Y: *ast.BasicLit {
13  .  .  .  ValuePos: -
14  .  .  .  Kind: STRING
15  .  .  .  Value: "\"四(3)班\""
16  .  .  }
17  .  }
18  .  OpPos: -
19  .  Op: &&
20  .  Y: *ast.BinaryExpr {
21  .  .  X: *ast.Ident {
22  .  .  .  NamePos: -
23  .  .  .  Name: "age"
24  .  .  .  Obj: *(obj @ 5)
25  .  .  }
26  .  .  OpPos: -
27  .  .  Op: >
28  .  .  Y: *ast.BasicLit {
29  .  .  .  ValuePos: -
30  .  .  .  Kind: INT
31  .  .  .  Value: "10"
32  .  .  }
33  .  }
34  }
```
### 分析表达式解析结果

 通过AST抽象语法树可以看到，表达式已经被准确地解析出来了。正如我们理解的，优先解析了最外层的 `&&` 。`&&` 前后的两个判断条件各自成为一个二进制节点。二进制节点

![](https://image.baidu.com/search/down?url=https://tva1.sinaimg.cn/large/008i3skNgy1gxrpsmy2e1j31he0u0432.jpg)


### 实现一个变量解析器（二元表达式）
有这么一个场景，用户发起了请求，传递了如下参数
```json
{
	"grade":"\"四(3)班\"",
	"name":"小明",
	"age": 10
}
```
需要用上面提到的表达式去判断是否符合条件。从抽象语法树上可以知道，不管在哪个层级，都是 X op Y的形式。所以可以通过递归的方式进行递归，直到根节点。


#### 判断是否是根节点
只有到根节点了，才能进行判断最终值。个节点是一个`*ast.BinaryExpr`节点，它符合`X.Type == *ast.Ident` 且 `Y.Type == *ast.BasicLit`
```golang
func isLeaf(node ast.Node) bool {
  expr, ok := node.(*ast.BinaryExpr)
  if !ok {
    return false
  }

  _, okL := expr.X.(*ast.Ident)
  _, okR := expr.Y.(*ast.BasicLit)
  if okL && okR {
    return true
  }

  return false
}
```

#### 递归判断结果值
```golang
func judge(node ast.Node, m map[string]interface{}) bool {
  if isLeaf(node) {
    // 断言成二元表达式
    expr := node.(*ast.BinaryExpr)
    x := expr.X.(*ast.Ident)    // 左边
    y := expr.Y.(*ast.BasicLit) // 右边

    // 仅处理表达式
    switch expr.Op {
    case token.GTR:
      left := cast.ToFloat64(m[x.Name])
      right := cast.ToFloat64(y.Value)
      return left > right

    case token.EQL:
      left, _ := m[x.Name]
      right := y.Value
      switch y.Kind {
      case token.INT:
        return cast.ToInt64(left) == cast.ToInt64(right)
      case token.FLOAT:
        return cast.ToFloat64(left) == cast.ToFloat64(right)
      case token.IMAG:
        return cast.ToFloat64(left) == cast.ToFloat64(right)
      case token.CHAR:
        return cast.ToInt(left) == cast.ToInt(right)
      case token.STRING:
        return cast.ToString(left) == cast.ToString(right)
      }
      return false
    }

    return false
  }

  // 不是叶子节点那么一定是 binary expression（我们目前只处理二元表达式）
  expr, ok := node.(*ast.BinaryExpr)
  if !ok {
    return false
  }

  // 递归地计算左节点和右节点的值
  switch expr.Op {
  case token.LAND:
    return judge(expr.X, m) && judge(expr.Y, m)
  case token.LOR:
    return judge(expr.X, m) || judge(expr.Y, m)
  }
  return false
}
```

#### 测试用例
```golang
`grade == "四(三)班" && age > 9`
```

```golang
`grade == "四(三)班" && age == 9`
```

```golang
`grade == "四(三)班" && age > 10`
```

```golang
`grade == "四(三)班" && age < 11`
```

```golang
`grade == "四(二)班" && age > 9`
```

```golang
`grade == "四(二)班" || age > 9`
```

### 扩展
#### 案例一
当传入的参数为如下，该如何解析`grade.group`
```json
{
    "grade":{
        "group":"\"四(三)班\""
    },
    "name":"小明",
    "age":10,
}
```



#### 案例二
当传入的参数为如下，又该如何解析
```json
{
    "jump": 160,
    "name":"小明",
    "age":10,
}
```
rule: `jump > 150 && jump / age > 16`



