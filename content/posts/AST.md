---
type: "post"
title: "From CST to AST"
date: 2021-10-12T21:50:57+08:00
draft: false
featured_image: "/images/Antlr/AST.jpg"
description: "关于编译器大作业。"
comment: true
tags: [编译器,笔记,语法树]
categories: 计算机科学
---

## AST vs CST

- Abstract Vs. Concrete Syntax Tree :

   Antlr4提供的是CST，为什么要将CST转换为AST再进行Semantic检查？

  [Reference Document]( https://eli.thegreenplace.net/2009/02/16/abstract-vs-concrete-syntax-trees/)

- CST: 将语法表示成树的形式 

  > A parse tree pictorially shows how the start symbol of a grammar derives a string in the language.

  CST是语言一对一到树上的映射。仅有叶子节点带有实际信息，中间节点均为Parser 解析出信息的过程。

- AST：AST are simplified syntactic representations of the source code, and they're most often expressed by the data structures of the language used for implementation. 

  > Abstract syntax trees, or simply *syntax trees*, differ from parse trees because superficial distinctions of form, unimportant for translation, do not appear in syntax trees.

  AST仅保留对**分析有用**的信息。内部节点也存有信息(如BinaryExprNode中间节点存有操作符)，而像`;`的词法符号都被抛弃，对分析没有任何帮助

- The front-end of a compiler can be seen as a process that transforms the input from **its most concrete form** (statements in the source language) **to its most abstract form** - stored in internal data structures and ready for **analysis & translation.**
  - CST更具体、接近于语法层面 而AST更抽象、方便于后续处理。

- CST是ANTLR根据`.g4`文件语法规则分析文本得到的满足形式的树，而我们需要的是内容上保留足够信息、形式上不拘泥便于处理的树。

## From CST to AST

- 将**需要的各种信息抽象成各类结点**：其中可以相互嵌套递归

  注意多态

  ```
  - ASTNode
  	- RootNode		// 根结点 由第一个规则产生
  	- StmtNode		// node as a statement
  		- varDefStmtNode
  		- returnStmtNode
  		- blockStmtNode
  		- exprStmtNode
  		- ifStmtNode
  		- forStmtNode
  		- whileStmtNode
  		...
  	- ExprNode		// node as an expression
  		- assignExprNode
  		- binaryExprNode
  		- noAryExprNode
  		- constExprNode
  		- cmpExprNode
  		- varExprNode
  		...
  	- TypeNode
    	- ArrayTypeNode
    	- ClassTypeNode
    - ClassDeclNode
    - FuncDeclNode
  ```

- 从`ParseTree parseTreeRoot = parser.program();`返回的CST的根节点进入开始深度优先遍历

  实际上等价于在CST上遍历同时抽象出需要的信息建立自己的AST结点

  > RootNode = astbuilder.visit(parseTreeRoot);

  每进入一层，实际上都是对其子节点的信息进行处理（才能获取到足够的信息new出点），也即递归进入下一层的节点处理，一路到最底层获得到足够信息建出一个基本的ASTNode，由此一路向上**回溯**时才建成每个点。

  ```java
  public ASTNode visitBinaryExpr(MxParser.BinaryExprContext ctx) {
          String op = ctx.op.getText();
          ExprNode ls = (ExprNode) visit(ctx.operand1); // 向下递归建点
          ExprNode rs = (ExprNode) visit(ctx.operand2); // 向下递归建点
    			//回溯时才真正建立本点
    			.....
          return new BinaryExprNode(tmpOp,new Position(ctx),ls,rs);
  }
  ```

- Tips:

  ```java
  //通过lambda表达式遍历ArrayList
  ctx.expression().forEach(tmp->arraySize.add((ExprNode) visit(tmp)));
  
  //通过for循环遍历ArrayList
  for(MxParser.SubProgramContext elements : ctx.subProgram()){...}
  
  //enhanced switch
  MonoExprNode.Op tmpOp = switch (op) {
  		case "++" -> MonoExprNode.Op.PREINC;
      case "--" -> MonoExprNode.Op.PREDEC
      case "!" -> MonoExprNode.Op.LNOT;
      case "~" -> MonoExprNode.Op.BITNOT;
      case "-" -> MonoExprNode.Op.NEG;
      case "+" -> MonoExprNode.Op.POS;
      default -> throw new RuntimeException("[debug] 3");
  };
  
  //利用已有的visit
  public ASTNode visitBaseVariableType(MxParser.BaseVariableTypeContext ctx) {
      return visit(ctx.baseType());
  }
  ```

  
