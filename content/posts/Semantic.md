---
type: "post"
title: "Semantic Notes"
date: 2021-10-28T17:57:57+08:00
draft: false
featured_image: "/images/Semantic/bg.jpeg"
description: "关于编译器大作业。"
comment: true
tags: [编译器,笔记,语法树]
categories: 计算机科学
---

# AST Node 信息结构

> 我的ASTNode结构

- Type Node：存储对象的类型 （Def by Induction）**只作为其他节点的成员变量出现而不单独作为结点出现**

  - (ClassType)基本类型+面向对象：字符串
  - (ArrayType)数组：存储基本类型+数组的维数信息

  - 函数类型：以上类型Node+void

- Constant:：仅仅存值，类型按照ClassType存储
- NewExprNode：是”New“表达式精简后的结点
  - 包括类型、维数和initList
  - 非递归结点，一次性获取了alloc的所有信息都一个结点
  
- Identifier：仅存储了对象的名字，具体信息应当去Scope里寻找

- ArrayAccess：**递归**产生结点，生成一棵树，维数需要递归获得

  对TypeNode的作用：维数减一，之后若为零，则转化为ClassTypeNode

- Block：存储ArrayList\<StmtNode>；产生新的Scope

- ReturnNode：返回类型从保存的Expression Node获得

- VariableDeclaration：一个VarDefStmtNode->ArrayList\<VarDefNode>

  VarDefNode: Type+String+Init

- FunctionDeclaration：Type+String+ParameterList+codeBlock

- ClassDeclaration：String+Func+Var

- RootNode：类定义、函数定义、全局变量定义；

  全部由ASTNode父类引用 而不是分成三个链表，这是为了保证顺序Check Semantic实现作用域是从声明处开始到作用域结束

  ```java
  public Class RootNode extends ASTNode{
    	public ArrayList<ASTNode> elements;//引用三种类型的声明语句
  }
  ```

# Semantic Question

> 需要检查的信息 (从README中提取)

- 作用域

  - 类以及函数的前向引用

    全局变量和局部变量**不支持**前向引用，作用域为**声明开始的位置**直到最近的一个块的结束位置

  - 类内部的成员变量、函数作用域都是整个类

  - 内层作用域对外层作用域的**覆盖**

  - 未定义/多重定义变量

  - `Return`必须在函数作用域； `break/continue`必须在勋在作用域内部

  - `{}`块、`函数入口`、`类的入口`引入一个新的作用域

    > for/if/while 即使没有{} 也会引入新的作用域

    ```java
    {
      int a;
      for(;;) int a;      //Success
      for(int a;;) int a; //Failed
      while(true) int a;  //Success
      if(flag) int a;     //Success
    }
    ```

- 命名空间：

  - 在同一个作用域里，变量，函数，和类都**分别**不能同名(即变量不能和变量同名，其余同理)，变量和函数可以**重名**，但是类不可以和变量、函数重名。

    > 注意 Expression DOT IDENTIFIER 的处理 可能是类成员函数，也可能是取成员变量

  - 内部作用域的命名空间可以遮掩外部作用域的命名空间

- Type匹配问题

  - 各种expression语句的要求

  - return与function type

  - `null`空值常量：引用类型 当作右值可以被任意Type接住；可以参与 `== null`与`!= null`

    ```java
    A a = null;   //Success
    int a = null; //Failed: null不能赋值给基本类型
    A foo(){
      return null; //Success
    }
    ```

  - 数组type匹配：类型和维数要相等

- Expression 中对象的左右值问题

  - 所有对象都是引用类型
  - 运算结果是右值
  - `++a`为左值，`a++`为右值

- 函数

  - 内建函数

  - 必须要有一个`int main()`在其中

    > 返回值是int 参数为空

  - 返回值不是void，则必须有return语句 (main除外)，必须要进行type-check

    是void，return必须单独使用，不能出现`return expr;`

- 数组

  - 内建函数 .Size()
  - 可以不初始化，声明为null
  - 交错数组创建的**维数**验证

- 类

  - 基本类型有**内建函数**；基本类型不能被赋值为null

  - 构造函数的定义和C++相同，无返回值无参数(有参数的未定义)，**可以没有构造函数**

  - `this`指针返回某个类的引用对象，关键字仅在**类作用域内**可以使用。

    不在类作用域内的 this 应当视为语法错误， **this 指针作为左值视为语法错误**。(设置为右值即可)

