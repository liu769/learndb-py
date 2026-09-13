#技术参考手册
# Reference 

The goal of this document is intended to provide a complete reference to learndb from the perspective of a user of 
the system. 

## Preface

### Overview

_Learndb_ is a RDBMS (relational database management system). 

Let's unpack this, _relational_ means it can be used to express relations between different entities, e.g. between
`transactions` and `users` involved in them. In some databases this is expressed through a foreign key constraint, 
which constrains the behavior/evolution of one table based on another table(s). 
This is not supported/yet-implemented in learndb [^1].

_Database_ is a collection of tables, each of which has a schema, and zero or more rows of records. The 
schema defines:
- the names of columns/fields 
- what types of data are supported in each field
- any constraint, e.g. if field data can be null or if the field is primary (i.e. must be unique and not null).

The _state_ of a single database (i.e. the schema of the tables in it, and the data within the tables) is persisted 
in a single file on the host filesystem.

The _management system_ manages multiple databases, i.e. multiple isolated collections of tables. The system exposes
interface(s) for: 
- creating and deleting databases
- creating, modifying, and deleting tables in a database
- adding, and removing data from tables


### Setup

Learndb can only be setup from the source repo (i.e. no installation from package repository, e.g. PyPI). The 
instructions are outlined in [README](../README. md) section `Hacking -> Install`

## Interacting with the Database 

Learndb is an _embedded database_. This means there is no standalone server process. The user/agent connects to the 
RDMBS via: 
- REPL
- python language library
- passing a file of commands to the engine  

Fundamentally, the system takes as input a set of statements and creates and modifies a database based on the system.

### REPL

The _REPL_ (read-evaluate-print loop) provides an interactive interface to provide statements the system can execute.
The user can provide: 1) SQL statements (spec below) or 2) meta commands. SQL statements operate on

#### Meta Commands
Meta commands are special commands that are processed by core engine. These include, commands like `.quit` which exits
the terminal.

But these commands more broadly expose non-standard commands, i.e. not part of sql spec - parser. Why some commands 
are meta commands, rather than part of the sql, e.g. `.nuke` which deletes the content of a database, is a 
peculiarity of how this codebase evolved.   

#### Output

Output is printed to console.

### Python Language Library

`interface.py` defines the `Learndb` class entity which can be imported.

TODO: generate code docs, and link interface.py::Learndb, Pipe here

Two important entities needed to programmatically interact with the database are `Learndb`, i.e. the class that 
represents a handle to the database, and `Pipe`

```
Learndb
  - 
 
 Pipe
  - 
```

```
# create handler instance
db = LearnDB(db_filepath)

# submit statement
resp = db.handle_input("select col_a from foo")
assert resp.success

# below are only needed to read results of statements that produce output
# get output pipe
pipe = db.get_pipe()

# print rows
while pipe.has_msgs():
    print(pipe.read())
    
# close handle - flushes any in-memory state
db.close()
```

#### Output

`Pipe` contains all records.

### Filesystem Storage 

The state of entire DB is stored on a single file. The database can be thought of as a logical entity, that is 
stored in some physical medium.

There is a 1 to 1 correspondence between a file and its database. Hence, we can consider the implied database, when 
discussing a database file, and vice versa. Within the context of a single file, there is a single, global, unnamed 
database. 

This means the language only has 1 part names for tables, i.e. no schema, no namespacing.

Further, deleting the `db.file` effectively equals dropping the entire database.

### ACID compliance

Atomic - not atomic. No transactions. Also, no guarantee database isn't left in an inconsistent state due to 
partial statement execution.

Consistent - strong consistency; storage layer updated synchronously

Isolated - guaranteed by database file being opened in exclusive read/write mode, and hence only a single connection to 
database exists.

Durable - As durable as files on underlying filesystem.

## The SQL Language (learndb-sql)

The learndb-sql grammar can be found at: `<repo_root>/learndb/lang_parser/grammar.py`. 

### Learndb-sql grammar specification

