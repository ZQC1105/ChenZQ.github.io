# 内存数据库提供程序：Microsoft.EntityFrameworkCore.InMemory

当我们需要进行集成测试时，理想的情况是使用一个**快速、轻量级、无需外部依赖**的数据库。这就是 **In-Memory Database Provider** 的用武之地。EF Core 官方提供了一个名为 `Microsoft.EntityFrameworkCore.InMemory` 的包，它允许我们将 EF Core 配置为使用一个完全驻留在内存中的数据库。

---

## 为什么使用 In-Memory Database 进行集成测试？

> **速度快**：所有数据操作都在内存中完成，没有磁盘 I/O 或网络延迟，测试执行非常迅速。

> **零外部依赖**：不需要安装 SQL Server、PostgreSQL 等数据库服务器。测试可以在任何环境中运行，包括 CI/CD 管道。

> **隔离性好**：每个测试都可以拥有自己独立的内存数据库实例，确保测试间互不影响。

> **接近真实**：它使用真正的 EF Core 查询管道，会执行 SQL 翻译（尽管是翻译成内存操作），因此可以发现查询逻辑错误、导航属性加载等问题。

---

## ⚠️ 重要注意事项与局限性

**并非 100% 真实**：In-Memory 数据库是一个简化版的数据库，**不强制执行真实的数据库约束**。

### 主要局限性：

- **外键约束 (Foreign Key Constraints)**：不会自动级联删除或阻止插入无效的外键
- **唯一约束 (Unique Constraints)**：不会阻止插入重复的值
- **索引**：没有真正的索引，因此无法测试索引相关的性能问题
- **SQL 特定功能**：某些特定于数据库的功能（如 SQL Server 的 `ROW_NUMBER()`）无法使用
- **数据持久性**：进程结束后，所有数据都会丢失
- **并发**：模拟多用户并发访问的能力有限

---

## 结论

In-Memory Database 是进行**大部分集成测试的绝佳选择**，尤其适合测试：
- CRUD 操作
- LINQ 查询的正确性
- 业务逻辑流程

但对于需要严格验证数据库约束、复杂事务或性能的场景，仍需考虑使用 **SQLite 或真实数据库**。

---

## 安装与基本使用

### 安装步骤

在你的测试项目中，通过 NuGet 安装包：

```bash
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

### 基本使用示例

在测试中配置 `DbContext` 使用 In-Memory 提供程序：

```csharp
// 在测试方法中
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase(databaseName: "TestDatabase") // 指定一个数据库名
    .Options;

using var context = new AppDbContext(options); // 使用配置好的选项创建 DbContext

// ... 执行你的测试操作 (Add, Save, Query, etc.)

// ... 进行断言

// using 语句确保 context 被释放，数据库被清理
```

---

## 数据库名的作用

`UseInMemoryDatabase` 使用**数据库名**来区分不同的内存数据库实例。如果两个 `DbContext` 实例使用相同的数据库名，它们将共享同一份内存数据。

> **最佳实践**：在测试中，通常为**每个测试或每个测试集合**使用唯一的数据库名，以保证隔离性。
>
> # 第4章：集成测试实践——使用 In-Memory Database

在上一章中，我们通过 Mocking 技术隔离了 `DbContext`，对业务逻辑进行了快速的单元测试。然而，这些测试无法验证 LINQ 查询是否能被正确翻译成 SQL、数据库约束是否生效等关键问题。本章将介绍如何使用 EF Core 的 **In-Memory Database Provider** 来进行集成测试，这是一种在真实（或近似真实）的数据库环境中验证数据访问代码的有效方法。

---

## 4.1 为什么选择 In-Memory Database 进行集成测试？

正如第2章所述，In-Memory Database 提供了一个完美的中间地带：

> **速度**：接近内存操作的速度，远快于连接真实的数据库服务器。

> **便利性**：无需安装和配置外部数据库，降低了开发和 CI/CD 环境的复杂性。

> **隔离性**：每个测试可以拥有独立的数据库实例，保证了测试的纯净和可重复性。

> **真实性**：它使用了完整的 EF Core 查询管道，会执行从 LINQ 到"内存查询"的翻译过程，能够发现许多 Mock 测试无法捕捉的问题。

虽然它不强制执行所有数据库约束，但对于大多数 CRUD 操作和查询逻辑的验证，它已经足够强大。

---

## 4.2 配置与初始化 In-Memory DbContext

进行集成测试的第一步是为测试创建一个配置好的 `DbContext` 实例。

### 步骤 1: 安装 NuGet 包

确保你的测试项目引用了 `Microsoft.EntityFrameworkCore.InMemory` 包。

### 步骤 2: 创建 DbContextOptions

这是最关键的一步。你需要创建一个 `DbContextOptionsBuilder` 并使用 `.UseInMemoryDatabase()` 方法。

```csharp
// TestBase.cs (可选的基础类)
public class TestBase
{
    protected AppDbContext CreateContext(string databaseName)
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: databaseName)
            .Options;
        return new AppDbContext(options);
    }
}

