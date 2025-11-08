## [关系简介](https://learn.microsoft.com/zh-cn/ef/core/modeling/relationships)

### [一对多关系](https://learn.microsoft.com/zh-cn/ef/core/modeling/relationships/one-to-many)
在 EF Core 中，外键（Foreign Key）是构建关系的核心机制。其作用可概括为：**建立关联、保证完整性、支撑导航与级联**。

---

## 一对多关系（Blog → Posts）

外键位于 **"多"端（Post）** ，存储"一"端（Blog）的主键值。

**核心作用：**

1. **建立表间关联**  
   `Post.BlogId` 指向 `Blog.Id`，明确每条帖子归属哪个博客。

2. **保证引用完整性**  
   数据库强制检查：`BlogId` 必须在 `Blog` 表中存在，防止出现"孤儿"记录。

3. **支撑导航与级联**  
   - 支持 `blog.Posts` 集合导航，代码可直接 `blog.Posts.ToList()` 获取所有帖子
   - 配置 `.OnDelete(DeleteBehavior.Cascade)` 后，删除 Blog 时自动级联删除其 Posts

**一句话总结**：外键是"多"端表里的字段，把两条记录拴在一起，既保证数据正确，又让 EF Core 能自动生成导航属性和级联逻辑。

---

## 一对一关系（User → UserProfile）

外键位于 **"依赖"端（UserProfile）** ，必须是**唯一键**，防止变成一对多。

**核心作用：**

1. **建立唯一关联**  
   `UserProfile.UserId` 既是外键又是唯一键，确保每个用户仅有一份资料。

2. **保证引用完整性**  
   数据库保证 `UserId` 必须在 `User` 表中存在。

3. **控制可选性与级联**  
   - **可空外键**：关系为可选，可配置 `DeleteBehavior.SetNull`
   - **不可空外键**：关系为必需，删除 User 时级联删除 UserProfile或抛异常

**一句话总结**：一对一的外键仍负责"挂靠"两条记录，但必须加唯一约束保证"一对一"，并通过是否允许 NULL 决定关系是否强制存在。

**"两条记录"指**：主表（User）一条主记录，从表（UserProfile）一条依赖记录，外键 UserId 将它们唯一绑定。

---

## 多对多关系（Student → Courses）

多对多需要 **中间表（联接表）** ，包含**两个外键**。

**核心作用：**

1. **建立多向关联**  
   `Enrollment` 表包含 `StudentId` 和 `CourseId` 两个外键，分别指向 `Student.Id` 和 `Course.Id`，表示一个学生可选多门课，一门课可有多个学生。

2. **保证引用完整性**  
   数据库强制两个外键必须存在于对应主表，防止出现无效关联。

3. **支撑双向导航与级联**  
   - 支持 `student.Courses` 和 `course.Students` 双向集合导航
   - EF Core 5.0+ 支持透明多对多：无需显式中间表实体，系统自动管理
   - 可配置级联删除：删除学生时自动删除其选课记录（中间表），但默认不删除课程本身

**关键差异点**：
- **两个外键**：分别指向两个主表
- **复合主键**：中间表通常设置 `(StudentId, CourseId)` 为复合主键或唯一索引，防止重复选课
- **透明映射**：EF Core 5.0+ 中，配置 `HasMany(...).WithMany(...)` 即可，无需定义中间表实体

**一句话总结**：多对多通过中间表的两个外键实现双向挂靠，既保证关联有效性，又支撑 EF Core 的双向导航和透明映射。

---

## 三种关系对比

| 关系类型   | 外键位置              | 外键唯一性   | 中间表   | 导航属性 | 典型场景                          |
| :--------- | :-------------------- | :----------- | :------- | :------- | :-------------------------------- |
| **一对多** | "多"端（Post）        | 非唯一       | 不需要   | 单方集合 | Blog-Posts, Order-OrderItems      |
| **一对一** | 依赖端（UserProfile） | **必须唯一** | 不需要   | 单方引用 | User-UserProfile, Person-Passport |
| **多对多** | 中间表（Enrollment）  | 复合唯一     | **必须** | 双方集合 | Student-Courses, Product-Customer |

**核心原则**：外键始终是依赖端指向主体端的"绳子"，唯一性决定关系类型，中间表决定关联维度。