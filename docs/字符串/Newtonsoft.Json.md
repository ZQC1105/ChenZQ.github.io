 Newtonsoft.Json（Json.NET）库的详细讲解，涵盖 日期格式、忽略属性 和 自定义转换器（Custom Converter） 的用法。

## 日期格式 (Date Formatting)

在 Newtonsoft.Json 中，有多种方式可以控制 DateTime 类型的序列化格式。
方法一：使用 JsonSerializerSettings 全局设置（推荐）

```csharp
using Newtonsoft.Json;

var settings = new JsonSerializerSettings
{
Formatting = Formatting.Indented, // 美化输出
DateFormatString = "yyyy-MM-dd HH:mm:ss", // 自定义日期格式
DateTimeZoneHandling = DateTimeZoneHandling.Unspecified, // 保持原始时区信息
NullValueHandling = NullValueHandling.Ignore // 忽略 null 值
};

// 序列化
string json = JsonConvert.SerializeObject(yourObject, settings);
```
✅ 优点：全局统一，代码简洁。

方法二：使用 [JsonConverter] 特性（属性级别）

```csharp
using Newtonsoft.Json;
using Newtonsoft.Json.Converters;

public class Event
{
public string Name { get; set; }

[JsonConverter(typeof(DateFormatConverter), "dd/MM/yyyy")]
public DateTime StartDate { get; set; }
}
```
⚠️ 注意：DateFormatConverter 是 Newtonsoft.Json 内置的转换器，但仅适用于 DateTime，且不支持 DateTime?。

方法三：自定义 DateTime 转换器（最灵活）

```csharp
using Newtonsoft.Json;
using System;

public class CustomDateTimeConverter : JsonConverter<DateTime>
{
private const string Format = "yyyy年MM月dd日 HH:mm";

public override void WriteJson(JsonWriter writer, DateTime value, JsonSerializer serializer)
{
writer.WriteValue(value.ToString(Format));
}

public override DateTime ReadJson(JsonReader reader, Type objectType, DateTime existingValue, bool hasExistingValue, JsonSerializer serializer)
{
var value = reader.Value?.ToString();
return DateTime.ParseExact(value, Format, null);
}
}


public class Event
{
public string Name { get; set; }

[JsonConverter(typeof(CustomDateTimeConverter))]
public DateTime StartDate { get; set; }
}
```

## 忽略属性 (Ignoring Properties)
方法一：[JsonIgnore] —— 无条件忽略

csharp
public class User
{
public string Name { get; set; }

[JsonIgnore] // 永远不会序列化
public string Password { get; set; }

public int Age { get; set; }
}

方法二：[JsonProperty] + NullValueHandling

```csharp
public class Product
{
public string Name { get; set; }

[JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
public string Description { get; set; }

[JsonProperty(DefaultValueHandling = DefaultValueHandling.Ignore)]
public int Stock { get; set; } = 0;
}
NullValueHandling.Ignore：当值为 null 时忽略该属性。
DefaultValueHandling.Ignore：当值为默认值（如 0, false）时忽略。
```
方法三：条件性忽略方法（ShouldSerializeXxx）

```csharp
public class Person
{
public string Name { get; set; }
public string Email { get; set; }

// 只有当 Email 不为 null 时才序列化
public bool ShouldSerializeEmail()
{
return !string.IsNullOrEmpty(Email);
}
}
```
✅ 这是 Newtonsoft.Json 特有的强大功能，System.Text.Json 不支持。

方法四：使用 JsonProperty 控制序列化行为

```csharp
public class Order
{
[JsonProperty(Required = Required.Always)] // 反序列化时必须存在
public string OrderId { get; set; }

[JsonProperty(Required = Required.AllowNull)] // 允许为 null
public string Status { get; set; }
}
```
## 自定义转换器（Custom Converter）

Newtonsoft.Json 的自定义转换器非常灵活，适用于复杂类型、枚举、集合等。
示例：将枚举序列化为字符串名称
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
```
步骤 2：创建自定义转换器
```csharp
using Newtonsoft.Json;
using System;

public class EnumToStringConverter : JsonConverter
{
public override bool CanConvert(Type objectType)
{
return objectType.IsEnum;
}

public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)
{
writer.WriteValue(value.ToString());
}

public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
{
var value = reader.Value?.ToString();
return Enum.Parse(objectType, value, true); // 忽略大小写
}
}
```
⚠️ 注意：JsonConverter（无泛型）需要重写 CanConvert 方法。
步骤 3：使用方式

方式 A：在属性上使用

```csharp
public class Order
{
public string Id { get; set; }

[JsonConverter(typeof(EnumToStringConverter))]
public Status Status { get; set; }
}
```
方式 B：在 JsonSerializerSettings 中注册

```csharp
var settings = new JsonSerializerSettings();
settings.Converters.Add(new EnumToStringConverter());
settings.Formatting = Formatting.Indented;

string json = JsonConvert.SerializeObject(order, settings);
```
更复杂示例：处理 TimeSpan

```csharp
public class TimeSpanConverter : JsonConverter<TimeSpan>
{
public override void WriteJson(JsonWriter writer, TimeSpan value, JsonSerializer serializer)
{
writer.WriteValue(value.ToString(@"hh\:mm\:ss"));
}

public override TimeSpan ReadJson(JsonReader reader, Type objectType, TimeSpan existingValue, JsonSerializer serializer)
{
return TimeSpan.Parse(reader.Value.ToString());
}
}
```
总结：Newtonsoft.Json vs System.Text.Json

功能 Newtonsoft.Json System.Text.Json
------ ------------------- --------------------
自定义日期格式 DateFormatString 或 [JsonConverter] 必须自定义 JsonConverter<DateTime>
忽略属性 [JsonIgnore], [JsonProperty], ShouldSerializeXxx [JsonIgnore], DefaultIgnoreCondition
条件忽略 ✅ ShouldSerializeXxx() 方法 ❌ 不支持
自定义转换器 灵活，支持泛型和非泛型 仅支持泛型 JsonConverter<T>
性能 较慢 更快
功能丰富度 极其丰富 基础功能完善

推荐实践
如果你已经使用 Newtonsoft.Json，可以继续使用它，特别是需要 ShouldSerializeXxx 或复杂转换逻辑时。
如果是新项目，建议优先尝试 System.Text.Json，性能更好，集成更紧密。
混合使用也是可行的（例如在 ASP.NET Core 中全局用 System.Text.Json，局部用 Newtonsoft.Json 处理复杂类型）。

如果你有具体的类型或格式需求（如 Guid 转小写、decimal 格式化等），我可以提供对应的 Newtonsoft.Json 自定义转换器示例。
