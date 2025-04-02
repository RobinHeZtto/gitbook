# Dart Analysis

## 1、相关概念

Dart Analysis是 Dart 语言的静态分析工具，它属于Dart SDK的一部分（代码在[Dart SDK](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fdart-lang%2Fsdk) `pkg/_fe_analyzer_shared` 和 `pkg/analyzer` 目录下），一般被用于检测和分析 Dart 代码的语法、类型、潜在问题和最佳实践等方面。

1. 静态分析：Dart Analysis 运用静态分析技术，通过分析代码的结构、类型注解和引用关系等，检查代码的合法性、语法错误、类型错误、潜在问题和最佳实践等方面。它可以在开发过程中自动检测问题，并提供实时的错误和警告提示，帮助开发者尽早发现和修复问题。
2. IDE 和编辑器支持：Dart Analysis 提供了对各种集成开发环境（IDE）和编辑器的支持，包括 Dart DevTools、DartPad、IntelliJ IDEA、Visual Studio Code 等。它与这些工具集成，提供了实时的代码分析、智能代码补全、代码导航、重构支持、错误提示等功能，提升了开发者的开发效率和代码质量。
3. 语义分析：Dart Analysis 不仅仅进行语法分析，还进行语义分析，以了解代码中的类型、类层次结构、继承关系、接口实现等信息。它可以根据这些信息进行更准确的类型检查、错误检测和代码建议。
4. Lint 规则：Dart Analysis 提供了一套默认的 Lint 规则集，用于检查和强制执行 Dart 代码中的最佳实践和代码风格。这些规则可以帮助开发者编写一致、易读、易维护的代码，并减少潜在的问题和错误。
5. 可配置性：Dart Analysis 允许开发者根据自己的需求进行配置和自定义。可以通过 `.analysis_options` 文件来配置 Lint 规则、代码风格、错误和警告级别等，以适应不同的项目和团队需求。

**AST**（**Abstract Syntax Tree** 抽象语法树）是通过对源代码进行语法分析而生成的一种树状结构，用来描述程序源代码语法结构。其中的每个节点代表源代码中的一个语法元素，例如表达式、语句、函数、类等，节点的属性和子节点表示了该语法元素的细节信息和嵌套结构。\


