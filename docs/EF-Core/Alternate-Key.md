# `HasAlternateKey()` 作用详解

`HasAlternateKey()` 是 Entity Framework Core（EF Core）中用于**配置替代键（Alternate Key）** 的 Fluent API 方法。

---

## 🔑 什么是“替代键”（Alternate Key）？

在数据库设计中：

- **主键（Primary Key）**：唯一标识表中每一行，**必须唯一且非空**，一个表只能有一个。
- **替代键（Alternate Key）**：也是**唯一且非空**的列（或列组合），**可以作为主键的候选**，但当前未被选为主键。

> 💡 简单说：**替代键 = 唯一约束（Unique Constraint） + 非空（NOT NULL）**，并且 EF Core 会将其视为“可用于外键引用的键”。

---

## ✅ `HasAlternateKey()` 的作用

1. **在数据库中创建唯一约束（Unique Constraint）**
2. **标记该列为“可被其他表作为外键引用”的键**
3. **与普通唯一索引（`HasIndex().IsUnique()`）的关键区别：是否支持外键引用**

---

## 🆚 `HasAlternateKey()` vs `HasIndex().IsUnique()`

| 特性 | `HasAlternateKey()` | `HasIndex().IsUnique()` |
|------|---------------------|--------------------------|
| 数据库效果 | 创建 **唯一约束（Unique Constraint）** | 创建 **唯一索引（Unique Index）** |
| 是否隐式 `NOT NULL` | ✅ 是（所有参与列自动设为非空） | ❌ 否（需手动 `.IsRequired()`） |
| 能否被其他表用作外键目标 | ✅ **可以** | ❌ **不可以** |
| EF Core 中的语义 | 表示“这是一个候选键” | 仅用于查询优化或业务唯一性 |

> ⚠️ 在 SQL Server 中，唯一约束底层也是通过唯一索引实现的，但**逻辑含义不同**。

---

## 🧪 示例说明

### 场景：用户表有 `Id`（主键）和 `Email`（唯一，且可被引用）

```csharp
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } // 唯一，且非空
}

public class AuditLog
{
    public int Id { get; set; }
    public string UserEmail { get; set; } // 外键指向 User.Email
}
```
### ❌ 错误做法（用唯一索引）：

```csharp
modelBuilder.Entity<User>(e =>
{
    e.HasIndex(u => u.Email).IsUnique(); // 只是唯一索引
});

modelBuilder.Entity<AuditLog>(e =>
{
    e.HasOne<User>()
     .WithMany()
     .HasForeignKey(l => l.UserEmail)
     .HasPrincipalKey(u => u.Email); // ❌ 报错！Email 不是替代键
});
```

# EF Core 外键配置错误示例文档

## 问题描述

在 Entity Framework Core 中，尝试使用非主键属性作为外键关联时，需要将该属性配置为**替代键（Alternate Key）**。仅配置唯一索引（Unique Index）是不够的。

## ❌ 错误做法示例

以下代码会导致运行时错误，因为 `Email` 属性未被正确配置为替代键：

```csharp
// 配置 User 实体
modelBuilder.Entity<User>(e =>
{
    e.HasIndex(u => u.Email).IsUnique(); // ⚠️ 这只是唯一索引，不是替代键
});

// 配置 AuditLog 实体，尝试使用 Email 作为外键关联
modelBuilder.Entity<AuditLog>(e =>
{
    e.HasOne<User>()
     .WithMany()
     .HasForeignKey(l => l.UserEmail)
     .HasPrincipalKey(u => u.Email); // ❌ 报错！Email 不是替代键
});
```

## 错误信息

运行时会抛出类似以下异常：
```
System.InvalidOperationException: The property 'Email' cannot be used as a principal key 
because it is not a key property. Configure 'Email' as a key or use the 'HasPrincipalKey' 
method to specify the principal key.
```

## ✅ 正确做法

需要将 `Email` 配置为替代键，而不仅仅是唯一索引：

```csharp
modelBuilder.Entity<User>(e =>
{
    e.HasAlternateKey(u => u.Email); // ✅ 正确：配置为替代键
});

modelBuilder.Entity<AuditLog>(e =>
{
    e.HasOne<User>()
     .WithMany()
     .HasForeignKey(l => l.UserEmail)
     .HasPrincipalKey(u => u.Email); // ✅ 现在可以正常工作
});
```

## 总结

| 配置方式 | 用途 | 是否支持外键关联 |
|---------|------|----------------|
| `HasIndex().IsUnique()` | 唯一索引，用于查询优化 | ❌ 不支持 |
| `HasAlternateKey()` | 替代键，可用于外键关联 | ✅ 支持 |

**关键点**：在 EF Core 中，作为 `HasPrincipalKey()` 参数的属性必须是实体的主键或替代键。