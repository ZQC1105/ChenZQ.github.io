
## [Record](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/builtin-types/record)
`record` 是 C# 9 引入的一个 **上下文关键字**（contextual keyword），用于声明一种特殊的引用类型，称为 **记录类型（record type）**。它本质上是一种不可变的、值语义的类或结构体，主要用于表示简单的数据载体（data carrier），比如 DTO、配置对象、消息等。

---

## 🔍 什么是“上下文关键字”？

`record` 是 **上下文关键字**，这意味着：

- 它 **只在特定上下文中被视为关键字**（即声明类型时）。
- 在其他地方，你可以用 `record` 作为变量名、方法名等（不推荐，但合法）。

```csharp
int record = 42; // 合法，但不推荐
```

---

## ✅ 基本语法

```csharp
public record Person(string FirstName, string LastName);
```

这行代码等价于：

```csharp
public class Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }

    public Person(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
    }

    // 自动生成 Equals, GetHashCode, ToString, Deconstruct, Clone, PrintMembers 等
}
```

---

## 🧠 核心特性详解

| 特性 | 描述 |
|------|------|
| **不可变性** | 属性默认是 `init` 只读的，只能在构造或初始化器中设置。 |
| **值相等性** | 基于属性值比较，而不是引用地址。 |
| **自动解构** | 支持解构语法，如 `var (f, l) = person;` |
| **with 表达式** | 支持非破坏性修改：`var newPerson = person with { FirstName = "John" };` |
| **ToString()** | 自动生成，输出所有属性值，便于调试。 |
| **继承支持** | record 支持继承，但有一些限制（如不能继承 class）。 |

---

## 🧪 示例详解

### 1. 定义与使用

```csharp
public record Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = new Point(1, 2);

Console.WriteLine(p1 == p2); // True，值相等
Console.WriteLine(p1.ToString()); // Point { X = 1, Y = 2 }
```

### 2. 使用 `with` 表达式（非破坏性修改）

```csharp
var p3 = p1 with { X = 5 };
Console.WriteLine(p3); // Point { X = 5, Y = 2 }
```

### 3. 解构

```csharp
var (x, y) = p1;
Console.WriteLine($"x={x}, y={y}");
```

---

## 🧱 record class vs record struct

从 C# 10 开始，支持 `record struct`：

| 类型 | 描述 |
|------|------|
| `record` 或 `record class` | 引用类型，继承自 `System.Object` |
| `record struct` | 值类型，继承自 `System.ValueType` |

```csharp
public record struct Point(int X, int Y);
```

---

## ⚠️ 注意事项

- record 类型默认是 **不可变** 的，但你可以手动添加可变属性（不推荐）。
- record 类型支持继承，但 **record 只能继承自另一个 record**，不能继承自普通 class。
- record 类型 **不能被 class 继承**（只能被 record 继承）。

---

## ✅ 总结一句话

> `record` 是 C# 为不可变数据模型提供的语法糖，它让你用一行代码写出值语义、不可变、可解构、可比较、可克隆的“完美 DTO”。

---

如果你有具体的使用场景（比如：Web API 返回模型、DDD 值对象、消息队列模型等），我可以给你更贴合的示例。

你说得**非常准确**，这段话已经触及了 `record` 的**一个隐藏陷阱**：  
> **计算属性（computed property）在 `with` 表达式里不会“重新计算”**，除非你把它们写成**真正“每次访问都重新计算”**的形式。

---

### 🔍 举个例子说明问题

```csharp
public record Sale(decimal UnitPrice, int Quantity)
{
    // ❌ 错误示范：在构造阶段就“算好”总价
    public decimal Total { get; init; } = UnitPrice * Quantity;
}

var s1 = new Sale(10m, 2);
var s2 = s1 with { Quantity = 5 };

Console.WriteLine(s2.Total);   // 输出 20，而不是 50！
```

- `Total` 在 **构造 `s1` 时就算好了**，`with` 只复制字段/自动属性，**不会重新跑 `init` 逻辑**。
- 结果 `s2.Total` 仍然是旧值 **20**，显然**数据不一致**。

---

### ✅ 正确做法：让属性**每次访问时重新计算**

```csharp
public record Sale(decimal UnitPrice, int Quantity)
{
    // ✅ 真正的计算属性，无状态，每次都算
    public decimal Total => UnitPrice * Quantity;
}
```

现在：

```csharp
var s2 = s1 with { Quantity = 5 };
Console.WriteLine(s2.Total);   // 输出 50，正确
```

---

### ✅ 小结（官方文档原意浓缩）

| 场景 | 建议 |
|------|------|
| 属性值**依赖其他成员**，且希望 `with` 后**自动反映新值** | 用 **表达式体属性**（`=>`），不要 `init` 或 `set`。 |
| 属性必须在构造时**一次性求值**，且后续**不再变化** | 考虑改用 **class** 并自己管理克隆逻辑，或接受 `with` 不会重新求值。 |

---

### 🧠 一句话记住

> 在 `record` 里，**只有“每次访问都重新计算”的属性才能保证 `with` 复制后仍然正确**；  
> 一旦你把计算结果“固化”到字段或 `init` 属性里，`with` 就**不会帮你再算一遍**。

---

如果你有更复杂的计算逻辑（依赖外部服务、缓存等），我们可以一起设计一个**线程安全且可克隆的方案**。

是的，**`record` 就是 C# 最近几年里“最香”的语法糖之一**——

- 不可变、值语义、线程安全  
- 写 DTO、消息、配置、领域值对象时**几乎零样板代码**  
- `with` 表达式让你**一行代码完成“非破坏性修改”**，比手写 `Clone`+手动赋值舒服太多  

从 C# 9 到 12，微软还在持续给它加餐：

| 版本 | 新增能力 |
|------|----------|
| C# 10 | `record struct`（值类型 record） |
| C# 11 | `required` 成员、字段关键字 `file`、自动缓存的 `record` 主构造函数参数 |
| C# 12 | 主构造函数（`record` 语法下沉到 `class`/`struct`） |

一句话：**用了就回不去**。  
下次写 DTO，直接 `record` 起手，准没错！