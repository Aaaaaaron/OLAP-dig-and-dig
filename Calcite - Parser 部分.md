---
title: Calcite - Parser 部分
date: 2020-02-08 20:49:53
tags: 
  - BigData
  - Calcite
  - OLAP
---

## 编译知识

**词法分析(lexing)**: 词法分析就是将文本分解成token，token就是具有特殊含义的原子单位, 如果语言的保留字、标点符号、数字、字符串等.

**语法分析(paring)**:语法分析器使用词法分析从输入中分离出一个个的token，并将token流作为其输入

- 根据某种给定的

  形式文法

  对由输入的 token 进行分析并确定其语法结构的过程

  - 自顶向下分析, 对应**LL分析器**
  - 自底向上分析, 对应**LR分析器**

## javacc

使用递归下降语法解析，LL(k)

- 第一个L表示从左到右扫描输入
- 第二个L表示每次都进行最左推导（在推导语法树的过程中每次都替换句型中最左的非终结符为终结符）
- k表示每次向前探索(lookahead) k个终结符
  - **LOOKAHEAD**: 设置在解析过程中面临 choice point 可以look ahead的token数量, 缺省的值是1
  - 调大 K 可以消除二义性, 但会减慢解析速度
  - 比如 **LOOKAHEAD**(2) 就表示要看两个 Token 才能决定下一步的动作

**基本结构**

```java
options {
    JavaCC的选项
}

PARSER_BEGIN(解析器类名)
package 包名;
import 库名;

public class 解析器类名 {
    任意的Java代码
}
PARSER_END(解析器类名)

词法描述器

语法分析器
```

### Option块和class声明块

```java
/* adder.jj Adding up numbers */
options {
    STATIC = false ;
    }

PARSER_BEGIN(Adder)
    class Adder {
        public static void main( String[] args ) throws ParseException, TokenMgrError {
            Adder parser = new Adder( System.in );
            parser.Start();
        }
    }
PARSER_END(Adder)
```

- STATIC默认是true，这里要将其修改为false，使得生成的函数不是static 的。
- ARSER_BEGIN(XXX)……PARSER_END(XXX)块，这里定义了一个名为 Adder的类

### 词法描述器

```java
// 忽略的字符
SKIP:{
    " "
    | "\t"
    | "\n"
    | "\r"
    | "\r\n"
}


// 关键字
TOKEN:{
    <PLUS :"+">
    | <NUMBER : (["0"-"9"])+ >
}
```

### 语法分析器

#### 语法介绍

- java代码块用`{}`声明

  ```java
  // 定义java代码块
  void javaCodeDemo():
  {}
  {
      {
          int i = 0;
          System.out.println(i);
      }
  }
  ```

- java函数
  需要使用JAVACODE声明

  ```java
  JAVACODE void print(Token t){
      System.out.println(t);
  }
  ```

- 条件

1. if语句 []

   ```java
   // if语句
   void ifExpr():
   {}
   {
       [
           <SELECT>
           {
               System.out.println("if select");
           }
       ]
   
       // 循环，出现一次
       (<SELECT>)?
   }
   ```

2. if else 语句 |

   ```java
   // if - else
   void ifElseExpr():
   {}
   {
       (
           <SELECT> {System.out.println("if else select");}
           |
           <UPDATE>  {System.out.println("if else update");}
           |
           <DELETE>  {System.out.println("if else delete");}
           |
           {
              System.out.println("other");
           }
       )
   }
   ```

3. 循环

- while 0~n

  ```java
  // while 0~n
  void whileExpr():
  {}
  {
      (<SELECT>)*
  }
  ```

```java
void Start() :
{}
{
    <NUMBER>
    (
        <PLUS>
        <NUMBER>
    )*
}
```

#### 例子

```java
options {
    STATIC = false ;
    }

PARSER_BEGIN(Adder)
    class Adder {
        public static void main( String[] args ) throws ParseException, TokenMgrError {
            Adder parser = new Adder( System.in );
            parser.Start();
        }
    }
PARSER_END(Adder)

SKIP : { " "| "\n" | "\r" | "\r\n" }

TOKEN : { < PLUS : "+" > }
TOKEN : { < NUMBER : (["0"-"9"])+ > }

void Start() :
{}
{
    <NUMBER>
    (
        <PLUS>
        <NUMBER>
    )*
    <EOF>
}
```