The grammar for learndb-sql is written using [lark](https://github.com/lark-parser/lark). Lark is a parsing library 
that allows defining a custom grammar, and parsers for text based on the grammar into an 
[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree). We'll go over Lark basics because statements in learndb-sql 
 are specified in lark [grammar language](https://lark-parser.readthedocs.io/en/latest/grammar.html). 

- Grammar rules are specified in a form similar to [EBNR notation](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form).
- the grammar is made up of terminals and rules.
- terminals are named with an uppercase name, and are defined with a literal or regex expression 
  - e.g. `IDENTIFIER : ("_" | ("a".."z") | ("A".."Z"))* ("_" | ("a".."z") | ("A".."Z") | ("0".."9"))+`
  - these define value literals, and keywords of the language
- grammar rules consist of `left-hand-side : right-hand-side`, where the left side has the name of the terminal or 
  rule, and the right side has one or more matching definition expressions   
- rules are named with a lowercase name, and are patterns of literals and symbols (terminals and rules)
- e.g. ```create_stmnt     : "create"i "table"i table_name "(" column_def_list ")" ```
- Here `"create"i`, `"("`, and `")"` are literals that matche `create`, `(`, an`)`,  respectively.
  - `table_name` and `column_def_list` are other rules with their own definitions


### Data Definition

#### Constraints

Tables can have the following constraints:

- `Not Null` - value cannot be null
- `Primary Key` - value cannot be not and must be unique

#### Data Types

Table columns can have the following types:

- `Integer`
  - 32 bit integer
- `Real`
  - single precision floating point number 
- `Text`
  - unlimited length character string
- `Boolean`
- `Null`

Note, how `Real` typed data is handled is different from how floats are typically
handled (i.e. [IEEE754]( https://en.wikipedia.org/wiki/IEEE_754)).

#### Create Table Statement

```
create_stmnt     : "create"i "table"i table_name "(" column_def_list ")"

?column_def_list  : (column_def ",")* column_def
?column_def       : column_name datatype primary_key? not_null?
datatype         : INTEGER | TEXT | BOOL | NULL | REAL
primary_key      : "primary"i "key"i
not_null         : "not"i "null"i
table_name       : SCOPED_IDENTIFIER
IDENTIFIER       : ("_" | ("a".."z") | ("A".."Z"))* ("_" | ("a".."z") | ("A".."Z") | ("0".."9"))+
SCOPED_IDENTIFIER : (IDENTIFIER ".")* IDENTIFIER
```
An example is 
```
Create table fruits (id integer primary key, name text, avg_weight real)
```

> NOTE: an integer primary key must be declared, i.e. it's declaration and datatype are mandatory 

#### Drop Table Statement

```
  drop_stmnt       : "drop"i "table"i table_name
```
An example is 
```
Drop table fruits
```

### Data Manipulation

#### Data Insertion

```
insert_stmnt     : "insert"i "into"i table_name "(" column_name_list ")" "values"i "(" value_list ")"

column_name_list : (column_name ",")* column_name
value_list       : (literal ",")* literal
column_name      : SCOPED_IDENTIFIER
literal          : INTEGER_NUMBER | REAL_NUMBER | STRING | TRUE | FALSE | NULL
```

An example is:

```
insert into fruits (id, name, avg_weight) values (1, 'apple', 4.2);
```


#### Data Deletion

```
delete_stmnt     : "delete"i "from"i table_name where_clause?

where_clause     : "where"i condition
condition        : or_clause
or_clause        : and_clause
                 | or_clause "or"i and_clause
and_clause       : predicate
                 | and_clause "and"i predicate
predicate        : comparison
                 | predicate ( EQUAL | NOT_EQUAL ) comparison
comparison       : term
                 | comparison ( LESS_EQUAL | GREATER_EQUAL | LESS | GREATER ) term
term             : factor
                 | term ( MINUS | PLUS ) factor
factor           : unary
                 | factor ( SLASH | STAR ) unary
unary            : primary
                 | ( BANG | MINUS ) unary

primary          : literal
                 | nested_select
                 | column_name
                 | func_call
```

An example is:

```
delete from fruits where id = 1;
```

### Queries

Let's consider how we can query tables.

```
select_stmnt     : select_clause from_clause?
select_clause    : "select"i selectable ("," selectable)*
selectable       : expr

from_clause      : "from"i source where_clause? group_by_clause? having_clause? order_by_clause? limit_clause?
where_clause     : "where"i condition
group_by_clause  : "group"i "by"i column_name ("," column_name)*
having_clause    : "having"i condition
order_by_clause  : "order"i "by"i (column_name ("asc"i|"desc"i)?)*
limit_clause     : "limit"i INTEGER_NUMBER ("offset"i INTEGER_NUMBER)?

source            : single_source
                  | joining

single_source      : table_name table_alias?

//split conditioned and unconditioned (cross) join as cross join does not have an on-clause
?joining          : unconditioned_join | conditioned_join
conditioned_join  : source join_modifier? "join"i single_source "on"i condition
unconditioned_join : source "cross"i "join"i single_source

join_modifier    : inner | left_outer | right_outer | full_outer

inner            : "inner"i
left_outer       : "left"i ["outer"i]
right_outer      : "right"i ["outer"i]
full_outer       : "full"i ["outer"i]
cross            : "cross"i

// `expr` is the de-facto root of the expression hierarchy
expr             : condition
```

#### Simple Queries

A select statement can contain `from`, `where`, `group by`, `having`, `limit` and `offset` clauses.

The simplest select statement has no `from` clause. This effectively, evaluates any expression. e.g.
```select 1+1```

The simplest select statement over a datasource is  a `select ... from ... ` without a where clause, e.g.
```select name from fruits```

This  will return all rows from the datasource.

#### Query with Conditions

Consider a query with a simple condition

```select name from fruits where id = 1```

Consider a query with a simple condition

```select name from fruits where avg_weight > 2.0 and avg_weight < 5.0 ```

Note, the condition can be composed of arbitrary logical operations, e.g.

```select name from fruits where avg_weight > 2.0 and avg_weight < 5.0 or name = 'apple' ```

#### Scoping

There is a global, assumed scope. All table names live in this global scope. 

Further, aliases for tables in the context of a query, are defined for the duration of the query.

### Functions

#### User-Defined Functions

Theoretically, a user can define functions in one of two ways: 
  - in learndb-sql (non-native); however, this is not yet implemented
  - in the implementation language, i.e. Python (native). For more details see [./functions.txt](./functions.txt)
  
## Internals 

### Storage Layer 

The storage layer consists of an on-disk btree. The btree is accessed through the below API. Any other backing data structure,
that implements the above API could easily replace the current implementation.

#### Storage API

The Storage API, is the implicit (not formally required by virtual machine) API exposed by the storage layer data structure.
The API consists of:
- insert(key, value)
- get(key)
- delete(key)


#### Btree implementation notes
- Many constants that control the layout of the btree are set in `constants.py`
- `LEAF_NODE_MAX_CELLS`, `INTERNAL_NODE_MAX_CELLS` control how many max children, leaf and internal nodes can have, respectively



## Unsupported Features
- at a single time, only a writer, per db; i.e. no multi writer
- no authentication
- floats implemented very crudely; expression eval uses a fixed epsilon

## Footnotes

[^1]: Arguably, a system can't be called _relational_ without foreign key constraints. But relations can still be 
modelled and foreign keys can still be used- just that the integrity of the constraints can't be enforced. So for 
simplicity, I will call this system an RDBMS. 


  
# 技术参考手册
# 参考

本文档旨在从系统用户的角度，提供对 learndb 的完整参考。

## 前言

### 概述

_Learndb_ 是一个 RDBMS（关系数据库管理系统）。

让我们来拆解这个概念，_关系型（relational）_ 意味着它可以用来表达不同实体之间的关系，例如 `transactions`（交易）与参与其中的 `users`（用户）之间的关系。在某些数据库中，这是通过外键约束来表达的，外键约束会根据一个表（或多个表）来约束另一个表的行为/演进。
这在 learndb 中不受支持/尚未实现 [^1]。

_数据库（Database）_ 是表的集合，每张表都有一个模式（schema），以及零行或多行记录。模式定义了：
- 列/字段的名称
- 每个字段支持哪些数据类型
- 任何约束，例如字段数据是否可以为 null，或者字段是否为主键（即必须唯一且非 null）。

单个数据库的_状态（state）_（即其中各表的模式，以及表中的数据）持久化在宿主文件系统上的单个文件中。

_管理系统（management system）_ 管理多个数据库，即多个相互隔离的表的集合。系统对外暴露的接口用于：
- 创建和删除数据库
- 在数据库中创建、修改和删除表
- 向表中添加数据，以及从表中移除数据


### 安装设置

Learndb 只能从源码仓库进行安装设置（即不能从包仓库安装，例如 PyPI）。相关说明在 [README](../README.md) 的 `Hacking -> Install` 章节中列出。

## 与数据库交互

Learndb 是一个_嵌入式数据库（embedded database）_。这意味着没有独立的服务器进程。用户/代理通过以下方式连接到该 RDBMS：
- REPL
- Python 语言库
- 向引擎传入一个命令文件

从根本上说，系统接收一组语句作为输入，并基于该系统创建和修改数据库。

### REPL

_REPL_（读取-求值-打印循环，read-evaluate-print loop）提供一个交互式界面，用于向系统提供可执行的语句。
用户可以输入：1）SQL 语句（规范见下文）或 2）元命令。SQL 语句操作于……

#### 元命令

元命令是由核心引擎处理的特殊命令。这些包括诸如 `.quit` 之类的命令，它会退出终端。

但更广泛地说，这些命令暴露了非标准命令，即不属于 SQL 规范——解析器的命令。为什么有些命令是元命令，而不是 SQL 的一部分，例如 `.nuke`（删除数据库的内容），这是该代码库演进方式的一个特性。

#### 输出

输出打印到控制台。

### Python 语言库

`interface.py` 定义了可被导入的 `Learndb` 类实体。

TODO：生成代码文档，并在此处链接 interface.py::Learndb、Pipe

以编程方式与数据库交互所需的两个重要实体是 `Learndb`，即表示数据库句柄的类，以及 `Pipe`

```
Learndb
  - 
 
 Pipe
  - 
```

```
# 创建句柄实例
db = LearnDB(db_filepath)

# 提交语句
resp = db.handle_input("select col_a from foo")
assert resp.success

# 以下仅在读取产生输出的语句的结果时才需要
# 获取输出管道
pipe = db.get_pipe()

# 打印行
while pipe.has_msgs():
    print(pipe.read())
    
# 关闭句柄 - 刷新任何内存中的状态
db.close()
```

#### 输出

`Pipe` 包含所有记录。

### 文件系统存储

整个数据库的状态存储在单个文件上。数据库可以被视为一个逻辑实体，存储在某种物理介质中。

文件与其数据库之间存在一一对应的关系。因此，在讨论数据库文件时，我们可以认为它隐含着一个数据库，反之亦然。在单个文件的上下文中，存在一个单一的、全局的、未命名的数据库。

这意味着该语言中表名只有 1 部分名称，即没有模式（schema），没有命名空间。

此外，删除 `db.file` 实际上等同于删除整个数据库。

### ACID 合规性

原子性（Atomic）——非原子性。没有事务。此外，不保证数据库不会因语句的部分执行而处于不一致状态。

一致性（Consistent）——强一致性；存储层同步更新

隔离性（Isolated）——由数据库文件以独占读写模式打开来保证，因此对数据库只存在单个连接。

持久性（Durable）——与底层文件系统上的文件一样持久。

## SQL 语言（learndb-sql）

learndb-sql 文法可在以下位置找到：`<repo_root>/learndb/lang_parser/grammar.py`。

### Learndb-sql 文法规范

learndb-sql 的文法是使用 [lark](https://github.com/lark-parser/lark) 编写的。Lark 是一个解析库，允许定义自定义文法，以及基于该文法将文本解析为 [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) 的解析器。我们将介绍 Lark 的基础知识，因为 learndb-sql 中的语句是用 lark [文法语言](https://lark-parser.readthedocs.io/en/latest/grammar.html) 指定的。

- 文法规则以类似于 [EBNF 记法](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) 的形式指定。
- 文法由终结符（terminals）和规则（rules）组成。
- 终结符用大写名称命名，并用字面量或正则表达式定义
  - 例如 `IDENTIFIER : ("_" | ("a".."z") | ("A".."Z"))* ("_" | ("a".."z") | ("A".."Z") | ("0".."9"))+`
  - 这些定义了该语言的值字面量和关键字
- 文法规则由 `左侧 : 右侧` 组成，其中左侧是终结符或规则的名称，右侧有一个或多个匹配的定义表达式
- 规则用小写名称命名，是字面量和符号（终结符和规则）的模式
- 例如 ```create_stmnt     : "create"i "table"i table_name "(" column_def_list ")" ```
- 这里 `"create"i`、`"("` 和 `")"` 是字面量，分别匹配 `create`、`(` 和 `)`。
  - `table_name` 和 `column_def_list` 是其他规则，有它们自己的定义


### 数据定义

#### 约束

表可以有以下约束：

- `Not Null`（非空）——值不能为 null
- `Primary Key`（主键）——值不能为 null，且必须唯一

#### 数据类型

表列可以有以下类型：

- `Integer`（整数）
  - 32 位整数
- `Real`（实数）
  - 单精度浮点数
- `Text`（文本）
  - 无限长度字符串
- `Boolean`（布尔）
- `Null`（空）

注意，`Real` 类型数据的处理方式与浮点数通常的处理方式（即 [IEEE754](https://en.wikipedia.org/wiki/IEEE_754)）不同。

#### 创建表语句

```
create_stmnt     : "create"i "table"i table_name "(" column_def_list ")"

?column_def_list  : (column_def ",")* column_def
?column_def       : column_name datatype primary_key? not_null?
datatype         : INTEGER | TEXT | BOOL | NULL | REAL
primary_key      : "primary"i "key"i
not_null         : "not"i "null"i
table_name       : SCOPED_IDENTIFIER
IDENTIFIER       : ("_" | ("a".."z") | ("A".."Z"))* ("_" | ("a".."z") | ("A".."Z") | ("0".."9"))+
SCOPED_IDENTIFIER : (IDENTIFIER ".")* IDENTIFIER
```
一个示例是
```
Create table fruits (id integer primary key, name text, avg_weight real)
```

> 注意：必须声明一个整数主键，即其声明和数据类型是强制性的

#### 删除表语句

```
  drop_stmnt       : "drop"i "table"i table_name
```
一个示例是
```
Drop table fruits
```

### 数据操作

#### 数据插入

```
insert_stmnt     : "insert"i "into"i table_name "(" column_name_list ")" "values"i "(" value_list ")"

column_name_list : (column_name ",")* column_name
value_list       : (literal ",")* literal
column_name      : SCOPED_IDENTIFIER
literal          : INTEGER_NUMBER | REAL_NUMBER | STRING | TRUE | FALSE | NULL
```

一个示例是：

```
insert into fruits (id, name, avg_weight) values (1, 'apple', 4.2);
```


#### 数据删除

```
delete_stmnt     : "delete"i "from"i table_name where_clause?

where_clause     : "where"i condition
condition        : or_clause
or_clause        : and_clause
                 | or_clause "or"i and_clause
and_clause       : predicate
                 | and_clause "and"i predicate
predicate        : comparison
                 | predicate ( EQUAL | NOT_EQUAL ) comparison
comparison       : term
                 | comparison ( LESS_EQUAL | GREATER_EQUAL | LESS | GREATER ) term
term             : factor
                 | term ( MINUS | PLUS ) factor
factor           : unary
                 | factor ( SLASH | STAR ) unary
unary            : primary
                 | ( BANG | MINUS ) unary

primary          : literal
                 | nested_select
                 | column_name
                 | func_call
```

一个示例是：

```
delete from fruits where id = 1;
```

### 查询

让我们考虑如何查询表。

```
select_stmnt     : select_clause from_clause?
select_clause    : "select"i selectable ("," selectable)*
selectable       : expr

from_clause      : "from"i source where_clause? group_by_clause? having_clause? order_by_clause? limit_clause?
where_clause     : "where"i condition
group_by_clause  : "group"i "by"i column_name ("," column_name)*
having_clause    : "having"i condition
order_by_clause  : "order"i "by"i (column_name ("asc"i|"desc"i)?)*
limit_clause     : "limit"i INTEGER_NUMBER ("offset"i INTEGER_NUMBER)?

source            : single_source
                  | joining

single_source      : table_name table_alias?

//将带条件和无条件的连接分开（交叉连接），因为交叉连接没有 on 子句
?joining          : unconditioned_join | conditioned_join
conditioned_join  : source join_modifier? "join"i single_source "on"i condition
unconditioned_join : source "cross"i "join"i single_source

join_modifier    : inner | left_outer | right_outer | full_outer

inner            : "inner"i
left_outer       : "left"i ["outer"i]
right_outer      : "right"i ["outer"i]
full_outer       : "full"i ["outer"i]
cross            : "cross"i

// `expr` 是表达式层次结构事实上的根
expr             : condition
```

#### 简单查询

一个 select 语句可以包含 `from`、`where`、`group by`、`having`、`limit` 和 `offset` 子句。

最简单的 select 语句没有 `from` 子句。这实际上是求值任何表达式。例如
```select 1+1```

对数据源的最简单 select 语句是 不带 where 子句的 `select ... from ... `，例如
```select name from fruits```

这将返回数据源中的所有行。

#### 带条件的查询

考虑一个带简单条件的查询

```select name from fruits where id = 1```

考虑一个带简单条件的查询

```select name from fruits where avg_weight > 2.0 and avg_weight < 5.0 ```

注意，条件可以由任意逻辑运算组成，例如

```select name from fruits where avg_weight > 2.0 and avg_weight < 5.0 or name = 'apple' ```

#### 作用域

存在一个全局的、假定的作用域。所有表名都位于这个全局作用域中。

此外，查询上下文中表的别名，在该查询的持续期间内被定义。

### 函数

#### 用户定义函数

理论上，用户可以通过以下两种方式之一来定义函数：
  - 用 learndb-sql（非原生）；然而，这尚未实现
  - 用实现语言，即 Python（原生）。更多细节见 [./functions.txt](./functions.txt)
  
## 内部实现

### 存储层

存储层由磁盘上的 btree 组成。btree 通过以下 API 访问。任何其他实现了上述 API 的后备数据结构，都可以轻松替换当前实现。

#### 存储 API

存储 API 是存储层数据结构暴露的隐式 API（虚拟机并未正式要求）。该 API 包括：
- insert(key, value)
- get(key)
- delete(key)


#### Btree 实现说明
- 控制 btree 布局的许多常量在 `constants.py` 中设置
- `LEAF_NODE_MAX_CELLS`、`INTERNAL_NODE_MAX_CELLS` 分别控制叶节点和内部节点最多可以有多少个子节点



## 不支持的特性
- 在某一时刻，每个数据库只有一个写入者；即不支持多写入者
- 没有身份认证
- 浮点数实现非常粗糙；表达式求值使用固定的 epsilon

## 脚注

[^1]: 可以说，没有外键约束的系统不能被称为_关系型_。但仍然可以对关系进行建模，也仍然可以使用外键——只是无法强制约束的完整性。所以为简单起见，我将此系统称为 RDBMS。