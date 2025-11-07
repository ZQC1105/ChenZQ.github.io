# 第6章：测试策略与最佳实践

至此，我们已经探讨了从单元测试到集成测试的各种技术。然而，拥有强大的工具并不等于能写出高效的测试套件。本章将总结一套全面的测试策略，并提供一系列经过验证的最佳实践，帮助你构建一个既快速又可靠的测试体系。

---

## 6.1 构建金字塔式的测试策略

一个健康的测试体系应该遵循"**测试金字塔**"模型：

```
    |‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|
    |   端到端测试      |  (少量)
    | (E2E / UI Tests)  |
    |___________________|
          |     |
    |‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|
    |   集成测试        |  (中等数量)
    | (Integration Tests)|
    |___________________|
            |
    |‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|
    |   单元测试        |  (大量)
    |   (Unit Tests)    |
    |___________________|
```

### 金字塔结构详解

**底层：大量的单元测试** (占 ~70%)
- **目标**：验证单个类、方法的内部逻辑
- **技术**：使用 Moq Mock `DbContext` 和外部服务
- **优点**：速度快（毫秒级），定位问题精确，易于维护
- **示例**：测试 `ProductService.CalculateDiscount()` 方法是否根据规则正确计算折扣

**中层：适量的集成测试** (占 ~20%)
- **目标**：验证多个组件协同工作，特别是与数据库的交互
- **技术**：使用 In-Memory Database 或 SQLite 内存模式
- **优点**：能发现查询翻译、数据映射、约束冲突等问题
- **示例**：测试 `OrderService.PlaceOrderAsync()` 是否能正确地在 `Orders` 和 `OrderItems` 表中创建记录

**顶层：少量的端到端测试** (占 ~10%)
- **目标**：模拟真实用户场景，从 API 入口到数据库再到响应
- **技术**：使用 `WebApplicationFactory<T>` 启动一个真实的 Web 服务器实例进行测试
- **优点**：最接近生产环境，能发现系统集成问题
- **缺点**：速度慢，维护成本高，容易因环境问题而失败
- **示例**：通过 HTTP POST 请求 `/api/orders` 来下订单，并验证返回状态码和数据库状态

### 核心原则

**尽可能用低层的测试来覆盖功能**。不要因为可以写 E2E 测试就放弃单元测试。一个由大量慢速 E2E 测试组成的"冰山"会严重拖累开发效率。

---

## 6.2 最佳实践清单

遵循以下最佳实践，可以显著提升你的测试代码质量和可维护性。

### 6.2.1 命名约定：清晰表达意图

测试方法的名称应该像一句完整的句子，清晰地描述**被测场景、输入条件和预期结果**。

```csharp
// ❌ 模糊不清
[Fact]
public void Test1() { ... }

// ❌ 仍然不够好
[Fact]
public void CreateProductTest() { ... }

// ✅ 推荐：Given_When_Then (GWT) 模式
[Fact]
public async Task CreateProductAsync_WithValidNameAndPrice_CreatesProductAndReturnsIt()
{
    // ...
}

// ✅ 推荐：直接描述行为
[Fact]
public async Task GetActiveProductsAsync_ExcludesInactiveProductsFromResult()
{
    // ...
}
```

---

### 6.2.2 组织结构：AAA 模式

始终将测试方法分为三个清晰的部分：

**Arrange (准备)**：设置测试前提条件。创建对象、配置 Mock、准备测试数据。

**Act (执行)**：调用被测试的方法。

**Assert (断言)**：验证结果是否符合预期。

```csharp
[Fact]
public async Task UpdateProductAsync_WithValidIdAndNewName_ChangesProductName()
{
    // Arrange
    var mockContext = new Mock<AppDbContext>();
    var mockDbSet = new Mock<DbSet<Product>>();
    mockContext.Setup(c => c.Products).Returns(mockDbSet.Object);
    
    var service = new ProductService(mockContext.Object);
    var existingProduct = new Product { Id = 1, Name = "Old", Price = 10 };
    var newName = "New";
    
    // Act
    await service.UpdateProductAsync(existingProduct.Id, newName, existingProduct.Price);
    
    // Assert
    mockDbSet.Verify(m => m.Update(It.Is<Product>(p => p.Name == newName)), Times.Once());
}
```

---

### 6.2.3 测试独立性：避免污染

每个测试都应该是独立的、可重复的。它不应该依赖于其他测试的执行顺序或状态。

