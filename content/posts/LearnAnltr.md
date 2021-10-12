---
type: "post"
title: "学用Antlr4"
date: 2021-10-10T21:36:57+08:00
draft: false
featured_image: "/images/Antlr/antlr-bg.png"
description: "关于编译器大作业。"
comment: true
tags: [编译器,笔记]
categories: 计算机科学
---

# 环境

> 采用Mac os + IntelliJ IDEA

1. IDE:Preference->Plugins->下载安装ANTLR v4

2. 从[官网](https://www.antlr.org/download.html)下载`Complete ANTLR 4.9.2 Java binaries jar`

   IDE: File->Project Structure->Libraries->`+` 选定刚刚下载的jar包



# 正则表达式

## What

正则表达式是用来**描述**一定数量字符文本的**模式串**，用于**匹配**字符串中符合描述的串。

## 规则

- 正则表达式是大小写敏感的

- 特殊保留字符`[ ] \ ^ $ . | ? * + ( )`

  > 想在正则表达式中将这些字符用作文本字符，需要用反斜杠\对其进行换码 。例如想匹配“1+1=2”，正确的表达式1\\+1=2.
  >
  > 而在字符集中只剩下部分字符有特殊含义

- Invisible char

  可以使用特殊字符序列来代表某些不可显示字符：

  `\t`代表Tab(0x09)

  `\r`代表回车符(0x0D)

  `\n`代表换行符(0x0A)

  >  要注意的是Windows中文本文件使用“\r\n”来结束一行而Unix使用“\n”。

- 两种正则导向引擎

  - **文本导向**：引擎在移动到输入的下一个字符之前尝试正则表达式的所有路径，它将返回**最长的匹配项**。例如，`(Set|SetValue)`输入“SetValue”将匹配“SetValue”。
  - **正则导向**：如果引擎在某个位置出现故障，它会回溯以尝试替代路径。路径按从左到右的顺序尝试。因此，即使稍后在另一条路径中有更长的匹配项，它也会返回**最左边的匹配项**。例如，`(Set|SetValue)`输入“SetValue”将匹配“Set”。

### 字符集

- 字符集：是由一对方括号“[]”括起来的字符集合。

  使用字符集时正则表达式引擎仅仅匹配**字符集中多个字符中的一个**。

  >  使用<<gr[ae]y>>匹配gray或grey(不会匹配graay或graey)
  >
  > \[A-Za-z_][A-Za-z_0-9]* 匹配Identifier
  >
  > 0\[xX][A-Fa-f0-9]* 匹配十六进制数字

  字符集中的字符无顺序，无优先级

- 取反字符集：将会对字符集取反。结果是将匹配任何不在字符集中的字符

  **取反字符集必须要匹配一个字符**。包括空格以及回车换行符

  > q\[^u]意味着：匹配一个q，后面跟着一个不是u的字符。所以它不会匹配“Iraq”中的q，而会匹配“Iraq is a country”中的q和一个空格符。事实上，空格符是匹配中的一部分，因为它是一个“不是u的字符”。

- 字符集中的元字符

  字符集中只有4个字符具有特殊含义`] \ ^ -`。“\”代表转义；“^”代表取反；“-”代表**范围定义。**

  其他常见的元字符在字符集定义内部都是正常字符，不需要转义。例如，要搜索星号\*或加号+，你可以用[+*]

- 字符集的重复

  如果用`? * +`将会重复**整个字符集**。而不仅是它上次匹配成功的那个字符。[0-9]+会匹配837以及222

### 使用? \* + 进行重复

- 含义：

  ?：告诉引擎匹配前导字符0次或一次。事实上是表示前导字符是可选的。

  +：告诉引擎匹配**前导字符**1次或多次

  *：告诉引擎匹配**前导字符**0次或多次

- 限制性重复

  允许定义对一个字符重复多少次。词法是：`{min,max}`。

  min和max都是非负整数。如果逗号有而max被忽略了，则max没有限制。如果逗号和max都被忽略了，则重复min次。

  >  因此{0,}和*一样，{1，}和+ 的作用一样。

  用`\b\[1-9][0-9]{3}\b`匹配1000~9999之间的数字(“\b”表示**单词边界**)。`\b\[1-9][0-9]{2,4}\b`匹配在100~99999之间的数字。

- 注意**贪婪性**

  假设用一个正则表达式匹配一个HTML标签`<html>`。

  如果采用正则表达式`<.+> `，则对于测试字符串`This is a \<html>first\</html> test`，会匹配“\<html>first\</html>”。很显然这不是我们想要的结果。原因在于“+”是**greedy**的，即“+”会导致正则表达式引擎试图尽可能的**重复满足前一个字符**。只有当这种重复会引起整个正则表达式匹配失败的情况下，引擎才会进行回溯。也就是说，它会放弃最后一次的“重复”，然后处理正则表达式余下的部分。

  **和`+`类似，`? *`重复也是贪婪的。**

  > 注：
  >
  > 第一个符号是“<”，第二个是通配符“.”，匹配了字符“h”，然后“+”**一直匹配任意字符(贪婪)**，直到换行符匹配失败(“.”不匹配换行符)。于是引擎开始对**下一个正则表达式符号**进行匹配，也即试图匹配“>”。到目前为止，“<.+”已经匹配了“\<html>first\</html> test”。引擎会试图将“>”与换行符进行匹配，结果失败了。于是引擎会进行**回溯**。则最终结果是“<.+”匹配“\<html>first</html”，“>”与“>”匹配。记住，正则导向的引擎是“**急切的**”，所以它会急着报告它找到的第一个匹配（而不是继续回溯），即使可能会有更好的匹配，例如“\<html>”。所以我们可以看到，由于“+”的贪婪性，使得正则表达式引擎返回了一个最长匹配。
  
- 用**懒惰性**取代**贪婪性**

  一个用于修正以上问题的方案是用“+”的惰性代替贪婪性。你可以在“+”后面紧跟一个问号“?”来达到这一点。`* ? {}`表示的重复也可以用这个方案。在上面的例子中使用`<.+?>`，引擎会尽可能少的去重复前一个字符，尽力匹配成功下一个字符。

  > 注:
  >
  > 再一次，正则表达式记号“<”会匹配字符串的第一个“<”。下一个正则记号是“.”。这次是一个**懒惰的**“+”来重复上一个字符。这告诉正则引擎，**尽可能少的重复上一个字符**。因此引擎匹配“.”和字符“h”，然后用“>”匹配“t”，结果失败了。引擎会进行回溯，和上一个例子不同，因为是惰性重复，所以引擎是**扩展惰性重复**而不是最终失败后的回溯减少，于是“<.+”现在被扩展为“<ht”，直至"<html"。引擎继续匹配下一个记号“>”与">"。这次得到了一个成功匹配。引擎于是报告“\<html>”是一个成功的匹配。

### `.`匹配几乎任意字符

`.`匹配任意单个字符(除换行符外)。等价于字符集`\[^\n\r]`(Window)

### 锚定

- 字符串开始和结束的锚定：它**不匹配**任何字符。相反，他们匹配的是字符之前或之后的**位置**。

- `^`匹配一行字符串第一个字符前的位置;`$`匹配字符串中最后一个字符的后面的位置

  > 如果想校验用户的输入为整数，用`^\d+$`。

### 选择符

- `|`表示选择。你可以用选择符匹配多个可能的正则表达式中的一个。

  > 搜索文字“cat”或“dog”，可以用`cat|dog`。可扩展列表`cat|dog|mouse|fish`
  >
  > 圆括号告诉正则引擎把括号内部(cat|dog)当成一个**正则表达式单位**来处理。
  >

### 组

把正则表达式的一部分放在圆括号内，你可以将它们形成组。然后你可以对整个组使用一些正则操作，例如**重复操作符**。

>  要注意的是，只有圆括号“()”才能用于形成组。“[]”用于定义字符集。“{}”用于定义重复操作。



# Antlr4

- Lexer(词法分析器)和Parser(语法分析器)：

  Lexer通过输入的字节流(01二进制流)按照词法规则来把字节流划分成一块一块的`token`。

  Parser通过`词素+符号`按照文法规则来识别语言结构，生成对应的`RuleNode`结点。

  > 区分词法规则和文法规则->词法规则以大写字母开头，文法规则以小写字母开头

- Antlr 可识别的语法模式(文法规则)

  - 序列模式:`ifStmt: If '(' expression ')' trueStmt=statement`

  - 选择模式:`loopStmt: forStmt | whileStmt;`

    ！ `#`为分支打标签

  - 词法符号依赖模式: 一个词法符号需要与另一个词法符号一起出现配对。如：`'(' xxx ')'`

  - 嵌套模式

    - 直接递归与间接递归

    - **优先级问题：位置靠前的分支优先级越高**

      > 运算符优先级依靠Expression语句的定义顺序规定

    - Antlr4能够处理直接左递归，但不能处理**间接左递归**

    - `<assoc=right>`指定右结合方式：**实际上就是优先递归生成右边的子树**

      ```java
      2 ^ 3 ^ 4
      // ParseTree
        	^									^
       ^ 			4	 ->     2				^
      2   3										3		 4
      ```

- 可以在文法规则当中为分支打标签 `#`

  如果备选分支上面没有标签，ANTLR就只为每条规则生成一个ParserRuleContext，备选分支的访问需要通过**父规则的成员函数和成员变量。**
  
  ```java
  subProgram
      :   functionDecl
      |   classDecl
      |   variableDecl ';'
      ;
  ```
  
  ```java
  	public static class SubProgramContext extends ParserRuleContext {
  		public FunctionDeclContext functionDecl();
  		public ClassDeclContext classDecl();
  		public VariableDeclContext variableDecl();
  		public SubProgramContext(ParserRuleContext parent, int invokingState);
  		@Override public <T> T accept(ParseTreeVisitor<? extends T> visitor);
  	}
  ```
  
  打标签=>必须为每个备选分支都打上标签。这样ANTLR对该规则每个分支生成一个ParserRuleContext(继承自父规则的Context)，而父规则并**无**通向分支的成员函数和成员变量，由此形成多态。
  
  ```java
  jumpStmt
      :   RETURN expression? ';'  #returnStmt
      |   BREAK ';'               #breakStmt
      |   CONTINUE ';'            #continueStmt
      ;
  ```
  
  ```java
  	public static class JumpStmtContext extends ParserRuleContext {
  		public JumpStmtContext(ParserRuleContext parent, int invokingState);
  		@Override public int getRuleIndex() { return RULE_jumpStmt; }
  		public JumpStmtContext() { }
  		public void copyFrom(JumpStmtContext ctx);
  	}
  	public static class BreakStmtContext extends JumpStmtContext {
  		public TerminalNode BREAK() { return getToken(MxParser.BREAK, 0); }
  		public BreakStmtContext(JumpStmtContext ctx) { copyFrom(ctx); }
  		@Override public <T> T accept(ParseTreeVisitor<? extends T> visitor);
  	}
  ```

## Lexer

- Lexer解决歧义问题的方法是**优先使用位置靠前的词法规则**

  ```java
  Int: 'int';
  ID: [a-zA-Z][a-zA-Z0-9_]*
    ;
  ```

  这样避免了Lexer把`int`分析成为一个ID

  > antlr为隐式生成的字符串常量词法规则放在显示定义的词法规则之前，所以他们总有最高的优先级。

  - ps: 同一优先级的运算符摆在同一个位置的候选分支 如('*' | '/' | '%')

- `fragment` 辅助规则

  表示该规则本身不是一个词法符号，它只会被其他词法规则使用而不会在文法规则里使用。

  ```java
  Float: Digit + '.' + Digit;
  fragment
  Digit: [1-9][0-9]* | 0 ;
  ```
  
- 非贪婪匹配(懒惰)

  `String: '"'.*?'"'`

  识别转义字符

  ```java
  String: '"'(ESC|.)*?'"'
  fragment
  ESC: '\\"' | '\\\\'; //antlr语法本身需要用\对\进行转义
  ```

- 处理注释和空白字符

  - 当Lexer处理到注释和空白字符时，希望将其丢弃而不是处理

  - 定义需要被丢弃的词法符号的方法和正常一样，只需增加` -> skip`指令

    > Lexer可以接受许多->指令，eg.Channel

  - `LINE_COMMENT: '\\' .*? '\r'?'\n';`

    `COMMENT: '/*' .*?  '*/';`

    `WS:[ \t\r\n]+ -> skip;` 匹配一个或多个空白字符并舍弃

- Boundary between Lexer and Parser

  - 并非很清晰，需要明确程序的需求
  - Lexer 处理 raw char string 丢弃不需要Parser知道的东西，并将词**抽象更高一级**。

- `'()'`与`'(' ')'`不等价，若`'()'`注册在前面，优先级会更高，`'()'`就不能被识别为`'('`了

  ```java
  allocFormat:baseType ('()')?；
  functionDecl: functionType? IDENTIFIER '(' parameterList? ')' block;
  无法识别 int foo();//ANTLR会因为优先识别为'()'而识别不到'('，无法识别为functionDecl语法
  ```

- ![一些基本词法](/images/Antlr/基础语法规则.png)

## Vistor

- 继承关系：

  ```java
  MxBaseVisitor->AbstractParseTreeVisitor
  			|                  |
    		v	                 v
  接口 MxVistor -> 接口  ParseTreeVisitor
  ```

  - `ParseTreeVisitor`：定义了`T visit(ParseTree var1);`、`T visitChildren(RuleNode var1);`等

  - `MxVisitor:`继承以上接口并新增`T visitProgram(MxParser.ProgramContext ctx);`等根据.g4生成的parsetree结点

  - `AbstractParseTreeVisitor`:实现`visit(ParseTree tree)`:tree.accept(this);

  - `MxBaseVisitor`:实现并继承以上

  ```java
  ParserRuleContext->RuleContext
                          |
                          v
                    接口 RuleNode->接口 ParseTree
  ```

  - `ParseTree`:定义了接口`<T> T accept(ParseTreeVisitor<? extends T> var1);`

  - `RuleContext`:实现了函数`getText()`转树结点内容为字符串、`getChildCount()`获得子结点的个数、`accept`(实际上无用，被覆写)

  - `ParserRuleContext`: 为`MxParser`文件中各`xxxxContext`的父类，`xxxContext`都各自覆写了对应的`accept`函数为调用对应的`visitXXXContext`，也即调用`MxBaseVisitor.visit(xxxContext)` => `xxxContext.accept(MxBaseVisitor)`=>`MxBaseVisitor.visitxxxContext(xxxContext)`

    而`MxBaseVisitor.visitxxxContext(xxxContext)`实际上被其子类(如ASTBuilder)所覆写。所以调用visit(xxx)等于调用我们自己实现的visitxxxContext(xxxContext);

- 注意**向上转型**

  调用的是实际所指向的对象的成员函数，借此实现多态

## Parser

- `ProgramContext`:

  -  用MxParser.program()可接到 ProgramContext(Derived from parse tree)

  - `public List<SubProgramContext> subProgram()` 返回子节点形成的`ArrayList<Sub..>`

    源于` program: subProgram *`

  - 覆写了`public <T> T accept(ParseTreeVisitor<? extends T> visitor)`函数

    在`AbstractParseTreeVistor`实现的**visit函数**即`ParseTree.accept(this)`实际上调用了该accept函数

    而该accept函数的实现又调用了`ParseTreeVistor.visitProgram`也即实际调用`ASTBuilder`中**覆写的函数**

    > => visit(Type ctx) 调用 ASTBuilder. visitType(ctx) 返回一个自定义的ASTNode

- `SubProgramContext`

  ```java
  subProgram
      :   functionDecl    #functionDeclaration
      |   classDecl       #classDeclaration
      |   variableDecl    #variableDeclaration
      ;
  ```

  - 实际上是作为`父类`发挥作用 提高代码复用率

    自然情况下**不会真正的出现类似的父类对象**，**都是父类引用对应的子类而实现多台**

    `FunctionDeclarationContext`、`ClassDeclarationContext`、`VariableDeclarationContext`均extends `subProgramCtx`

  - 其子类均实现了`accept`函数(等价于ASTBuilder.visit(子类ctx)返回子类节点)

    由于向上转型 传一个父类的引用进入函数当中是可以调用其`accept`函数的 (由此实现多态)

  - Usage：

    ```java
    ArrayList<ASTNode(or other father node)> _List;
    ProgramCtx.subProgram.forEach(ptr->_List.add((ASTNode) visit(ptr)));
    ```

- `FunctionDeclarationContext`、`ClassDeclarationContext`、`VariableDeclarationContext`

  - 类似于`ProgramContext`覆写`accept`需要实现visitxxx(xxx ctx);
  - 其成员函数即为返回对应的`functionDeclCtx` 、`classDeclCtx`、 `variableDeclCtx`

  - **！这样打标签实在是多此一举！** (进行一个modify

    - 打标签的实质是：告诉ANTLR不要为`subProgram`生成实际的`accept`而是把其当作父类，其各种组件打上标签及都要生成对应的`ParseRuleContext && accept && extends subProgram`。也即这个文法实际上由其他文法定义而成，只不过分类在一起。

      ```java
      //After modificatin
      subProgram
          :   functionDecl
          |   classDecl
          |   variableDecl
          ;
      public static class SubProgramContext extends ParserRuleContext {
      		public FunctionDeclContext functionDecl(); //访问子Context
      		public ClassDeclContext classDecl();
      		public VariableDeclContext variableDecl();
      		public SubProgramContext(ParserRuleContext parent, int invokingState);
      		@Override public <T> T accept(ParseTreeVisitor<? extends T> visitor);
      }
      //既有accept 又能向下到子节点 实为文法
      ```

      不打标签的文法规则可以**通过成员函数/成员变量**访问到子Context

      打了标签只能充当多态的二传手了

    - **Necessity** for tag Expressions :

      给每一种分支都覆写一个`accept`函数，而`expression`本身当作父类起到多态作用。

      即abstract class ExprNode: BinaryExprNode/NoaryExprNode extends ExprNode.

# 一些语法：

- java 泛型的实现是：擦拭法 有许多缺点，[详见](https://www.liaoxuefeng.com/wiki/1252599548343744/1265104600263968)

- `<T>`等价于`template <typename T>`

- ```java
  ArrayList<Int> 可向上转型为 List<Int>
  ArrayList<Int> 不可向上转型为 ArrayList<Number>
  但实际上Int 继承与Number 有相同的实现 ，理论上可以转型，实现方法：
  ArrayList<? extends Number>
  则所有ArrayList<T> T继承与Number的都可以转型被接到
  ```

- ```java
  class xxx extends yyy;
  yyy a;
  xxx b = a;
  a instanceof yyy return true;
  ```

**参考资料:**

> 1. https://www.cnblogs.com/dragon/archive/2006/05/09/394923.html
> 2. 《antlr4 权威指南》