Dapper - .Net的一个简单对象映射器
========================================
[![Build status](https://ci.appveyor.com/api/projects/status/8rbgoxqio76ynj4h?svg=true)](https://ci.appveyor.com/project/StackExchange/dapper)

发行说明
-------------

[Located at stackexchange.github.io/Dapper](https://stackexchange.github.io/Dapper/)


特征
--------
Dapper 是一个 [NuGet 库](https://www.nuget.org/packages/Dapper) 可以添加到您的项目中，这可以扩展 `IDbConnection` 接口.

他提供了三个帮助:

执行查询并将结果映射到强类型列表
------------------------------------------------------------

```csharp
public static IEnumerable<T> Query<T>(this IDbConnection cnn, string sql, object param = null, SqlTransaction transaction = null, bool buffered = true)
```
使用示例:

```csharp
public class Dog
{
    public int? Age { get; set; }
    public Guid Id { get; set; }
    public string Name { get; set; }
    public float? Weight { get; set; }

    public int IgnoredProperty { get { return 1; } }
}            
            
var guid = Guid.NewGuid();
var dog = connection.Query<Dog>("select Age = @Age, Id = @Id", new { Age = (int?)null, Id = guid });
            
dog.Count()
    .IsEqualTo(1);

dog.First().Age
    .IsNull();

dog.First().Id
    .IsEqualTo(guid);
```

执行查询并将其映射到动态对象列表
-------------------------------------------------------

```csharp
public static IEnumerable<dynamic> Query (this IDbConnection cnn, string sql, object param = null, SqlTransaction transaction = null, bool buffered = true)
```
此方法将执行SQL并返回动态列表。

使用示例:

```csharp
var rows = connection.Query("select 1 A, 2 B union all select 3, 4");

((int)rows[0].A)
   .IsEqualTo(1);

((int)rows[0].B)
   .IsEqualTo(2);

((int)rows[1].A)
   .IsEqualTo(3);

((int)rows[1].B)
    .IsEqualTo(4);
```

执行不返回结果的命令
-----------------------------------------

```csharp
public static int Execute(this IDbConnection cnn, string sql, object param = null, SqlTransaction transaction = null)
```

使用示例:

```csharp
connection.Execute(@"
  set nocount on 
  create table #t(i int) 
  set nocount off 
  insert #t 
  select @a a union all select @b 
  set nocount on 
  drop table #t", new {a=1, b=2 })
   .IsEqualTo(2);
```

执行命令多次
--------------------------------

相同的签名还允许您方便有效地执行命令多次（例如批量加载数据）

使用示例:

```csharp
connection.Execute(@"insert MyTable(colA, colB) values (@a, @b)",
    new[] { new { a=1, b=1 }, new { a=2, b=2 }, new { a=3, b=3 } }
  ).IsEqualTo(3); // 3 rows inserted: "1,1", "2,2" and "3,3"
```
这适用于为某些T实现IEnumerable<T>的任何参数。

性能
-----------

Dapper的一个关键特性是性能。以下指标显示对DB执行500 SELECT语句所需的时间，并映射返回到对象的数据。

性能测试分为3个列表:

- 支持从DB拉取静态类型对象的框架的POCO序列化。使用原始SQL.
- 支持返回对象动态列表的框架的动态序列化.
- 典型的框架使用。典型的框架使用通常与最佳使用性能不同。通常它不会涉及编写SQL.

### 500次迭代的SELECT映射的性能--POCO序列化

<table>
  <tr>
  	<th>方法</th>
		<th>持续时间</th>		
		<th>备注</th>
	</tr>
	<tr>
		<td>Hand coded (using a <code>SqlDataReader</code>)</td>
		<td>47ms</td>
		<td rowspan="9"><a href="http://www.toptensoftware.com/blog/posts/94-PetaPoco-More-Speed">Can be faster</a></td>
	</tr>
	<tr>
		<td>Dapper <code>ExecuteMapperQuery<Post></code></td>
		<td>49ms</td>
	</tr>
	<tr>
		<td><a href="https://github.com/ServiceStack/ServiceStack.OrmLite">ServiceStack.OrmLite</a> (QueryById)</td>
		<td>50ms</td>
	</tr>
	<tr>
		<td><a href="http://www.toptensoftware.com/petapoco/">PetaPoco</a></td>
		<td>52ms</td>
	</tr>
	<tr>
		<td>BLToolkit</td>
		<td>80ms</td>
	</tr>
	<tr>
		<td>SubSonic CodingHorror</td>
		<td>107ms</td>
	</tr>
	<tr>
		<td>NHibernate SQL</td>
		<td>104ms</td>
	</tr>
	<tr>
		<td>Linq 2 SQL <code>ExecuteQuery</code></td>
		<td>181ms</td>
	</tr>
	<tr>
		<td>Entity framework <code>ExecuteStoreQuery</code></td>
		<td>631ms</td>
	</tr>
</table>

### 500次迭代的SELECT映射的性能 - 动态序列化

<table>
	<tr>
		<th>Method</th>
		<th>Duration</th>		
		<th>Remarks</th>
	</tr>
	<tr>
		<td>Dapper <code>ExecuteMapperQuery</code> (dynamic)</td>
		<td>48ms</td>
		<td rowspan="3">&nbsp;</td>
	</tr>
	<tr>
		<td><a href="https://github.com/FransBouma/Massive">Massive</a></td>
		<td>52ms</td>
	</tr>
	<tr>
		<td><a href="https://github.com/markrendle/Simple.Data">Simple.Data</a></td>
		<td>95ms</td>
	</tr>
</table>


### 超过500次迭代的SELECT映射的性能 - 典型用法

<table>
	<tr>
		<th>Method</th>
		<th>Duration</th>		
		<th>Remarks</th>
	</tr>
	<tr>
		<td>Linq 2 SQL CompiledQuery</td>
		<td>81ms</td>
		<td>Not super typical involves complex code</td>
	</tr>
	<tr>
		<td>NHibernate HQL</td>
		<td>118ms</td>
		<td>&nbsp;</td>
	</tr>
	<tr>
		<td>Linq 2 SQL</td>
		<td>559ms</td>
		<td>&nbsp;</td>
	</tr>
	<tr>
		<td>Entity framework</td>
		<td>859ms</td>
		<td>&nbsp;</td>
	</tr>
	<tr>
		<td>SubSonic ActiveRecord.SingleOrDefault</td>
		<td>3619ms</td>
		<td>&nbsp;</td>
	</tr>
</table>

性能可以查看 [这里](https://github.com/StackExchange/dapper-dot-net/blob/master/Dapper.Tests.Performance/PerformanceTests.cs).

随时提交包含其他ORM的修补程序 - 运行基准时，请确保在Release中编译，而不是附加调试器（ctrl F5）。

Alternatively, you might prefer Frans Bouma's [RawDataAccessBencher](https://github.com/FransBouma/RawDataAccessBencher) test suite or [OrmBenchmark](https://github.com/InfoTechBridge/OrmBenchmark).

参数化查询
---------------------

参数作为匿名类传递。这允许您轻松地命名参数，并使您能够简单地剪切并粘贴SQL代码段并在查询分析器中运行它们。

```csharp
new {A = 1, B = "b"} // A will be mapped to the param @A, B to the param @B 
```

列表支持
------------
Dapper 允许您传入IEnumerable并自动参数化查询。

例如:

```csharp
connection.Query<int>("select * from (select 1 as Id union all select 2 union all select 3) as X where Id in @Ids", new { Ids = new int[] { 1, 2, 3 } });
```

将被翻译成:

```csharp
select * from (select 1 as Id union all select 2 union all select 3) as X where Id in (@Ids1, @Ids2, @Ids3)" // @Ids1 = 1 , @Ids2 = 2 , @Ids2 = 3
```

缓冲与无缓冲 的读取
---------------------
Dapper的默认行为是执行您的sql并在返回时缓冲整个阅读器。这在大多数情况下是理想的，因为它可以最小化数据库中的共享锁并减少数据库网络时间。

但是，当执行大量查询时，您可能需要最小化内存占用，并且只需要根据需要加载对象。要做到这一点，通过缓冲：false进入Query方法。

多重映射
---------------------
Dapper 允许您将单个行映射到多个对象。如果您想避免无关的查询和热切的负载关联，这是一个关键功能。

例如:

有 2 类: `Post` 和 `User`

```csharp
class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public User Owner { get; set; }
}

class User
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

现在让我们说，我们要映射一个加入posts和users表的查询。直到现在，如果我们需要结合2个查询的结果，我们需要一个新的对象来表达它，但是在这种情况下，将 `User` 对象放在 `Post` 对象中更有意义。

这是用于多映射的用例。你告诉短小精悍该查询返回`Post`和`User`对象，然后给它描述要与每个含有一个行做什么的功能`Post`和`User`对象。在我们的例子中，我们想把用户对象放在`post`对象中。所以我们写功能：

```csharp
(post, user) => { post.Owner = user; return post; }
```

该`Query`方法的3类参数指定dapper应使用什么对象反序列化行以及将要返回的对象。我们要将两行解释为一个组合`Post`，`User`并且我们返回一个`Post`对象。因此，类型声明变为

```csharp
<Post, User, Post>
```

一切都放在一起，看起来像这样：

```csharp
var sql = 
@"select * from #Posts p 
left join #Users u on u.Id = p.OwnerId 
Order by p.Id";
 
var data = connection.Query<Post, User, Post>(sql, (post, user) => { post.Owner = user; return post;});
var post = data.First();
 
post.Content.IsEqualTo("Sams Post1");
post.Id.IsEqualTo(1);
post.Owner.Name.IsEqualTo("Sam");
post.Owner.Id.IsEqualTo(99);
```

Dapper可以通过假设您的`Id`列命名`Id`或拆分返回的行`id`。如果您的主键不同，或者您想在除了其他位置之外分割行`Id`，请使用可选`splitOn`参数。

多个结果
---------------------
Dapper 允许您在单个查询中处理多个结果网格。

例如:

```csharp
var sql = 
@"
select * from Customers where CustomerId = @id
select * from Orders where CustomerId = @id
select * from Returns where CustomerId = @id";
 
using (var multi = connection.QueryMultiple(sql, new {id=selectedId}))
{
   var customer = multi.Read<Customer>().Single();
   var orders = multi.Read<Order>().ToList();
   var returns = multi.Read<Return>().ToList();
   ...
} 
```

存储程序
---------------------
Dapper完全支持存储过程：

```csharp
var user = cnn.Query<User>("spGetUser", new {Id = 1}, 
        commandType: CommandType.StoredProcedure).SingleOrDefault();
```

如果你想要更多的东西，你可以做：

```csharp
var p = new DynamicParameters();
p.Add("@a", 11);
p.Add("@b", dbType: DbType.Int32, direction: ParameterDirection.Output);
p.Add("@c", dbType: DbType.Int32, direction: ParameterDirection.ReturnValue);

cnn.Execute("spMagicProc", p, commandType: CommandType.StoredProcedure); 

int b = p.Get<int>("@b");
int c = p.Get<int>("@c"); 
```

Ansi字符串和varchar
---------------------
Dapper支持varchar参数，如果您使用param执行varchar列上的where子句，请务必以此方式传递：

```csharp
Query<Thing>("select * from Thing where Name = @Name", new {Name = new DbString { Value = "abcde", IsFixedLength = true, Length = 10, IsAnsi = true });
```

在SQL Server上，当查询unicode时查询unicode和ansi时，使用unicode至关重要。

类型每行切换
---------------------

通常，您需要将给定表中的所有行都视为相同的数据类型。然而，在某些情况下，能够将不同的行解析为不同的数据类型是有用的。这是在哪里`IDataReader.GetRowParser`派上用场。

想象一下，你有一个名为“形状”与列的数据库表：`Id`，`Type`，和`Data`，你想它的行解析成`Circle`，`Square`或者`Triangle`基于类型列的值对象。

```csharp
var shapes = new List<IShape>();
using (var reader = connection.ExecuteReader("select * from Shapes"))
{
    // Generate a row parser for each type you expect.
    // The generic type <IShape> is what the parser will return.
    // The argument (typeof(*)) is the concrete type to parse.
    var circleParser = reader.GetRowParser<IShape>(typeof(Circle));
    var squareParser = reader.GetRowParser<IShape>(typeof(Square));
    var triangleParser = reader.GetRowParser<IShape>(typeof(Triangle));
  	
  	var typeColumnIndex = reader.GetOrdinal("Type");
  	
  	while (reader.Read())
    {
        IShape shape;
        var type = (ShapeType)reader.GetInt32(typeColumnIndex);
        switch (type)
        {
          	case ShapeType.Circle:
            	shape = circleParser(reader);
            	break;
            case ShapeType.Square:
            	shape = squareParser(reader);
            	break;
          	case ShapeType.Triangle:
            	shape = triangleParser(reader);
            	break;
          	default:
            	throw new NotImplementedException();
        }
      
      	shapes.Add(shape);
    }
}
```

限制和注意事项
---------------------
Dapper缓存关于它运行的每个查询的信息，这样可以快速实现对象并快速处理参数。当前的实现将此信息缓存在ConcurrentDictionary对象中。它存储的对象不会被刷新。如果您不使用参数即时生成SQL字符串，则可能会遇到内存问题。我们可以将字典转换为LRU缓存。

Dapper的简单意味着ORM的许多功能都被剥离。它担心95％的情况，并为您提供大部分时间需要的工具。它不试图解决每个问题。

Dapper会与我的DB提供商一起工作吗？
---------------------
Dapper没有具体的DB实现细节，它适用于所有.NET ADO提供程序，包括SQLite，SQL CE，Firebird，Oracle，MySQL，PostgreSQL和SQL Server。

你有全面的例子列表吗？
---------------------
Dapper 有一个全面的测试项目在 [test project](https://github.com/StackExchange/dapper-dot-net/blob/master/Dapper.Tests)

谁在使用这个？
---------------------
Dapper在生产中使用：

[Stack Overflow](https://stackoverflow.com/), [helpdesk](https://www.jitbit.com/web-helpdesk/)

(if you would like to be listed here let me know)