// 在具体的测试类中
public class ProductServiceIntegrationTests : TestBase
{
    [Fact]
    public async Task CreateProductAsync_ValidInput_PersistsToDatabase()
    {
        // Arrange: 为这个测试创建一个唯一的数据库名
        const string dbName = "CreateProductTestDb";
        using var context = CreateContext(dbName); // 使用 using 确保释放
        
        var service = new ProductService(context);
        
        // Act
        var product = await service.CreateProductAsync("New Phone", 999.99m);
        
        // Assert: 直接查询数据库验证
        var savedProduct = await context.Products.FindAsync(product.Id);
        Assert.NotNull(savedProduct);
        Assert.Equal("New Phone", savedProduct.Name);
        Assert.Equal(999.99m, savedProduct.Price);
    }
}
```

### 关键点解释

- **databaseName 参数**：这个名称用于区分不同的内存数据库。如果两个 `DbContext` 实例使用相同的名称，它们会共享同一份数据。因此，在自动化测试中，最好为**每个测试生成一个唯一的名称**（例如使用 `Guid.NewGuid().ToString()`），或者在测试结束后清理数据库。
- **using 语句**：`AppDbContext` 实现了 `IDisposable`。使用 `using` 可以确保在测试结束时，上下文被正确释放，相关的内存数据库资源也会被清理。这对于**防止内存泄漏和确保测试间隔离**至关重要。

---

## 4.3 编写基础的 CRUD 集成测试

现在，我们可以编写测试来验证真正的数据库操作。

### 4.3.1 Create (创建) 测试

```csharp
[Fact]
public async Task CreateProductAsync_ValidInput_IncrementsCountAndSetsId()
{
    // Arrange
    const string dbName = "CreateProductCountTest";
    using var context = CreateContext(dbName);
    var service = new ProductService(context);
    
    // 获取初始计数
    var initialCount = await context.Products.CountAsync();
    
    // Act
    await service.CreateProductAsync("Laptop", 1200.00m);
    
    // Assert
    var finalCount = await context.Products.CountAsync();
    Assert.Equal(initialCount + 1, finalCount);
    
    // 验证新实体的 Id 是否被设置（通常由数据库生成）
    var createdProduct = await context.Products.FirstAsync(p => p.Name == "Laptop");
    Assert.True(createdProduct.Id > 0); // 假设使用的是整数主键
}
```

### 4.3.2 Read (读取) 测试

```csharp
[Fact]
public async Task GetProductByIdAsync_ExistingId_ReturnsCorrectProduct()
{
    // Arrange
    const string dbName = "GetProductByIdTest";
    using var context = CreateContext(dbName);
    
    // 手动添加测试数据到数据库
    var testProduct = new Product { Name = "Mouse", Price = 25.00m };
    context.Products.Add(testProduct);
    await context.SaveChangesAsync(); // 必须先保存
    
    var service = new ProductService(context);
    
    // Act
    var result = await service.GetProductByIdAsync(testProduct.Id);
    
    // Assert
    Assert.NotNull(result);
    Assert.Equal(testProduct.Id, result.Id);
    Assert.Equal("Mouse", result.Name);
}

