# 2.2 Mocking 框架：Moq

在进行单元测试时，我们需要将被测代码（通常是业务服务）与其依赖项（如 `DbContext`）**隔离开来**。这就是 **Mocking 框架**发挥作用的地方。

**Moq** 是 .NET 中最受欢迎、最成熟的 Mocking 框架。

## 什么是 Mocking？

Mocking 是一种技术，它创建一个对象的“替身”（Mock），这个替身可以**模拟真实对象的行为**。我们可以对这个 Mock 对象进行配置（`Setup`），指定当某个方法被调用时应该返回什么值，或者记录该方法是否被调用以及被调用了多少次。

## 为什么选择 Moq？

- **LINQ 风格的 API**：Moq 使用 LINQ 表达式树来设置期望和行为，语法直观且类型安全。
- **功能全面**：支持方法调用、属性、事件、回调、验证调用次数等几乎所有常见的 Mocking 场景。
- **易于使用**：API 设计简洁，学习曲线平缓。
- **与 xUnit.net 完美集成**：两者都是现代 .NET 开发的标准配置。

---

## 核心概念与示例

假设我们有一个 `ProductService`，它依赖于 `AppDbContext` 来操作 `Products` 表。

### 被测类：`ProductService.cs`

```csharp
// ProductService.cs
public class ProductService
{
    private readonly AppDbContext _context;

    public ProductService(AppDbContext context) // 通过构造函数注入
    {
        _context = context;
    }

    public async Task<Product> CreateProductAsync(string name, decimal price)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name is required.", nameof(name));
        if (price <= 0)
            throw new ArgumentException("Price must be greater than zero.", nameof(price));

        var product = new Product { Name = name, Price = price };
        _context.Products.Add(product);
        await _context.SaveChangesAsync(); // 关键点：这里会与数据库交互
        return product;
    }
}
```
现在，我们要对 CreateProductAsync 方法进行单元测试。

我们不希望这个测试真正去连接数据库（那会变成集成测试，速度慢且不稳定），而只想测试它的内部逻辑（参数验证、是否调用了 Add 和 SaveChangesAsync）。

测试类：ProductServiceTests.cs
```csharp
// ProductServiceTests.cs
using Moq;
using Xunit;

public class ProductServiceTests
{
    [Fact]
    public async Task CreateProductAsync_ValidInput_CallsAddAndSaveChanges()
    {
        // Arrange: 准备测试环境
        // 1. 创建 DbContext 的 Mock 对象
        var mockContext = new Mock<AppDbContext>();
        // 2. 创建 DbSet<Product> 的 Mock 对象
        var mockDbSet = new Mock<DbSet<Product>>();
        // 3. 将 mockDbSet 设置到 mockContext 的 Products 属性上
        mockContext.Setup(c => c.Products).Returns(mockDbSet.Object);
        // 4. 创建被测服务，并注入 Mock 的 DbContext
        var service = new ProductService(mockContext.Object);

        // Act: 执行被测操作
        await service.CreateProductAsync("New Phone", 999.99m);

        // Assert: 验证结果
        // 验证 Products DbSet 的 Add 方法是否被调用了一次
        mockDbSet.Verify(m => m.Add(It.IsAny<Product>()), Times.Once());
        // 验证 SaveChangesAsync 方法是否被调用了一次
        mockContext.Verify(m => m.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once());
    }

    [Fact]
    public async Task CreateProductAsync_EmptyName_ThrowsArgumentException()
    {
        // Arrange
        var mockContext = new Mock<AppDbContext>();
        var service = new ProductService(mockContext.Object);

        // Act & Assert
        // 验证传入空名称时是否抛出正确的异常
        await Assert.ThrowsAsync<ArgumentException>(
            () => service.CreateProductAsync("", 100));
    }
}
```
**关键点说明**  
- new Mock<T>() 创建了 T 类型的 Mock 实例。
- .Setup(...) 用于配置 Mock 对象的行为。在这里，我们配置了 mockContext.Products 属性的 getter，让它返回我们创建的 mockDbSet.Object。
- .Verify(...) 用于断言某个方法或属性是否以预期的方式被调用。这是单元测试中验证“交互”的关键。
**Moq 的局限性**
⚠️ Moq 只能 Mock 虚方法 (virtual) 或接口 (interface)。

EF Core 的 DbContext 和 DbSet<T> 的许多关键方法（如 SaveChangesAsync）都是虚方法，因此可以被 Moq 成功 Mock。

但如果要 Mock 的类或方法：

不是 virtual
来自密封类（sealed）
是静态方法
那么 Moq 就无能为力。

✅ 幸运的是，对于 EF Core 的场景，这通常不是问题。