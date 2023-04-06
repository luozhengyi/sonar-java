Sonar Java 解读
==============

编译开发环境
-----------
* win7/master/7.18.0-SNAPSHOT : https://github.com/SonarSource/sonar-java
* maven 3.6.2
* JDK17
* cywin64
* mvn install -DskipTests
* ideaIC-2020.3.4
* [its Downloading](https://github.com/AdoptOpenJDK/openjdk8-binaries/releases/download/jdk8u265-b01/OpenJDK8U-jre_x86-32_windows_hotspot_8u265b01.zip)

一级目录结构图
-------------

```
├── docs                         // 参见该模块readme如何自定义checker // 重点
├── external-reports
├── its
├── java-checks                  // checker 实现 和 rule的test cases // 重点
├── java-checks-test-sources     // refer to README in this modlue
├── java-checks-testkit
├── java-frontend                // engine frontend // 重点
├── java-jsp               
├── java-surefire
├── java-symbolic-execution
├── jdt                          // 封装 eclipse jdt
└── sonar-java-plugin            // 生成 maven / gradle plugin ？
```

java-checks 结构
----------------
```shell
sonar-java/java-checks$ tree -L  3
```
```
.
├── pom.xml
├── src
│   ├── main
│   │   ├── java       // checker 实现
│   │   └── resources  // checker metadata
│   └── test
│       ├── files      // checker的测试用例
│       ├── java       // checker的单元测试
````


全局检索checker SwitchCaseWithoutBreakCheck
-------------------------------------------
```java
java-checks\src\main\java\org\sonar\java\checks\CheckList.java:     // list of all checkers
  625      SwitchCaseTooBigCheck.class,
  626:     SwitchCaseWithoutBreakCheck.class,
  627      SwitchCasesShouldBeCommaSeparatedCheck.class,
```
```java
java-checks\src\main\java\org\sonar\java\checks\SwitchCaseWithoutBreakCheck.java:   // checker实现
  42  @Rule(key = "S128")                                                                // 后续检索S128
  43: public class SwitchCaseWithoutBreakCheck extends IssuableSubscriptionVisitor {
  44    @Override
```
```java
java-checks\src\test\files\filters\SuppressWarningFilter.java:
  12   * - ObjectFinalizeCheck
  13:  * - SwitchCaseWithoutBreakCheck
  14   * - RedundantTypeCastCheck
```
```java
java-checks\src\test\java\org\sonar\java\checks\SwitchCaseWithoutBreakCheckTest.java: // automatic test
  24  
  25: class SwitchCaseWithoutBreakCheckTest {
  26  

  29      CheckVerifier.newVerifier()
  30:       .onFile("src/test/files/checks/SwitchCaseWithoutBreakCheck.java")  // checker test cases file
  31:       .withCheck(new SwitchCaseWithoutBreakCheck())
  32        .verifyIssues();
```
```java
java-checks\src\test\java\org\sonar\java\filters\SuppressWarningFilterTest.java:
  39  import org.sonar.java.checks.SuppressWarningsCheck;
  40: import org.sonar.java.checks.SwitchCaseWithoutBreakCheck;
  41  import org.sonar.java.checks.SynchronizedOverrideCheck;

  72        new ObjectFinalizeCheck(),
  73:       new SwitchCaseWithoutBreakCheck(),
  74        new RedundantTypeCastCheck(),
```

检索 S128
---------
```java
its\ruling\src\test\java\org\sonar\java\it\AutoScanTest.java:
  81    private static final Comparator<String> RULE_KEY_COMPARATOR = (k1, k2) -> Integer.compare(
  82:     // "S128" should be before "S1028"
  83      Integer.parseInt(k1.substring(1)),
```
```java
its\ruling\src\test\resources\autoscan\autoscan-diff-by-rules.json:
  92    {
  93:     "ruleKey": "S128",
  94      "hasTruePositives": true,
```
```java
java-checks\src\main\java\org\sonar\java\checks\SwitchCaseWithoutBreakCheck.java:
  41  
  42: @Rule(key = "S128")
  43  public class SwitchCaseWithoutBreakCheck extends IssuableSubscriptionVisitor {
```
```java
java-checks\src\main\java\org\sonar\java\filters\SuppressWarningFilter.java:
  59        .put("empty", SetUtils.immutableSetOf("java:S1116", "java:S108"))
  60:       .put("fallthrough", Collections.singleton("java:S128"))
  61        .put("finally", Collections.singleton("java:S1143"))
```
```java
java-checks\src\main\resources\org\sonar\l10n\java\rules\java\S128.json:
  15    "ruleSpecification": "RSPEC-128",
  16:   "sqKey": "S128",
  17    "scope": "All",
```
```java
java-checks\src\main\resources\org\sonar\l10n\java\rules\java\Sonar_way_profile.json:
  18      "S127",
  19:     "S128",
  20      "S131",
```
```java
java-checks\src\test\files\checks\SwitchCaseWithoutBreakCheck.java: // checker test cases file(同上)
  131  
  132: class S128 {
  133  
```
```java
java-checks\src\test\files\filters\SuppressWarningFilter.java:
  236  
  237: // fallthrough suppresses S128
  238  @SuppressWarnings("fallthrough") // WithIssue
```

jdt 使用细节
-----------

在 pom.xml 文件中检索 org.eclipse.jdt
```xml
its\plugin\projects\package-info-annotations\pom.xml:  // 不关心
  17      <dependency>
  18:       <groupId>org.eclipse.jdt</groupId>
  19:       <artifactId>org.eclipse.jdt.annotation</artifactId>
  20        <version>2.1.100</version>
```
```xml
java-checks-test-sources\pom.xml: // 不关心, 具体参见它的 README
  515      <dependency>
  516:       <groupId>org.eclipse.jdt</groupId>
  517:       <artifactId>org.eclipse.jdt.annotation</artifactId>
  518        <version>2.2.600</version>
```
```xml
java-jsp\pom.xml:  // 不关心
  31          <exclusion>
  32:           <groupId>org.eclipse.jdt</groupId>
  33            <artifactId>ecj</artifactId>
```
```xml
java-symbolic-execution\pom.xml: // 不关心
  121                  <artifactItem>
  122:                   <groupId>org.eclipse.jdt</groupId>
  123:                   <artifactId>org.eclipse.jdt.annotation</artifactId>
  124                    <version>2.1.100</version>
```
```xml
jdt\pom.xml: // 封装 eclipse jdt 成为 org.sonarsource.java.jdt, 然后仅供被java-frontend/.../model使用
  16        <dependency>
  17:         <groupId>org.eclipse.jdt</groupId>
  18:         <artifactId>org.eclipse.jdt.core</artifactId>
  19          <version>3.30.0</version>

  74      <dependency>
  75:       <groupId>org.eclipse.jdt</groupId>
  76:       <artifactId>org.eclipse.jdt.core</artifactId>
  77      </dependency>
```

jdt dependencies in pom.xml:
```java
 org.eclipse.jdt.core  // import: 只在 java-frontend/.../model 除了 ASTUtils.java
 org.eclipse.jdt.annotation
```