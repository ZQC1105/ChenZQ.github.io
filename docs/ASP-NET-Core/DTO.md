## 在 ASP.NET Core 项目里，DTO（Data Transfer Object）不是“可有可无的贫血类”，而是边界契约的核心。
---

1. 职责单一：DTO 只负责“跨进程/跨层搬运数据”，不含任何业务逻辑，因此天然防贫血领域模型。  
2. 隔离领域：通过 DTO↔Domain Entity 显式转换（手工或 AutoMapper/Mapster），把外部变化挡在应用层之外，领域层零依赖 MVC/WebAPI 契约。  
3. 防过度提交：Input DTO 与 Output DTO 分家，再配合 `record/init` 或 `[JsonIgnore]`，杜绝“用户多提交一个字段就把数据库覆盖” 的 Mass Assignment 漏洞。  
4. 版本兼容：给 API 升级时，新建 V2 DTO 并保留 V1 DTO，Controller 同时存在，实现零停机灰度发布；老客户端不会炸。  
5. 性能/安全双杀：POST 用 Input DTO + 校验特性，GET 用 Output DTO + 投影查询，既避免“序列化整个 Entity 导致循环引用/懒加载 N+1”，又不把敏感字段意外吐出去。

---

面试一句话总结（背下来）

> “我们在 WebAPI 里坚持‘DTO 只进不出，Entity 只出不进’：Controller 收 Input DTO，应用层做校验→映射→业务，返回 Output DTO；这样领域无依赖、API 可版本、数据不外泄、性能 N+1 消失，上线两年零回滚。”

“防过度提交”的核心是：让客户端只能拿到/修改“我允许你动”的字段。

ASP.NET Core 里把 Input DTO 与 Output DTO 彻底分家，再用 `record/init` + `[JsonIgnore]` 组合锁，就能在编译期 + 运行时两层堵住 Mass Assignment（批量赋值）漏洞。

下面按“场景 → 危险代码 → 安全重构 → 验证”四步详解。

---

1. 场景：用户更新个人资料

需求：只允许他改昵称和手机号，禁止自己改余额、角色、创建时间。

---

2. 危险代码（一步到位 Entity）  

```csharp
// PUT /users/{id}
[HttpPut("{id}")]
public async Task<IActionResult> Update(int id, User entity)   // ← 直接收 Entity
{
    _db.Users.Update(entity);     // 客户端多传一个字段就把数据库覆盖
    await _db.SaveChangesAsync();
    return NoContent();
}
```

攻击：  

```http
PUT /users/1
{
  "nickName": "Hacker",
  "balance": 999999          // ← 多传一个字段，钱直接改了
}
```

结果：EF Core 把整行全更新，`balance` 被覆盖。

---

3. 安全重构：Input/Output DTO 分家 + 最小化赋值

3.1 定义两个 DTO  

```csharp
// 只给客户端“想改”的字段
public record UpdateUserRequest(
    [Required] string NickName,
    [Phone] string PhoneNumber
);

// 返回给客户端的视图
public record UserDto(
    int Id,
    string NickName,
    string PhoneNumber,
    decimal Balance,
    DateTime CreateTime
);
```

3.2 Controller 显式映射  

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> Update(
    int id,
    [FromBody] UpdateUserRequest dto)     // ← 只收 DTO
{
    var user = await _db.Users.FindAsync(id);
    if (user == null) return NotFound();

    // 手工映射：只动允许动的字段
    user.NickName = dto.NickName;
    user.PhoneNumber = dto.PhoneNumber;

    await _db.SaveChangesAsync();
    return NoContent();
}
```

3.3 查询接口用 Output DTO 投影  

```csharp
[HttpGet("{id}")]
public async Task<UserDto> Get(int id) =>
    await _db.Users
             .Where(u => u.Id == id)
             .Select(u => new UserDto(
                 u.Id,
                 u.NickName,
                 u.PhoneNumber,
                 u.Balance,
                 u.CreateTime))
             .FirstAsync();
```

- 不会序列化密码、盐等敏感列。  
- 查询只取 5 个字段，杜绝“select ”。

---

4. 再加一道锁：init-only + [JsonIgnore]

4.1 如果一定想在同一个类里区分读写，可用 `init` 或 `[JsonIgnore]`：  

```csharp
public class UserDto
{
    public int Id { get; init; }            // 客户端传了也忽略
    public string NickName { get; set; }    // 允许改
    [JsonIgnore]                            // 返回时序列化，接收时忽略
    public decimal Balance { get; set; }
}
```

4.2 验证  

```http
PUT /users/1
{
  "id": 999,           // ← 想改主键
  "nickName": "OK",
  "balance": 999999    // ← 想改余额
}
```

结果：  
- `id` 被 `init` 保护，模型绑定后仍是 1；  
- `balance` 被 `[JsonIgnore]` 拒绝绑定，保持原值；  
- 数据库只更新 `NickName`，Mass Assignment 失败。

---

一句话总结

“Input DTO 只放可写字段，Output DTO 只放可见字段；再用 init / [JsonIgnore] 把门焊死，就算黑客多传字段，模型绑定也接不住，EF 更不会意外更新。”