**解决方案**：
- 使用 **Per-Test 数据库**（为每个测试使用唯一名称的 In-Memory DB）
- 在测试开始前清理或重建数据库状态
- 避免使用静态变量存储共享状态

---

### 6.2.4 只测试公共契约

单元测试应只关注类的**公共接口（Public Methods）**。不要测试私有方法（Private Methods）的实现细节。

**为什么？**
- 实现细节是会变的。如果你的测试紧密耦合到私有方法，那么重构代码时就会导致大量测试失败，即使最终功能没有改变

**怎么做？**
- 通过测试公共方法的行为来间接验证私有逻辑
- 如果某个私有逻辑非常复杂，考虑将其提取到一个新的、可独立测试的服务类中

---

### 6.2.5 避免过度 Mocking

Mock 应该用于隔离真正的外部依赖（如数据库、HTTP 服务、文件系统）。不要为了 Mock 而 Mock。

```csharp
// 不要 Mock 一个简单的 DTO 或纯数据类
var mockProduct = new Mock<Product>(); // ❌ 没必要

// ✅ 正确做法：直接创建实体的实例作为测试数据
var product = new Product { Name = "Test", Price = 10 }; // ✅
```

---

### 6.2.6 保持测试的单一职责

一个测试方法应该只**验证一个明确的行为**。

```csharp
// ❌ 反例：一个测试做了太多事
[Fact]
public void TestProductService()
{
    // 测试创建
    // 测试更新
    // 测试删除
    // 测试查询
}

// ✅ 正例：拆分成多个小测试
[Fact] 
public async Task CreateProductAsync_CreatesNewRecord() { ... }

[Fact] 
public async Task UpdateProductAsync_UpdatesExistingRecord() { ... }

[Fact] 
public async Task DeleteProductAsync_RemovesRecord() { ... }
```

**这样做的好处**：当测试失败时，你能立即知道是哪个具体功能出了问题。

---

### 6.2.7 善用测试初始化和清理

利用 xUnit.net 的 `IClassFixture<T>` 或 `ICollectionFixture<T>` 来管理昂贵的资源（如共享的 SQLite 连接、大型测试数据集）。

对于需要在每个测试前后执行的代码，可以使用构造函数和 `IDisposable.Dispose()` 方法。

```csharp
public class MyTests : IDisposable
{
    private readonly AppDbContext _context;
    
    public MyTests()
    {
        // 所有测试开始前运行
        _context = CreateInMemoryContext("MyTests");
        SeedTestData();
    }
    
    public void Dispose()
    {
        // 所有测试结束后运行
        _context?.Dispose();
    }
    
    [Fact]
    public async Task Test1() { /* 使用 _context */ }
    
    [Fact]
    public async Task Test2() { /* 使用 _context */ }
}
```

---

### 6.2.8 监控测试覆盖率，但不迷信

使用工具（如 **Coverlet**）来测量代码覆盖率。目标是达到一个合理的水平（例如 **80%**），但不要盲目追求 100%。

**不需要测试的代码**：
- 自动生成的代码（如 EF Core 的 `OnModelCreating`）
- 简单的 Getter/Setter
- 明显的、无逻辑的 DTO 类

**重点测试的代码**：
- 复杂的业务逻辑
- 条件分支和循环
- 错误处理和异常路径

---

## 6.3 总结：选择正确的工具

最后，让我们回顾一下三种主要测试技术的适用场景：

| 场景                   | Mocking | In-Memory Database | SQLite       |
| ---------------------- | ------- | ------------------ | ------------ |
| **测试纯业务逻辑**     | ✅ 首选  | ❌ 不需要           | ❌ 不需要     |
| **快速验证 LINQ 查询** | ⚠️ 有限  | ✅ 首选             | ⚠️ 较慢但可以 |
| **验证完整 CRUD 流程** | ❌ 无法  | ✅ 非常适合         | ✅ 非常适合   |
| **测试数据库约束**     | ❌ 无法  | ❌ 无法             | ✅ 首选       |
| **持续集成 (CI)**      | ✅ 极佳  | ✅ 极佳             | ✅ 极佳       |
| **测试复杂事务**       | ⚠️ 困难  | ⚠️ 有限             | ✅ 首推       |

### 终极建议

**以 Mocking 为主，In-Memory 为辅，关键路径用 SQLite 加固**。

根据你的应用特点和风险点，灵活组合这些技术，构建一个高效、可靠的测试防护网。