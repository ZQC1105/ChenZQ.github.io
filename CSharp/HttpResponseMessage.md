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