- bool 类型仅可以做 == 和 != 比较，出现 <=, >=, <, > 的比较是语法错误的。

  对象可以和 null 比较(仅限 !=, == )，但是不能运算。

- IF/FOR/WHILE语句当中的Condition**要求**是不同的
- 单目运算符除`++ -- `以外对左右值没有要求 对操作对象的类型有要求



# Design

### Scope

- “并查集”结构。每个结点只记录其父亲。

  > 每个Scope都只需要在本地找，找不到向上找；

  ```java
  if(Variable_Table.containsKey(identifier)) return Variable_Table.get(identifier);
  else if(this.parent != null) return parent.fetch_Variable_Type(identifier);
  else return null;
  ```

- HashMap<String,value>存什么value：

  - Variables：存typenode信息即可. Semantic Check仅需要知道identifier对应数据的类型
  - Functions：将整个FuncDefNode存入 以便Check函数调用的正确性
  - Class：将类的Scope存进去(其中包括成员变量、成员函数) 

- 全局记录一个当前作用域`cScope`

  进入一个新作用域时 `cScope = new Scope(cScope);`

  离开当前作用域时 `cScope = cScope.parent;`

  从而模拟一个Stack的效果 ，以实现命名空间的**覆盖**

- 全局存储变量、类和函数；类作用域存储变量和函数；局部作用域只存储局部变量

  > 调用变量：声明变量看当前作用域hashmap中有无value；调用变量要从当前向上回溯获取类型
  >
  > 调用类相关：从GlobalScope获取类的Scope并从其中读取信息
  >
  > 调用函数：从GlobalScope中Fetch_Funciton(String identifier)



### 前向引用

- 实现前向引用所需要的信息：类的成员变量**标识符**、类的成员**函数签名**、全局函数签名

- 在Semantic Check之前先**扫一遍**，将全局函数和类(包括成员函数的**签名**也即FuncDefNode、成员变量的**类型**和Identifier)读入到Scope里面。

  此时对数据的类型是否合法**不做检查**（信息不够），但进行一些基础的语法检查，包括是否重命名、有没有main函数等等

- 这之后在Semantic check时已有的信息：所有合法的类型、函数的签名、类成员变量的标识符，即实现了前向引用。

- 非前向引用：按照正常**线性顺序处理**即非前向引用，不用特殊处理，但要保证根结点当存储的子节点**是按照线性顺序的**



### 内建函数

为了提高代码复用率，我是在前向引用信息收集前，向GlobalScope当中塞对应的信息。从而达到后续处理相同。

```java
//built-in Function print 构造一个FuncDefNode 塞入全局作用域
ArrayList<VarDefNode> tmp_List = new ArrayList<>();
tmp_List.add(new VarDefNode(new ClassTypeNode("string",new Position(-1,-1)),"str",null,new Position(-1,-1)));
FuncDefNode tmp = new FuncDefNode(new VoidTypeNode(new Position(-1,-1)),"print",tmp_List,null,_pos);
initScope.define_Function("print",tmp);
```



### Type

- 依赖于`TypeNode`来对数据的类型进行区分

  - int/bool/string以及面向对象都是`ClassTypeNode` 仅保存字符串

    对于内置类型的处理，预处理阶段塞入globalScope的class当中

  - `arrayType:` "baseTypeNode" + 维数
  - 函数返回值:  baseType+ArrayType+VOID

