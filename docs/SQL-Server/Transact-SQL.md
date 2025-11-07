## [Transact-SQL 参考](https://learn.microsoft.com/zh-cn/sql/t-sql/language-reference?view=sql-server-ver17)

- T-SQL 语言元素（如控制流语句、运算符、函数等）
- 数据定义语言（DDL）、数据操作语言（DML）、数据控制语言（DCL）语法
- 系统存储过程、内置函数、数据类型等详细说明


Transact-SQL（通常缩写为 T-SQL）是微软和Sybase公司共同开发的一种关系数据库通用语言SQL的扩展。它在标准SQL的基础上增加了编程功能，如变量、流程控制、函数等，主要用于Microsoft SQL Server和Sybase Adaptive Server等数据库管理系统中。

### T-SQL 的主要特点：

1. **数据操作语言 (DML) 扩展**：
   - 支持标准的 `SELECT`, `INSERT`, `UPDATE`, `DELETE` 语句。
   - 提供更强大的 `SELECT` 功能，如 `TOP`, `OFFSET-FETCH`, 窗口函数（`ROW_NUMBER()`, `RANK()`, `OVER()` 等）。

2. **数据定义语言 (DDL) 支持**：
   - 用于创建、修改和删除数据库对象，如表、视图、索引等。
   - 示例：`CREATE TABLE`, `ALTER TABLE`, `DROP INDEX`。

3. **数据控制语言 (DCL)**：
   - 用于管理数据库权限和安全性。
   - 示例：`GRANT`, `REVOKE`, `DENY`。

4. **流程控制语句**：
   - `IF...ELSE`
   - `WHILE` 循环
   - `BEGIN...END` 块
   - `TRY...CATCH` 异常处理
   - `GOTO`（不推荐使用）

5. **变量与批处理**：
   - 局部变量：以 `@` 开头，如 `@name NVARCHAR(50)`
   - 全局变量（系统函数）：以 `@@` 开头，如 `@@VERSION`, `@@ROWCOUNT`, `@@ERROR`

6. **内置函数**：
   - 字符串函数：`LEN()`, `SUBSTRING()`, `REPLACE()`
   - 数值函数：`ROUND()`, `CEILING()`, `FLOOR()`
   - 日期函数：`GETDATE()`, `DATEADD()`, `DATEDIFF()`
   - 聚合函数：`SUM()`, `COUNT()`, `AVG()`, `MAX()`, `MIN()`

7. **存储过程与函数**：
   - 存储过程（Stored Procedure）：预编译的SQL代码块，可带参数，提高性能和复用性。
   - 用户定义函数（UDF）：可返回标量值或表。

8. **事务控制**：
   - `BEGIN TRANSACTION`
   - `COMMIT`
   - `ROLLBACK`
   - 支持事务的ACID特性。

---

### 示例代码：

```sql
-- 声明变量
DECLARE @EmployeeID INT = 101;
DECLARE @Salary DECIMAL(10,2);

-- 使用 SELECT 查询赋值
SELECT @Salary = Salary 
FROM Employees 
WHERE ID = @EmployeeID;

-- 流程控制
IF @Salary > 5000
    PRINT 'High Salary';
ELSE
    PRINT 'Normal Salary';

-- 异常处理示例
BEGIN TRY
    BEGIN TRANSACTION;
    UPDATE Accounts SET Balance -= 100 WHERE AccountID = 1;
    UPDATE Accounts SET Balance += 100 WHERE AccountID = 2;
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION;
    PRINT 'Transaction failed: ' + ERROR_MESSAGE();
END CATCH;
```

---
下面给你 3 条“能直接下载、用 SSDT 打开就能跑”的开源 Transact-SQL 演练项目，按“先易后难”排好，5 分钟就能在 Visual Studio 里按 F5 调试，最适合快速把 T-SQL + SSDT 的完整工作流跑通。

---

1. **Northwind-SSDT**（微软经典库，零门槛）  
仓库：https://github.com/microsoft/sql-server-samples/tree/master/samples/databases/northwind-pubs  
- 已拆成 .sql 脚本，直接“文件→新建→SQL Server 数据库项目→导入脚本”即可生成完整项目。  
- 表、视图、存储过程、触发器一应俱全，适合练“单步调试存储过程”“重构索引”等基础操作。

2. **WideWorldImporters-SSDT**（微软官方“现代”示例，带 CI/CD）  
**1 分钟拿到源码**：打开官方仓库 https://github.com/microsoft/sql-server-samples → 右侧 Releases → 找到“WideWorldImporters SSDT sample database”（标签名 wwi-ssdt）→ 下载 Source code (zip)。  
解压后路径：`samples\databases\wide-world-importers\wide-world-importers-ssdt\WideWorldImporters.sqlproj`  
双击 .sqlproj 即可用 Visual Studio + SSDT 打开，F5 直接跑。

- 官方 SSDT 项目（.sqlproj），下载即用，README 里自带 Azure DevOps yaml，可顺带把“Git → Build → Publish”整条 DevOps 流水线跑一遍。  
- 包含大量 2016+ 新语法（JSON、列存、CCI、安全策略），想练“现代 T-SQL”就它。

3. **tSQLt-SSDT-Template**（单元测试框架，进阶必备）  
仓库：https://github.com/tSQLt-org/tSQLt-SSDT-Template  
- 把主流单元测试框架 tSQLt 打包成 SSDT 项目，F5 后自动部署 tSQLt 与示例测试。  
- 可练“测试先行”模式：写测试类→写实现→调试→提交 Git，完美模拟企业级数据库开发流程。

