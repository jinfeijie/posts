---
title: Go抽象语法树（AST）实用指南 （一）
seo_title: Go抽象语法树（AST）实用指南 （一）
categories:
  - Go语言实战系列
book: Strcpy
book_title: Go抽象语法树（AST）实用指南 （一）
date: 2021-12-25 22:58:00
description:
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/infra/Go%E6%8A%BD%E8%B1%A1%E8%AF%AD%E6%B3%95%E6%A0%91%EF%BC%88AST%EF%BC%89%E5%AE%9E%E7%94%A8%E6%8C%87%E5%8D%97%20%EF%BC%88%E4%B8%80%EF%BC%89.md
id: post-2112252258 
---

AST抽象语法树在平时开发一般不太能用到。较多使用场景为代码生成，代码动态解析等等。
抽象到具体场景
- 生成MySQL的model
- 规则引擎动态解析脚本

本文以另一个使用场景呈现AST的应用场景。

### Java下枚举

```Java
package com.demo.risk.demo;

import lombok.NoArgsConstructor;

@NoArgsConstructor
public enum Status {

    StatusProcessing(1, "流程中"),
    StatusPass(2, "通过"),
    StatusReject(3, "拒绝");

    public int Id;
    public String Name;

    Status(int id, String name) {
        this.Id = id;
        this.Name = name;
    }
}
```

以上代码定义了三个枚举变量，可以很方便地获取到枚举内的id和name

```Java
public void Demo() {
    Status process = Status.StatusProcessing;
    int id = process.Id;
    String name = process.Name;
}
```


### Go常规枚举定义与实现
```golang
const (
  StatusBeg = iota
  StatusProcessing
  StatusPass
  StatusReject
  StatusEnd
)

var StatusName = map[int]string{
  StatusBeg:        "Unknown",
  StatusProcessing: "流程中",
  StatusPass:       "通过",
  StatusReject:     "拒绝",
  StatusEnd:        "Unknown",
}
```

如果想要实现则需要去手写一套这样的代码，去实现功能。


实际上以上的功能，通过AST可以非常简单地解决掉这个问题。

可能实现的方式不会有太大的变化，但是通过AST可以通过解析注释，动态地解析生成对应的代码。

### 实现原理

```golang
package main

//go:generate  gen-const-status
const (
  StatusBeg        = 0 // Unknown
  StatusProcessing = 1 // 流程中
  StatusPass       = 2 // 通过
  StatusReject     = 3 // 拒绝
  StatusEnd        = 4 // Unknown
)
```
有上面一段代码，用来定义状态流转枚举。

