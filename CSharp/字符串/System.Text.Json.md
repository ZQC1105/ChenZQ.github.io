

## 日期格式 (Date Formatting)

默认情况下，System.Text.Json 会使用 ISO 8601 格式（如 "2025-10-31T16:46:00Z"）来序列化 DateTime 类型。但很多时候我们希望使用更易读或符合特定规范的格式。
方法一：通过 JsonSerializerOptions 全局设置

```csharp
using System;
using System.Text.Json;

var options = new JsonSerializerOptions
{
WriteIndented = true,
// 设置日期格式为 "yyyy-MM-dd HH:mm:ss"
Converters = { new DateTimeConverter() }
};

// 示例类
public class Event
{
public string Title { get; set; }
public DateTime OccurredAt { get; set; }
}

// 使用
var evt = new Event
{
Title = "会议",
OccurredAt = DateTime.Now
};

string json = JsonSerializer.Serialize(evt, options);
Console.WriteLine(json);
```
// 输出: {"Title":"会议","OccurredAt":"2025-10-31 16:46:00"}
方法二：创建自定义 DateTime 转换器

```csharp
using System;
using System.Text.Json;
using System.Text.Json.Serialization;

public class DateTimeConverter : JsonConverter<DateTime>
{
private const string Format = "yyyy-MM-dd HH:mm:ss";

public override DateTime Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
{
return DateTime.ParseExact(reader.GetString(), Format, null);
}

public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options)
{
writer.WriteStringValue(value.ToString(Format));
}
}
```
⚠️ 注意：如果 JSON 中的日期字符串格式不匹配，Read 方法会抛出异常。建议在生产环境中添加异常处理。

## 忽略属性 (Ignoring Properties)

有时我们不希望某些属性被序列化到 JSON 中，比如敏感信息或临时状态。
方法一：使用 [JsonIgnore] 特性

```csharp
using System.Text.Json.Serialization;

public class User
{
public string Name { get; set; }

[JsonIgnore] // 序列化时忽略此属性
public string Password { get; set; }

public int Age { get; set; }
}

var user = new User { Name = "Alice", Password = "secret123", Age = 30 };
string json = JsonSerializer.Serialize(user);
Console.WriteLine(json);
```
// 输出: {"Name":"Alice","Age":30} —— Password 被忽略
方法二：条件性忽略（[JsonIgnore(Condition = ...)]）

使用 JsonIgnoreCondition 可以根据值是否为 null、默认值或相等来决定是否忽略。

```csharp
using System.Text.Json.Serialization;

public class Product
{
public string Name { get; set; }

[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
public string Description { get; set; }

[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]
public int? Stock { get; set; } // null 时忽略
}
```
可用条件：
Always：总是忽略
Never：从不忽略
WhenWritingNull：当值为 null 时忽略
WhenWritingDefault：当值为默认值（如 0, null, false）时忽略
方法三：全局忽略 null 值

```csharp
var options = new JsonSerializerOptions
{
DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};
```
这会全局忽略所有 null 值的属性。

3. 自定义转换器 (Custom Converter)

当默认的序列化行为无法满足需求时（如枚举转字符串、特殊类型处理、兼容旧格式等），可以编写自定义转换器。
示例：将枚举序列化为字符串而不是数字

默认情况下，枚举会被序列化为整数。我们希望它输出为字符串名称。
步骤 1：定义枚举

```csharp
public enum Status
{
Pending,
Approved,
Rejected
}

public class Order
{
public string Id { get; set; }
public Status Status { get; set; }
}
步骤 2：创建自定义转换器


using System;
using System.Text.Json;
using System.Text.Json.Serialization;

public class EnumToStringConverter<T> : JsonConverter<T> where T : struct, Enum
{
public override T Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
{
var value = reader.GetString();
return (T)Enum.Parse(typeof(T), value, true); // 忽略大小写
}

public override void Write(Utf8JsonWriter writer, T value, JsonSerializerOptions options)
{
writer.WriteStringValue(value.ToString());
}
}
```
步骤 3：使用转换器

方式 A：在属性上使用特性

```csharp
public class Order
{
public string Id { get; set; }

[JsonConverter(typeof(EnumToStringConverter<Status>))]
public Status Status { get; set; }
}

方式 B：在 JsonSerializerOptions 中注册

csharp
var options = new JsonSerializerOptions();
options.Converters.Add(new EnumToStringConverter<Status>());

var order = new Order { Id = "O001", Status = Status.Approved };
string json = JsonSerializer.Serialize(order, options);
Console.WriteLine(json);
// 输出: {"Id":"O001","Status":"Approved"}

更复杂的自定义转换器示例：处理 TimeSpan


public class TimeSpanConverter : JsonConverter<TimeSpan>
{
public override TimeSpan Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
{
return TimeSpan.Parse(reader.GetString());
}

public override void Write(Utf8JsonWriter writer, TimeSpan value, JsonSerializerOptions options)
{
writer.WriteStringValue(value.ToString(@"hh\:mm\:ss"));
}
}
```
然后在属性上使用 [JsonConverter(typeof(TimeSpanConverter))]。

总结

需求 推荐方式
------ ----------
日期格式 创建 JsonConverter<DateTime> 并注册到 options.Converters
忽略属性 使用 [JsonIgnore] 或 [JsonIgnore(Condition = ...)]
自定义逻辑 继承 JsonConverter<T> 实现 Read 和 Write 方法

这些技巧可以让你更灵活地控制 JSON 的输入输出，满足 API 兼容性、可读性或性能需求。如果你有具体的类型或格式需求，我可以提供更针对性的转换器代码。