---

## 5 分钟上手步骤（以 WideWorldImporters 为例）

1. **装 SSDT**：打开 Visual Studio Installer → 勾选“SQL Server Data Tools”工作负载。  
2. **拿代码**：下载解压后，双击 `.sqlproj` 打开。  
3. **配调试库**：右键项目 → 属性 → 调试 → 新建 LocalDB 或指向自己的 SQL Server 实例。  
4. **F5 一键部署+调试**：选“Start (调试)”，VS 会自动生成 .dacpac、publish 到调试库、附加调试器，随后即可在存储过程里打断点、单步、看变量。

---

## 两库对比

| 对比维度          | Northwind-SSDT（经典版）                  | WideWorldImporters-SSDT（现代版）                                                                            |
| :---------------- | :---------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| **设计年代**      | 1996 年随 Office 发布，90 年代小商户场景  | 2016 年随 SQL Server 2016 设计，面向云端与现代 ERP                                                           |
| **目标 SQL 版本** | SQL Server 2000-2012 语法为主             | SQL Server 2016+ / Azure SQL DB，需 2016 SP1+ 跑全功能                                                       |
| **项目体积**      | ≈ 1-2 MB，几十张表，数据 1-10 MB 级       | 空库 30 MB，生成 3.5 年数据后 ≈ 160 MB（可扩到 GB 级）                                                       |
| **新特性覆盖**    | 无（无列存、内存、JSON、时态表、分区）    | 全部都有：列存、内存 OLTP、时态表、JSON、分区、数据掩码、行级安全等                                          |
| **SSDT 项目结构** | 手工拆分脚本，简单直观，适合“第一次按 F5” | 官方原生 `.sqlproj`，预置 300+ 对象，分 Schema 文件夹，企业级模板                                            |
| **数据生成方式**  | 静态 INSERT 脚本，数据量固定              | 内置 `DataLoadSimulation.PopulateDataToCurrentDate` 存储过程，可参数化按日生成订单，支持周末系数、静默模式等 |
| **场景定位**      | ① 零基础学 T-SQL ② 教课 ③ 做演示用        | ① 练“现代”T-SQL/新特性 ② 练 CI/CD 发布 ③ 做性能测试（列存、内存）                                            |
| **调试体验**      | 表少、逻辑简单，存储过程断点一目了然      | 对象多，但自带示例测试存储过程，可练“大库”下调试 & 单元测试                                                  |
| **与 CI/CD 结合** | 无官方 yaml，需自己写                     | 同仓库直接给 Azure DevOps/GitHub Actions 样例，push 即发布                                                   |

---

## 学习路线图（按工作日每天 1.5h 估算）

### 起点 A：只会简单 SELECT / 没用过 SSDT

| 阶段         | 目标                  | 任务清单（可量化）                                                                           | 耗时                    |
| :----------- | :-------------------- | :------------------------------------------------------------------------------------------- | :---------------------- |
| ① 入门       | 把 SSDT 玩顺          | 克隆 Northwind-SSDT → F5 成功 → 单步调试 5 个存储过程                                        | 3 天                    |
| ② 基础 SQL   | 写 CRUD + 视图 + 事务 | 在 Northwind 里自己写 10 个查询、3 个视图、2 个带事务的存储过程并通过 tSQLt 测试             | 7 天                    |
| ③ 迁移现代库 | 体验“大数据量”        | 把 Northwind 订单表导入 WideWorldImporters，跑通 `PopulateDataToCurrentDate` 生成 100 万订单 | 3 天                    |
| ④ 新特性     | 掌握 2016+ 功能       | 在 WideWorldImporters 里各做一次：列存索引、内存 OLTP、时态表、JSON 查询、行级安全           | 10 天                   |
| ⑤ 实战       | 端到端 DevOps         | 用 GitHub Actions 把 WideWorldImporters 自动发布到 Docker 版 SQL 2025，加单元测试闸门        | 5 天                    |
| **合计**     |                       |                                                                                              | **≈ 4 周（20 工作日）** |

### 起点 B：已会 T-SQL，第一次用 SSDT
跳过阶段 ①②，直接从阶段 ③ 开始， **≈ 2 周** 可完成。

### 起点 C：熟手，只想快速补新特性 / CI/CD
只跑阶段 ④⑤， **≈ 1 周** 就够。

---

## 加速技巧（通用）

1. 每天任务拆成 ≤30 min 的小块，用 tSQLt 测试保证一次做对。  
2. 把 WideWorldImporters 数据生成脚本参数调小（`@YearNumberToLoad=1`），笔记本也能 3 min 跑完。  
3. 用 VS Code 的 mssql 扩展 + SSDT 同时开，查询窗口和项目文件切换更快。  
4. 周末集中 2 h 做“综合演练”——写个迷你业务需求（如“给 Northwind 加会员积分表”）从需求→建表→存储过程→测试→Git 提交一条龙，一次性把知识缝起来。

---

## 验收标准（你能独立做出即算“掌握”）

- 不用向导，手工新建 SSDT 项目，把任意业务库（≥15 张表）反向工程进来并推送到 Git。  
- 能用列存 + 分区把 1 亿行销售表查询压到 1 s 内。  
- 一条 `git push` 自动跑 tSQLt 测试→生成发布脚本→部署到 Docker SQL 实例→回写结果。

达到这三条，两个示例库就可以毕业，接下来直接拿生产项目练手即可。祝你 1 个月后就能上台讲“SQL Server 现代开发工作流”！