**介绍下生成的一些类:**

- Adder 是语法分析器
- TokenMgrError 是一个简单的定义错误的类，它是Throwable类的子类，用于定义在词法分析阶段检测到的错误。
- ParseException是另一个定义错误的类。它是Exception 和Throwable的子类，用于定义在语法分析阶段检测到的错误。
- Token类是一个用于表示token的类。我们在.jj文件中定义的每一个token（PLUS, NUMBER, or EOF），在Token类中都有对应的一个整数属性来表示，此外每一个token都有名为image的string类型的属性，用来表示token所代表的从输入中获取到的真实值。
- SimpleCharStream是一个转接器类，用于把字符传递给语法分析器。
  AdderConstants是一个接口，里面定义了一些词法分析器和语法分析器中都会用到的常量。AdderTokenManager 是词法分析器。

#### 一个更复杂的例子

```java
PARSER_BEGIN(Calculator)
package com.github.aaaaaron.parser.javacc.calc;

import java.io.* ;

public class Calculator {

    public Calculator(String expr) {
        this((Reader)(new StringReader(expr)));
    }

    public static void main(String[] args)   {
       Calculator calc = new Calculator(args[0]);
       System.out.println(calc.calc());
    }
}

PARSER_END(Calculator)



// 忽略的字符
SKIP:{
    " "
    | "\t"
    | "\n"
    | "\r"
    | "\r\n"
}


// 关键字
TOKEN:{
    <ADD :"+">
    | <SUB :"-">
    | <MUL :"*">
    | <DIV :"/">
}

TOKEN : {
    <NUMBER : <DIGITS>
            | <DIGITS> "." <DIGITS>
            | <DIGITS> "."
            | "." <DIGITS>>
}

// DIGITS这个并不是token，这意味着在后面生成的Token类中，将不会有DIGITS对应的属性，而在语法分析器中也无法使用DIGITS
TOKEN : { < #DIGITS : (["0"-"9"])+ > }

// 计算
double calc():
{
    double value ;
    double result = 0.0;
}
{
  result = mulDiv()
  // 加减
  (
      <ADD>
      value = mulDiv()
      {result += value;}
      |
      <SUB>
      value =mulDiv()
      {result -= value;}
  )*
  {return result;}
}

// 乘除
double mulDiv():
{
    double value;
    double result;
}
{
    result = getNumber()
    (
       <MUL>
        value = getNumber()
        {result *= value;}
       |
       <DIV>
       value = getNumber()
       {result /= value;}
    )*
    {return result;}

}

// 获取字符串
double getNumber():
{
    double number;
    Token t;
}
{
    t = <NUMBER>
    {number = Double.parseDouble(t.image);
    return number;}
}
```

# Calcite

## 基础

```java
graph LR;
    SQL-- Parser -->A(SqlNode);
    A-- Validate -->B(SqlNode);
    B-- SqlToRelConverter --> C(RelNode);
    C-- Optimizer --> RelNode;
```

- SqlNode: 抽象语法树(AST), 树状结构.
- RelNode: 逻辑执行计划节点, 如TableScan, Project, Sort, Join等, 树状结构.
  - 继承自 RelOptNode, 代表能被优化器进行优化
    - `RelTraitSet#getTraitSet();`用来定义逻辑表的物理相关属性(分布/排序)

## SQL 解析阶段（SQL–>SqlNode）

