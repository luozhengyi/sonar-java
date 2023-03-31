Writing Custom Java Rules 101
==========

你正在使用 SonarQube 和它的 Java Analyzer 来分析你的项目, 但是没有规则可以用于你的公司项目检查的目的? 那么你可以定制的自己的规则来实现你的逻辑检查.

本文档主要指导你如何定制你个人的 SonarQube Java Analyzer. 它会覆盖理解和开发有效的规则所需要的的基于 SonarSource Analyzer for Java 提供的 API 的静态分析的主要概念.

## Content

* [Getting Started](#getting-started)
  * [Looking at the pom](#looking-at-the-pom)
* [Writing a rule](#writing-a-rule)
  * [Three files to forge a rule](#three-files-to-forge-a-rule)
  * [A specification to make it right](#a-specification-to-make-it-right)
  * [A test file to rule them all](#a-test-file-to-rule-them-all)
  * [A test class to make it pass](#a-test-class-to-make-it-pass)
  * [First version: Using syntax trees and API basics](#first-version-using-syntax-trees-and-api-basics)
  * [Second version: Using semantic API](#second-version-using-semantic-api)
  * [What you can use, and what you can't](#what-you-can-use-and-what-you-cant)
* [Registering the rule in the custom plugin](#registering-the-rule-in-the-custom-plugin)
  * [Rule Metadata](#rule-metadata)
  * [Rule Activation](#rule-activation)
  * [Rule Registrar](#rule-registrar)
* [Testing a custom plugin](#testing-a-custom-plugin)
  * [How to define rule parameters](#how-to-define-rule-parameters)
  * [How to test sources requiring external binaries](#how-to-test-sources-requiring-external-binaries)
  * [How to test precise issue location](#how-to-test-precise-issue-location)
  * [How to test the Source Version in a rule](#how-to-test-the-source-version-in-a-rule)
* [References](#references)

## Getting started

你将要开发的规则会通过依赖于 **SonarSource Analyzer for Java API** 的专用的自定义插件来提交. 为了工作效率, 我们提供了一个 maven 项目模板, 你可以根据本文档的指导来填充该模板.

你可以克隆仓库 (https://github.com/SonarSource/sonar-java) 然后将子模块 [java-custom-rules-examples](https://github.com/SonarSource/sonar-java/tree/master/docs/java-custom-rules-example)导入你的IDE来获取模板项目.
这个项目已经包含一些定制规则的例子. 我们的目标是添加额外的规则!

### Looking at the POM

一个自定义插件是一个 Maven  项目，在看代码之前，有必要先关注下与你的 soon-to-be-released 自定义插件的配置相关的一些行. Maven 项目的根配置文件是 `pom.xml`.

我们的情况中有3个根配置文件:
* `pom.xml`: use a snapshot version of the Java Analyzer
* `pom_SQ_8_9_LTS.xml`: self-contained `pom` file, configured with dependencies matching SonarQube `8.9 LTS` requirements

These 3 `pom`s correspond different use-cases, depending on which instance of SonarQube you will target with your custom-rules plugin. In this tutorial, **we will only use the file named `pom_SQ_8_9_LTS.xml`**, as it is completely independent from the build of the Java Analyzer, is self-contained, and will target the latest release of SonarQube.

让我们使用以下命令来开始构建自定义插件:

```
mvn clean install -f pom_SQ_8_9_LTS.xml
```

注意你也可以决定 **删除** 原来的 pom.xml 文件 (**NOT RECOMMENDED**), 然后将 `pom_SQ_8_9_LTS.xml` 重命名为 `pom.xml`. 那么你也可以使用以下简单的命令:

```
mvn clean install
```

Looking inside the `pom`, you will see that both versions of SonarQube and the Java Analyzer are hard-coded. This is because SonarSource's analyzers are directly embedded in the various SonarQube versions and are shipped together. For instance, SonarQube `7.9` (previous LTS) is shipped with the version `6.3.2.22818` of the Java Analyzer, while SonarQube `8.9` (LTS) is shipped with a much more recent version `6.15.1.26025` of the Java Analyzer. **These versions can not be changed**.

```xml
<properties>
  <sonarqube.version>8.9.0.43852</sonarqube.version>
  <sonarjava.version>6.15.1.26025</sonarjava.version>
  <!-- [...] -->
</properties>
```

Other tags such as `<groupId>`, `<artifactId>`, `<version>`, `<name>` and `<description>` can be freely modified.

```xml
  <groupId>org.sonar.samples</groupId>
  <artifactId>java-custom-rules-example</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <name>SonarQube Java :: Documentation :: Custom Rules Example</name>
  <description>Java Custom Rules Example for SonarQube</description>
```

在下面的代码片段中, 请注意 **entry point of the plugin** 使用 java 类 `MyJavaRulesPlugin` 的 qualified name 被提供为 sonar-packaging-maven 插件的配置中的 `<pluginClass>` .
如果你重构你的代码, 例如重名令或者移动继承于 `org.sonar.api.SonarPlugin` 的类, 那么你将必须改变这个配置.
`<sonarQubeMinVersion>` 属性也是用来保证你的插件与你的目标  SonarQube instance 之间的兼容性.

```xml
<plugin>
  <groupId>org.sonarsource.sonar-packaging-maven-plugin</groupId>
  <artifactId>sonar-packaging-maven-plugin</artifactId>
  <version>1.20.0.405</version>
  <extensions>true</extensions>
  <configuration>
    <pluginKey>java-custom</pluginKey>
    <pluginName>Java Custom Rules</pluginName>
    <pluginClass>org.sonar.samples.java.MyJavaRulesPlugin</pluginClass>
    <sonarLintSupported>true</sonarLintSupported>
    <sonarQubeMinVersion>${sonarqube.version}</sonarQubeMinVersion>
    <requirePlugins>java:${sonarjava.version}</requirePlugins>
  </configuration>
</plugin>
```

## Writing a rule

在本节中，我们将从头开始编写一个自定义规则。为此, 我们将使用  [Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development) (TDD) 方法, 即先写test cases, 接着再实现规则方案.

### Three files to forge a rule

实现规则时，我们总需要最少创建3个不同的文件:
  1. A test file, 规则的测试用例
  1. A test class, 规则的单元测试（调用规则来检测测试用例）.
  1. A rule class, 规则实现文件.
  
为了创建我们的第一个定制规则 (usually called a "*check*"), 让我们首先在模板项目中创建三个文件开始, 如下所述:

  1. 在文件夹 `/src/test/files` 中, 创建空的测试用例文件  `MyFirstCustomCheck.java`, 然后拷贝粘贴以下代码.
```java
class MyClass {
}
```

  2. In package `org.sonar.samples.java.checks` of `/src/test/java`, 创建单元测试文件 `MyFirstCustomCheckTest` 然后拷贝粘贴以下代码.
```java
package org.sonar.samples.java.checks;
 
import org.junit.jupiter.api.Test;

class MyFirstCustomCheckTest {

  @Test
  void test() {
  }

}
```

  3. In package `org.sonar.samples.java.checks` of `/src/main/java`, 创建继承于 Java Plugin API 提供的 `org.sonar.plugins.java.api.IssuableSubscriptionVisitor` 的规则文件 `MyFirstCustomCheck`. 接着, 使用以下代码片段替换 `nodesToVisit()` 方法的内容. 这个文件会在实现规则时继续描述!
```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.Tree.Kind;
import java.util.Collections;
import java.util.List;

@Rule(key = "MyFirstCustomRule")
public class MyFirstCustomCheck extends IssuableSubscriptionVisitor {

  @Override
  public List<Kind> nodesToVisit() {
    return Collections.emptyList();
  }
}

```

>
> :question: **More files...**
> 
> 如果说上面描述的3个文件是写规则的最基本需求，那么有些情况下也会需要额外的文件. 例如, 当一个规则使用参数时, 或它的行为依赖于检测到的java 版本, 那么可能需要多个测试文件. 使用额外的文件来描述 rule metadata 也是有可能的, 例如 HTML 格式的描述. 这些情况会在本文档的其它主题中描述.
>

### A specification to make it right

当然, 继续深入前, 我们需要一个写规则的关键元素: 一份规则规范(规则描述)! 为了练习的目的, 我们考虑使用来自著名 Guru 的名言来作为我们定制规则的规范, 因为这当然是绝对正确和无可争议的.

>
> **Gandalf - Why Program When Magic Rulez (WPWMR, p.42)**
>
> *“For a method having a single parameter, the types of its return value and its parameter should never be the same.”*
>

### A test file to rule them all

因为我们选择 TDD 方法, 我们第一件事就是写规则的测试用例. 在这个文件中, 我们考虑我们规则在分析时可能遇到的各种大量的例子，然后标记出我们规则需要报出问题的行. 我们通常在有问题的行的尾部注释上  `// Noncompliant` 作为标记. 为什么 *Noncompliant*? 因为标记行 do not *comply* with the rule.

覆盖所用可能的用例不是必要的需求，测试用例文件的目的是覆盖分析时可能碰到的所有情形，每种情形都是需要抽象掉无关的细节. 注意该用例文件应该结构正确，并且所有的代码都能编译.

在之前创建的测试用例文件 `MyFirstCustomCheck.java` 中, 拷贝粘贴以下代码: 
```java
class MyClass {
  MyClass(MyClass mc) { }

  int     foo1() { return 0; }
  void    foo2(int value) { }
  int     foo3(int value) { return 0; } // Noncompliant
  Object  foo4(int value) { return null; }
  MyClass foo5(MyClass value) {return null; } // Noncompliant

  int     foo6(int value, String name) { return 0; }
  int     foo7(int ... values) { return 0;}
}
```

测试用例文件现在包含以下例子:
* **line 2:** A constructor, to differentiate the case from a method;
* **line 4:** 不带参数的方法 (`foo1`);
* **line 5:** 返回void的方法 (`foo2`);
* **line 6:** A method returning the same type as its parameter (`foo3`), which will be noncompliant;
* **line 7:** A method with a single parameter, but a different return type (`foo4`);
* **line 8:** Another method with a single parameter and same return type, but with non-primitive types (`foo5`), therefore non compliant too;
* **line 10:** 多于一个参数的方法 (`foo6`);
* **line 11:** A method with a variable arity argument (`foo7`);

### A test class to make it pass

待测试文件更新后, 让我们开始更新我们的单元测试文件来使用它，并且同时连接我们的规则(not yet implemented). 为此, 返回单元测试类 `MyFirstCustomCheckTest`, 然后使用以下的代码更新它的 `test()` 方法， (你可能需要导入 `org.sonar.java.checks.verifier.CheckVerifier`):

```java
  @Test
  void test() {
    CheckVerifier.newVerifier()
      .onFile("src/test/files/MyFirstCustomCheck.java")
      .withCheck(new MyFirstCustomCheck())
      .verifyIssues();
  }
```

正如你可能观察到的, 单元测试类只包含一个单一测试, 这个测试目的是验证我们即将要实现的规则的行为. 为此, 它依赖于 Java Analyzer rule-testing API 提供的 `CheckVerifier` 类. `CheckVerifier` 类提供了验证规则的有用方法, 使得我们能完全抽象所有与分析器初始化相关的机制. 注意，当验证规则时, *verifier* 会收集标记为 *Noncompliant* 的行, 然后验证规则报出预期的问题, 并且 *只有* 这些问题.

现在, 让我们继续处理 TDD 的后续步骤: make the test fail!

为此, 使用 JUnit 简单执行单元测试即可. 测试应该 **fail** 并输出错误 "**At least one issue expected**", 如下所示. 因为我们的规则还没实现, 不会有问题报出, 所以这是预期的结果.

```
java.lang.AssertionError: No issue raised. At least one issue expected
    at org.sonar.java.checks.verifier.InternalCheckVerifier.assertMultipleIssues(InternalCheckVerifier.java:291)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.checkIssues(InternalCheckVerifier.java:231)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyAll(InternalCheckVerifier.java:222)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyIssues(InternalCheckVerifier.java:167)
    at org.sonar.samples.java.checks.MyFirstCustomCheckTest.test(MyFirstCustomCheckTest.java:13)
    ...
```

### First version: Using syntax trees and API basics

在我们开始实现规则前，你需要一些背景知识.

运行任何规则前, SonarQube Java Analyzer 解析传入的 Java code file 并产生一个等价的数据结构: the **Syntax Tree**. Java 语言的每个结构都可以被表示为一种特定的 Syntax Tree 用来详细说明其每一个特点. 每个结构都关联着一种特定类别和一个显示描述它所有特点的接口. 例如, 与方法声明相关的类别是 `org.sonar.plugins.java.api.tree.Tree.Kind.METHOD`, 然后它的接口被定义在  `org.sonar.plugins.java.api.tree.MethodTree`. 所有的类别列在了 [`Kind` enum of the Java Analyzer API](https://github.com/SonarSource/sonar-java/blob/6.13.0.25138/java-frontend/src/main/java/org/sonar/plugins/java/api/tree/Tree.java#L47).

当创建规则类时, 我们选择实现来自 API 的 `IssuableSubscriptionVisitor` 类. 这个类除了提供了一系列用来报出问题的方法之外，还 **defines the strategy which will be used when analyzing a file**. 正如它的名字所告诉我们的, 它是基于订阅机制的, 允许我们指定规则应该对何种类型的树作出反应. 通过 `nodesToVisit()` 方法指定要覆盖的结点类型的list. 在先前的步骤中, 我们修改了方法返回一个空的 list, 即不订阅语法树的任何结点.

现在是时候开始实现我们的第一条规则了! 回到 `MyFirstCustomCheck` class, 并修改 the list of Kinds returned by the `nodesToVisit()` method. 因为我们的规则关注 method declarations, 我们只需要访问 methods. 为此, simply make sure that we return a singleton list containing only `Kind.METHOD` as a parameter of the returned list, 如下代码所示.

```java
@Override
public List<Kind> nodesToVisit() {
  return Collections.singletonList(Kind.METHOD);
}
```

一旦要访问的结点指定后, we have to implement how the rule will react when encountering method declarations. To do so, override method `visitNode(Tree tree)`, inherited from `SubscriptionVisitor` through `IssuableSubscriptionVisitor`.

```java
@Override
public void visitNode(Tree tree) {
}
```

因为我们注册规则去访问 Method 结点, 所以我们可以知道该方法每次调用时，树的参数必然是 `org.sonar.plugins.java.api.tree.MethodTree` (the interface tree associated with the `METHOD` kind). 第一步，我们就可以安全地直接将树强转为 MethodTree, 如下所示. 注意，如果我们注册了多种结点种类，就必须得在强转前使用 `Tree.is(Kind ... kind)` 对结点种类进行测试.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
}
```

现在, 让我们收缩规则的焦点, 检查方法是否只有一个参数，然后报出问题.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    reportIssue(method.simpleName(), "Never do that!");
  }
}
```

The method `reportIssue(Tree tree, String message)` from `IssuableSubscriptionVisitor` 可以在给定的树上报出带有具体信息问题. 此例中，我们选择报在方法名的精确位置上.

Now, let's test our implementation by executing `MyFirstCustomCheckTest.test()` again.

```
java.lang.AssertionError: Unexpected at [5, 7, 11]
    at org.sonar.java.checks.verifier.InternalCheckVerifier.assertMultipleIssues(InternalCheckVerifier.java:303)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.checkIssues(InternalCheckVerifier.java:231)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyAll(InternalCheckVerifier.java:222)
    at org.sonar.java.checks.verifier.InternalCheckVerifier.verifyIssues(InternalCheckVerifier.java:167)
    at org.sonar.samples.java.checks.MyFirstCustomCheckTest.test(MyFirstCustomCheckTest.java:13)
    ...
```

当然, 测试再次失败... The `CheckVerifier` reported that lines 5, 7 and 11 are raising unexpected issues, as visible in the stack-trace above. By looking back at our test file, it's easy to figure out that raising an issue line 5 is wrong because the return type of the method is `void`, line 7 is wrong because `Object` is not the same as `int`, and line 11 is also wrong because of the variable *arity* of the method. 然而根据我们的实现，是应该报出这些问题, 因为我们没有检查参数类型和返回类型. 为了处理 type, 我们需要使用语法树知识，获取更多的依赖信息. 现在我们需要使用 semantic API!

>
> :question: **IssuableSubscriptionVisitor and BaseTreeVisitor**
>
> 为了实现这个规则, 我们选择 `IssuableSubscriptionVisitor` 作为实现的基础. This visitor offers an easy approach to writing quick and simple rules, because it allows us to narrow the focus of our rule to a given set of Kinds to visit by subscribing to them. However, this approach is not always the most optimal one. In such a situation, it could be useful to take a look at another visitor provided with the API: `org.sonar.plugins.java.api.tree.BaseTreeVisitor`. The `BaseTreeVisitor` contains a `visit()` method dedicated for each and every kind of the syntax tree, and is particularly useful when the visit of a file has to be fine tuned.
>
> In [rules already implemented in the Java Plugin](https://github.com/SonarSource/sonar-java/tree/5.12.1.17771/java-checks/src/main/java/org/sonar/java/checks), you will be able to find multiple rule using both approaches: An `IssuableSubscriptionVisitor` as entry point, helped by simple `BaseTreeVisitor`(s) to identify pattern in other parts of code.
>

### Second version: Using semantic API

Up to now, our rule implementation only relied on the data provided directly by syntax tree that resulted from the parsing of the code. However, the SonarAnalyzer for Java provides a lot more regarding the code being analyzed, because it also constructs a ***semantic model*** of the code. This semantic model provides information related to each ***symbol*** being manipulated. For a method, for instance, the semantic API will provide useful data such as a method's owner, its usages, the types of its parameters and its return type, the exception it may throw, etc. Don't hesitate to explore the [semantic package of the API](https://github.com/SonarSource/sonar-java/tree/6.13.0.25138/java-frontend/src/main/java/org/sonar/plugins/java/api/semantic) in order to have an idea of what kind of information you will have access to during analysis!

现在，返回到规则的实现中，并利用语义的优势.

一旦我们知道我们的方法只有一个参数, 让我们使用 `symbol()` 方法 从 `MethodTree` 获取 method 的 symbol.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    MethodSymbol symbol = method.symbol();
    reportIssue(method.simpleName(), "Never do that!");
  }
}

```

From the symbol, 很容易检索 **the type of its first parameter**, 和 **return type** (You may have to import `org.sonar.plugins.java.api.semantic.Symbol.MethodSymbol` and `org.sonar.plugins.java.api.semantic.Type`).

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    Symbol.MethodSymbol symbol = method.symbol();
    Type firstParameterType = symbol.parameterTypes().get(0);
    Type returnType = symbol.returnType().type();
    reportIssue(method.simpleName(), "Never do that!");
  }
}
```

因为规则只应该在两种类型相同时报出问题，那么我们只需要在报出问题之前简单地使用 `Type` class中的 `is(String fullyQualifiedName)` 测试两者是否相同.

```java
@Override
public void visitNode(Tree tree) {
  MethodTree method = (MethodTree) tree;
  if (method.parameters().size() == 1) {
    Symbol.MethodSymbol symbol = method.symbol();
    Type firstParameterType = symbol.parameterTypes().get(0);
    Type returnType = symbol.returnType().type();
    if (returnType.is(firstParameterType.fullyQualifiedName())) {
      reportIssue(method.simpleName(), "Never do that!");
    }
  }
}
```

Now, **execute the test** class again.

Test passed? If not, then check if you somehow missed a step.

If it passed...

>
> :tada: **Congratulations!** :confetti_ball:
>
> [*You implemented your first custom rule for the SonarQube Java Analyzer!*](resources/success.jpg)
>

### What you can use, and what you can't

When writing custom Java rules, you can only use classes from package [`org.sonar.plugins.java.api`](https://github.com/SonarSource/sonar-java/tree/6.13.0.25138/java-frontend/src/main/java/org/sonar/plugins/java/api).

When browsing the existing 600+ rules from the SonarSource Analyzer for Java, you will sometime notice use of some other utility classes, not part of the API. While these classes could be sometime extremely useful in your context, **these classes are not available at runtime** for custom rule plugins. It means that, while your unit tests are still going to pass when building your plugin, your rules will most likely make analysis **crash at analysis time**.

Note that we are always open to discussion, so don't hesitate to reach us and participate to threads, through our [community forum](https://community.sonarsource.com/), to suggest features and API improvement!

## Registering the rule in the custom plugin

OK, 此刻你可能非常高兴, 因为规则是按我们的预期运行的... 但是，我们还没有结束. 在将我们的规则用于实际的项目之前, 我们必须要在自定义插件中注册我们的规则来完成规则的创建.

### Rule Metadata
第一件要做的事是提供规则的所有 metadata, 此举能使我们在 SonarQube Platform 上正确地注册我们的规则.
There are 2 ways to add metadata for your rule: annotation and static documentation.
While annotation provides a handy way to document the rule, static documentation offers the possibility for richer information.
Incidentally, static documentation is also the way rules in the sonar-java analyzer are described.

To provide metadata for your rule, you need to create an HTML file, where you can provide an extended textual description of the rule, and a JSON file, with the actual metadata.
In the case of `MyFirstCustomRule`, you will head to the `src/main/resources/org/sonar/l10n/java/rules/java/` folder to create `MyFirstCustomRule.html` and `MyFirstCustomRule.json`.

We first need to populate the HTML file with some information that will help developers fix the issue.
```html
<p>For a method having a single parameter, the types of its return value and its parameter should never be the same.</p>

<h2>Noncompliant Code Example</h2>
<pre>
class MyClass {
  int doSomething(int a) { // Noncompliant
    return 42;
  }
}
</pre>

<h2>Compliant Solution</h2>
<pre>
class MyClass {
  int doSomething() { // Compliant
    return 42;
  }
  long doSomething(int a) { // Compliant
    return 42L;
  }
}
</pre>
```

We can now add metadata to `src/main/resources/org/sonar/l10n/java/rules/java/MyFirstCustomRule.json`:
```json
{
  "title": "Return type and parameter of a method should not be the same",
  "type": "Bug",
  "status": "ready",
  "tags": [
    "bugs",
    "gandalf",
    "magic"
  ],
  "defaultSeverity": "Critical"
}
```
With this example, we have a concise but descriptive `title` for our rule, the `type` of issue it highlights, its `status` (ready or deprecated), the `tags` that should bring it up in a search and the `severity` of the issue.


### Rule Activation
第二件要做的事是在插件中激活我们的规则. To do so, open class `RulesList` (`org.sonar.samples.java.RulesList`). In this class, you will notice methods `getJavaChecks()` and `getJavaTestChecks()`. These methods are used to register our rules with alongside the rule of the Java plugin. Note that rules registered in `getJavaChecks()` will only be played against source files, while rules registered in `getJavaTestChecks()` will only be played against test files. To register the rule, simply add the rule class to the list builder, as in the following code snippet:

```java
public static List<Class<? extends JavaCheck>> getJavaChecks() {
  return Collections.unmodifiableList(Arrays.asList(
      // other rules...
      MyFirstCustomCheck.class
    ));
}

```


### Rule Registrar

Because your rules are relying on the SonarSource Analyzer for Java API, 你需要告诉父 Java plugin, 一些新的规则也需要被检索. 如果你是使用模板自定义插件作为本教程的基础, 那么此刻你已经做完了, but feel free to have a look at the `MyJavaFileCheckRegistrar.java` class, which connects the dots. 最后, 确保将注册类作为你的自定义插件的扩展加入到了你的 Plugin definition class (`MyJavaRulesPlugin.java`)中了.

```java
/**
 * Provide the "checks" (implementations of rules) classes that are going be executed during
 * source code analysis.
 *
 * This class is a batch extension by implementing the {@link org.sonar.plugins.java.api.CheckRegistrar} interface.
 */
@SonarLintSide
public class MyJavaFileCheckRegistrar implements CheckRegistrar {
 
  /**
   * Register the classes that will be used to instantiate checks during analysis.
   */
  @Override
  public void register(RegistrarContext registrarContext) {
    // Call to registerClassesForRepository to associate the classes with the correct repository key
    registrarContext.registerClassesForRepository(MyJavaRulesDefinition.REPOSITORY_KEY, checkClasses(), testCheckClasses());
  }
 
 
  /**
   * Lists all the main checks provided by the plugin
   */
  public static List<Class<? extends JavaCheck>> checkClasses() {
    return RulesList.getJavaChecks();
  }
 
  /**
   * Lists all the test checks provided by the plugin
   */
  public static List<Class<? extends JavaCheck>> testCheckClasses() {
    return RulesList.getJavaTestChecks();
  }
 
}
```

现在，我们已经添加了一条新的规则，我们也需要更新测试以确保我们的规则确实被包含进去. To do so, navigate to its corresponding test class, named `MyJavaFileCheckRegistrarTest`, and update the expected number of rules from 8 to 9.

```java

class MyJavaFileCheckRegistrarTest {

  @Test
  void checkNumberRules() {
    CheckRegistrar.RegistrarContext context = new CheckRegistrar.RegistrarContext();

    MyJavaFileCheckRegistrar registrar = new MyJavaFileCheckRegistrar();
    registrar.register(context);

    assertThat(context.checkClasses()).hasSize(8); // change it to 9, we added a new one!
    assertThat(context.testCheckClasses()).isEmpty();
  }
}
```

### Rules repository

With the actions taken above, your rule is activated, registered and should be ready to test.
But before doing so, you may want to customize the repository name your rule belongs to.

This repository's key and name are defined in `MyJavaRulesDefinition.java` and can be customized to suit your needs.
```java
public class MyJavaRulesDefinition implements RulesDefinition {
  // ...
  public static final String REPOSITORY_KEY = "fellowship-inc";

  public static final String REPOSITORY_NAME = "The Fellowship's custom rules";
  // ...
}
```

## Testing a custom plugin

>
> :exclamation: **Prerequisite**
> 
> For this chapter, you will need a local instance of SonarQube. If you don't have a SonarQube platform installed on your machine, now is time to download its latest version from [HERE](https://www.sonarqube.org/downloads/)!
>

此刻, 我们已经完成了第一个定制规则，并将其注册到自定义插件中了. 剩下的步骤就是直接用 SonarQube platform 测试它，并尝试分析一个工程! 

Start by building the project using maven. Note that here we are using the self-contained `pom` file targeting SonarQube `8.9` LTS. If you renamed it into `pom.xml`, remove the `-f pom_SQ_8_9_LTS.xml` part of the following command):


```
$ pwd
/home/gandalf/workspace/sonar-java/docs/java-custom-rules-example
  
$ mvn clean install -f pom_SQ_8_9_LTS.xml
[INFO] Scanning for projects...
[INFO]                                                                        
[INFO] ------------------------------------------------------------------------
[INFO] Building SonarQube Java :: Documentation :: Custom Rules Example 1.0.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
  
...
 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 8.762 s
[INFO] Finished at: 2021-03-02T12:17:28+01:00
[INFO] ------------------------------------------------------------------------
```

Then, grab the jar file `java-custom-rules-example-1.0.0-SNAPSHOT.jar` from the `target` folder of the project, and move it to the extensions folder of your SonarQube instance, which will be located at `$SONAR_HOME/extensions/plugins`.

>
> :exclamation: **SonarQube Java Plugin compatible version**
>
> Before going further, be sure to have the adequate version of the SonarQube Java Analyzer with your SonarQube instance. The dependency over the Java Analyzer of our custom plugin is defined in its `pom`, as seen in the first chapter of this tutorial. We consequently provide two distinct `pom` files mapping both the `7.9` previous LTS version of SonarQube, as well as the latest LTS release, version `8.9`.
>
> * If you are using a SonarQube `8.9` and updated to the latest LTS already, then you won't have the possibility to update the Java Analyzer independently anymore. Consequently, use the file `pom_SQ_8_9_LTS.xml` to build the project.
>

Now, (re-)start your SonarQube instance, log as admin and navigate to the ***Rules*** tab.

From there, under the language section, select "**Java**", and then "**The Fellowship's custom rules**" (or "**MyCompany Custom Repository**" if you did not change it) under the repository section. Your rule should now be visible (with all the other sample rules). 

![Selected rules](resources/rules_selected.png)

Once activated (not sure how? see [quality-profiles](https://docs.sonarqube.org/latest/instance-administration/quality-profiles/)), the only step remaining is to analyze one of your project!

When encountering a method returning the same type as its parameter, the issue will now raise issue, as visible in the following picture:

![Issues](resources/issues.png)

### How to define rule parameters

You have to add a `@RuleProperty` to your Rule.

Check this example: [SecurityAnnotationMandatoryRule.java](https://github.com/SonarSource/sonar-java/blob/master/docs/java-custom-rules-example/src/main/java/org/sonar/samples/java/checks/SecurityAnnotationMandatoryRule.java)

### How to test sources requiring external binaries

In the `pom.xml`, define in the `Maven Dependency Plugin` part all the JARs you need to run your Unit Tests. For example, if you sample code used in your Unit Tests is having a dependency on Spring, add it there.

See: [pom.xml#L137-L197](./java-custom-rules-example/pom_SQ_8_9_LTS.xml#L137-L197)

### How to test precise issue location

You can raise an issue on a given line, but you can also raise it at a specific Token. Because of that, you may want to specify, in your sample code used by your Unit Tests, the exact location, i.e. in between which 2 specific Columns, where you are expecting the issue to be raised.

This can be achieved using the special keywords `sc` (start-column) and `ec` (end-column) in the `// Noncompliant` comment. In the following example, we are expecting to have the issue being raised between the column 27 and 32 (i.e. exactly on "Order" variable type):

```java
public String updateOrder(Order order) { // Noncompliant [[sc=27;ec=32]] {{Don't use Order here because it's an @Entity}}
```

### How to test the Source Version in a rule

Starting from **Java Plugin API 3.7** (October 2015), the java source version can be accessed directly when writing custom rules. This can be achieved by simply calling the method `getJavaVersion()` from the context. Note that the method will return null only when the property is not set. Similarly, it is possible to specify to the verifier a version of Java to be considered as runtime execution, calling method `verify(String filename, JavaFileScanner check, int javaVersion)`.

```java
@Beta
public interface JavaFileScannerContext {

  // ...
  
  @Nullable
  Integer getJavaVersion();
  
}
```

## References

* [Analysis of Java code documentation](https://docs.sonarqube.org/latest/analysis/languages/java/)
* [SonarQube Platform](http://www.sonarqube.org/)
* [SonarSource Code Quality and Security for Java Github Repository](https://github.com/SonarSource/sonar-java)
* [SonarQube Java Custom-Rules Example](https://github.com/SonarSource/sonar-java/tree/master/docs/java-custom-rules-example)
