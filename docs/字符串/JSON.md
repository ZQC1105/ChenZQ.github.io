在 .NET 中，JSON 序列化是将 .NET 对象转换为 JSON 字符串，以及将 JSON 字符串转换为 .NET 对象的过程。.NET 提供了多种 JSON 序列化库，最常用的是 System.Text.Json 和 Newtonsoft.Json（也称为 Json.NET）。

以下是关于这两种主流库的简要介绍和使用示例：

## System.Text.Json（推荐，现代 .NET 的首选）

这是 .NET Core 3.0+ 内置的高性能 JSON 库，位于 System.Text.Json 命名空间中。它是跨平台的，性能优秀，且与 .NET 生态深度集成。
基本用法

```csharp
using System;
using System.Text.Json;

public class Person
{
public string Name { get; set; }
public int Age { get; set; }
}

class Program
{
static void Main()
{
// 序列化对象为 JSON
var person = new Person { Name = "Alice", Age = 30 };
string json = JsonSerializer.Serialize(person);
Console.WriteLine(json); // 输出: {"Name":"Alice","Age":30}

// 反序列化 JSON 为对象
string jsonInput = "{\"Name\":\"Bob\",\"Age\":25}";
Person person2 = JsonSerializer.Deserialize<Person>(jsonInput);
Console.WriteLine(person2.Name); // 输出: Bob
}
}
```
高级选项

你可以通过 JsonSerializerOptions 控制序列化行为：

```csharp
var options = new JsonSerializerOptions
{
PropertyNamingPolicy = JsonNamingPolicy.CamelCase, // 属性名转为 camelCase
WriteIndented = true // 格式化输出
};

string json = JsonSerializer.Serialize(person, options);
属性映射

使用 [JsonPropertyName] 特性指定 JSON 属性名：

csharp
public class Person
{
[JsonPropertyName("full_name")]
public string Name { get; set; }
}
```
## Newtonsoft.Json（Json.NET，广泛使用）

这是最流行的第三方 JSON 库，功能非常强大，支持更复杂的场景（如动态对象、LINQ to JSON 等）。
安装

通过 NuGet 安装：
bash
Install-Package Newtonsoft.Json
基本用法

```csharp
using Newtonsoft.Json;

// 序列化
string json = JsonConvert.SerializeObject(person);

// 反序列化
Person person = JsonConvert.DeserializeObject<Person>(json);
```
高级功能
支持 JsonProperty 特性
支持 JsonObject、JObject 等动态解析
更灵活的日期格式、空值处理等

```csharp
public class Person
{
[JsonProperty("full_name")]
public string Name { get; set; }
}
``
如何选择？

特性 System.Text.Json Newtonsoft.Json
------ --------------------- -------------------
性能 更快 稍慢
内置 .NET 3.0+ 需要 NuGet 包
功能丰富度 基础功能完善 功能更全面
兼容性 推荐新项目 旧项目或复杂需求
建议：新项目优先使用 System.Text.Json，除非需要 Newtonsoft.Json 的特定功能（如 ShouldSerializeXxx 方法、部分动态特性等）。

注意事项
确保类的属性有 public 的 getter 和 setter。
处理循环引用时需配置选项。
注意日期、枚举、空值的序列化行为。

如果你有具体的序列化需求（如日期格式、忽略属性、自定义转换器等），可以进一步说明，我可以提供更详细的代码示例。
