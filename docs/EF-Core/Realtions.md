## [关系简介](https://learn.microsoft.com/zh-cn/ef/core/modeling/relationships)

### [一对多关系](https://learn.microsoft.com/zh-cn/ef/core/modeling/relationships/one-to-many)
在 EF Core 的一对多关系里，外键的核心作用可以概括为三点：

1. **建立表间关联**  
   外键字段放在“多”端表中，用来存储“一”端表的主键值，从而把两条记录绑定在一起。例如 Blog-Post 关系里，Post 表的 `BlogId` 外键指向 Blog 表的 `Id`，表示这条帖子属于哪个博客。

2. **保证引用完整性**  
   数据库会强制检查：插入或更新 Post 记录时，其 `BlogId` 必须在 Blog 表中真实存在，否则操作失败。这防止了出现“孤儿”记录。

3. **支撑导航与级联行为**  
   - 有了外键，EF Core 才能生成 `Blog.Posts` 这样的集合导航属性，让代码可以直接 `blog.Posts.ToList()` 拉取所有相关帖子。  
   - 同时可在 `OnModelCreating` 里通过 `.HasForeignKey().OnDelete(DeleteBehavior.Cascade)` 配置级联删除：删除 Blog 时，数据库自动把该 Blog 的所有 Post 一并删除，保持数据一致。

一句话总结：外键是“多”端表里的一个字段，它把两条记录拴在一起，既保证数据正确，又让 EF Core 能自动生成导航属性和级联逻辑。

### [一对一](https://learn.microsoft.com/zh-cn/ef/core/modeling/relationships/one-to-one)

在 EF Core 的一对一关系里，外键的作用与一对多类似，但有两点关键差异：

1. **外键位置与唯一性** 
外键放在“依赖”端（从表），并必须加唯一约束，确保“一”对“一”的映射不会变成“一”对“多”。 
例如：User-UserProfile 关系中，UserProfile 表的 `UserId` 字段既是外键又是唯一键，表示每个用户只能有一个资料。

2. **级联与可选性** 
- 外键列是否允许 NULL 决定了关系是“必需”还是“可选”。 
- 若外键不允许 NULL，则删除主表记录时，从表记录也会被级联删除（或抛异常），保证不会留下孤立记录。 
- 若外键允许 NULL，则关系是可选的，级联删除行为需要显式配置。

一句话总结：一对一的外键仍负责“挂靠”两条记录，但必须加唯一约束来保证“一对一”，并通过是否允许 NULL 来决定关系是否强制存在。

两条记录指的是一对一关系中的“主”与“从”两条数据。  
以 User（主）和 UserProfile（从）为例：

1. **主记录**：User 表中的一条用户数据，主键为 Id。  
2. **从记录**：UserProfile 表中的一条资料数据，包含外键 UserId，且该字段加唯一约束，确保同一个用户不能拥有两条资料记录。

外键 UserId 把 UserProfile 的这条记录“挂”在 User 上，形成一对一映射。