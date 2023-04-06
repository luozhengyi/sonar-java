Java Frontend 解读
=================
```cmd
sonar-java/java-frontend$ tree -d -L 6 -I "test|target"
```
```java
.
└── src/main/java/org
                    ├── eclipse/jdt/core/dom   // only contains one "ASTUtils.java"
                    └── sonar
                        ├── java
                        └── plugins/java/api  // 提供给外部的的接口(写checker...)
```
```cmd
sonar-java/java-frontend/src/main/java/org/sonar/java$ tree -L 4 -d
```
```java
.
├── annotations
├── ast
│   ├── api
│   ├── parser
│   └── visitors
├── caching
├── cfg
├── classpath  // 可以分析 .class 文件
├── collections
├── exceptions
├── filters
├── matcher
├── model // model data of "concrete syntax tree"
│   ├── declaration // class declaration; method declaration...
│   ├── expression  // 带有 operator(new +-/* ...) 的表达式
│   ├── location // 每一个词法分析的token的位置信息，静态分析必须要这个
│   ├── pattern  // JDK17中的预览语言特征 JEP 409
│   └── statement  // 不带 operator的语句(ifstatement returnstatement...)
├── regex
├── reporting
└── testing
```
```cmd
sonar-java/java-frontend/src/main/java/org/sonar/plugins/java/api$ tree -L 5 -d
```
```java
.
├── caching
├── cfg
├── internal
├── location
├── semantic
└── tree         // 全是各种tree(ast node)的抽象类
```

[JDK17 中的预览语言特征 JEP 409](https://blog.csdn.net/weixin_38833041/article/details/125450755)

model 分析
----------

* model 大致是根据 java language 的 EBNF 来定义的. 可以参考 ClassTreeImpl.java
* public interface Tree{} 是所有 syntax tree 的 node 的总接口
* Java EBNF中比较顶层的 非终端符号 大概分三类:
  - declaration: binds a non-blank identifier to a constant, type, variable, function.
  - expression: specifies the computation of a value by applying operators and functions to operands.
  - statement: control execution.