[Fact]
public async Task GetProductsByPriceRangeAsync_ValidRange_ReturnsFilteredProducts()
{
    // Arrange
    const string dbName = "GetProductsByPriceRangeTest";
    using var context = CreateContext(dbName);
    
    // 添加多条测试数据
    var products = new List<Product>
    {
        new() { Name = "Cheap Item", Price = 10.00m },
        new() { Name = "Mid Item", Price = 50.00m },
        new() { Name = "Expensive Item", Price = 200.00m }
    };
    context.Products.AddRange(products);
    await context.SaveChangesAsync();
    
    var service = new ProductService(context);
    
    // Act
    var results = await service.GetProductsByPriceRangeAsync(20, 100);
    
    // Assert
    Assert.Collection(results,
        item => Assert.Equal("Mid Item", item.Name)); // 应该只有 Mid Item 符合条件
    Assert.DoesNotContain(results, r => r.Name == "Cheap Item");
    Assert.DoesNotContain(results, r => r.Name == "Expensive Item");
}
```

### 4.3.3 Update (更新) 和 Delete (删除) 测试

```csharp
[Fact]
public async Task UpdateProductAsync_ValidChanges_UpdatesDatabaseRecord()
{
    // Arrange
    const string dbName = "UpdateProductTest";
    using var context = CreateContext(dbName);
    var product = new Product { Name = "Old Name", Price = 100.00m };
    context.Products.Add(product);
    await context.SaveChangesAsync();
    
    var service = new ProductService(context);
    
    // Act
    await service.UpdateProductAsync(product.Id, "New Name", 150.00m);
    
    // Assert
    var updatedProduct = await context.Products.FindAsync(product.Id);
    Assert.NotNull(updatedProduct);
    Assert.Equal("New Name", updatedProduct.Name);
    Assert.Equal(150.00m, updatedProduct.Price);
}

[Fact]
public async Task DeleteProductAsync_ExistingId_RemovesFromDatabase()
{
    // Arrange
    const string dbName = "DeleteProductTest";
    using var context = CreateContext(dbName);
    var product = new Product { Name = "To Be Deleted", Price = 1.00m };
    context.Products.Add(product);
    await context.SaveChangesAsync();
    
    var service = new ProductService(context);
    
    // Act
    await service.DeleteProductAsync(product.Id);
    
    // Assert
    var deletedProduct = await context.Products.FindAsync(product.Id);
    Assert.Null(deletedProduct); // 应该已被删除
}
```

---

## 4.4 处理关联关系的集成测试

In-Memory Database 能够很好地处理导航属性和关联关系的加载。

### 示例：加载订单及其项目

```csharp
[Fact]
public async Task GetOrderWithItemsAsync_ExistingOrder_LoadsItemsCollection()
{
    // Arrange
    const string dbName = "GetOrderWithItemsIntegrationTest";
    using var context = CreateContext(dbName);
    
    // 手动构建关联数据
    var order = new Order 
    {
        OrderDate = DateTime.Today,
        Items = new List<OrderItem>
        {
            new OrderItem { ProductName = "Widget", Quantity = 2 },
            new OrderItem { ProductName = "Gadget", Quantity = 1 }
        }
    };
    context.Orders.Add(order);
    await context.SaveChangesAsync(); // 保存时，EF Core 会处理外键
    
    var service = new OrderService(context);
    
    // Act
    var result = await service.GetOrderWithItemsAsync(order.Id);
    
    // Assert
    Assert.NotNull(result);
    Assert.Equal(2, result.Items.Count);
    Assert.Contains(result.Items, i => i.ProductName == "Widget");
    Assert.Contains(result.Items, i => i.ProductName == "Gadget");
}
```

在这个测试中，当我们调用 `context.SaveChangesAsync()` 时，EF Core 会自动为 `OrderItem` 实体设置正确的 `OrderId`。然后，当我们在服务中使用 `.Include(o => o.Items)` 查询时，In-Memory 提供程序能够正确地返回包含项目的完整订单对象。

---

## 4.5 测试数据管理策略

随着测试数量的增加，如何高效地准备和清理测试数据成为一个挑战。

### 4.5.1 每个测试独立数据库 (Per-Test Isolation)

这是我们目前采用的策略。为**每个测试使用一个唯一的数据库名**。

**优点**：
- 提供了最强的隔离性
- 测试完全独立，不会因数据污染而相互影响

**缺点**：
- 可能会占用更多内存（尽管很小，每个内存数据库的开销很低）

### 4.5.2 测试集合共享数据库 (Per-TestCollection)

xUnit.net 允许你定义 `IClassFixture<T>` 或 `ICollectionFixture<T>`。你可以创建一个共享的 `DbContext` 实例供同一个测试类或测试集合中的所有测试使用。

```csharp
// SharedDatabaseFixture.cs
public class SharedDatabaseFixture : IDisposable
{
    public AppDbContext Context { get; }
    
