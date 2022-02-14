![An Introduction to Annotations and Annotation Processing in Java](https://github.com/DamionDang/D_Notes/blob/205b2250bd41ed4b3700f96176c1edb141cf3902/AnnotationsAndAnnotation/AnnotationsAndAnnotationInJava/image/image01.png)

# Java中的注解和注解处理简介

**注释**是与 Java 源代码元素（例如类、方法和变量）相关联的构造。注释在编译时或运行时向程序提供信息，程序可以根据这些信息采取进一步的行动。**注释处理器**在编译时或运行时处理这些注释，以提供代码生成、错误检查等功能。

该`java.lang`包提供了一些核心注解，还使我们能够创建可以用注解处理器处理的自定义注解。

在本文中，我们将讨论注释的主题，并通过一个真实的示例展示注释处理的强大功能。

## 注释基础

注释前面有`@`符号。注释的一些常见示例是`@Override`和`@SuppressWarnings`。`java.lang`这些是 Java 通过包提供的内置注解。我们可以进一步扩展核心功能以提供我们的自定义注释。

注释本身不执行任何操作。它只是提供可在编译时或运行时用于执行进一步处理的信息。

我们以`@Override`注解为例：

```java
public class ParentClass {
  public String getName() {...}
}

public class ChildClass extends ParentClass {
  @Override
  public String getname() {...}
}
```

我们使用`@Override`注解来标记存在于父类中但我们想在子类中覆盖的方法。上面的程序在编译时抛出一个错误，因为`getname()`in 的方法`ChildClass`被注释了，`@Override`即使它没有覆盖 from 的方法`ParentClass`（因为没有`getname()`方法 in `ParentClass`）。

通过在 中添加`@Override`注解`ChildClass`，编译器可以强制执行规则，即子类中的覆盖方法应该与父类中的方法具有相同的区分大小写的名称，因此程序会在编译时抛出错误，从而捕获一个即使在运行时也可能未被检测到的错误。

## 标准注释

以下是我们可用的一些最常见的注释。这些是 Java 作为`java.lang`包的一部分提供的标准注释。要查看它们的全部效果，最好从命令行运行代码片段，因为大多数 IDE 都提供了更改警告级别的自定义选项。

### `@SuppressWarnings`

我们可以使用`@SuppressWarnings`注解来指示代码编译的警告应该被忽略。我们可能想要抑制使构建输出混乱的警告。`@SuppressWarnings("unchecked")`，例如，抑制与原始类型相关的警告。

让我们看一个我们可能想要使用的例子`@SuppressWarnings`：

```java
public class SuppressWarningsDemo {

  public static void main(String[] args) {
    SuppressWarningsDemo swDemo = new SuppressWarningsDemo();
    swDemo.testSuppressWarning();
  }

  public void testSuppressWarning() {
    Map testMap = new HashMap();
    testMap.put(1, "Item_1");
    testMap.put(2, "Item_2");
    testMap.put(3, "Item_3");
  }
}
```

如果我们使用编译器开关从命令行运行该程序`-Xlint:unchecked`以接收完整的警告列表，我们会收到以下消息：

```text
javac -Xlint:unchecked ./com/reflectoring/SuppressWarningsDemo.java
Warning:
unchecked call to put(K,V) as a member of the raw type Map
```

上面的代码块是遗留 Java 代码（在 Java 5 之前）的示例，我们可能有集合，我们可能会在其中意外地存储混合类型的对象。为了引入编译时错误检查，引入了泛型。所以为了让这个遗留代码编译没有错误，我们会改变：

```java
Map testMap = new HashMap();
```

到

```java
Map<Integer, String> testMap = new HashMap<>();
```

如果我们有一个庞大的遗留代码库，我们不会想要进行大量代码更改，因为这将意味着大量的 QA 回归测试。因此，我们可能希望将`@SuppressWarning `注释添加到类中，这样日志就不会被多余的警告消息弄得一团糟。我们将添加如下代码：

```java
@SuppressWarnings({"rawtypes", "unchecked"})
public class SuppressWarningsDemo {
  ...
}
```

现在，如果我们编译程序，控制台就没有警告了。

### `@Deprecated`

我们可以使用`@Deprecated`注解来标记方法或类型已被更新的功能替换。

IDE 利用注解处理在编译时发出警告，通常用删除线指示已弃用的方法，以告诉开发人员他们不应再使用此方法或类型。

以下类声明了一个不推荐使用的方法：

```java
public class DeprecatedDemo {

  @Deprecated(since = "4.5", forRemoval = true)
  public void testLegacyFunction() {

    System.out.println("This is a legacy function");
  }
}
```

注释中的属性`since`告诉我们该元素在哪个版本中被弃用，并`forRemoval`指示该元素是否将在下一个版本中删除。

现在，如下调用遗留方法将触发编译时警告，指示需要替换方法调用：

```java
./com/reflectoring/DeprecatedDemoTest.java:8: warning: [removal] testLegacyFunction() in DeprecatedDemo has been deprecated and marked for removal
    demo.testLegacyFunction();
      ^           
1 warning
```

### `@Override`

我们已经看过`@Override`上面的注释。我们可以使用它来指示一个方法将覆盖父类中具有相同签名的方法。它用于在诸如字母大小写错误之类的情况下引发编译时错误，如以下代码示例所示：

```java
public class Employee {
  public void getEmployeeStatus(){
    System.out.println("This is the Base Employee class");
  }
}

public class Manager extends Employee {
  public void getemployeeStatus(){
    System.out.println("This is the Manager class");
  }
}
```

我们打算重写该`getEmployeeStatus()`方法，但我们拼错了方法名称。这可能会导致严重的错误。上面的程序将毫无问题地编译和运行，而不会捕获该错误。

如果我们将注解添加`@Override`到`getemployeeStatus()`方法中，我们会得到一个编译时错误，这会导致编译错误并迫使我们立即更正拼写错误：

```text
./com/reflectoring/Manager.java:5: error: method does not override or implement a method from a supertype
  @Override
  ^
1 error
```

### `@FunctionalInterface`

注解用于表示一个`@FunctionalInterface`接口不能有多个抽象方法。如果有多个抽象方法，编译器会抛出错误。Java 8 中引入了函数式接口，以实现 Lambda 表达式并确保它们不使用一种以上的方法。

即使没有`@FunctionalInterface`注解，如果您在接口中包含多个抽象方法，编译器也会抛出错误。`@FunctionalInterface`那么，如果它不是强制性的，我们为什么需要它呢？

让我们以下面的代码为例：

```java
@FunctionalInterface
interface Print {
  void printString(String testString);
}
```

如果我们`printString2()`向`Print`接口添加另一个方法，编译器或 IDE 将抛出错误，这将立即显而易见。

现在，如果`Print`接口在一个单独的模块中，并且没有`@FunctionalInterface`注释怎么办？该其他模块的开发人员可以轻松地向接口添加另一个功能并破坏您的代码。此外，现在我们必须弄清楚这两个函数中的哪一个是我们案例中的正确函数。通过添加`@FunctionalInterface`注释，我们会在 IDE 中立即收到警告，例如：

```text
Multiple non-overriding abstract methods found in interface com.reflectoring.Print
```

因此，最好始终包含`@FunctionalInterface`接口是否可以用作 Lambda。

### `@SafeVarargs`

可变参数功能允许创建具有可变参数的方法。在 Java 5 之前，创建带有可选参数的方法的唯一选择是创建多个方法，每个方法具有不同数量的参数。Varargs 允许我们创建一个方法来处理可选参数，语法如下：

```java
// we can do this:
void printStrings(String... stringList)

// instead of having to do this:
void printStrings(String string1, String string2)
```

但是，在参数中使用泛型时会引发警告。`@SafeVarargs`允许抑制这些警告：

```java
package com.reflectoring;

import java.util.Arrays;
import java.util.List;

public class SafeVarargsTest {

   private void printString(String test1, String test2) {
    System.out.println(test1);
    System.out.println(test2);
  }

  private void printStringVarargs(String... tests) {
    for (String test : tests) {
      System.out.println(test);
    }
  }

  private void printStringSafeVarargs(List<String>... testStringLists) {
    for (List<String> testStringList : testStringLists) {
      for (String testString : testStringList) {
        System.out.println(testString);
      }
    }
  }

  public static void main(String[] args) {
    SafeVarargsTest test = new SafeVarargsTest();

    test.printString("String1", "String2");
    test.printString("*******");

    test.printStringVarargs("String1", "String2");
    test.printString("*******");

    List<String> testStringList1 = Arrays.asList("One", "Two");
    List<String> testStringList2 = Arrays.asList("Three", "Four");

    test.printStringSafeVarargs(testStringList1, testStringList2);
  }
}
```

在上面的代码中，`printString()`也`printStringVarargs()`达到了同样的效果。然而，编译代码会产生一个警告，`printStringSafeVarargs()`因为它使用了泛型：

```text
javac -Xlint:unchecked ./com/reflectoring/SafeVarargsTest.java

./com/reflectoring/SafeVarargsTest.java:28: warning: [unchecked] Possible heap pollution from parameterized vararg type List<String>
  private void printStringSafeVarargs(List<String>... testStringLists) {
                            ^
./com/reflectoring/SafeVarargsTest.java:52: warning: [unchecked] unchecked generic array creation for varargs parameter of type List<String>[]
    test.printStringSafeVarargs(testStringList1, testStringList2);
                   ^
2 warnings
```

通过添加如下 SafeVarargs 注释，我们可以摆脱警告：

```java
@SafeVarargs
private void printStringSafeVarargs(List<String>... testStringLists) {
```

## 自定义注释

这些是为特定目的而定制的注释。我们可以自己创建它们。我们可以使用自定义注解

1. 减少重复，
2. 自动生成样板代码，
3. 在编译时捕获错误，例如潜在的空指针检查，
4. 基于自定义注释的存在自定义运行时行为。

自定义注释的一个例子是这个`@Company`注释：

```java
@Company{  
  name="ABC"
  city="XYZ"
}
public class CustomAnnotatedEmployee { 
  ... 
}
```

当创建`CustomAnnotatedEmployee`该类的多个实例时，所有实例都将包含同一个 company`name`和`city`，因此不再需要将该信息添加到构造函数中。

要创建自定义注解，我们需要使用`@interface`关键字声明它：

```java
public @interface Company{
}
```

要指定有关注解范围及其目标区域的信息，例如编译时间或运行时，我们需要将元注解添加到自定义注解中。

例如，要指定注解仅适用于类，我们需要添加`@Target(ElementType.TYPE)`，指定此注解仅适用于类，以及`@Retention(RetentionPolicy.RUNTIME)`，指定此注解必须在运行时可用。一旦我们运行这个基本示例，我们将讨论有关元注释的更多细节。

使用元注释，我们的注释看起来像这样：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Company{
}
```

接下来，我们需要将字段添加到自定义注释中。在这种情况下，我们需要`name`和`city`。所以我们添加如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Company{
	String name() default "ABC";
	String city() default "XYZ";
}
```

将它们放在一起，我们可以创建一个`CustomAnnotatedEmployee`类并将注释应用到它，如下所示：

```java
@Company
public class CustomAnnotatedEmployee {

  private int id;
  private String name;

  public CustomAnnotatedEmployee(int id, String name) {
    this.id = id;
    this.name = name;
  }

  public void getEmployeeDetails(){
    System.out.println("Employee Id: " + id);
    System.out.println("Employee Name: " + name);
  }
}
```

现在我们可以创建一个测试类来在运行时读取`@Company`注解：

```java
import java.lang.annotation.Annotation;

public class TestCustomAnnotatedEmployee {

  public static void main(String[] args) {

    CustomAnnotatedEmployee employee = new CustomAnnotatedEmployee(1, "John Doe");
    employee.getEmployeeDetails();

    Annotation companyAnnotation = employee
            .getClass()
            .getAnnotation(Company.class);
    Company company = (Company)companyAnnotation;

    System.out.println("Company Name: " + company.name());
    System.out.println("Company City: " + company.city());
  }
}
```

这将给出以下输出：

```text
Employee Id: 1
Employee Name: John Doe
Company Name: ABC
Company City: XYZ
```

因此，通过在运行时自省注解，我们可以访问所有员工的一些公共信息，并且在我们必须构造大量对象时避免大量重复。

## 元注释

元注释是应用于其他注释的注释，这些注释向编译器或运行时环境提供有关注释的信息。

元注释可以回答以下关于注释的问题：

1. 子类可以继承注解吗？
2. 注释是否需要显示在文档中？
3. 注释可以多次应用于同一个元素吗？
4. 注释适用于哪些特定元素，例如类、方法、字段等？
5. 注释是在编译时还是运行时处理的？

### `@Inherited`

默认情况下，注解不会从父类继承到子类。将`@Inherited`元注释应用于注释允许它被继承：

```java
@Inherited
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Company{
  String name() default "ABC";
  String city() default "XYZ";
}

@Company
public class CustomAnnotatedEmployee {

  private int id;
  private String name;

  public CustomAnnotatedEmployee(int id, String name) {
    this.id = id;
    this.name = name;
  }

  public void getEmployeeDetails(){
    System.out.println("Employee Id: " + id);
    System.out.println("Employee Name: " + name);
  }
}

public class CustomAnnotatedManager extends CustomAnnotatedEmployee{
  public CustomAnnotatedManager(int id, String name) {
    super(id, name);
  }
}
```

由于`CustomAnnotatedEmployee`具有`@Company`注解并`CustomAnnotatedManager`从其继承，因此`CustomAnnotatedManager`该类不需要包含它。

现在，如果我们为 Manager 类运行测试，我们仍然可以访问注解信息，即使 Manager 类没有注解：

```java
public class TestCustomAnnotatedManager {

  public static void main(String[] args) {
    CustomAnnotatedManager manager = new CustomAnnotatedManager(1, "John Doe");
    manager.getEmployeeDetails();

    Annotation companyAnnotation = manager
            .getClass()
            .getAnnotation(Company.class);
    Company company = (Company)companyAnnotation;

    System.out.println("Company Name: " + company.name());
    System.out.println("Company City: " + company.city());
  }
}
```

### `@Documented`

`@Documented`确保自定义注释出现在 JavaDocs 中。

通常，当我们在类上运行 JavaDoc 时，`CustomAnnotatedManager`注释信息不会显示在文档中。但是当我们使用`@Documented`注解时，它会：

```
@Inherited
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Company{
  String name() default "ABC";
  String city() default "XYZ";
}
```

### `@Repeatable`

`@Repeatable`允许在方法、类或字段上进行多个重复的自定义注释。要使用`@Repeatable`注解，我们需要将注解包装在一个容器类中，该类将其称为数组：

```java
@Target(ElementType.TYPE)
@Repeatable(RepeatableCompanies.class)
@Retention(RetentionPolicy.RUNTIME)
public @interface RepeatableCompany {
  String name() default "Name_1";
  String city() default "City_1";
}
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface RepeatableCompanies {
  RepeatableCompany[] value() default{};
}
```

我们声明我们的主类如下：

```java
@RepeatableCompany
@RepeatableCompany(name =  "Name_2", city = "City_2")
public class RepeatedAnnotatedEmployee {
}
```

如果我们对它进行如下测试：

```java
public class TestRepeatedAnnotation {

  public static void main(String[] args) {

    RepeatableCompany[] repeatableCompanies = RepeatedAnnotatedEmployee.class
            .getAnnotationsByType(RepeatableCompany.class);
    for (RepeatableCompany repeatableCompany : repeatableCompanies) {
      System.out.println("Name: " + repeatableCompany.name());
      System.out.println("City: " + repeatableCompany.city());
    }
  }
}
```

我们得到以下输出，它显示了多个`@RepeatableCompany`注释的值：

```text
Name: Name_1
City: City_1
Name: Name_2
City: City_2
```

### `@Target`

`@Target`指定可以在哪些元素上使用注释，例如在上面的示例中，注释`@Company`仅用于定义`TYPE`，因此只能应用于类。

让我们看看如果我们将`@Company`注解应用于方法会发生什么：

```java
@Company
public class Employee {

  @Company
  public void getEmployeeStatus(){
    System.out.println("This is the Base Employee class");
  }
}
```

如果我们将`@Company`注释应用于上述方法`getEmployeeStatus()`，我们会得到一个编译器错误说明：`'@Company' not applicable to method.`

各种不言自明的目标类型是：

- `ElementType.ANNOTATION_TYPE`
- `ElementType.CONSTRUCTOR`
- `ElementType.FIELD`
- `ElementType.LOCAL_VARIABLE`
- `ElementType.METHOD`
- `ElementType.PACKAGE`
- `ElementType.PARAMETER`
- `ElementType.TYPE`

### `@Retention`

`@Retention`指定何时丢弃注释。

- `SOURCE`- 注释在编译时使用并在运行时丢弃。
- `CLASS`- 注释在编译时存储在类文件中，并在运行时丢弃。
- `RUNTIME`- 注释在运行时保留。

如果我们需要一个注解来仅在编译时提供错误检查`@Override`，我们将使用`SOURCE`. 如果我们需要一个注解来在运行时提供功能，比如`@Test`在 Junit 中，我们会使用`RUNTIME`. 要查看真实示例，请在 3 个单独的文件中创建以下注释：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface ClassRetention {
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
public @interface SourceRetention {
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface RuntimeRetention {
}
```

现在创建一个使用所有 3 个注释的类：

```java
@SourceRetention
@RuntimeRetention
@ClassRetention
public class EmployeeRetentionAnnotation {
}
```

要验证在运行时是否只有运行时注解可用，请按如下方式运行测试：

```java
public class RetentionTest {

  public static void main(String[] args) {

    SourceRetention[] sourceRetention = new EmployeeRetentionAnnotation()
            .getClass()
            .getAnnotationsByType(SourceRetention.class);
    System.out.println("Source Retentions at runtime: " + sourceRetention.length);

    RuntimeRetention[] runtimeRetention = new EmployeeRetentionAnnotation()
            .getClass()
            .getAnnotationsByType(RuntimeRetention.class);
    System.out.println("Runtime Retentions at runtime: " + runtimeRetention.length);

    ClassRetention[] classRetention = new EmployeeRetentionAnnotation()
            .getClass()
            .getAnnotationsByType(ClassRetention.class);
    System.out.println("Class Retentions at runtime: " + classRetention.length);
  }
}
```

输出如下：

```text
Source Retentions at runtime: 0
Runtime Retentions at runtime: 1
Class Retentions at runtime: 0
```

所以我们验证了只有`RUNTIME`注解在运行时被处理。

## 注释类别

注释类别根据我们传递给它们的参数数量来区分注释。通过将注解分类为无参数、单值或多值，我们可以更轻松地思考和讨论注解。

### 标记注释

标记注释不包含任何成员或数据。我们可以`isAnnotationPresent()`在运行时使用该方法来确定标记注释的存在或不存在，并根据注释的存在做出决定。

例如，如果我们公司有几个客户具有不同的数据传输机制，我们可以在类中使用一个注解来指示数据传输的方法，如下所示：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface CSV {
}
```

客户端类可以使用如下注解：

```java
@CSV
public class XYZClient {
    ...
}
```

我们可以对注解进行如下处理：

```java
public class TestMarkerAnnotation {

  public static void main(String[] args) {

  XYZClient client = new XYZClient();
  Class clientClass = client.getClass();

    if (clientClass.isAnnotationPresent(CSV.class)){
        System.out.println("Write client data to CSV.");
    } else {
        System.out.println("Write client data to Excel file.");
    }
  }
}
```

根据`@CSV`注释是否存在，我们可以决定是将信息写入 CSV 文件还是 Excel 文件。上面的程序会产生这个输出：

```text
Write client data to CSV.
```

### 单值注释

单值注解只包含一个成员，参数是成员的值。必须命名单个成员`value`。

让我们创建一个`SingleValueAnnotationCompany`仅使用值字段作为名称的注释，如下所示：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface SingleValueAnnotationCompany {
  String value() default "ABC";
}
```

创建一个使用注释的类，如下所示：

```java
@SingleValueAnnotationCompany("XYZ")
public class SingleValueAnnotatedEmployee {

  private int id;
  private String name;

  public SingleValueAnnotatedEmployee(int id, String name) {
    this.id = id;
    this.name = name;
  }

  public void getEmployeeDetails(){
    System.out.println("Employee Id: " + id);
    System.out.println("Employee Name: " + name);
  }
}
```

运行如下测试：

```java
public class TestSingleValueAnnotatedEmployee {

  public static void main(String[] args) {
    SingleValueAnnotatedEmployee employee = new SingleValueAnnotatedEmployee(1, "John Doe");
    employee.getEmployeeDetails();

    Annotation companyAnnotation = employee
            .getClass()
            .getAnnotation(SingleValueAnnotationCompany.class);
    SingleValueAnnotationCompany company = (SingleValueAnnotationCompany)companyAnnotation;

    System.out.println("Company Name: " + company.value());
  }
}
```

单个值 'XYZ' 覆盖默认注释值，输出如下：

```text
Employee Id: 1
Employee Name: John Doe
Company Name: XYZ
```

### 完整注释

它们由多个名称值对组成。例如`Company(name="ABC", city="XYZ")`. 考虑我们最初的公司示例：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Company{
  String name() default "ABC";
  String city() default "XYZ";
}
```

让我们创建`MultiValueAnnotatedEmployee`如下的类。指定参数和值如下。默认值将被覆盖。

```java
@Company(name = "AAA", city = "ZZZ")
public class MultiValueAnnotatedEmployee {
  
}
```

运行如下测试：

```java
public class TestMultiValueAnnotatedEmployee {

  public static void main(String[] args) {

    MultiValueAnnotatedEmployee employee = new MultiValueAnnotatedEmployee();

    Annotation companyAnnotation = employee.getClass().getAnnotation(Company.class);
    Company company = (Company)companyAnnotation;

    System.out.println("Company Name: " + company.name());
    System.out.println("Company City: " + company.city());
  }
}
```

输出如下，并覆盖了默认注释值：

```text
Company Name: AAA
Company City: ZZZ
```

## 构建真实世界的注释处理器

对于我们真实世界的注解处理器示例，我们将对`@Test`JUnit 中的注解进行简单的模拟。通过使用`@Test`注释标记我们的函数，我们可以在运行时确定测试类中的哪些方法需要作为测试运行。

我们首先创建注解作为方法的标记注解：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) 
public @interface Test {
}
```

接下来，我们创建一个类`AnnotatedMethods`，我们将对`@Test`方法应用注释`test1()`。这将使该方法能够在运行时执行。该方法`test2()`没有注释，不应在运行时执行。

```java
public class AnnotatedMethods {

  @Test
  public void test1() {
    System.out.println("This is the first test");
  }

  public void test2() {
    System.out.println("This is the second test");
  }
}
```

现在我们创建测试来运行`AnnotatedMethods`该类：

```java
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

public class TestAnnotatedMethods {

  public static void main(String[] args) throws Exception {

    Class<AnnotatedMethods> annotatedMethodsClass = AnnotatedMethods.class;

    for (Method method : annotatedMethodsClass.getDeclaredMethods()) {

      Annotation annotation = method.getAnnotation(Test.class);
      Test test = (Test) annotation;

      // If the annotation is not null
      if (test != null) {

        try {
          method.invoke(annotatedMethodsClass
                  .getDeclaredConstructor()
                  .newInstance());
        } catch (Throwable ex) {
          System.out.println(ex.getCause());
        }

      }
    }
  }
}
```

通过调用`getDeclaredMethods()`，我们得到了我们`AnnotatedMethods`类的方法。然后，我们将遍历这些方法并检查每个方法是否带有`@Test`注解。最后，我们对标识为用 注释的方法执行运行时调用`@Test`。

我们想验证该`test1()`方法是否会运行，因为它被注释了`@Test`，并且`test2()`不会运行，因为它没有被注释`@Test`。

输出是：

```text
This is the first test
```

因此，我们验证了`test2()`没有` @Test`注释的 没有打印其输出。

## 结论

我们对注释进行了概述，然后是注释处理的简单真实示例。

我们可以进一步使用注释处理的强大功能来执行更复杂的自动化任务，例如在编译时为一组 POJO 创建构建器源文件。构建器是 Java 中的一种设计模式，用于在涉及大量参数或需要多个带有可选参数的构造器时为构造器提供更好的替代方案。如果我们有几十个 POJO，注释处理器的代码生成功能将通过在编译时创建相应的构建器文件为我们节省大量时间。

通过充分利用注释处理的力量，我们将能够跳过大量重复并节省大量时间。

> 部分文章、数据、图片来自互联网,一切版权均归原网站或原作者所有。
> 如果侵犯了你的权益请来信告知删除。
