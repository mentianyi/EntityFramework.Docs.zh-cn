---
title: 原生 SQL 查询 - EF Core
author: rowanmiller
ms.author: divega
ms.date: 10/27/2016
ms.assetid: 70aae9b5-8743-4557-9c5d-239f688bf418
ms.technology: entity-framework-core
uid: core/querying/raw-sql
ms.openlocfilehash: 29b7e20e875bf791a88a92636c1df4bc4e31656b
ms.sourcegitcommit: 038acd91ce2f5a28d76dcd2eab72eeba225e366d
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 05/14/2018
ms.locfileid: "34163208"
---
# <a name="raw-sql-queries"></a>原生 SQL 查询

通过 Entity Framework Core 可以在使用关系数据库时下拉到原生 SQL 查询。 这在无法使用 LINQ 表示要执行的查询或在使用 LINQ 查询导致 SQL 发送到数据库的效率低时非常有用。

> [!TIP]  
> 可在 GitHub 上查看此文章的[示例](https://github.com/aspnet/EntityFramework.Docs/tree/master/samples/core/Querying)。

## <a name="limitations"></a>限制

使用原生 SQL 查询时需注意以下几个限制：
* SQL 查询只能用于返回属于你的模型的实体类型。 积压工作有一个增强功能，即[可从原生 SQL 查询中返回临时类型](https://github.com/aspnet/EntityFramework/issues/1862)。

* SQL 查询必须返回实体或查询类型的所有属性的数据。

* 结果集中的列名必须与属性映射到的列名称匹配。 请注意，这与 EF6 不同，其中已忽略原生 SQL 查询的属性/列映射，且结果集列名称必须与属性名称匹配。

* SQL 查询不能包含相关数据。 但是，在许多情况下可以使用 `Include` 运算符在查询顶部进行编写以返回相关数据（请参阅[包括相关数据](#including-related-data)）。

* 传递到此方法的 `SELECT` 语句通常应该可以进行编写：如果 EF Core 需要计算服务器上的其他查询运算符（例如，转换 `FromSql` 后应用的 LINQ 运算符），则提供的 SQL 将被视为子查询。 这意味着传递的 SQL 不应包含子查询上无效的任何字符或选项，如：
  * 尾随分号
  * 在 SQL Server 上，尾随的查询级提示，如 `OPTION (HASH JOIN)`
  * 在 SQL Server 上，不附带 `SELECT` 子句中的 `TOP 100 PERCENT` 的 `ORDER BY` 子句

* 除 `SELECT` 以外的其他 SQL 语句自动识别为不可编写。 因此，存储过程的完整结果将始终返回到客户端，且在内存中计算 `FromSql` 后应用的任何 LINQ 运算符。 

## <a name="basic-raw-sql-queries"></a>基本原生 SQL 查询

可以使用 *FromSql* 扩展方法基于原生 SQL 查询开始 LINQ 查询。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var blogs = context.Blogs
    .FromSql("SELECT * FROM dbo.Blogs")
    .ToList();
```

原生 SQL 查询可用于执行存储过程。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var blogs = context.Blogs
    .FromSql("EXECUTE dbo.GetMostPopularBlogs")
    .ToList();
```

## <a name="passing-parameters"></a>传递参数

正如接受 SQL 的任何 API 一样，务必参数化任何用户输入以抵御 SQL 注入攻击。 可以将参数占位符包含在 SQL 查询字符串中，然后提供参数值作为其他参数。 提供的任何参数值将自动转换为 `DbParameter`。

下面的示例将一个参数传递到存储过程。 尽管这看上去可能像 `String.Format` 语法，但提供的值包装在参数中，且生成的参数名称插入在指定 `{0}` 占位符的位置。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = "johndoe";

var blogs = context.Blogs
    .FromSql("EXECUTE dbo.GetMostPopularBlogsForUser {0}", user)
    .ToList();
```

这是相同的查询，但使用 EF Core 2.0 及更高版本支持的字符串内插语法：

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = "johndoe";

var blogs = context.Blogs
    .FromSql($"EXECUTE dbo.GetMostPopularBlogsForUser {user}")
    .ToList();
```

还可以构造 DbParameter 并将其作为参数值提供。 这样可以在 SQL 查询字符串中使用命名参数

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = new SqlParameter("user", "johndoe");

var blogs = context.Blogs
    .FromSql("EXECUTE dbo.GetMostPopularBlogsForUser @user", user)
    .ToList();
```

## <a name="composing-with-linq"></a>使用 LINQ 编写

如果在数据库中可以在 SQL 查询上进行编写，则可以使用 LINQ 运算符在初始原生 SQL 查询顶部进行编写。 可以使用 `SELECT` 关键字在其上进行编写的 SQL 查询。

下面的示例使用可从表值函数 (TVF) 中选择的原生 SQL 查询，然后使用 LINQ 在其上进行编写以执行筛选和排序。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Blogs
    .FromSql($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .Where(b => b.Rating > 3)
    .OrderByDescending(b => b.Rating)
    .ToList();
```

### <a name="including-related-data"></a>包括相关数据

使用 LINQ 运算符编写可用于将相关数据包含在查询中。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Blogs
    .FromSql($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .Include(b => b.Posts)
    .ToList();
```

> [!WARNING]  
> **始终为原生 SQL 查询使用参数化：** 接受原生 SQL 字符串（如 `FromSql` 和 `ExecuteSqlCommand`）的 API 允许将值作为参数轻松传递。 除了验证用户输入以外，始终为原生 SQL 查询/命令中使用的所有值使用参数化。 如果使用字符串串联来动态生成查询字符串的任何部分，则你将负责验证所有输入以抵御 SQL 注入攻击。
