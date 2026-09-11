#架构文档
# Architecture of LearnDB

The goal of this document is to give a breakdown of the different components of `Learndb`.

We can consider Learndb an RDBMS (relational database management system)- a system for managing the storage of structured data.

## Data Flow

![](./leardb_architecture.png)

To better understand the architecture, let's consider how the user interacts with the system in general.
1. The user interacts via one of the many interfaces
2. Interface receives either: 1) meta command (administrative tasks) or 2) sql program (operate on database)
3. If the input in 2 was SQL, this is parsed into an AST by the SQL Parser module.
4. ^
5. AST is executed by Virtual Machine
6. Which operates on a stack of abstractions of storage, which ground out on a single file on the local file system
    - State Manager
    - B-Tree
    - Pager
    - File

## Component Breakdown

Learndb can be decomposed into the four logical areas for:
- storing data
- interfacing with user
- parsing user input (SQL)
- computing user queries over stored data

### Storage

#### Filesystem

- File system provides access to create database file
- lowest layer of storage hierarchy
- a single db corresponds to a single file
- state of the database is persisted in a single file. But for the execution
of some statement, the state is held across memory and disk. Only when the system is closed, is the state of the database
persisted to disk.

#### Pager
- manages IO to database file
- expose db file as a set of pages (fixed size blocks)
- pages are referenced by their page_number
- a page (with page number page_num) are the bytes in the file from byte offset `page_num * PAGE_SIZE` to  `(page_num + 1) * PAGE_SIZE]`

#### B-tree
- represents an ordered set of key-value pairs
- on-disk data structure, optimized for efficient insertion and retrieval of ordered data (O(lgn))
- one table corresponds to one b-tree
- a b-tree consists of multiple nodes organized in a tree structure
- a node contains many key-value pairs.
- key is the unique, orderable primary key associated with the table; value is the structure/mapping of the other column names and values.
- b-tree interfaces with pager to get pages. 
- each b-tree node corresponds 1:1 with one page
- each row (in a table) is encoded such that the key is the primary key of the table, and the value is an encoding of the rest of the columns/fields in the row 

#### State Manager
- provides a higher level abstraction over database
- understands that a database has many tables; each with a different schema and b-tree
- provides lookup to schema and b-tree, by table name, such that virtual machine can operate on them
- Understands how to read the catalog
  - catalog is a special table with a hardcorded location (page_number), which contains the definition of other tables and their locations
  - location refers to the page number
