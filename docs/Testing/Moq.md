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

# 第3章：单元测试实践——Mocking DbContext

本章将深入探讨如何使用 **Moq** 框架对 `DbContext` 及其相关的 `DbSet<T>` 进行 Mock，从而编写出高效、可靠的单元测试。我们的核心目标是**隔离业务逻辑**，确保测试只关注被测方法本身的正确性，而不受数据库连接、网络延迟或外部数据状态的影响。

---

## 3.1 设计可测试的代码：依赖注入 (DI) 是前提

在进行 Mock 之前，一个至关重要的前提是：**你的代码必须设计成易于测试**。这通常意味着要遵循**依赖注入（Dependency Injection, DI）**原则。

回顾我们在第2章中提到的 `ProductService`：

```csharp
public class ProductService
{
    private readonly AppDbContext _context;
    
    public ProductService(AppDbContext context) // 构造函数注入
    {
        _context = context;
    }
    
    // ... 方法
}
```

这种设计模式是单元测试的基础，因为：

- **可替换性**：在生产环境中，DI 容器会注入一个真实的 `AppDbContext` 实例
- **可 Mock 性**：在测试环境中，我们可以手动创建一个 `ProductService` 的实例，并传入一个 Mock 的 `AppDbContext` 对象

### ❌ 不可测试的设计

如果 `ProductService` 像下面这样直接创建 `DbContext`，那么它将变得几乎无法进行单元测试：

```csharp
public class BadProductService
{
    public async Task<Product> CreateProductAsync(string name, decimal price)
    {
        using var context = new AppDbContext(); // 直接 new，无法替换
        // ... 操作 context
        await context.SaveChangesAsync();
    }
}
```

**结论**：始终通过构造函数注入 `DbContext` 或其抽象（如仓储接口），这是编写可测试代码的第一步。

---

## 3.2 Mock DbSet：基础操作的模拟

`DbSet<T>` 是我们与实体集交互的主要入口。最常见的操作是 `Add`、`Update`、`Remove` 和查询（通过 LINQ）。我们需要学会如何 Mock 这些行为。

### 3.2.1 模拟查询操作 (IQueryable<T>)

这是最常见也最关键的场景。许多业务逻辑都基于查询数据库。

**问题**：`DbSet<T>` 实现了 `IQueryable<T>`。Moq 不能直接让一个 `Mock<DbSet<T>>` 表现出完整的 `IQueryable` 行为（例如支持 `.Where()`、`.Select()` 等 LINQ 操作）。

**解决方案**：我们需要创建一个支持 LINQ to Objects 的 `IQueryable<T>` 集合，并将其设置为 `DbSet<T>` 的返回值。

```csharp
[Fact]
public async Task GetActiveProductsAsync_ReturnsOnlyActiveProducts()
{
    // Arrange
    var mockContext = new Mock<AppDbContext>();
    
    // 1. 准备测试数据
    var testData = new List<Product>
    {
        new Product { Id = 1, Name = "Phone", IsActive = true },
        new Product { Id = 2, Name = "Laptop", IsActive = false }, // 非活跃
        new Product { Id = 3, Name = "Tablet", IsActive = true }
    }.AsQueryable(); // 转换为 IQueryable
    
    // 2. 创建 Mock 的 DbSet
    var mockDbSet = new Mock<DbSet<Product>>();
    
    // 3. 关键步骤：配置 mockDbSet 表现得像一个 IQueryable
    //    我们需要设置它的提供程序和表达式
    mockDbSet.As<IQueryable<Product>>().Setup(m => m.Provider).Returns(testData.Provider);
    mockDbSet.As<IQueryable<Product>>().Setup(m => m.Expression).Returns(testData.Expression);
    mockDbSet.As<IQueryable<Product>>().Setup(m => m.ElementType).Returns(testData.ElementType);
    mockDbSet.As<IQueryable<Product>>().Setup(m => m.GetEnumerator()).Returns(testData.GetEnumerator);
    
    // 4. 将 mockDbSet 设置到 mockContext
    mockContext.Setup(c => c.Products).Returns(mockDbSet.Object);
    
    // 5. 创建服务
    var service = new ProductService(mockContext.Object);
    
    // Act
    var result = await service.GetActiveProductsAsync();
    
    // Assert
    Assert.Equal(2, result.Count()); // 应该只有 2 个活跃产品
    Assert.All(result, p => Assert.True(p.IsActive)); // 断言所有结果都是活跃的
}
```

