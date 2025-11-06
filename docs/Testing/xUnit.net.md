#  xUnit.net

在 .NET 平台，有三大主流的测试框架：**xUnit.net**、**MSTest** 和 **NUnit**。  
虽然三者都能胜任工作，但 **xUnit.net** 因其现代化的设计、简洁的 API 和强大的扩展性，已成为 **.NET Core/.NET 5+ 社区的事实标准**，也是我们推荐的选择。

## 为什么选择 xUnit.net？

### ✅ 设计理念先进
- **无静态状态**：与 MSTest 不同，xUnit.net 的 `TestCollection` 默认是隔离的。这意味着同一个类中的不同测试方法**不会共享一个实例**，从而避免了测试间的副作用。每个测试方法都会创建一个新的测试类实例。

### ✅ 数据驱动测试更优雅
- 通过 `[Theory]` 和 `[InlineData]`、`[MemberData]` 等特性，可以非常方便地编写参数化测试，用不同的输入数据运行同一个测试逻辑。

### ✅ 丰富的断言库
- xUnit.net 自带了一个强大且流畅的断言库（`Assert` 类），提供了比其他框架更细致的断言方法，例如：
  - `Assert.Contains`
  - `Assert.DoesNotContain`
  - `Assert.InRange`
  - `Assert.IsType`
- 它也完美兼容第三方断言库，如 **FluentAssertions**，后者能提供更具可读性的“流式”断言。

### ✅ 强大的扩展模型
- xUnit.net 拥有极佳的扩展性。你可以轻松创建自定义的特性（Attributes）、数据源、断言等。这对于构建复杂的测试夹具（Fixture）和共享上下文至关重要。

### ✅ 社区与生态
- 作为 **ASP.NET Core 官方项目使用的测试框架**，xUnit.net 拥有最活跃的社区支持和最丰富的文档、教程资源。

---

## 核心特性示例

```csharp
using Xunit;

public class CalculatorTests
{
    // 基本测试
    [Fact]
    public void Add_TwoNumbers_ReturnsSum()
    {
        var calculator = new Calculator();
        var result = calculator.Add(2, 3);
        Assert.Equal(5, result); // 断言相等
    }

    // 数据驱动测试 (Theory)
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    public void Add_MultipleScenarios_ReturnsCorrectSum(int a, int b, int expected)
    {
        var calculator = new Calculator();
        var result = calculator.Add(a, b);
        Assert.Equal(expected, result);
    }

    // 异常测试
    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        var calculator = new Calculator();
        Assert.Throws<DivideByZeroException>(() => calculator.Divide(5, 0));
    }
}
```
**关键特性说明**
- [Fact]：标记一个方法是一个测试事实（必须通过）。
- [Theory]：标记一个方法是一个理论，需要数据来验证。
- [InlineData(...)]：为 [Theory] 提供一行内联数据。