通过AST抽象语法树解析得到如下结果：
（下面代码块中`//# `后的为作者注释）
```golang
  0  *ast.File {
  1  .  Package: code.go:1:1 //# 意味着包定义从1:1开始
  2  .  Name: *ast.Ident {
  3  .  .  NamePos: code.go:1:9
  4  .  .  Name: "main"
  5  .  }
  6  .  Decls: []ast.Decl (len = 1) { //# 意味着有一个区块的常量被定义，换句话说有一个const关键字
  7  .  .  0: *ast.GenDecl {
  8  .  .  .  Doc: *ast.CommentGroup { //# 对于const上方的注释的描述
  9  .  .  .  .  List: []*ast.Comment (len = 1) { //# 意味着有一行注释
 10  .  .  .  .  .  0: *ast.Comment {
 11  .  .  .  .  .  .  Slash: code.go:3:1
 12  .  .  .  .  .  .  Text: "//go:generate  gen-const-status" //# 注释内容
 13  .  .  .  .  .  }
 14  .  .  .  .  }
 15  .  .  .  }
 16  .  .  .  TokPos: code.go:4:1
 17  .  .  .  Tok: const
 18  .  .  .  Lparen: code.go:4:7
 19  .  .  .  Specs: []ast.Spec (len = 5) { //# 意味着const内定义了5个常量
 20  .  .  .  .  0: *ast.ValueSpec {
 21  .  .  .  .  .  Names: []*ast.Ident (len = 1) {
 22  .  .  .  .  .  .  0: *ast.Ident {
 23  .  .  .  .  .  .  .  NamePos: code.go:5:2
 24  .  .  .  .  .  .  .  Name: "StatusBeg" //# 常量的名字叫StatusBeg
 25  .  .  .  .  .  .  .  Obj: *ast.Object {
 26  .  .  .  .  .  .  .  .  Kind: const //# 常量类型
 27  .  .  .  .  .  .  .  .  Name: "StatusBeg" //# 常量的名字叫StatusBeg
 28  .  .  .  .  .  .  .  .  Decl: *(obj @ 20) //# 描述在 @20 开始 // @20 指代指向20行的指针
 29  .  .  .  .  .  .  .  .  Data: 0
 30  .  .  .  .  .  .  .  }
 31  .  .  .  .  .  .  }
 32  .  .  .  .  .  }
 33  .  .  .  .  .  Values: []ast.Expr (len = 1) {
 34  .  .  .  .  .  .  0: *ast.BasicLit {
 35  .  .  .  .  .  .  .  ValuePos: code.go:5:21
 36  .  .  .  .  .  .  .  Kind: INT //# 常量是int类型
 37  .  .  .  .  .  .  .  Value: "0"  //# 常量的值为0
 38  .  .  .  .  .  .  }
 39  .  .  .  .  .  }
 40  .  .  .  .  .  Comment: *ast.CommentGroup {
 41  .  .  .  .  .  .  List: []*ast.Comment (len = 1) {
 42  .  .  .  .  .  .  .  0: *ast.Comment {
 43  .  .  .  .  .  .  .  .  Slash: code.go:5:23
 44  .  .  .  .  .  .  .  .  Text: "// Unknown" //# 常量的注释为 "// Unknown"
 45  .  .  .  .  .  .  .  }
 46  .  .  .  .  .  .  }
 47  .  .  .  .  .  }
 48  .  .  .  .  }
 49  .  .  .  .  1: *ast.ValueSpec {
 50  .  .  .  .  .  Names: []*ast.Ident (len = 1) {
 51  .  .  .  .  .  .  0: *ast.Ident {
 52  .  .  .  .  .  .  .  NamePos: code.go:6:2
 53  .  .  .  .  .  .  .  Name: "StatusProcessing"
 54  .  .  .  .  .  .  .  Obj: *ast.Object {
 55  .  .  .  .  .  .  .  .  Kind: const
 56  .  .  .  .  .  .  .  .  Name: "StatusProcessing"
 57  .  .  .  .  .  .  .  .  Decl: *(obj @ 49)
 58  .  .  .  .  .  .  .  .  Data: 1
 59  .  .  .  .  .  .  .  }
 60  .  .  .  .  .  .  }
 61  .  .  .  .  .  }
 62  .  .  .  .  .  Values: []ast.Expr (len = 1) {
 63  .  .  .  .  .  .  0: *ast.BasicLit {
 64  .  .  .  .  .  .  .  ValuePos: code.go:6:21
 65  .  .  .  .  .  .  .  Kind: INT
 66  .  .  .  .  .  .  .  Value: "1"
 67  .  .  .  .  .  .  }
 68  .  .  .  .  .  }
 69  .  .  .  .  .  Comment: *ast.CommentGroup {
 70  .  .  .  .  .  .  List: []*ast.Comment (len = 1) {
 71  .  .  .  .  .  .  .  0: *ast.Comment {
 72  .  .  .  .  .  .  .  .  Slash: code.go:6:23
 73  .  .  .  .  .  .  .  .  Text: "// 流程中"
 74  .  .  .  .  .  .  .  }
 75  .  .  .  .  .  .  }
 76  .  .  .  .  .  }
 77  .  .  .  .  }
 78  .  .  .  .  2: *ast.ValueSpec {
 79  .  .  .  .  .  Names: []*ast.Ident (len = 1) {
 80  .  .  .  .  .  .  0: *ast.Ident {
 81  .  .  .  .  .  .  .  NamePos: code.go:7:2
 82  .  .  .  .  .  .  .  Name: "StatusPass"
 83  .  .  .  .  .  .  .  Obj: *ast.Object {
 84  .  .  .  .  .  .  .  .  Kind: const
 85  .  .  .  .  .  .  .  .  Name: "StatusPass"
 86  .  .  .  .  .  .  .  .  Decl: *(obj @ 78)
 87  .  .  .  .  .  .  .  .  Data: 2
 88  .  .  .  .  .  .  .  }
 89  .  .  .  .  .  .  }
 90  .  .  .  .  .  }
 91  .  .  .  .  .  Values: []ast.Expr (len = 1) {
 92  .  .  .  .  .  .  0: *ast.BasicLit {
 93  .  .  .  .  .  .  .  ValuePos: code.go:7:21
 94  .  .  .  .  .  .  .  Kind: INT
 95  .  .  .  .  .  .  .  Value: "2"
 96  .  .  .  .  .  .  }
 97  .  .  .  .  .  }
 98  .  .  .  .  .  Comment: *ast.CommentGroup {
 99  .  .  .  .  .  .  List: []*ast.Comment (len = 1) {
100  .  .  .  .  .  .  .  0: *ast.Comment {
101  .  .  .  .  .  .  .  .  Slash: code.go:7:23
102  .  .  .  .  .  .  .  .  Text: "// 通过"
103  .  .  .  .  .  .  .  }
104  .  .  .  .  .  .  }
105  .  .  .  .  .  }
106  .  .  .  .  }
107  .  .  .  .  3: *ast.ValueSpec {
108  .  .  .  .  .  Names: []*ast.Ident (len = 1) {
109  .  .  .  .  .  .  0: *ast.Ident {
110  .  .  .  .  .  .  .  NamePos: code.go:8:2
111  .  .  .  .  .  .  .  Name: "StatusReject"
112  .  .  .  .  .  .  .  Obj: *ast.Object {
113  .  .  .  .  .  .  .  .  Kind: const
114  .  .  .  .  .  .  .  .  Name: "StatusReject"
115  .  .  .  .  .  .  .  .  Decl: *(obj @ 107)
116  .  .  .  .  .  .  .  .  Data: 3
117  .  .  .  .  .  .  .  }
118  .  .  .  .  .  .  }
119  .  .  .  .  .  }
120  .  .  .  .  .  Values: []ast.Expr (len = 1) {
121  .  .  .  .  .  .  0: *ast.BasicLit {
122  .  .  .  .  .  .  .  ValuePos: code.go:8:21
123  .  .  .  .  .  .  .  Kind: INT
124  .  .  .  .  .  .  .  Value: "3"
125  .  .  .  .  .  .  }
126  .  .  .  .  .  }
127  .  .  .  .  .  Comment: *ast.CommentGroup {
128  .  .  .  .  .  .  List: []*ast.Comment (len = 1) {
129  .  .  .  .  .  .  .  0: *ast.Comment {
130  .  .  .  .  .  .  .  .  Slash: code.go:8:23
131  .  .  .  .  .  .  .  .  Text: "// 拒绝"
132  .  .  .  .  .  .  .  }
133  .  .  .  .  .  .  }
134  .  .  .  .  .  }
135  .  .  .  .  }
136  .  .  .  .  4: *ast.ValueSpec {
137  .  .  .  .  .  Names: []*ast.Ident (len = 1) {
138  .  .  .  .  .  .  0: *ast.Ident {
139  .  .  .  .  .  .  .  NamePos: code.go:9:2
140  .  .  .  .  .  .  .  Name: "StatusEnd"
141  .  .  .  .  .  .  .  Obj: *ast.Object {
142  .  .  .  .  .  .  .  .  Kind: const
143  .  .  .  .  .  .  .  .  Name: "StatusEnd"
144  .  .  .  .  .  .  .  .  Decl: *(obj @ 136)
145  .  .  .  .  .  .  .  .  Data: 4
146  .  .  .  .  .  .  .  }
147  .  .  .  .  .  .  }
148  .  .  .  .  .  }
149  .  .  .  .  .  Values: []ast.Expr (len = 1) {
150  .  .  .  .  .  .  0: *ast.BasicLit {
151  .  .  .  .  .  .  .  ValuePos: code.go:9:21
152  .  .  .  .  .  .  .  Kind: INT
153  .  .  .  .  .  .  .  Value: "4"
154  .  .  .  .  .  .  }
155  .  .  .  .  .  }
156  .  .  .  .  .  Comment: *ast.CommentGroup {
157  .  .  .  .  .  .  List: []*ast.Comment (len = 1) {
158  .  .  .  .  .  .  .  0: *ast.Comment {
159  .  .  .  .  .  .  .  .  Slash: code.go:9:23
160  .  .  .  .  .  .  .  .  Text: "// Unknown"
161  .  .  .  .  .  .  .  }
162  .  .  .  .  .  .  }
163  .  .  .  .  .  }
164  .  .  .  .  }
165  .  .  .  }
166  .  .  .  Rparen: code.go:10:1
167  .  .  }
168  .  }
169  .  Scope: *ast.Scope {
170  .  .  Objects: map[string]*ast.Object (len = 5) {
171  .  .  .  "StatusBeg": *(obj @ 25)
172  .  .  .  "StatusProcessing": *(obj @ 54)
173  .  .  .  "StatusPass": *(obj @ 83)
174  .  .  .  "StatusReject": *(obj @ 112)
175  .  .  .  "StatusEnd": *(obj @ 141)
176  .  .  }
177  .  }
178  .  Comments: []*ast.CommentGroup (len = 6) {
179  .  .  0: *(obj @ 8)
180  .  .  1: *(obj @ 40)
181  .  .  2: *(obj @ 69)
182  .  .  3: *(obj @ 98)
183  .  .  4: *(obj @ 127)
184  .  .  5: *(obj @ 156)
185  .  }
186  }
```


### 简单实现通过解析树解析定义的数据

通过抽象解析树，可以非常清晰地知道代码的整体结构。也可以通过解析抽象树来实现常量值与常量注释的映射。
```golang
package main

import (
  "fmt"
  "go/ast"
  "go/parser"
  "go/token"
  "strings"
)

func main() {
  t := token.NewFileSet()
  f, err := parser.ParseFile(t, "code.go", nil, parser.ParseComments)
  if err != nil {
    panic(err)
  }
  
  for _, object := range f.Scope.Objects {
    obj := object.Decl.(*ast.ValueSpec)
    constVal := obj.Values[0].(*ast.BasicLit)
    fmt.Printf("常量名：%s, 常量值：%s, 常量类型：%s, 常量注释：%s\n", obj.Names, constVal.Value, constVal.Kind.String(), strings.Trim(obj.Comment.Text(), " \n\t"))
  }

}
```

### 扩展
可以试试看以下的数据定义，尝试是否可以通过ast解析出来
```golang
package main

//go:generate  gen-const-status
const (
  StatusBeg        = iota // Unknown
  StatusProcessing        // 流程中
  StatusPass              // 通过
  StatusReject            // 拒绝
  StatusEnd               // Unknown
)
```