[AST在线预览网站](https://astexplorer.net/) 可以比较直观的查看AST结构（可惜暂时不支持Dart :joy:）



## 2、工作流程

Dart Analysis 的工作流程如下：

<figure><img src="../.gitbook/assets/image (262).png" alt=""><figcaption><p>工作流程图</p></figcaption></figure>

1. 词法分析（Lexical Analysis）：首先，Dart Analysis 对源代码进行词法分析。词法分析器将源代码分解为一系列的词法单元（Tokens），表示编程语言中的关键字、标识符、运算符、字面量等。词法分析器根据语言的语法规则将源代码划分为词法单元流。
2. 语法分析（Parsing）：在词法分析的基础上，Dart Analysis 进行语法分析。语法分析器将词法单元流转换为抽象语法树（Abstract Syntax Tree，AST）。AST 表示源代码的语法结构和语义信息。语法分析器根据语言的语法规则构建 AST，通过识别语法规则和组织词法单元。
3. 语义分析（Semantic Analysis）：在语法分析的基础上，Dart Analysis 进行语义分析。语义分析器对 AST 进行类型检查、符号解析、作用域分析等。它通过访问 AST 中的节点，收集类型信息、解析标识符、检查类型一致性等。语义分析器使用编程语言的语义规则和类型系统来验证代码的语义正确性。
4. Lint 规则检查：除了语法和语义分析外，Dart Analysis 还进行 Lint 规则的检查。Lint 规则是一组编码规范和最佳实践，用于检测代码中的潜在问题和风格问题。Dart Analysis 根据预定义的 Lint 规则集，检查源代码并生成相应的建议或警告。
5. 生成分析结果：在完成分析过程后，Dart Analysis 生成分析结果。分析结果包括有关源文件的错误、警告、建议等信息。这些分析结果可以用于代码编辑器的实时错误提示、代码导航、重构建议等，帮助开发者改善代码质量。



## 3、analyzer库的简单使用

除了使用 代码格式化工具 [dart format](https://github.com/dart-lang/dart_style)、代码文档生成工具[dart doc](https://github.com/dart-lang/dartdoc)、代码语法预分析服务[Dart Analysis Server](https://github.com/dart-lang/sdk/tree/main/pkg/analysis_server)工具外，我们也可直接使用 [analyzer](https://pub.dev/packages/analyzer) 库提供的api定制自己的功能。以将Dart源代码转换成AST json为例，看一下我们如何使用analyzer api。

首先，pubspec.yaml 中集成analyzer库

```yaml
dependencies:
  analyzer: ^6.0.0
```

然后通过提供的工具方法 `parseFile`解析指定的Dart代码文件，文件的路径作为`path`参数传值，返回`ParseStringResult`对象

<figure><img src="../.gitbook/assets/image (256).png" alt=""><figcaption><p><code>parseFile实现</code></p></figcaption></figure>

其中`ParseStringResult`的定义如下：

<figure><img src="../.gitbook/assets/image (226).png" alt=""><figcaption><p><code>ParseStringResult定义</code></p></figcaption></figure>

在`ParseStringResult`定义中可以看到，其成员`unit`类型为`CompilationUnit`，表示一个整个源文件对应的编译单元。其定义如下：

<figure><img src="../.gitbook/assets/image (227).png" alt=""><figcaption><p><code>CompilationUnit</code></p></figcaption></figure>

可以看到`CompilationUnit`实际继承于`AstNode` ，我们可以通过 `analyzer` 提供的 `AstVisitor`来访问该`CompilationUnit下的所有AstNode` ，如下:

<figure><img src="../.gitbook/assets/image (413).png" alt=""><figcaption><p>MyAstVisitor</p></figcaption></figure>

最后，在`main`中实现如下调用即可：\


<figure><img src="../.gitbook/assets/image (492).png" alt=""><figcaption></figcaption></figure>

## 4、analyzer库详解



常见的 AST 节点类型：

1. 程序结构相关：
   * CompilationUnit：表示整个编译单元，包含了整个源代码文件的结构。
   * LibraryDirective：表示库声明，用于指定当前文件的库名和导入的其他库。
   * ImportDirective：表示导入声明，用于导入其他库的功能。
   * ExportDirective：表示导出声明，用于将当前库中的内容导出给其他库使用。
2. 类和成员相关：
   * ClassDeclaration：表示类声明，包含类名、继承关系、成员列表等信息。
   * MethodDeclaration：表示方法声明，包含方法名、参数列表、返回类型、方法体等信息。
   * FieldDeclaration：表示字段声明，包含字段名、字段类型、初始值等信息。
3. 表达式和语句相关：
   * Expression：表示表达式，可以是算术表达式、逻辑表达式、函数调用等。
   * Statement：表示语句，例如赋值语句、条件语句、循环语句等。
4. 标识符和字面量相关：
   * SimpleIdentifier：表示简单标识符，用于表示变量名、函数名、类名等。
   * Literal：表示字面量，例如整数字面量、浮点数字面量、字符串字面量等。



常见的 AST 访问器类：

1. `AstVisitor`: `AstVisitor` 是一个抽象类，用作 AST 访问器的基类。它定义了一组 visit 方法，对应不同类型的 AST 节点，例如 `visitExpression`、`visitStatement`、`visitDeclaration` 等。可以根据需要继承 `AstVisitor` 并实现所需的 visit 方法来处理特定的节点类型。
2. `SimpleAstVisitor`: `SimpleAstVisitor` 是 `AstVisitor` 的子类，提供了一个简化的实现，其中每个 visit 方法默认为空实现。这个类适用于只关注特定节点类型的访问器，可以通过重写感兴趣的节点类型的 visit 方法来实现相应的操作。
3. `RecursiveAstVisitor`: `RecursiveAstVisitor` 是 `AstVisitor` 的子类，提供了递归访问 AST 结构的功能。它会自动遍历 AST 中的子节点，并调用相应的 visit 方法。通过继承 `RecursiveAstVisitor` 并实现 visit 方法，可以对整个 AST 树进行递归遍历和访问。
4. `ThrowingAstVisitor`: `ThrowingAstVisitor` 是 `AstVisitor` 的子类，它对每个 visit 方法都提供了默认的实现，但是在每个 visit 方法中都会抛出异常。这个类适用于快速检查 AST 结构是否包含未处理的节点类型，可以通过继承 `ThrowingAstVisitor` 并选择性地重写感兴趣的节点类型的 visit 方法来实现特定的操作。



|                                   |                                                                                                   |
| --------------------------------- | ------------------------------------------------------------------------------------------------- |
| visitCompilationUnit              | 访问编译单元（编译单元是指一个 Dart 程序的最顶层单位，通常对应于一个 Dart 文件。编译单元由一系列顶级声明组成，例如类声明、函数声明、变量声明等）                    |
| visitMapLiteralEntry              | 访问 Map 字面量                                                                                        |
| visitSetOrMapLiteral              | 访问 Set 或 Map 字面量                                                                                  |
| visitAnnotation                   | 访问注解                                                                                              |
| visitListLiteral                  | 访问列表字面量                                                                                           |
| visitInterpolationExpression      | 访问插值表达式（将变量或表达式的值插入到字符串字面量中，由 `${expression}` 形式的语法构成）                                            |
| visitStringInterpolation          | 访问字符串插值（`visitStringInterpolation` 用于访问整个字符串插值节点，而 `visitInterpolationExpression` 用于访问其中的插值表达式节点） |
| visitPostfixExpression            | 访问后缀表达式（常见的后缀表达式包括函数调用、成员访问和递增/递减操作符）                                                             |
| visitExpressionStatement          | 访问表达式语句                                                                                           |
| visitFieldDeclaration             | 访问字段声明（包含访问修饰符、类型、名称和初始值等信息）                                                                      |
| visitBlock                        | 访问代码块（花括号 `{}` 包围的语句集合）                                                                           |
| visitBlockFunctionBody            | 访问块函数体（`visitBlockFunctionBody` 用于访问函数体中的块函数体，而 `visitBlock` 用于访问任意的代码块）                          |
| visitVariableDeclaration          | 访问单个变量声明                                                                                          |
| visitVariableDeclarationStatement | 访问变量声明语句（变量声明语句用于声明一个或多个变量，并可选地包含初始值，通常包含在一个代码块或函数体中）                                             |
| visitVariableDeclarationList      | 访问变量声明列表                                                                                          |
| visitSimpleIdentifier             | 访问简单标识符（表示变量名、函数名、类名等）                                                                            |
| visitIntegerLiteral               | 访问整数字面量                                                                                           |
| visitDoubleLiteral                | 访问浮点数字面量                                                                                          |
| visitFunctionDeclaration          | 访问函数声明                                                                                            |
| visitFunctionDeclarationStatement | 访问函数声明语句                                                                                          |
| visitExpressionFunctionBody       | 访问表达式函数体（()=>方法）                                                                                  |
| visitFunctionExpression           |                                                                                                   |
| visitSimpleFormalParameter        |                                                                                                   |
| visitDefaultFormalParameter       |                                                                                                   |
| visitFormalParameterList          |                                                                                                   |
| visitNamedType                    |                                                                                                   |
| visitReturnStatement              |                                                                                                   |
|                                   |                                                                                                   |
|                                   |                                                                                                   |
|                                   |                                                                                                   |

