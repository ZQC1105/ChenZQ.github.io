下面把 `with` 表达式的行为一次说透，并给出可直接运行的代码片段，方便验证。

1. 作用范围  
   `with` 可以“改”两类属性：  
   - 位置属性（主构造函数参数）  
   - 手动写的标准属性  
   前提：该属性**必须可见且可写**（即拥有 `init` 或 `set` 访问器）。  
   只读字段/只 get 属性不会被 `with` 复制，也不会被修改。

2. 复制深度  
   `with` 做的是**浅拷贝（shallow copy）**。  
   如果属性是引用类型，原记录和新记录将指向**同一个子对象**；修改子对象的内容对两边都可见。

3. 代码验证  
```csharp
// 引用类型子对象
public record Address(string City);

// 记录包含值类型位置属性 + 引用类型属性
public record Person(string FirstName, string LastName)
{
    public Address Addr { get; init; }        // 需要 init 才能出现在 with 里
    public int Age { get; set; }              // 普通 set 也可以
}

var ada = new Person("Ada", "Lovelace")
{
    Addr = new Address("London"),
    Age  = 36
};

// with 修改位置属性 + 引用属性 + 普通属性
var alan = ada with { FirstName = "Alan", Addr = new Address("New York"), Age = 40 };

// 验证浅拷贝：如果共享引用，修改原对象会影响副本
ada.Addr.City = "Paris";

Console.WriteLine($"ada.Addr.City = {ada.Addr.City}");   // Paris
Console.WriteLine($"alan.Addr.City = {alan.Addr.City}"); // New York  （已替换为新实例，未受影响）

// 验证值类型字段默认复制
Console.WriteLine($"ada.Age = {ada.Age}, alan.Age = {alan.Age}");  // 36 vs 40
```

4. 一句话总结  
`with` 表达式可以同时针对**位置属性**和**显式属性**做非破坏性修改，但只复制对象本身一层；引用型子对象默认**共享**，需要手动在 `with` 里替换成新实例才能实现“深度不可变”。