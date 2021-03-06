### 项目层次说明

> Overt.Core.Data v1.0.3.5

#### 1. 项目目录

```
|-Attrubute                     特性
|  |-SubmeterAttrubute.cs       分表标识特性
|
|-Contract                      契约层
|  |-IPropertyAssist.cs         统一的接口定义文件，所有的数据层接口均继承
|  |-IBaseRepository<T>.cs      基础接口类，包含增删改查，以及继承IDbRepository
|
|-DataContext                   数据库连接层
|  |-DataContext.cs             数据库连接工厂类
|  |-DataContextConfig.cs       数据库配置信息获取类
|
|-Enums                         枚举
|  |-DatabaseType.cs            数据库类型
|  |-FieldSortType.cs           Asc / Desc
|
|-Expressions                   表达式解析器（略）
|
|-Extensions                    Dapper扩展
|  |-Dapper.Extension.cs        扩展类
|
|-Params                        参数实体
|  |-OrderByField.cs            排序类
|
|-Repositry                     契约实现层
|  |-PropertyAssist.cs          抽象类，继承IPropertyAssist，virtual 
|  |-BaseRepository<T>.cs       抽象类，实现 IBaseRepository<T>，virtual
```


#### 2. 版本及支持

> * Nuget版本：V 1.0.3.5
> * 框架支持： Framework4.6 - NetStandard 2.0
> * 数据库支持：MySql / SqlServer / SQLite [使用详见下文]


#### 3. 项目依赖

> * Framework 4.6

```
Dapper 1.50.2  
MySql.Data 6.10.4
System.Data.SQLite 1.0.106
System.ComponentModel.DataAnnotations
```

> * NetStandard 2.0

```
Dapper 1.50.2  
MySql.Data 6.10.4
Microsoft.Data.Sqlite 2.0.0
```


### 使用

#### 1. 配置信息

> * 支持 IConfiguration 对象注入
> * 支持默认配置文件appsettings.json
> * Core(DbType=MySql|SqlServer|SQLite): 

```
[mysql]: DataSource=127.0.0.1;Database=TestDb;uid=root;pwd=123456;Allow Zero Datetime=True;DbType=MySql
[sqlserver]: Data Source=127.0.0.1;Initial Catalog=TestDb;Persist Security Info=True;User ID=sa;Password=123456;DbType=SqlServer  
[sqlite]: Data Source=testdb.db;DbType=SQLite; // 默认使用App_Data目录
```

> * Framework: 正常的连接字符串，使用PrividerName来区分数据库类型


#### 2. Nuget包引用

```
Install-Package Overt.Core.Data -Version 1.0.3.5
```


#### 3. 约定

> * 服务层契约需继承IBaseRepository<>
> * 服务层契约实现实现自定义契约，并继承BaseRepository<T>
> * 外部执行Sql使用 BaseRepository 中 Execute 方法 内部管理连接

```
return await Execute(async (connection) =>
{
    // 执行方法
}, true);
```

#### 4. 分表实现

```
IBaseRepository的实现

// 自定义重写TableNameFunc
// 使用GetTableName()可获取到实际的表名
// 内置方法均从上述方法中获取表名，默认表名为实体定义的[Table("主表名")]
public override Func<string> TableNameFunc => () =>
{
    var tableName = $"{GetMainTableName()}_{DateTime.Now.ToString("yyyyMMdd")}";
    return tableName;
};
        
// 重写创建表的脚本，可在调用过程中，自动创建表，一般用于动态分表
public override Func<string, string> CreateScriptFunc => (tableName) =>
{
    return "创建表的Sql脚本"; // 将在增删改操作中执行，查询操作中，表不存在则直接返回空数据  
}        
```

#### 5. 分库实现

> * 连接字符串配置中以key - value 模式定义，key使用默认的读写分离关键字【master / secondary】表示写入连接字符串和读取连接字符串
> * 前缀添加 xxx.即可简单定义不同数据库，代码中，对于IBaseRepository的实现，在构造函数中直接常量定义 xxx，如下所示
```
using Overt.Core.Data;
using Overt.User.Domain.Contracts;
using Overt.User.Domain.Entities;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Dapper;
using System.Linq;
using Microsoft.Extensions.Configuration;

namespace Overt.User.Domain.Repositories
{
    public class UserRepository : BaseRepository<UserEntity>, IUserRepository
    {
        public UserRepository(IConfiguration configuration) 
            : base(configuration, "xxx") // dbStoreKey 可用于不同数据库切换，连接字符串key前缀：xxx.master xxx.secondary
        {
        }
    }
}

```


#### 6. Lambda表达式支持

> * ==
> * !
> * !=
> * null
> * In
> * Equals
> * Contains
> * StartWith
> * EndWith
> * &&
> * ||

##### 支持案例
```
var ary = new string[]{ "1", "2" };
var list = _repository.GetList(1, 1, oo=>ary.Contains(oo.UserId));

var ary = new List<string>(){ "1", "2" };
var list = _repository.GetList(1, 1, oo=>ary.Contains(oo.UserId));

var key = "abc";
var list = _repository.GetList(1, 1, oo=>oo.UserName.Contains(key));

var list = _repository.GetList(1, 1, oo=>oo.UserName.Equals(key));

var list = _repository.GetList(1, 1, oo=>oo.UserName.Contains("abc"));

var list = _repository.GetList(1, 1, oo=>oo.UserName.StartWith("abc"));

var list = _repository.GetList(1, 1, oo=>oo.UserName.EndWith("abc"));

var list = _repository.GetList(1, 1, oo=>ary.Contains(oo.UserId) && !IsSex);

var list = _repository.GetList(1, 1, oo=>oo.UserName != null);

var list = _repository.GetList(1, 1, oo=>oo.UserName == null);
```

##### 不支持案例

~~var list = _repository.GetList(1, 1, oo=>"test".Contains(oo.UserName));~~  
~~var list = _repository.GetList(1, 1, oo=>oo.UserName.IndexOf("abc") > -1);~~

---
