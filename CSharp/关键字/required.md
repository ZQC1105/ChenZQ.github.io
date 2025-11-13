## [required 修饰符](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/required)
下面把官方文档里散落在各处的“required 成员”规则，汇总成一份**一页式速查表**。  
直接拿来写代码、做 Code Review 或给同事科普都够用。示例用 C# 11 / .NET 7+ 语法。

------------------------------------------------
required 成员（C# 11+）一句话定位  
“**必须在对象初始值设定项或构造函数里显式赋值**（可赋 null），否则编译失败；同时仍保留对象初始值设定符的简洁语法。”

------------------------------------------------
1. 能贴在哪  
- 字段、属性、索引器  
- class、struct、record、record struct  
- **不能**贴在 interface 成员、显式接口实现、位置 record 参数上。

2. 可见性要求  
- 成员的可访问性 **≥** 所在类型的可访问性。  
  ```csharp
  public class Foo
  {
      protected required string Bar;   // ❌ CS9039
  }
  ```
- required 属性必须至少有一个 **set** 或 **init** 访问器，且该访问器可见性同样 ≥ 类型可见性。

3. 初始化规则  
- 任何创建实例的表达式（`new`、`with`、反射的 `Activator.CreateInstance`）都必须给所有 required 成员赋值；否则 **编译期错误**（CS9035）。  
- 可以显式赋 null；对不可为 null 的引用类型会给出 **警告**（CS8618、CS9036）。  
- 构造函数内部不必再次赋值——只要**最终**被初始化即可。

4. 继承/重写  
- 派生类 **不能**用 `new` 隐藏基类 required 成员。  
- 重写基类 required 属性时，**必须**保留 `required` 关键字；  
  可以在重写时**追加** `required`（基类不是 required，派生类改成 required 是允许的），但**不能**拿掉。  
  ```csharp
  public class Base
  {
      public virtual string Name { get; init; }
  }
  public class Derived : Base
  {
      public override required string Name { get; init; }   // ✅
  }
  ```

5. 泛型约束死角  
- 如果类型参数有 `where T : new()` 约束，则 **不能** 把带 required 成员的类型实参传进去——编译器无法保证泛型代码里会初始化那些成员（CS9034）。  
  解决：改用 `where T : new() => T.Create()` 工厂模式，或删掉 `new()` 约束。

6. 与 record 位置参数的关系  
- 位置参数天生自带“必传”语义，因此 **不允许**再贴 `required`。  
  如果既想要位置语法又想加 required，可手动声明属性：  
  ```csharp
  public record Person(string FirstName, string LastName)
  {
      public required string MiddleName { get; init; }
  }
  ```

7. 常见错误码速览  
| 错误码 | 含义                                                |
| ------ | --------------------------------------------------- |
| CS9035 | 忘记在对象初始值设定项里给 required 成员赋值        |
| CS9036 | 把不可为 null 的 required 成员显式设成 null         |
| CS9039 | required 成员可见性比类型低                         |
| CS9034 | 带 new() 约束的泛型参数使用了含 required 成员的类型 |

------------------------------------------------
一张图记心里  
“**required = 编译期强制初始化 + 仍享对象初始值设定符**；  
看不见、去不掉、泛型 new() 要绕开。”

把两个问题拆开说：

------------------------------------------------
一、required 成员 vs“传统构造函数必填参数”

| 维度            | required 成员                                                        | 构造函数必填参数                               |
| --------------- | -------------------------------------------------------------------- | ---------------------------------------------- |
| **调用语法**    | 保留对象初始值设定符的“键值对”风格<br>`new Person { Name = "Mads" }` | 必须走构造函数实参列表<br>`new Person("Mads")` |
| **参数顺序**    | 无顺序要求，可只填一部分，可混用默认值                               | 顺序固定，全必填时必须一次给齐                 |
| **向后兼容**    | 新增字段/属性时老代码**无需改动**                                    | 新增参数会导致**所有已有调用点**编译失败       |
| **可组合性**    | 与 `with` 表达式天然友好                                             | 需要手写 `Copy` 构造或工厂方法                 |
| **反射/序列化** | 需要**无参构造**+ 赋值路径                                           | 需要**匹配签名的构造**                         |
| **性能**        | 多一次赋值（先 0 再写值）                                            | 直接进字段，理论少一次写                       |