- 对于表达式的类型检查

  - TypeNode：每个表达式结点带一个TypeNode，表示运算后的类型，以便其父节点调用、检查类型匹配

    类型的初始化：回溯时候初始化，即当前表达式结点 运算之后的类型由其**子节点**的类型以及当前结点的**操作**共同决定(要处理当前结点，必须先处理子节点，回溯之后子节点的类型已经确定）

    叶子结点的类型：常量则直接确定，变量则从Scope中获取；

  - `isAssignable`: 每个表达式结点都持有，以便父节点检查语法。

    根据当前操作来确定当前结点运算之后是左值还是右值。

    > 如加减乘除则将自身设置为右值，赋值操作将自身设置为左值

- 对于数组类型的处理

  - `[]`每一个ArrayAccess结点都将自身的类型设置为`维数=原数组维数-1`

    如果维数为0，则把数组类型转换为一般数据类型

### Feature

- Semantic Check本质上是在一棵树上进行遍历，提取想要的信息并根据规则判断合法性。

  其信息的传输分为**两种**，一种是深度优先遍历时从上层结点带给下层结点的，另外一种是回溯时从下层结点带回上层结点的。两种信息传输都可以充分使用。

- 对于类型和左右值的处理就是采取**回溯时**得到的信息的方法

- 对于`对象取用`结点可以采取从上到下传入信息

  ```java
  // in ASTNode
  public class ObjectMemberExprNode extends ExprNode{
      public ExprNode base;
      public String member;
      public boolean forFunc;				//如果父节点是FunctionCallNode，则打标记将此值置为true，默认为false
      public FuncDefNode funcInfo;	//在处理当前结点时，发现被打了标记，则去class的scope里面找函数，并且把函数的签名赋值带回；此外都找变量
    	...
  }
  // in SemanticChecker
  public void visit(FuncCallExprNode node) {
      FuncDefNode checkBase;
      if(node.Func instanceof ObjectMemberExprNode){		
          ((ObjectMemberExprNode) (node.Func)).forFunc = true;	//打标记
          node.Func.accept(this);	//向下处理对象取用结点
          checkBase = ((ObjectMemberExprNode) (node.Func)).funcInfo; //从下面结点中获取函数签名
      }
    	...
  }
  ```

- 对于`Return 语句`的处理

  因为Lambda表达式可以嵌套，且为了代码复用，采取了全局栈的方式

  ```java
  public Stack<ASTNode> FuncStation;
  //	处理函数或者lambda结点时将对应的包括所有信息的ASTNode压入栈中
  //  处理return语句时很方便
  public void visit(ReturnStmtNode node) {
      if(FuncStation.empty()) throw new SemanticError;
      if(FuncStation.peek() instanceof FuncDefNode) {
        	....
      }else{
          LambdaExprNode nowFunc = (LambdaExprNode) FuncStation.peek();
        	....
      }
  }
  ```

  

# Debug Log

> 细枝末节 : )

- `For`语句产生了两个作用域

  ```java
  for(int i = 1;;){int i;} //判断不出重定义
  ```

  ```java
  public void visit(BlockStmtNode node){
    cScope = new Scope(cScope);
    node.stmtList.forEach(tmp->tmp.accept(this));
  }
  public void visit(ForStmtNode node){
    ...
    if(node.loopBody != null) node.loopBody.accept(this);//BlockStmt的话会产生新作用域
    ... //modified
    if(node.loopBody instanceof BlockStmtNode) 
      ((BlockStmtNode) node.loopBody).stmtList.forEach(tmp->tmp.accept(this));
    else node.loopBody.accept(this);
    ...
  }
  ```

- ArrayList使用前未判`null`，直接forEach遍历，导致空指针调用

- 单目运算符对左右值的要求考虑出错:

  `+5` `-5` `!false`  均允许

- 赋值运算符右边可以为`null`，基本类型不能赋值为`null`

- class 内部调用函数时，忽略了对类内部Scope中的函数进行检查，只去判断了全局作用域，导致找不到函数

- 对`{{{;};{}}};`的处理

  ```java
  // g4设计 保证能识别; 不出现syntax问题
  statement
    :	block    #codeBlock
    | .......
    | ;        #blankStmt
    ;
  // ASTBuilder 设计添加特判 遇到blankStmt不要向stmtList里面添加元素
  public ASTNode visitBlock(MxParser.BlockContext ctx) {
      ArrayList<StmtNode> _List = null;
      if(ctx.statement() != null){
          for(MxParser.StatementContext ele : ctx.statement()){
              if(!(ele instanceof MxParser.BlankStmtContext)){ // 处理
                  if(_List == null) _List = new ArrayList<>();
                  _List.add((StmtNode) visit(ele));
              }
          }
      }
      return new BlockStmtNode(_List,new Position(ctx));
  }
  ```

- 函数参数可以传`null`  ; return可以`null` ;  `== !=`可以与`null`