这个模式很繁琐。我们可以创建一个帮助类来简化它：

```csharp
// TestHelper.cs
public static class TestHelper
{
    public static Mock<DbSet<T>> CreateMockDbSet<T>(IQueryable<T> data) where T : class
    {
        var mockSet = new Mock<DbSet<T>>();
        mockSet.As<IQueryable<T>>().Setup(m => m.Provider).Returns(data.Provider);
        mockSet.As<IQueryable<T>>().Setup(m => m.Expression).Returns(data.Expression);
        mockSet.As<IQueryable<T>>().Setup(m => m.ElementType).Returns(data.ElementType);
        mockSet.As<IQueryable<T>>().Setup(m => m.GetEnumerator()).Returns(data.GetEnumerator());
        return mockSet;
    }
}

// 在测试中使用
var testData = new List<Product>{ /* ... */ }.AsQueryable();
var mockDbSet = TestHelper.CreateMockDbSet(testData);
mockContext.Setup(c => c.Products).Returns(mockDbSet.Object);
```

### 3.2.2 模拟 CUD 操作 (Create, Update, Delete)

对于 `Add`、`Update`、`Remove` 操作，我们通常不关心它们的"返回值"，而是关心它们是否被调用，以及被调用了多少次。这正是 `.Verify()` 方法的用武之地。

```csharp
[Fact]
public async Task DeleteProductAsync_ExistingId_CallsRemoveAndSaveChanges()
{
    // Arrange
    var mockContext = new Mock<AppDbContext>();
    var mockDbSet = new Mock<DbSet<Product>>();
    mockContext.Setup(c => c.Products).Returns(mockDbSet.Object);
    
    var service = new ProductService(mockContext.Object);
    var productId = 1;
    
    // Act
    await service.DeleteProductAsync(productId);
    
    // Assert
    // 验证 Remove 方法被调用了一次，且参数是正确的 Product 对象
    // 注意：这里我们不知道具体的 Product 实例，所以用 It.IsAny<Product>()
    mockDbSet.Verify(m => m.Remove(It.IsAny<Product>()), Times.Once());
    
    // 验证 SaveChangesAsync 被调用了一次
    mockContext.Verify(m => m.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once());
}

[Fact]
public async Task UpdateProductAsync_ValidInput_CallsUpdateAndSaveChanges()
{
    // Arrange
    var mockContext = new Mock<AppDbContext>();
    var mockDbSet = new Mock<DbSet<Product>>();
    mockContext.Setup(c => c.Products).Returns(mockDbSet.Object);
    
    var service = new ProductService(mockContext.Object);
    var productToUpdate = new Product { Id = 1, Name = "Old Name", Price = 100 };
    
    // Act
    await service.UpdateProductAsync(productToUpdate, "New Name", 200);
    
    // Assert
    mockDbSet.Verify(m => m.Update(It.IsAny<Product>()), Times.Once());
    mockContext.Verify(m => m.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once());
}
```

---

## 3.3 处理复杂的导航属性和关联关系

现实世界中的实体往往通过导航属性相互关联（一对多、多对多等）。Mock 这些关系的查询需要更精细的控制。

### 示例场景

