## [HttpResponseMessage 类](https://learn.microsoft.com/zh-cn/dotnet/api/system.net.http.httpresponsemessage?view=net-9.0)

## [HttpStatusCode 枚举](https://learn.microsoft.com/zh-cn/dotnet/api/system.net.httpstatuscode?view=net-9.0)

`HttpResponseMessage.StatusCode` 是服务器返回的 **HTTP 状态码**，类型为枚举 `System.Net.HttpStatusCode`，告诉你请求“成功还是失败、哪种失败”。

### 1. 常用值一览（背住这几个就够）

| 枚举值 | 数字 | 含义 |
|--------|------|------|
| `OK` | 200 | 成功 |
| `Created` | 201 | 创建成功（POST） |
| `NoContent` | 204 | 成功但无返回体 |
| `BadRequest` | 400 | 请求参数错误 |
| `Unauthorized` | 401 | 未登录/令牌失效 |
| `Forbidden` | 403 | 登录了但无权限 |
| `NotFound` | 404 | 资源不存在 |
| `Conflict` | 409 | 冲突（如重复键） |
| `InternalServerError` | 500 | 服务器内部错误 |
| `BadGateway` | 502 | 网关/代理错误 |
| `ServiceUnavailable` | 503 | 服务器暂不可用 |

### 2. 代码里怎么用

```csharp
var resp = await client.PostAsync("/api/log", content);

// 方法 1：直接判断
if (resp.StatusCode == HttpStatusCode.OK) { ... }

// 方法 2：快速“成功”判定（2xx 都算）
resp.EnsureSuccessStatusCode();   // 非 2xx 会抛 HttpRequestException

// 方法 3：switch 精细化处理
switch (resp.StatusCode)
{
    case HttpStatusCode.OK:
        var dto = await resp.Content.ReadFromJsonAsync<LogDto>();
        break;
    case HttpStatusCode.BadRequest:
        var bad = await resp.Content.ReadAsStringAsync();
        MessageBox.Show($"参数错误：{bad}");
        break;
    case HttpStatusCode.Unauthorized:
        // 跳登录
        break;
    default:
        // 其他统一处理
        break;
}
```

### 3. 数字 ↔ 枚举互转

```csharp
HttpStatusCode code = resp.StatusCode;
int numeric = (int)code;               // 200
HttpStatusCode back = (HttpStatusCode)numeric;
```

### 4. 小结
- `StatusCode` 是 **第一手可判断依据**；  
- `EnsureSuccessStatusCode()` 图省事，但 **不会给你细节**；  
- 对 4xx/5xx 做 **分支处理**，才能给出友好提示。
下面把 `HttpResponseMessage.IsSuccessStatusCode` 补进去，并给出它与 `EnsureSuccessStatusCode()` 的核心区别，方便你“一眼选型”。

---

### 5. 更方便的“成功”判定——`IsSuccessStatusCode`

| 成员                        | 返回值 | 抛异常                      | 说明                                        |
| --------------------------- | ------ | --------------------------- | ------------------------------------------- |
| `IsSuccessStatusCode`       | `bool` | ❌                           | 只判断，不抛；**2xx 返回 true**，其余 false |
| `EnsureSuccessStatusCode()` | `void` | ✅（`HttpRequestException`） | 非 2xx 直接抛，**拿不到响应体**             |

```csharp
var resp = await client.PostAsync("/api/log", content);

// 写法 1：IsSuccessStatusCode —— 想自己掌控流程
if (resp.IsSuccessStatusCode)          // 2xx
{
    var dto = await resp.Content.ReadFromJsonAsync<LogDto>();
}
else                                   // 4xx / 5xx
{
    var err = await resp.Content.ReadAsStringAsync();
    _logger.LogError("服务端返回 {Status}：{Error}", resp.StatusCode, err);
}

// 写法 2：EnsureSuccessStatusCode —— 图省事，异常即失败
try
{
    resp.EnsureSuccessStatusCode();    // 2xx 直接过
    var dto = await resp.Content.ReadFromJsonAsync<LogDto>();
}
catch (HttpRequestException ex)        // 非 2xx
{
    // 这里已经读不到响应体，只能看 ex.Message
    _logger.LogError(ex, "请求失败");
}
```

---

### 小结（更新版）
1. 只想知道“成没成功”——用 `IsSuccessStatusCode`；  
2. 想“失败就抛”——用 `EnsureSuccessStatusCode()`，但**丢了响应体**；  
3. 要精细提示——`switch (resp.StatusCode)` 或 `if (IsSuccessStatusCode)` 后再分支。