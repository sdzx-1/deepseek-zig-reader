嗯，我需要总结这个Zig代码中的主要函数流程。首先，我得仔细看一下代码的结构，了解各个函数的作用和它们之间的调用关系。看起来这是一个WGSL（WebGPU Shading Language）的解析器，用Zig语言写的。

首先，入口点应该是`translationUnit`函数，它负责整个解析过程。里面调用了`parameterizeTemplates`，可能用于处理模板参数。然后，通过循环调用`globalDirectiveRecoverable`来处理全局指令，比如启用扩展。接着，通过`expectGlobalDeclRecoverable`来解析全局声明，如结构体、函数、变量等。最后，将解析的节点信息保存到AST中。

`parameterizeTemplates`函数的作用是处理模板列表，将`<`和`>`转换为特定的模板标记，比如`.template_left`和`.template_right`。这里用了栈来跟踪未闭合的模板，处理嵌套的情况。

接下来是`globalDirective`函数，处理`enable`指令，比如启用f16扩展。`expectGlobalDecl`函数则处理各种全局声明，包括结构体、函数、常量等。这里通过多个函数如`structDecl`、`fnDecl`、`globalVar`等来分别处理不同类型的声明。

在解析结构体声明时，`structDecl`会处理结构体成员，而`structMember`则处理每个成员的属性和类型。对于函数声明`fnDecl`，处理参数列表、返回类型和函数体。变量声明`globalVar`和`globalOverrideDecl`处理变量和覆盖声明的不同类型。

在语句解析部分，`statement`函数处理各种语句，比如循环、条件、返回等。`block`函数处理代码块，`statementList`收集块内的所有语句。其他如`ifStatement`、`forStatement`处理具体的控制流结构。

表达式解析涉及多个函数，如`expression`处理整个表达式，分解为`unaryExpr`、`bitwiseExpr`、`relationalExpr`等。`primaryExpr`处理基本表达式，比如字面量、标识符、括号表达式等。`componentOrSwizzleSpecifier`处理成员访问和数组索引。

错误处理和恢复函数如`findNextGlobalDirective`、`findNextGlobalDecl`在遇到错误时跳过无效的标记，继续解析后续内容。辅助函数如`addNode`和`addExtra`用于构建AST节点和存储额外信息。

总的来说，代码结构清晰，每个函数负责解析特定的语法结构，通过递归下降的方式逐步构建抽象语法树。错误处理机制帮助在遇到语法错误时尽可能恢复，继续解析后续代码。需要注意的是模板处理、作用域嵌套以及各种声明和语句的解析流程。
================================================
### 主要函数流程总结

#### 1. **入口函数 `translationUnit`**
- **功能**：解析整个 WGSL 源文件，构建 AST（抽象语法树）。
- **流程**：
  1. 调用 `parameterizeTemplates` 处理模板语法（如将 `<` 和 `>` 转换为模板标记）。
  2. 循环解析全局指令（`globalDirectiveRecoverable`），例如 `enable f16`。
  3. 循环解析全局声明（`expectGlobalDeclRecoverable`），包括结构体、函数、变量等。
  4. 将解析结果存入 AST 的 `nodes` 和 `extra` 字段，最终生成根节点。

---

#### 2. **模板处理 `parameterizeTemplates`**
- **功能**：将 WGSL 中的 `<` 和 `>` 转换为模板标记（`.template_left` 和 `.template_right`）。
- **流程**：
  1. 遍历 Token 流，识别模板起始标记（如 `ident<`）。
  2. 使用栈（`discovered_tmpls`）跟踪未闭合的模板，处理嵌套和括号作用域。
  3. 遇到 `>>` 时，拆分为两个 `.template_right` 标记。

---

#### 3. **全局指令解析 `globalDirective`**
- **功能**：解析 `enable` 指令（如启用 `f16` 扩展）。
- **流程**：
  1. 匹配 `enable` 关键字。
  2. 验证扩展名称（如 `f16`），更新解析器的扩展配置（`extensions`）。
  3. 错误处理：无效扩展会触发语法错误。

---

#### 4. **全局声明解析 `expectGlobalDecl`**
- **功能**：解析全局声明（结构体、函数、变量等）。
- **流程**：
  1. 跳过多余的分号。
  2. 解析属性列表（`attributeList`）。
  3. 按优先级尝试匹配以下声明：
     - **结构体**：`structDecl` 处理结构体定义。
     - **函数**：`fnDecl` 处理函数声明（含参数、返回类型、函数体）。
     - **变量**：`globalVar` 处理全局变量（含类型、初始值）。
     - **类型别名**：`typeAliasDecl` 处理类型别名。
     - **常量**：`constDecl` 和 `letDecl` 处理常量和不可变变量。
     - **断言**：`constAssert` 处理编译时断言。

---

#### 5. **结构体解析 `structDecl`**
- **功能**：解析结构体定义。
- **流程**：
  1. 匹配 `struct` 关键字和结构体名称。
  2. 解析结构体成员（`structMember`），支持属性和类型标注。
  3. 检查成员列表非空，生成结构体节点。

---

#### 6. **函数解析 `fnDecl`**
- **功能**：解析函数声明。
- **流程**：
  1. 匹配 `fn` 关键字和函数名。
  2. 解析参数列表（`parameterList`）和返回类型（含属性）。
  3. 解析函数体（`block`），生成函数原型节点。

---

#### 7. **表达式解析 `expression`**
- **功能**：处理复杂表达式，包括运算符优先级和结合性。
- **流程**：
  1. 解析一元表达式（`unaryExpr`）。
  2. 按优先级处理位运算（`bitwiseExpr`）、关系运算（`expectRelationalExpr`）、逻辑短路（`expectShortCircuitExpr`）。
  3. 最终生成表达式树节点。

---

#### 8. **错误恢复机制**
- **函数**：`findNextGlobalDirective`、`findNextGlobalDecl`、`findNextStmt`。
- **功能**：在语法错误后跳过无效标记，恢复解析。
- **实现**：
  - 根据作用域层级（括号、花括号）跳过无关 Token。
  - 寻找下一个有效声明或语句的起始标记（如 `fn`、`struct`）。

---

#### 9. **AST 构建辅助函数**
- **`addNode`**：将节点添加到 `nodes` 列表，返回索引。
- **`addExtra`**：将复杂数据（如结构体成员列表）存入 `extra` 列表。
- **`listToSpan`**：将节点列表转换为 `span` 节点，记录起始和结束位置。

---

### 关键特点
- **递归下降解析**：通过函数递归匹配语法规则。
- **错误恢复**：跳过无效 Token，尽量继续解析。
- **模板处理**：显式处理嵌套模板和括号作用域。
- **模块化设计**：每个语法结构（如函数、变量）由独立函数处理，代码可读性高。