有一个 `Order` 实体，它包含多个 `OrderItem`：

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public virtual ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public virtual Order Order { get; set; }
    public string ProductName { get; set; }
    public int Quantity { get; set; }
}
```

现在，我们要测试一个服务方法，它根据订单 ID 获取订单及其所有项目：

```csharp
public async Task<Order> GetOrderWithItemsAsync(int orderId)
{
    return await _context.Orders
        .Include(o => o.Items) // 包含 Items 导航属性
        .FirstOrDefaultAsync(o => o.Id == orderId);
}
```

为了测试这个方法，我们需要 Mock 两个 `DbSet`：`Orders` 和 `OrderItems`，并确保 `Include` 操作能正确地"连接"它们。

```csharp
[Fact]
public async Task GetOrderWithItemsAsync_ExistingOrder_ReturnsOrderWithItems()
{
    // Arrange
    var mockContext = new Mock<AppDbContext>();
    
    // 准备 Orders 数据
    var ordersData = new List<Order>
    {
        new Order { Id = 1, OrderDate = DateTime.Today }
        // 注意：这里 Items 集合是空的，但我们会通过 Include 来"填充"
    }.AsQueryable();
    
    // 准备 OrderItems 数据
    var orderItemsData = new List<OrderItem>
    {
        new OrderItem { Id = 1, OrderId = 1, ProductName = "Widget", Quantity = 2 },
        new OrderItem { Id = 2, OrderId = 1, ProductName = "Gadget", Quantity = 1 }
    }.AsQueryable();
    
    // 创建 Mock 的 DbSets
    var mockOrdersSet = TestHelper.CreateMockDbSet(ordersData);
    var mockOrderItemsSet = TestHelper.CreateMockDbSet(orderItemsData);
    
    // 设置 Context
    mockContext.Setup(c => c.Orders).Returns(mockOrdersSet.Object);
    mockContext.Setup(c => c.OrderItems).Returns(mockOrderItemsSet.Object);
    
    var service = new OrderService(mockContext.Object);
    
    // Act
    var result = await service.GetOrderWithItemsAsync(1);
    
    // Assert
    Assert.NotNull(result);
    Assert.Equal(1, result.Id);
    Assert.Equal(2, result.Items.Count); // 断言 Items 集合被正确填充
    Assert.Contains(result.Items, i => i.ProductName == "Widget");
}
```

在这个例子中，EF Core 的查询管道会处理 `Include` 语句。由于我们 Mock 的 `DbSet<Order>` 和 `DbSet<OrderItem>` 都提供了 `IQueryable`，Moq 允许 EF Core 的查询执行引擎去"执行"这个包含查询。虽然这不是真正的 SQL JOIN，但 Moq 提供的 `IQueryable` 支持基本的内存中连接操作，足以满足大多数测试需求。

---

## 3.4 验证方法调用与参数

`.Verify()` 方法非常强大，可以精确地验证方法是如何被调用的。

### 验证调用次数

```csharp
mockContext.Verify(m => m.SaveChangesAsync(), Times.Once());      // 恰好一次
mockContext.Verify(m => m.SaveChangesAsync(), Times.Exactly(2)); // 恰好两次
mockContext.Verify(m => m.SaveChangesAsync(), Times.AtLeastOnce()); // 至少一次
mockContext.Verify(m => m.SaveChangesAsync(), Times.Never());     // 从未被调用
```

### 验证参数

使用 `It.Is<T>()` 或 `It.IsAny<T>()`：

```csharp
// 验证 Remove 被调用时，传入的 Product 的 Id 是 5
mockDbSet.Verify(m => m.Remove(
    It.Is<Product>(p => p.Id == 5)),
    Times.Once());

// 验证 Add 被调用时，传入的 Product 的 Name 不为空
mockDbSet.Verify(m => m.Add(
    It.Is<Product>(p => !string.IsNullOrEmpty(p.Name))),
    Times.Once());
```

### 回调 (Callback)

如果你想在 Mock 方法被调用时执行一些自定义逻辑（例如，修改传入的对象），可以使用 `Callback`：

```csharp
Product capturedProduct = null;
mockDbSet.Setup(m => m.Add(It.IsAny<Product>()))
         .Callback<Product>(p => capturedProduct = p); // 捕获传入的 Product

// ... 执行操作 ...

// 断言被捕获的对象
Assert.NotNull(capturedProduct);
Assert.Equal("Expected Name", capturedProduct.Name);
```

---

## 3.5 单元测试的局限性总结

尽管 Mocking 功能强大，但它也有明显的**局限性**：

- **不验证 SQL**：你无法知道你的 LINQ 查询会被翻译成什么样的 SQL。一个低效的查询在单元测试中可能表现完美，但在生产环境中却成为性能瓶颈。
- **不验证数据库约束**：外键、唯一性等约束不会被强制执行，因此无法发现因违反约束而导致的 `DbUpdateException`。
- **不验证事务行为**：Mock 无法模拟真实的数据库事务隔离级别和并发控制。
- **过度 Mock**：如果 Mock 了太多细节，测试可能会变得脆弱，一旦内部实现改变（即使功能不变），测试就会失败。

**结论**：单元测试应该与集成测试结合使用。**单元测试确保逻辑正确，集成测试确保与数据库的集成正确**。