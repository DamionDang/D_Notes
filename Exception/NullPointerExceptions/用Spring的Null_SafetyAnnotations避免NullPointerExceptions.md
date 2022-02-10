# 使用 Spring 的 Null-Safety Annotations 保护您的代码免受 NullPointerExceptions 的影响

`NullPointerExceptions`（通常简称为“NPE”）是每个 Java 程序员的噩梦。

我们可以在互联网上找到大量解释如何编写Null-safety代码的文章。Null-safety确保我们在代码中添加了适当的检查，以**确保对象引用不能为空，或者在对象为空时采取可能的安全措施**。

由于`NullPointerException`是运行时异常，因此在代码编译期间很难找出这种情况。Java 的类型系统没有办法快速消除危险的空对象引用。

幸运的是，Spring Framework 提供了一些注释来解决这个确切的问题。在本文中，我们将学习如何使用这些注解来使用Spring Boot编写 null-safe代码。

## Spring中的Null-Safety Annotations

Spring核心包下`org.springframework.lang`，有4个这样的注解：

- `@NonNull`
- `@NonNullFields`
- `@Nullable`
- `@NonNullApi`

**Eclipse 和 IntelliJ IDEA 等流行的 IDE 可以理解这些注释。**他们可以在编译期间警告开发人员潜在的问题。

我们将在本教程中使用 IntelliJ IDEA。让我们通过一些代码示例了解更多信息。

要创建基础项目，我们可以使用Spring Initializr。我们只需要 Spring Boot 启动器，无需添加任何额外的依赖项。

## IDE Configuration

**请注意，并非所有开发工具都可以显示这些编译警告。如果您没有看到相关警告，请检查 IDE 中的编译器设置。**

### IntelliJ

对于 IntelliJ，我们可以在 'Build, Execution, Deployment -> Compiler' 下激活注解检查：