    public SharedDatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: "SharedTestDb")
            .Options;
        Context = new AppDbContext(options);
        
        // 可以在这里一次性 Seed 共享的测试数据
        SeedData();
    }
    
    private void SeedData()
    {
        if (!Context.Products.Any())
        {
            Context.Products.AddRange(new List<Product>
            {
                new() { Name = "Default Product 1", Price = 10.00m },
                new() { Name = "Default Product 2", Price = 20.00m }
            });
            Context.SaveChanges();
        }
    }
    
    public void Dispose() => Context?.Dispose();
}

// 在测试类中使用
public class ProductServiceSharedTests : IClassFixture<SharedDatabaseFixture>
{
    private readonly AppDbContext _context;
    
    public ProductServiceSharedTests(SharedDatabaseFixture fixture)
    {
        _context = fixture.Context;
        
        // 注意：在使用共享上下文时，需要在每个测试开始前清理数据，
        // 或者设计测试使其不互相干扰。
        ClearTestData(); 
    }
    
    private void ClearTestData()
    {
        // 删除本测试可能添加的特定数据
        var testProducts = _context.Products.Where(p => p.Name.StartsWith("Test_"));
        _context.Products.RemoveRange(testProducts);
        _context.SaveChanges();
    }
    
    [Fact]
    public async Task SomeTest()
    {
        // 使用 _context ...
    }
}
```

> **注意**：使用共享数据库时，必须非常小心地管理数据状态，避免测试间的污染。

### 4.5.3 使用种子数据 (Seed Data)

对于有大量固定参考数据的应用，可以在测试启动时一次性播种。EF Core 支持 `OnModelCreating` 中的 `HasData()` 方法，或者在测试的 `Setup` 阶段手动添加。

---

## 4.6 In-Memory Database 的局限性再审视

尽管 In-Memory Database 功能强大，但我们必须时刻牢记它的局限性：

### 无约束检查

如前所述，唯一约束、外键约束不会被强制执行。下面的测试在 In-Memory 中会意外地通过，但在真实数据库中会失败：

```csharp
[Fact]
public async Task InsertDuplicateUniqueKey_DoesNotThrow() // ❌ 错误的期望！
{
    // Arrange
    const string dbName = "DuplicateKeyTest";
    using var context = CreateContext(dbName);
    context.Products.Add(new Product { Name = "UniqueName", Price = 10 });
    await context.SaveChangesAsync();
    
    // Act & Assert
    context.Products.Add(new Product { Name = "UniqueName", Price = 20 }); // 重复的 Name
    // 下面这行在真实数据库中会抛出异常，但在 In-Memory 中不会！
    await context.SaveChangesAsync(); // 不会抛出 DbUpdateException
}
```

> **结论**：对于涉及约束的关键业务逻辑，必须使用 **SQLite 或真实数据库**进行补充测试。

### 有限的 SQL 功能

- 不支持存储过程、用户自定义函数、窗口函数等高级 SQL 特性
- 与真实数据库的行为可能存在微小差异

### 浮点数精度

内存中的浮点运算可能与特定数据库（如 SQL Server 的 DECIMAL/NUMERIC 类型）行为略有不同。