- Two levels of access: metadata (resolving table tree from name) and data (operate on a sequence of rows, that allow fast (lgn) search along certain dimensions 


### User Interface

- REPL
- file

### SQL Parser

### Parser
- converts user specified sql into an AST.
- The AST is the representation that the VM operates on.

### Compute

#### Virtual Machine (VM)
- VM executes user sql on database state
- The VM takes an AST (instructions), and a database
(represented by a file) and runs the instructions over the database, in the process evolving the database.


## Flows
Next, we will consider some typical flow, to highlight how different components interact
- define a database
- define a table
- insert some records into table
- delete some records
- read contents of a table

### Creating a Database
Currently, a database is associated with a single db file. So a database file implicitly corresponds to one database.
The database file has a header; when the header is set- database is initialized
- a file has pages
- a page is a fixed size contiguous chunk of the file.

### Defining a Table
There is a hardcoded table called catalog. Hardcoded means that it has a fixed root page number for the tree.
When the user requests a new table be created, the Virtual Machine creates a b-tree corresponding to this table; i.e. the root node of the tree is allocated and page number of the root is the location of the table.
The location along with the schema are stored in the catalog.

### Inserting Record

Virtual Machine (VM) looks up the schema of the target tables. And checks the input for schema compatibility, e.g. primary key is unique.
The key (the primary key of the table) and value (struct/dictionary of all other column name-value pairs) are serialized.
The VM then invokes the insert method on the b-tree.

### Reading Records
Virtual Machine (VM) looks up the b-tree of the requested tables. It iterates over the b-tree using a cursor and fills
a buffer with record objects. It then does a second pass over the records to create new record objects with only the requested fields.

### Deleting Records
Virtual Machine (VM) looks up the b-tree of the requested tables. It then invokes delete on the b-tree.


# 架构文档
# LearnDB 的架构

本文档旨在分解说明 `Learndb` 的各个组成部分。

我们可以将 Learndb 视为一个 RDBMS（关系数据库管理系统）——一个用于管理结构化数据存储的系统。

## 数据流

![](./leardb_architecture.png)

为了更好地理解架构，让我们考虑用户通常是如何与系统交互的。
1. 用户通过众多接口之一进行交互
2. 接口接收以下两种输入之一：1）元命令（管理任务）或 2）SQL 程序（对数据库进行操作）
3. 如果第 2 点中的输入是 SQL，则由 SQL 解析器模块将其解析为 AST。
4. ^
5. AST 由虚拟机执行
6. 虚拟机在一组存储抽象栈上操作，这些抽象最终落到本地文件系统上的单个文件
    - 状态管理器
    - B-树
    - 分页器
    - 文件

## 组件分解

Learndb 可以分解为四个逻辑领域，分别用于：
- 存储数据
- 与用户交互
- 解析用户输入（SQL）
- 在已存储的数据上计算用户查询

### 存储

#### 文件系统

- 文件系统提供创建数据库文件的访问能力
- 存储层次结构中的最底层
- 一个数据库对应一个文件
- 数据库的状态持久化在单个文件中。但在执行某些语句时，状态同时保存在内存和磁盘上。只有当系统关闭时，数据库的状态才会持久化到磁盘。

#### 分页器
- 管理对数据库文件的 IO
- 将数据库文件暴露为一组页（固定大小的块）
- 页通过其 page_number（页号）来引用
- 一个页（页号为 page_num）是文件中从字节偏移量 `page_num * PAGE_SIZE` 到 `(page_num + 1) * PAGE_SIZE]` 的字节
- 分页器与 B-树交互以获取页。

#### B-树
- 表示一个有序的键值对集合
- 一种磁盘数据结构，针对有序数据的高效插入和检索进行了优化（O(lgn)）
- 一张表对应一棵 B-树
- 一棵 B-树由多个以树形结构组织的节点组成
- 一个节点包含许多键值对。
- 键是与表关联的唯一、可排序的主键；值是其他列名和值的结构/映射。
- B-树与分页器交互以获取页。
- 每个 B-树节点与一个页一一对应
- 每一行（表中的）被编码为：键是表的主键，值是行中其余列/字段的编码

#### 状态管理器
- 在数据库之上提供更高层次的抽象
- 理解一个数据库有许多表；每张表有不同的模式和 B-树
- 按表名提供对模式和 B-树的查找，以便虚拟机可以操作它们
- 理解如何读取目录
  - 目录是一张特殊表，具有硬编码位置（页号），其中包含其他表的定义及其位置
  - 位置指的是页号
- 两个访问层次：元数据（根据名称解析表树）和数据（操作一系列行，允许沿某些维度进行快速（lgn）搜索）

### 用户界面

- REPL
- 文件

### SQL 解析器

### 解析器
- 将用户指定的 SQL 转换为 AST。
- AST 是虚拟机操作所基于的表示形式。

### 计算

#### 虚拟机（VM）
- 虚拟机在数据库状态上执行用户 SQL
- 虚拟机接收 AST（指令）和一个数据库（由文件表示），并在数据库上运行这些指令，在此过程中演进数据库。

## 流程
接下来，我们将考虑一些典型流程，以突出不同组件之间的交互方式
- 定义数据库
- 定义表
- 向表中插入一些记录
- 删除一些记录
- 读取表的内容

### 创建数据库
目前，一个数据库与单个数据库文件关联。因此，一个数据库文件隐式地对应一个数据库。
数据库文件有一个头部；当头部被设置时——数据库即被初始化
- 文件由页组成
- 页是文件中固定大小的连续块。

### 定义表
有一张名为 catalog 的硬编码表。硬编码意味着它有一个固定的树根页号。
当用户请求创建新表时，虚拟机会创建一棵对应该表的 B-树；即分配树的根节点，根节点的页号就是该表的位置。
该位置连同模式一起存储在目录中。

### 插入记录

虚拟机（VM）查找目标表的模式。并检查输入是否符合模式，例如主键是否唯一。
键（表的主键）和值（所有其他列名-值对的结构/字典）被序列化。
然后虚拟机在 B-树上调用插入方法。

### 读取记录
虚拟机（VM）查找所请求表的 B-树。它使用游标遍历 B-树，并用记录对象填充缓冲区。然后它对记录进行第二遍处理，以创建仅包含所请求字段的新记录对象。

### 删除记录
虚拟机（VM）查找所请求表的 B-树。然后它在 B-树上调用删除操作。