Calcite 使用 JavaCC 做 SQL 解析，JavaCC 根据 Calcite 中定义的 [Parser.jj](https://github.com/apache/calcite/blob/master/core/src/main/codegen/templates/Parser.jj) 文件，生成一系列的 java 代码，生成的 Java 代码会把 SQL 转换成 AST 的数据结构（这里是 SqlNode 类型）

> 与 Javacc 相似的有 ANTLR，JavaCC 中的 jj 文件跟 ANTLR 中的 G4文件类似，Apache Spark 中使用ANTLR

```java
String sql = "select * from emps where id = 1 group by locationid order by empno limit 100";
//quoting, quotedCasing, unquotedCasing, caseSensitive
SqlParser.Config mysqlConfig = SqlParser.configBuilder().setLex(Lex.MYSQL).build();
SqlParser parser = SqlParser.create(sql, mysqlConfig);
SqlNode sqlNode = parser.parseStmt();
```

### 源码分析

```java
public SqlNode parseQuery() throws SqlParseException {
  try {
    return parser.parseSqlStmtEof();
  } catch (Throwable ex) {
    throw handleException(ex);
  }
}
```

这里的 parser 就是 javacc 文件生成的解析类:

##### 生成代码

```java
//org.apache.calcite.sql.parser.impl.SqlParserImpl
//generated from Parser.jj by JavaCC.

public SqlNode parseSqlStmtEof() throws Exception {
  return SqlStmtEof();
}

final public SqlNode SqlStmtEof() throws ParseException {
  SqlNode stmt;
  stmt = SqlStmt();
  jj_consume_token(0);
      {if (true) return stmt;}
  throw new Error("Missing return statement in function");
}

final public SqlNode SqlStmt() throws ParseException {
  SqlNode stmt;
  if (jj_2_34(2)) {
    stmt = SqlSetOption(Span.of(), null);
  } else if (jj_2_35(2)) {
    stmt = SqlAlter();
  } else if (jj_2_36(2)) {
    // select语句
    stmt = OrderedQueryOrExpr(ExprContext.ACCEPT_QUERY);
  } else if (jj_2_37(2)) {
    stmt = SqlExplain();
  } else if (jj_2_38(2)) {
    stmt = SqlDescribe();
  } else if (jj_2_39(2)) {
    stmt = SqlInsert();
  } else if (jj_2_40(2)) {
    stmt = SqlDelete();
  } else if (jj_2_41(2)) {
    stmt = SqlUpdate();
  } else if (jj_2_42(2)) {
    stmt = SqlMerge();
  } else if (jj_2_43(2)) {
    stmt = SqlProcedureCall();
  } else {
    jj_consume_token(-1);
    throw new ParseException();
  }
      {if (true) return stmt;}
  throw new Error("Missing return statement in function");
}
```

##### javacc 定义

```java
/**
 * Parses an SQL statement.
 */
SqlNode SqlStmt() :
{
    SqlNode stmt;
}
{
    (
<#-- Add methods to parse additional statements here -->
<#list parser.statementParserMethods as method>
        LOOKAHEAD(2) stmt = ${method}
    |
</#list>
        stmt = SqlSetOption(Span.of(), null)
    |
        stmt = SqlAlter()
    |
<#if parser.createStatementParserMethods?size != 0>
        stmt = SqlCreate()
    |
</#if>
<#if parser.dropStatementParserMethods?size != 0>
        stmt = SqlDrop()
    |
</#if>
        stmt = OrderedQueryOrExpr(ExprContext.ACCEPT_QUERY) //select语句
    |
        stmt = SqlExplain()
    |
        stmt = SqlDescribe()
    |
        stmt = SqlInsert()
    |
        stmt = SqlDelete()
    |
        stmt = SqlUpdate()
    |
        stmt = SqlMerge()
    |
        stmt = SqlProcedureCall()
    )
    {
        return stmt;
    }
}

/**
 * Parses a leaf SELECT expression without ORDER BY.
 */
SqlSelect SqlSelect() :
{
    final List<SqlLiteral> keywords = new ArrayList<SqlLiteral>();
    final SqlNodeList keywordList;
    List<SqlNode> selectList;
    final SqlNode fromClause;
    final SqlNode where;
    final SqlNodeList groupBy;
    final SqlNode having;
    final SqlNodeList windowDecls;
    final List<SqlNode> hints = new ArrayList<SqlNode>();
    final Span s;
}
{
    <SELECT>
    {
        s = span();
    }
    [
        <HINT_BEG>
        CommaSepatatedSqlHints(hints)
        <COMMENT_END>
    ]
    SqlSelectKeywords(keywords)
    (
        <STREAM> {
            keywords.add(SqlSelectKeyword.STREAM.symbol(getPos()));
        }
    )?
    (
        <DISTINCT> {
            keywords.add(SqlSelectKeyword.DISTINCT.symbol(getPos()));
        }
    |   <ALL> {
            keywords.add(SqlSelectKeyword.ALL.symbol(getPos()));
        }
    )?
    {
        keywordList = new SqlNodeList(keywords, s.addAll(keywords).pos());
    }
    selectList = SelectList()
    (
        <FROM> fromClause = FromClause()
        where = WhereOpt()
        groupBy = GroupByOpt()
        having = HavingOpt()
        windowDecls = WindowOpt()
    )
    {
        return new SqlSelect(s.end(this), keywordList,
            new SqlNodeList(selectList, Span.of(selectList).pos()),
            fromClause, where, groupBy, having, windowDecls, null, null, null,
            new SqlNodeList(hints, getPos()));
    }
}

/**
 * Parses an ORDER BY clause.
 */
SqlNodeList OrderBy() :
{
    List<SqlNode> list;
    SqlNode e;
    final Span s;
}
{
    <ORDER> {
        s = span();
    }
    <BY> e = OrderItem() {
        list = startList(e);
    }
    {
        return new SqlNodeList(list, s.addAll(list).pos());
    }
}

/**
 * Parses one list item in an ORDER BY clause.
 */
SqlNode OrderItem() :
{
    SqlNode e;
}
{
    e = Expression(ExprContext.ACCEPT_SUB_QUERY)
    (
        <ASC>
    |   <DESC> {
            e = SqlStdOperatorTable.DESC.createCall(getPos(), e);
        }
    )
    {
        return e;
    }
}
```

## 生成逻辑执行计划（SqlNode->RelNode）

SqlToRelConverter 中的 `convertQuery()` 将 SqlNode 转换为 RelRoot, 其实现如下:

```java
/**
 * Recursively converts a query to a relational expression.
 *
 * @param query         Query
 * @param top           Whether this query is the top-level query of the
 *                      statement
 * @param targetRowType Target row type, or null
 * @return Relational expression
 */
protected RelRoot convertQueryRecursive(SqlNode query, boolean top,
    RelDataType targetRowType) {
  final SqlKind kind = query.getKind();
  switch (kind) {
  case SELECT:
    return RelRoot.of(convertSelect((SqlSelect) query, top), kind);
  case INSERT:
    return RelRoot.of(convertInsert((SqlInsert) query), kind);
  case DELETE:
    return RelRoot.of(convertDelete((SqlDelete) query), kind);
  case UPDATE:
    return RelRoot.of(convertUpdate((SqlUpdate) query), kind);
  case MERGE:
    return RelRoot.of(convertMerge((SqlMerge) query), kind);
  case UNION:
  case INTERSECT:
  case EXCEPT:
    return RelRoot.of(convertSetOp((SqlCall) query), kind);
  case WITH:
    return convertWith((SqlWith) query, top);
  case VALUES:
    return RelRoot.of(convertValues((SqlCall) query, targetRowType), kind);
  default:
    throw new AssertionError("not a query: " + query);
  }
}

protected void convertSelectImpl(
    final Blackboard bb,
    SqlSelect select) {
  convertFrom(
      bb,
      select.getFrom());
  convertWhere(
      bb,
      select.getWhere());

  final List<SqlNode> orderExprList = new ArrayList<>();
  final List<RelFieldCollation> collationList = new ArrayList<>();
  gatherOrderExprs(
      bb,
      select,
      select.getOrderList(),
      orderExprList,
      collationList);
  final RelCollation collation =
      cluster.traitSet().canonize(RelCollations.of(collationList));

  if (validator.isAggregate(select)) {
    convertAgg(
        bb,
        select,
        orderExprList);
  } else {
    convertSelectList(
        bb,
        select,
   
     orderExprList);
  }

  if (select.isDistinct()) {
    distinctify(bb, true);
  }

  convertOrder(
      select, bb, collation, orderExprList, select.getOffset(),
      select.getFetch());

  bb.setRoot(bb.root, true);
}
```

最后生成的逻辑执行计划

```sql
SQL: 

SELECT EMPNO, GENDER, NAME
FROM EMPS
WHERE GENDER = 'F' AND EMPNO > 125
ORDER BY NAME 
LIMIT 10

====================================================

LogicalSort(sort0=[$2], dir0=[ASC], fetch=[10])
  LogicalProject(EMPNO=[$0], GENDER=[$3], NAME=[$1])
    LogicalFilter(condition=[AND(=($3, 'F'), >($0, 125))])
      LogicalTableScan(table=[[SALES, EMPS]])
```