一句话：  
“required 成员把‘语法糖’留给你，把‘必填’留给编译器；  
构造函数参数把‘必填’和‘语法’都硬编码在方法签名里。”

------------------------------------------------
二、与 JSON 反序列化共舞（System.Text.Json）

1. **必备前提**  
   必须有无参构造函数——无论 `public` 还是 `internal`，否则反序列化器连对象都造不出来。

2. **版本要求**  
   - .NET 7 起：内置支持 required 成员（`JsonSerializer.Deserialize` 会检查 required 并抛 `JsonException`）。  
   - .NET 6 及以前：不认 required，漏字段不会报错，需要自定义 `JsonConverter` 或迁移到 .NET 7+。

3. **典型代码**

   ```csharp
   public class Person
   {
       public required string Name { get; init; }
       public int Age { get; init; } = 0;   // 非 required，可缺
   }

   string json = """{"Age":29}""";
   try
   {
       var p = JsonSerializer.Deserialize<Person>(json);   // .NET 7 会抛
   }
   catch (JsonException ex) when (ex.Message.Contains("required"))
   {
       Console.WriteLine("缺少 required 字段");
   }
   ```

4. **与构造函数参数方案对比**  
   若用“全参构造”：
   ```csharp
   public Person(string name, int age = 0) { ... }
   ```
   必须写自定义 `JsonConstructor` 告诉反序列化器把 JSON 属性映射到构造参数：
   ```csharp
   [JsonConstructor]
   public Person(string name, int age) : this(name) { Age = age; }
   ```
   每新增一个参数就要改构造，**对版本演进不友好**。

5. **最佳实践速记**  
   - 对外 DTO：用 `required` + 无参构造 → 保持序列化兼容，编译期检查必填。  
   - 对内领域对象：若追求极致不可变，可保留“全参构造”+ 私有 `init`，再写工厂或 `JsonConstructor`。  
   - 升级路径：老项目先补无参构造，再贴 `required`，最后把 `.NET SDK` 升到 7+，就能同时得到  
     ① 编译期检查 ② 反序列化期检查 ③ 对象初始值设定符的清爽语法。

------------------------------------------------
一句话收工  
“required 让‘编译器 + 序列化器’帮你一起把关必填字段，而构造函数必填参数只能让‘编译器’把关，还把序列化器拒之门外。”

`with` 表达式（C# 9 起）的底层逻辑是：

1. 先**调用当前实例的“无参构造函数”**（编译器生成的 `protected` 拷贝构造）。  
2. 再把**所有需要改变的属性**按名字逐个赋值（普通赋值，不是 `init` 也不是 `set`）。  
3. 返回一个新实例。

因此，只要属性具备 **init** 访问器，`with` 就能写；  
而 `required` 只约束“构造阶段必须赋值”，**不限制后续 `with` 再次赋值**——  
因为 `with` 是在**新实例的构造流程里**完成写入，正好落在 `init` 允许的“构造期”时间窗口内。

------------------------------------------------
看代码最直观

```csharp
public record Person   // record 自动给所有 init 属性生成 with 支持
{
    public required string Name { get; init; }   // 必须初始化
    public int Age { get; init; }
}

var p1 = new Person { Name = "Mads", Age = 50 };   // 构造阶段满足 required

var p2 = p1 with { Age = 51 };                     // ✅ 合法
// 编译器生成的大致 IL：
// var tmp = new Person();   // 1. 无参构造
// tmp.Name = p1.Name;       // 2. 拷贝旧值
// tmp.Age = 51;             // 3. 把 Age 改掉
// return tmp;               // 4. 构造完成，required 检查通过
```

------------------------------------------------
对比普通 class（非 record）

即使不是 `record`，只要属性是 `init`，你就可以手动写扩展方法或自己调用构造函数 + 对象初始值设定符，但 `with` 语法糖仅限 `record` 和 `struct` 自动享受。  
要点不变：**`init` 允许在“构造期”写入，`with` 正好处于这一时间段**，所以两者“天然友好”。