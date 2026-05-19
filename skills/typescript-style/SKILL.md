---
name: typescript-style
description: |
  TypeScript 编码规范技能。当用户编写、审查、重构 TypeScript 代码，或询问 TypeScript 最佳实践、类型系统问题、代码风格时，**必须立即调用此技能进行辅助**。

  触发场景包括但不限于：
  - 编写新的 TypeScript 文件、函数、类、接口、type
  - 代码审查、review、重构
  - 询问泛型、泛型约束、条件类型、映射类型
  - 处理 async/await、Promise、回调
  - 定义或使用 interface、type、enum
  - 编写 JSDoc 注释
  - 使用 ESLint/Prettier 配置
  - 遇到类型错误需要帮助

  **重要：无论用户是否明确要求，只要涉及 TypeScript 代码，就必须主动调用此技能。**
---

# TypeScript 编码规范

**角色：** TypeScript 代码审查员。确保代码符合团队规范。

**本 skill 规则分为两类：**
- **硬性规则（MUST）** — 违反即阻止合并
- **建议规则（SHOULD）** — 强烈推荐，但不阻塞

**核心原则：** 可读性优先于技巧，简单优先于复杂，一致性优先于个人习惯。

---

## 一、项目配置

编写代码前必须了解项目已有的 ESLint + Prettier 配置，这些规则**已通过 CI 检查**：

### Prettier
```json
{
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 120
}
```
> 注意：`semi`（分号）依赖 TypeScript 默认行为，无需手动配置。

### ESLint
```javascript
{
  "@typescript-eslint/no-explicit-any": "off",  // 项目允许 any
  "@typescript-eslint/no-floating-promises": "warn",
  "@typescript-eslint/no-unsafe-argument": "warn",
  "prettier/prettier": ["error", { "endOfLine": "auto" }]
}
```

**本 skill 的硬性规则以此配置为基础，不与其冲突。**

---

## 二、硬性规则（MUST）

以下情况必须阻止并要求修复：

### 2.1 语法基础
- 禁止使用 `var`（使用 `const` / `let`）
- 禁止空 `catch`（必须处理或重新抛出异常）
- 禁止提交未通过 ESLint 检查的代码

### 2.2 类型系统
- **公共 API（export 的函数/类/接口）必须声明返回类型**
- 禁止使用包装类型：`new Boolean()`、`new String()`、`new Number()`
- 禁止使用大写原始类型：`Object`、`String`、`Number`、`Boolean`（用 `object`、`string`、`number`、`boolean`）

### 2.3 函数规范
- 参数不得超过 **4 个**（超出必须用配置对象）
- 禁止使用布尔参数控制行为
- `async` 函数内禁止使用 `.then().catch()` 链式调用（必须用 `try/catch`）

### 2.4 导入规范
- 禁止 `import * as xxx`（除非 from 无法拆分的类型声明文件）
- 禁止导入后未使用的模块
- 同一模块的 import 必须合并

### 2.5 命名规范
- 禁止在命名中包含类型信息（如 `userListArray`、`nameString`）
- 禁止使用 `_` 前后缀

### 2.6 代码结构
- 单个函数不得超过 **200 行**
- 嵌套层级不得超过 **4 层**
- 禁止循环依赖

### 2.7 变量使用
- 禁止定义未使用的局部变量。如果不确定未来是否会使用，可以暂时注释掉。
- 如果函数没有在文件内被调用，且不是预期被外部导入使用，请为其添加 `export` 关键字。如果确定无需导出，则删除该函数。
- 对于回调函数的参数（如 `(req, res, next)` 中的 `next`），如果未使用，请重命名为 `_`。
- 生成代码后，主动移除所有 `tsc` 报告为 “unused” 的声明。
---

## 三、建议规则（SHOULD）

以下规则强烈推荐，但不阻塞合并：

### 3.1 类型使用
- 优先使用 `interface`，其次 `type`（两者等价时选 `interface`）
- 简单类型优先使用类型推断，避免冗余声明
- 使用 `unknown` 替代 `any`（项目允许 `any`，但应节制）

### 3.2 函数规范
- 单一职责——每个函数只做一件事
- 使用卫语句（Early Exit）减少嵌套

### 3.3 对象与数据
- 优先使用浅层解构（不超过 2 层）
- 禁止直接修改入参，使用扩展运算符
- 优先使用不可变数据

### 3.4 空值处理
- 使用 `??` 而非 `||` 处理 null/undefined
- 非空断言 `!` 应尽量避免

### 3.5 注释规范
- 所有 `export` 的类型、函数、类必须有中文 JSDoc，需包含业务背景
- 注释说明"为什么"，而非"是什么"
- 复杂核心逻辑必须有注释

---

## 四、代码审查清单

完成代码后自检（硬性规则快速检查）：

- [ ] 无 `var`
- [ ] 无空 `catch`
- [ ] `export` 函数有返回类型
- [ ] 无 `new Boolean()`、`new String()` 等包装类型
- [ ] 无 `Object`、`String`、`Number`、`Boolean` 大写原始类型
- [ ] 参数不超过 4 个
- [ ] 无布尔参数
- [ ] async 函数内无 `.then()` 链
- [ ] 无 `import * as`
- [ ] 无未使用导入
- [ ] 命名无类型前缀
- [ ] 函数不超过 150 行
- [ ] 嵌套不超过 4 层
- [ ] 已通过 `npm run lint` 检查
- [ ] tsc编译无报错