![IntelliJ 编译器配置](https://reflectoring.io/images/posts/spring-boot-null-safety-annotations/intellij-compiler-settings_hud188e08fdc0139feec7b91a21ff944e5_145688_1866x0_resize_box_3.png)

### Eclipse

对于 Eclipse，我们可以在 'Java -> Compiler -> Errors/Warnings' 下找到设置：

![Eclipse 编译器配置](https://reflectoring.io/images/posts/spring-boot-null-safety-annotations/eclipse-compiler-settings_hu8534a04dde40f239dc7f4772e1ef3337_850180_2084x0_resize_box_3.png)

## 示例代码

我们使用一个普通的`Employee`类来理解注解：

```java
package io.reflectoring.nullsafety;

// imports 

class Employee {
  String id;
  String name;
  LocalDate joiningDate;
  String pastEmployment;

  // standard constructor, getters, setters
}
```

## `@NonNull`

大多数情况下，该`id`字段（在`Employee`类中）将是一个不可为空的值。因此，为了避免任何潜在的可能性，`NullPointerException`我们可以将此字段标记为[`@NonNull`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/lang/NonNull.html)：

```java
class Employee {

  @NonNull 
  String id;

  //...
}
```

现在，如果我们不小心尝试`id`在代码中的任何位置将值设置为 null，IDE 将显示编译警告：

![NonNull 的 IDE 警告](https://reflectoring.io/images/posts/spring-boot-null-safety-annotations/nonnull-ide-warning_hu5348352c54dfc5689093e49bbc5394b7_33745_1138x0_resize_box_3.png)

**`@NonNull`注释可以在方法、参数或字段级别使用。**

此时，您可能会想“如果一个类有多个非空字段怎么办？”。如果我们必须`@NonNull`在每个之前添加注释，是不是太罗嗦了？

[`@NonNullFields`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/lang/NonNullFields.html)我们可以通过使用注解来解决这个问题。

`@NonNull`的快速摘要：

| 注释元素  | 影响                       |
| --------- | -------------------------- |
| field     | 当字段为空时显示警告       |
| parameter | 当参数为空时显示警告       |
| method    | 当方法返回 null 时显示警告 |
| package   | 不适用                     |

## `@NonNullFields`

让我们创建一个`package-info.java`文件以在包级别应用非空字段检查。`@NonNullFields`该文件将包含带有注释的根包名称：

```java
@NonNullFields
package io.reflectoring.nullsafety;

import org.springframework.lang.NonNullFields;
```

现在，**我们不再需要使用`@NonNull` annotation**来注释字段。因为默认情况下，该包中类的所有字段现在都被视为非空。而且，我们仍然会看到与以前相同的警告：![NonNullFields 的 IDE 警告](https://reflectoring.io/images/posts/spring-boot-null-safety-annotations/nonnull-ide-warning_hu5348352c54dfc5689093e49bbc5394b7_33745_1138x0_resize_box_3.png)

这里要注意的另一点是，如果有任何未初始化的字段，那么我们将看到初始化这些字段的警告：![NonNull 的 IDE 警告](https://reflectoring.io/images/posts/spring-boot-null-safety-annotations/nonnullfields-ide-warning_huf1e2dea76ed1bc462a15a89ca8946d6c_20352_612x0_resize_box_3.png)

`@NonNullFields`快速摘要：

| 注释元素  | 影响                                 |
| --------- | ------------------------------------ |
| field     | 不适用                               |
| parameter | 不适用                               |
| method    | 不适用                               |
| package   | 如果应用包的任何字段为空，则显示警告 |

## `@NonNullApi`

到现在为止，您可能已经发现了另一个要求，即对方法参数或返回值进行类似的检查。这里[`@NonNullApi`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/lang/NonNullApi.html)会来救我们。

与 类似`@NonNullFields`，我们可以使用`package-info.java`文件并`@NonNullApi`为预期的包添加注释：

```java
@NonNullApi
package io.reflectoring.nullsafety;

import org.springframework.lang.NonNullApi;
```

现在，如果我们在方法返回 null 的地方编写代码：

```java
package io.reflectoring.nullsafety;

// imports

class Employee {

  String getPastEmployment() {
    return null;
  }

  //...
}  
```

我们可以看到 IDE 现在警告我们不可为空的返回值：![NonNullApi 的 IDE 警告](https://reflectoring.io/images/posts/spring-boot-null-safety-annotations/nonnullapi-method-ide-warning_hu0f179a30c6c723f579ce7eeee2a50ca2_40298_1084x0_resize_box_3.png)

`@NonNullApi`的快速摘要：

| 注释元素  | 影响                                            |
| --------- | ----------------------------------------------- |
| field     | 不适用                                          |
| parameter | 不适用                                          |
| method    | 不适用                                          |
| package   | 如果应用包的任何参数或返回值为 null，则显示警告 |

## `@Nullable`

但这里有个问题。**在某些情况下，特定字段可能为空**（无论我们多么想避免它）。

例如，该`pastEmployment`字段在类中可以为空`Employee`（对于以前没有工作过的人）。但根据我们的安全检查，IDE 认为不可能。

`@Nullable`我们可以使用字段上的注释来表达我们的意图。这将告诉 IDE 该字段在某些情况下可以为空，因此无需触发警报。正如[JavaDoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/lang/Nullable.html)建议的那样：

”可用于关联`@NonNullApi`或`@NonNullFields`覆盖默认的不可为空语义为可空。“

与 类似`NonNull`，`Nullable`注解可以应用于方法、参数或字段级别。

我们现在可以将该`pastEmployment`字段标记为可为空：

```java
package io.reflectoring.nullsafety;

// imports

class Employee {

  @Nullable
  String pastEmployment;

  @Nullable String getPastEmployment() {
    return pastEmployment;
  }

  //...
}  
```

`@Nullable`的快速摘要：

| 注释元素  | 影响                 |
| --------- | -------------------- |
| field     | 表示该字段可以为空   |
| parameter | 表示参数可以为空     |
| method    | 表示方法可以返回null |
| package   | 不适用               |

## Automated Build检查

到目前为止，我们正在讨论现代 IDE 如何让编写null-safe代码变得更加容易。但是，如果我们想在构建管道中进行一些自动代码检查，这在某种程度上也是可行的。

[SpotBugs](https://spotbugs.github.io/)（著名但废弃的[FindBugs](http://findbugs.sourceforge.net/)项目的转世）提供了一个 Maven/Gradle 插件，可以检测代码由于可空性而导致一些问题。让我们看看如何使用它。

对于 Maven 项目，我们需要更新`pom.xml`以添加[SpotBugs Maven 插件](https://spotbugs.readthedocs.io/en/latest/maven.html)：

```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.5.2.0</version>
  <dependencies>
    <!-- overwrite dependency on spotbugs if you want to specify the version of spotbugs -->
    <dependency>
      <groupId>com.github.spotbugs</groupId>
      <artifactId>spotbugs</artifactId>
      <version>4.5.3</version>
    </dependency>
  </dependencies>
</plugin>
```

构建项目后，我们可以使用此插件中的以下目标：

- 目标分析`spotbugs`目标项目。
- `check`目标运行目标并在`spotbugs`发现任何错误时使构建失败。

如果您使用 Gradle 而不是 Maven，则可以在文件中配置[SpotBugs Gradle 插件](https://spotbugs.readthedocs.io/en/latest/gradle.html)`build.gradle`：

```groovy
dependencies {
  spotbugsPlugins 'com.h3xstream.findsecbugs:findsecbugs-plugin:1.11.0'
}

spotbugs {
  toolVersion = '4.5.3'
}
```

项目更新后，我们可以使用`gradle check`命令运行检查。

SpotBugs 提供了一些规则，通过`@NonNull`在 Maven 构建期间处理注释来标记潜在问题。您可以浏览详细的[错误描述](https://spotbugs.readthedocs.io/en/latest/bugDescriptions.html)列表。

例如，如果任何带有注释的方法`@NonNull`意外返回 null，则 SpotBugs 检查将失败并出现类似以下的错误：

```text
[ERROR] High: io.reflectoring.nullsafety.Employee.getJoiningDate() may return null, but is declared @Nonnull [io.reflectoring.nullsafety.Employee] At Employee.java:[line 36] NP_NONNULL_RETURN_VIOLATION
```

## 结论

这些注释确实是 Java 程序员减少`NullPointerException`运行时出现的可能性的福音。但是请记住，这并不能保证完全的零安全性。

[Kotlin](https://kotlinlang.org/docs/null-safety.html)使用这些注解来推断 Spring API 的可空性。

我希望你现在已经准备好在 Spring Boot 中编写 null 安全代码了！

