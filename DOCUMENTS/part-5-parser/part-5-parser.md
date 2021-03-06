牙医教你 450 行代码自制编程语言 - 5, 递归下降语法解析器.md
----------------------------------------------------

@version    20210105:1  
@author     karminski <work.karminski@outlook.com>  


上一篇  [牙医教你 450 行代码自制编程语言 - 4, 实现 Lexer 下篇](), 我们终于完成了 Lexer, 本篇我们就来讲 Parser 怎么实现.

本教程的所有代码都可以在 [https://github.com/karminski/pineapple](https://github.com/karminski/pineapple) 找到.  


再次推荐这本书, 本教程就是类似这本书的简化版本, 想要仔细学习的话可以考虑看原作:  

- [《自己动手实现Lua：虚拟机、编译器和标准库》](https://union-click.jd.com/jdc?e=jdext-1331174943460048896-0&p=AyIGZRhfEQAUAlEZWBAyEgZUGF4SAhIFUBJaEQQiQwpDBUoyS0IQWhkeHAxfEE8HCllHGAdFBwsCEwZWHlwVAhACXBpfEx1LQglGa2lVWnpcTwhRYXZHBkIzFXRIXT1jGHUOHjdVElsXChMGVRxYJQITBlUfXhYBFAZlK1sQMlNpXBhdFAUaN1QrWxICEwdRHFIXCxYPUitbHQYi0fuPjp29y7fwzfG715%2B3gJLwwbyUN2UrWCVZR1McXkcVABAHVR1eHQcQAlIaWhALGw9SB1olAhMGVx9ZFAUaBzseWxQDEwNdGlkXbBAGVBlaFAAVAVYrWyUBIlk7GggVUhVVAEw1T1lTBxAeWxdsEgRdHFwRBBA3VxpaFwA%3D)


# 按图索骥 

我们再次把已经看到烦的 EBNF 贴在这里, 接下来用这个, 我们就能飞速的构建 Parser 了. 最难的 Lexer 我们已经搞定, 接下来请系好安全带, 我们开始发车了!  

```ebnf
SourceCharacter ::=  #x0009 | #x000A | #x000D | [#x0020-#xFFFF] /* /[\u0009\u000A\u000D\u0020-\uFFFF]/ */
Name            ::= [_A-Za-z][_0-9A-Za-z]*
StringCharacter ::= SourceCharacter - '"'
String          ::= '"' '"' Ignored | '"' StringCharacter '"' Ignored
Variable        ::= "$" Name Ignored
Assignment      ::= Variable Ignored "=" Ignored String Ignored
Print           ::= "print" "(" Ignored Variable Ignored ")" Ignored
Statement       ::= Print | Assignment
SourceCode      ::= Statement+ 
```

## Name 

首先是 ```Name``` 语句, 很简单, 我们的 Lexer 能自动解析出 ```Name``` Token, 因此:

```go
// Name ::= [_A-Za-z][_0-9A-Za-z]*
func parseName(lexer *Lexer) (string, error) {
    _, name := lexer.NextTokenIs(TOKEN_NAME)
    return name, nil
}
```

5 行搞定! 甚至还包含了一行注释! 我们只需要断言 ```parseName()``` 的时候, 接下来的 Token 必须是 ```Name``` 就可以了! 如果不是怎么办? 不是那就是输入的代码语法错误了! 与我们无关~  

然后返回的真正的 name 是个 string 类型, 直接返回就好了.  

## String  

```String``` 复杂一些, 它要么是空字符串, 要么是双引号包裹的字符串 ```StringCharacter```. 而 ```StringCharacter``` 是由不包含双引号的 ```SourceCharacter``` 构成.  

```go
// String ::= '"' '"' Ignored | '"' StringCharacter '"' Ignored
func parseString(lexer *Lexer) (string, error) {
    str := "" 
    switch lexer.LookAhead() {
    case TOKEN_DUOQUOTE:
        lexer.NextTokenIs(TOKEN_DUOQUOTE)
        lexer.LookAheadAndSkip(TOKEN_IGNORED)
        return str, nil 
    case TOKEN_QUOTE:
        lexer.NextTokenIs(TOKEN_QUOTE)
        str = lexer.scanBeforeToken(tokenNameMap[TOKEN_QUOTE])
        lexer.NextTokenIs(TOKEN_QUOTE)
        lexer.LookAheadAndSkip(TOKEN_IGNORED)
        return str, nil
    default:
        return "", errors.New("parseString(): not a string.")
    }
} 
```

我们来按照这两种情况书写, 既然有两种情况, 我们只好先 ```LookAhead()``` 一下, 看看接下来的是什么 Token.  

首先, 如果是 ```TOKEN_DUOQUOTE``` 空字符串的情况, 那么就断言是空字符串 ```lexer.NextTokenIs(TOKEN_DUOQUOTE)```. 然后跳过后面的 ```Ignored``` Token 即: ```lexer.LookAheadAndSkip(TOKEN_IGNORED)```.  

然后如果只是个单引号 ```TOKEN_QUOTE```, 那接下来就是字符串本体了, 我们用 ```scanBeforeToken()``` 扫描直到遇到下一个单引号. 然后断言双引号, 最后跳过 ```Ignored```. 

如果都不是, 那我们直接返回错误, 这不是 string. 不用怀疑, 肯定是出语法错误了, 让写代码的人自己检查去吧. 哈哈哈, 是不是有种翻身了的感觉?  

## Variable  

同理, ```Variable```  是美元符号 ```TOKEN_VAR_PREFIX``` 和 ```Name``` 构成:  

```go
// Variable ::= "$" Name Ignored
func parseVariable(lexer *Lexer) (*Variable, error) {
    var variable Variable
    var err      error 
    
    variable.LineNum = lexer.GetLineNum()
    lexer.NextTokenIs(TOKEN_VAR_PREFIX)
    if variable.Name, err = parseName(lexer); err != nil {
        return nil, err
    }
    lexer.LookAheadAndSkip(TOKEN_IGNORED)
    return &variable, nil
}
```

这里有些不同, 我们声明了 ```Variable``` 类型, 它是这样的:  

```go
type Variable struct {
    LineNum int 
    Name    string 
}
```

可见很忠实的还原了 EBNF 定义的结构, ```Variable``` 包含了 ```Name```.

回到代码, 我们这里还用 ```variable.LineNum = lexer.GetLineNum()``` 获取了行号, 这个主要是提供给源代码编写者报错用的. 可以不必太在意. 后续优化的话可以给到精确的行号和列号.  

这里除了开头是美元符号, 我们用了 ```variable.Name, err = parseName(lexer)```. 这样就把解析 ```Name``` 的工作交给了刚刚完成的 ```parseName()``` 方法. 这个过程就是递归下降的过程. 把很大的目标拆分成递归形式的小目标, 可以让我们更好的完成工作.  


## SourceCode & Statements 

由于 ```parseAssignment()```, ```parsePrint()``` 很相似, 我们就不赘述了. 我们只把他们的数据结构定义写下来:  

```go
type Assignment struct {
    LineNum   int 
    Variable *Variable
    String    string 
}

type Print struct {
    LineNum   int 
    Variable *Variable
}
```

接下来看 ```parseStatement()```:  

```go
// Statement ::= Print | Assignment
func parseStatement(lexer *Lexer) (Statement, error) {
    switch lexer.LookAhead() {
    case TOKEN_PRINT:
        return parsePrint(lexer)
    case TOKEN_VAR_PREFIX:
        return parseAssignment(lexer)
    default:
        return nil, errors.New("parseStatement(): unknown Statement.")
    }
}
```

```Statement``` 可以是 ```Print``` 或 ```Assignment```, 需要先 ```LookAhead()``` 一下, 如果是 ```TOKEN_PRINT``` 那么就解析 ```Print```, 如果是 ```TOKEN_VAR_PREFIX``` 就解析 ```Assignment```.  

然后我们看 ```SourceCode``` 的定义 ```SourceCode ::= Statement+ ``` 它可以是多个 ```Statement```. 所以我们这样定义:  

```go
type SourceCode struct {
    LineNum      int 
    Statements []Statement
}
```

对, 我们定义了一个 ```Statements []Statement``` 一个盛放 Statement 的 Slice (不会 Go 的同学理解成数组也行). 然后 ```Statement``` 是这样的:  

```go
type Statement interface{}

var _ Statement = (*Print)(nil)
var _ Statement = (*Assignment)(nil)
```

由于 ```Statement``` 可以是不同数据类型, 因此我们定义成了 ```interface{}```. 然后 Go 的初学者可能看不懂接下来的是什么.  

简单来讲, ```var _ Statement = (*Print)(nil)``` 可以简单理解为 ```*Print 属于 Statement```. 这样 ```Statement``` 的具体实现就被限定到了 ```*Print```, ```*Assignment``` 这两种类型. 对后续的编写安全是一种很好的保障, 否则 interface{} 满天飞, 很容易出错.  

定义完毕之后, 我们来看如何搞定 ```SourceCode ::= Statement+``` 中的 ```Statement+```:  

```go
func parseStatements(lexer *Lexer) ([]Statement, error) {
    var statements []Statement 
    
    for !isSourceCodeEnd(lexer.LookAhead()) {
        var statement Statement
        var err       error
        if statement, err = parseStatement(lexer); err != nil {
            return nil, err 
        }
        statements = append(statements, statement)
    }
    return statements, nil
}

func isSourceCodeEnd(token int) bool {
    if token == TOKEN_EOF {
        return true
    }
    return false
}
```

我们设置了个有条件的死循环, ```for !isSourceCodeEnd(lexer.LookAhead()) {``` 不断向前判断 Token, 直到遇到源码末尾. 判断末尾的 ```isSourceCodeEnd()``` 也很简单, 直接判断是不是  ```TOKEN_EOF``` 就可以了.  

然后不断 ```statement, err = parseStatement(lexer)``` 然后把结果放到 Slice 里面 ```statements = append(statements, statement)```.  这样, 就完成了!  

# 长舒一口气  

最后, 我们把调用方法封装一下:  

```go
func parse(code string) (*SourceCode, error) {
    var sourceCode *SourceCode
    var err         error 

    lexer := NewLexer(code)
    if sourceCode, err = parseSourceCode(lexer); err != nil {
        return nil, err 
    }
    lexer.NextTokenIs(TOKEN_EOF)
    return sourceCode, nil
}
```

到此, 我们就完成了编译器的前端 (对, 相对后端的都叫前端. 编译器的前端一般就指 Lexer 和 Parser).  

Parser 的源码详见:  [https://github.com/karminski/pineapple/blob/main/src/parser.go](https://github.com/karminski/pineapple/blob/main/src/parser.go).  

相关数据结构的定义详见: [https://github.com/karminski/pineapple/blob/main/src/definition.go](https://github.com/karminski/pineapple/blob/main/src/definition.go).  

下一节我们就要开始写编译器的后端了! 就可以让代码跑起来了! 我们下一篇再见~  
