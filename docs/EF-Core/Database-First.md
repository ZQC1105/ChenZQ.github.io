## Database First

\*简短回答：在 Database First 模式下，生成的 OnModelCreating 中的 Fluent API 配置 技术上不是绝对必须的，但对于正确映射复杂的数据库结构来说，它几乎是必不可少的，也是工具生成它的根本原因。

让我们深入解释一下：
为什么 Database First 会生成 Fluent API 配置？

虽然 EF Core 有强大的约定 (Conventions)，但现实世界的数据库（尤其是遗留系统或由 DBA 设计的数据库）往往不符合这些约定。Scaffolding 工具需要确保生成的模型能精确地、无歧义地反映现有数据库的每一个细节。

### Fluent API 提供了最强大和最精确的配置方式，可以覆盖所有约定无法处理或处理错误的情况。以下是一些常见的、必须使用 Fluent API 配置的例子：

1. 不符合约定的主键/外键名称:
   约定: 主键期望名为 Id 或 <类名>Id。
   现实: 数据库主键可能是 Blog_PK, blog_id, RecordId 等。
   Fluent API: modelBuilder.Entity<Blog>().HasKey(b => b.Blog_PK);

2. 不符合约定的表名或列名:
   约定: 类 Blog 映射到表 Blogs，属性 Url 映射到列 Url。
   现实: 表可能是 tblBlog，列可能是 blog_url。
   Fluent API: modelBuilder.Entity<Blog>().ToTable("tblBlog"); 和 modelBuilder.Entity<Blog>().Property(b => b.Url).HasColumnName("blog_url");

3. 复杂的数据类型和精度:
   约定: string 属性可能被映射为 nvarchar(max) 或默认长度。
   现实: 列 Title 是 nvarchar(100)，Price 是 decimal(18,2)。
   Fluent API: modelBuilder.Entity<Post>().Property(p => p.Title).HasMaxLength(100); 和 modelBuilder.Entity<Post>().Property(p => p.Price).HasPrecision(18, 2);

4. 复杂的外键关系和级联删除:
   约定: 可能无法正确推断自引用关系或非标准的外键名称。
   现实: 需要明确配置外键属性、导航属性以及 OnDelete 行为（如 Restrict, Cascade）。
   Fluent API: modelBuilder.Entity<Post>().HasOne(p => p.Blog).WithMany(b => b.Posts).HasForeignKey("blog_fk_id").OnDelete(DeleteBehavior.Cascade);

5. 索引:
   约定: 不会自动为非主键列创建索引。
   现实: 数据库中已有 IX_Blog_Url 索引。
   Fluent API: modelBuilder.Entity<Blog>().HasIndex(b => b.Url).HasDatabaseName("IX_Blog_Url");
   如果没有 Fluent API 配置会怎样？

如果 Scaffolding 工具不生成这些 Fluent API 配置，而是完全依赖约定，那么：

生成的模型将无法准确反映数据库结构。
当 EF Core 尝试查询或保存数据时，会因为表名、列名、主键不匹配而抛出 SqlException。
数据类型不匹配可能导致数据截断或转换错误。
关系无法建立，导致导航属性无法正常工作。
为什么不用数据注释 (Data Annotations)？

### 你可能会问，为什么不用 [Table], [Column], [Key] 这些特性呢？这也是个好问题。

Fluent API 更强大: 对于非常复杂的配置（如复杂的索引、特定的约束、某些数据库特定的配置），数据注释可能无能为力，而 Fluent API 可以。
保持实体类“干净”: 数据注释会直接“污染”你的实体类。而 Fluent API 集中在 OnModelCreating 方法中，使实体类保持为更纯粹的 POCO (Plain Old CLR Object)。
工具生成的便利性: Scaffolding 工具更容易将所有配置集中写在一个地方（OnModelCreating），而不是分散到多个类的多个属性上。
结论

因此，虽然 EF Core 的核心模型最终是内存中的元数据，而这个元数据可以通过约定、数据注释或 Fluent API 三种方式中的任何一种来构建，但在 Database First 场景下：

Scaffolding 工具选择生成 Fluent API 配置是经过深思熟虑的最佳实践。
它确保了生成的模型能够精确、完整、可靠地映射到任意复杂或非标准的现有数据库。
它避免了依赖可能失败的约定，并提供了数据注释无法比拟的灵活性和控制力。

所以，虽然从技术上讲，如果数据库完美符合所有 EF Core 的约定，你可能不需要 Fluent API，但在实际的 Database First 项目中，这些生成的 Fluent API 配置是保证模型正确性的关键，是不可或缺的。
