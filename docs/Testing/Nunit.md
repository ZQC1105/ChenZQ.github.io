<!-- 请输出C# Nunit的功能介绍 -->
## Nunit 功能介绍
NUnit 是一个流行的开源单元测试框架，专为 .NET 平台设计。它提供了一套丰富的功能，帮助开发者编写、组织和运行单元测试，从而提高代码质量和可靠性。以下是 NUnit 的一些主要功能：
1. 丰富的断言库
NUnit 提供了多种断言方法，允许开发者验证代码行为是否符合预期。例如，Assert.AreEqual、Assert.IsTrue、Assert.Throws 等，覆盖了各种常见的测试场景。
```csharp
[Test]
public void TestAddition()
{
    int result = Calculator.Add(2, 3);
    Assert.AreEqual(5, result);
}
```
2. 灵活的测试组织
NUnit 支持通过属性（Attributes）来组织测试用例。开发者可以使用 [TestFixture] 来定义测试类，使用 [Test] 来标记测试方法，还可以使用 [SetUp] 和 [TearDown] 来进行测试前后的初始化和清理工作。
[SetUp]作用：在测试方法执行之前执行，通常用于初始化测试环境。
[TearDown] 作用：在测试方法执行之后执行，通常用于清理测试环境。

```csharp
[TestFixture]
public class CalculatorTests
{
    private Calculator calculator;

    [SetUp]
    public void Setup()
    {
        calculator = new Calculator();
    }

    [Test]
    public void TestAddition()
    {
        int result = calculator.Add(2, 3);
        Assert.AreEqual(5, result);
    }

    [TearDown]
    public void Teardown()
    {
        // 清理资源
    }
}
```

3. 参数化测试
NUnit 支持参数化测试，允许开发者使用不同的输入数据运行同一个测试方法。可以使用 [TestCase] 属性来定义多个测试用例。
```csharp
[Test]
[TestCase(2, 3, 5)]
[TestCase(-1, 1, 0)]
[TestCase(0, 0, 0)]
public void TestAddition(int a, int b, int expected)
{
    int result = calculator.Add(a, b);
    Assert.AreEqual(expected, result);
}
```

4. 并行测试执行
NUnit 支持并行执行测试用例，能够显著减少测试运行时间。开发者可以通过配置属性来启用并行测试。
```csharp
[TestFixture]
[Parallelizable(ParallelScope.All)]
public class CalculatorTests
{
    // 测试方法
}
```

5. 测试结果报告
NUnit 提供了多种格式的测试结果报告，方便开发者查看测试结果和分析失败原因。可以生成 XML、HTML 等格式的报告。
```bash
nunit3-console.exe MyTests.dll --result=TestResult.xml;format=nunit2
```
6. 集成支持
NUnit 可以与多种持续集成工具（如 Jenkins、Azure DevOps、GitHub Actions 等）集成，方便自动化测试流程。
```yaml
- name: Run NUnit Tests
  run: |
    dotnet test MyTests.csproj --logger "trx;LogFileName=test_results.trx"
```
7. 扩展性
NUnit 提供了丰富的扩展点，允许开发者创建自定义的断言、属性和测试运行器，以满足特定的测试需求。
```csharp
public class CustomAssert
{
    public static void IsPositive(int value)
    {
        Assert.IsTrue(value > 0, "Value should be positive");
    }
}
```

Assert的断言方法
在 NUnit 新的约束模型中，`Assert` 类提供了多种断言方法用于验证测试条件。以下是常用的断言方法分类和说明：
## Assert.Multiple
Assert.Multiple
NUnit 提供了一个名为 Assert.Multiple 的方法，允许在单个测试方法中运行多个断言。其作原理是，在测试方法中，使用 Assert.Multiple 方法包裹多个断言，然后使用 Assert.Multiple.End 方法结束。区别于单个断言，多个断言可以一起运行，如果其中某个断言失败，则整个测试方法会失败。

## 基本断言

- `Assert.That(actual, Is.EqualTo(expected))` - 验证两个值是否相等
- `Assert.That(actual, Is.Not.EqualTo(expected))` - 验证两个值是否不相等
- `Assert.That(actual, Is.SameAs(expected))` - 验证两个对象是否引用同一个实例
- `Assert.That(actual, Is.Not.SameAs(expected))` - 验证两个对象是否引用不同实例
- `Assert.That(object, Is.Null)` - 验证对象是否为 null
- `Assert.That(object, Is.Not.Null)` - 验证对象是否不为 null
- `Assert.That(condition, Is.True)` - 验证条件是否为 true
- `Assert.That(condition, Is.False)` - 验证条件是否为 false

## 数值比较断言

- `Assert.That(value, Is.GreaterThan(expected))` - 验证值是否大于期望值
- `Assert.That(value, Is.GreaterThanOrEqualTo(expected))` - 验证值是否大于等于期望值
- `Assert.That(value, Is.LessThan(expected))` - 验证值是否小于期望值
- `Assert.That(value, Is.LessThanOrEqualTo(expected))` - 验证值是否小于等于期望值
- `Assert.That(value, Is.InRange(min, max))` - 验证值是否在指定范围内

## 字符串断言

- `Assert.That(actual, Does.Contain(expected))` - 验证字符串是否包含指定子串
- `Assert.That(actual, Does.StartWith(expected))` - 验证字符串是否以指定前缀开始
- `Assert.That(actual, Does.EndWith(expected))` - 验证字符串是否以指定后缀结束
- `Assert.That(actual, Is.EqualTo(expected).IgnoreCase)` - 验证字符串是否相等（忽略大小写）
- `Assert.That(actual, Does.Not.Contain(unexpected))` - 验证字符串是否不包含指定子串

## 集合断言

- `Assert.That(collection, Is.EquivalentTo(expected))` - 验证两个集合是否包含相同元素
- `Assert.That(collection, Contains.Item(element))` - 验证集合是否包含指定元素
- `Assert.That(collection, Does.Not.Contain(element))` - 验证集合是否不包含指定元素
- `Assert.That(collection, Is.Empty)` - 验证集合是否为空
- `Assert.That(collection, Is.Not.Empty)` - 验证集合是否不为空
- `Assert.That(collection, Has.Count.EqualTo(expected))` - 验证集合元素数量

## 文件断言

- `Assert.That(file, Does.Exist)` - 验证文件是否存在
- `Assert.That(file, Does.Not.Exist)` - 验证文件是否不存在

## 异常断言

- `Assert.That(code, Throws.TypeOf<ExceptionType>())` - 验证代码是否抛出指定类型的异常
- `Assert.That(code, Throws.Nothing)` - 验证代码是否不抛出异常
- `Assert.That(code, Throws.Exception.With.Message.Contains(expected))` - 验证异常消息

## 逻辑组合断言

- `Assert.That(actual, Is.Not.EqualTo(unexpected))` - 逻辑非
- `Assert.That(actual, Is.GreaterThan(min).And.LessThan(max))` - 逻辑与
- `Assert.That(actual, Is.LessThan(min).Or.GreaterThan(max))` - 逻辑或

这些断言方法使用 NUnit 的约束模型，提供了更自然的语法和更好的错误消息，帮助开发者编写更具表达力的单元测试。


这些断言方法可以帮助开发者编写全面的单元测试，验证各种条件和场景。
总结
NUnit 是一个功能强大且灵活的单元测试框架，提供了丰富的功能来支持各种测试需求。通过使用 NUnit，开发者可以更轻松地编写高质量的测试用例，从而提升代码的可靠性和维护性。


