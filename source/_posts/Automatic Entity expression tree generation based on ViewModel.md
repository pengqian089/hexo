---
title: 过滤html标签、属性，防止XSS攻击
date: 2021-11-20 04:18:12
tags: "asp.net core"
categories: "asp.net core"
summary: "关于根据ViewModel自动生成Entity表达式树的尝试"
---

网站用的Mongodb，Entity和ViewModel分离

项目中所有Entity都实现`IBaseEntity`接口
``` csharp
public interface IBaseEntity
{

}

//IBaseEntity 默认实现
public class BaseEntity : IBaseEntity
{
	[JsonConverter(typeof(ObjectIdConverter))]
	public ObjectId Id { get; set; }
}
```

项目中所有的ViewModel都实现`IMapFrom<T>`接口或者`IHaveCustomMapping`接口，目的是使用AutoMapper创建Mapper，这里只实现`IMapFrom<T>`的表达式生成（[详见](/article/read/612cdc0f6334bcb72527a88d.html "详见")）。
``` csharp
public interface IMapFrom<T>
{

}

public interface IHaveCustomMapping
{
	void CreateMappings(MapperConfigurationExpression cfg);
}
```

首先定义比较的枚举：
``` csharp
public enum ExpressComparison
{
	/// <summary>
	/// 等于
	/// </summary>
	Equal,
	/// <summary>
	/// 包含
	/// </summary>
	Contains,
	/// <summary>
	/// 不等于
	/// </summary>
	NoEqual,
	/// <summary>
	/// 大于
	/// </summary>
	Gt,
	/// <summary>
	/// 小于
	/// </summary>
	Lt,
	/// <summary>
	/// 大于或等于
	/// </summary>
	GtOrEqual,
	/// <summary>
	/// 小于或等于
	/// </summary>
	LtOrEqual,

	/// <summary>
	/// 自定义规则
	/// </summary>
	Custom
}
```

再定义特性，用来标注（`Attribute`）ViewModel中需要在Entity中比较的属性（`Property`）
``` csharp
//只能标注与属性上
[AttributeUsage(AttributeTargets.Property)]
public class ViewModelLabelAttribute:Attribute
{
	public ExpressComparison Comparison { get; private set; }

	/// <summary>
	/// 生成表达式树，需要比较的方式
	/// </summary>
	/// <param name="expressComparison">比较方式</param>
	public ViewModelLabelAttribute(ExpressComparison expressComparison)
	{
		Comparison = expressComparison;
	}
}
```

处理需要查询的字段
``` csharp
class QueryField
{
	//ViewModel上标注的比较方式
	public ViewModelLabelAttribute Label { get; set; }

	//比较的属性。因为是默认方式，所以ViewModel和Entity上的属性类型应该一致
	public PropertyInfo Property { get; set; }

	//比较的值
	public object Value { get; set; }
	
	//处理各种比较方式
	Expression HandleField(ParameterExpression parameter)
	{
		//如果需要比较的值为null，那么返回Empty表达式
		if (Value == null) return Expression.Empty();
		//约定如果需要比较的值是枚举类型，但是值为-1的情况下，过滤掉
		if (Property.PropertyType.IsEnum && Convert.ToInt32(Value) == -1)
			return Expression.Empty();
		//属性成员表达式
		var memberExpr = Expression.Property(parameter, Property);
		switch (Label.Comparison)
		{
			//相等的比较
			case ExpressComparison.Equal:
				return Expression.Equal(memberExpr, Expression.Constant(Value));
			//大于
			case ExpressComparison.Gt:
				return Expression.GreaterThan(memberExpr, Expression.Constant(Value));
			//大于或者等于
			case ExpressComparison.GtOrEqual:
				return Expression.GreaterThanOrEqual(memberExpr, Expression.Constant(Value));
			//小于
			case ExpressComparison.Lt:
				return Expression.LessThan(memberExpr, Expression.Constant(Value));
			//小于或者等于
			case ExpressComparison.LtOrEqual:
				return Expression.LessThanOrEqual(memberExpr, Expression.Constant(Value));
			//不等于
			case ExpressComparison.NoEqual:
				return Expression.NotEqual(memberExpr, Expression.Constant(Value));
			//包含
			case ExpressComparison.Contains:
				//如果比较的类型是string类型，那么反射获取string类型的Contains方法，生成表达式
				if (Property.PropertyType == typeof(string) && Value is string)
				{
					var containsMethod = typeof(string).GetMethod("Contains", new[] { typeof(string) });
					if (containsMethod != null)
					{
						return Expression.Call(memberExpr, containsMethod, Expression.Constant(Value));
					}
				}
				//如果比较的类型是集合，那么反射获取linq中的Contains方法，生成表达式
				if (typeof(IEnumerable).IsAssignableFrom(Property.PropertyType))
				{
					var containsMethod = typeof(Enumerable)
						.GetMethods(BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic)
						.FirstOrDefault(x =>
							x.Name == "Contains" && x.ReturnType == typeof(bool) &&
							x.GetParameters().Length == 2);

					if (containsMethod != null)
					{
						containsMethod = containsMethod.MakeGenericMethod(Value.GetType());
						return Expression.Call(containsMethod, memberExpr, Expression.Constant(Value));
					}
				}
				//Todo 暂未实现
				throw new NotImplementedException("string、IEnumerable以外类型的Contains表达式未实现！");
			case ExpressComparison.Custom:
				//Todo 暂未实现
				throw new NotImplementedException("暂无自定义实现");
		}
		//默认为Empty表达式
		return Expression.Empty();
	}
}
```

扩展一个ViewMode的方法，来生成Entity类型的表达式树
``` csharp
/*
* TEntity，泛型，Entity的类型
* IMapForm<TEntity>，ViewModel的类型，ViewModel实现了IMapForm<IBaseEntiy>接口
*/
public static Expression<Func<TEntity, bool>> GenerateExpressTree<TEntity>(this IMapFrom<TEntity> mapFrom)
	where TEntity : IBaseEntity, new()
{
	//表达式树形参
	var parameter = Expression.Parameter(typeof(TEntity), "__q");
	//获取ViewModel的所有属性
	var queryFieldsExpression = (from x in mapFrom.GetType().GetProperties()
		//获取属性所标注的ViewModelLabelAttribute特性
		let attr = x.GetCustomAttribute<ViewModelLabelAttribute>()
		where attr != null //过滤掉没有标注的特性
		let field = new QueryField
		{
			Label = attr,//特性
			Property = typeof(TEntity).GetProperty(x.Name),//属性
			Value = x.GetValue(mapFrom)//属性值
		}
		//处理查询字段
		select field.HandleField(parameter)).Where(x => x.NodeType != ExpressionType.Default).ToList();
	//创建一个默认表达式树
	var expression =
		(Expression)Expression
			.Constant(true);
	//遍历所有需要查询的字段
	foreach (var expr in queryFieldsExpression)
	{
		//第一个表达式树节点赋值给expression
		if (queryFieldsExpression.IndexOf(expr) == 0)
		{
			expression = expr;
			continue;
		}
		//其余表达式树节点以 && 形式生成
		expression = Expression.AndAlso(expression, expr);
	}
	//生成Lambda表达式树
	return Expression.Lambda<Func<TEntity, bool>>(expression, parameter);
}
```

单元测试

定义Entity和ViewModel
```
public enum Sex
{
	Man,
	Wuman
}

public class VmUser : IMapFrom<UserTest>
{
	[ViewModelLabel(ExpressComparison.Equal)]
	public string Id { get; set; }

	[ViewModelLabel(ExpressComparison.Contains)]
	public string Name { get; set; }

	public string Avatar { get; set; }

	[ViewModelLabel(ExpressComparison.Equal)]
	public Sex Sex { get; set; }

	[ViewModelLabel(ExpressComparison.Contains)]
	public string Groups { get; set; }
}

public class UserTest : IBaseEntity
{
	public string Id { get; set; }

	public string Name { get; set; }

	public string Avatar { get; set; }

	public Sex Sex { get; set; }

	public string[] Groups { get; set; }
}
```

直接从内存从加载相关数据
``` csharp
var list = new List<UserTest>
{
	new UserTest
	{
		Avatar = "http://localhost:9527/images/laola.png",
		Id = "pengqian",
		Name = "彭迁",
		Sex = Sex.Man,
		Groups = new[] {"111", "222", "333"}
	},
	new UserTest
	{
		Avatar = "http://localhost:9527/images/laola.png",
		Id = "pengqian",
		Name = "dpz",
		Sex = Sex.Man,
		Groups = new[] {"444", "555", "666"}
	},
	new UserTest
	{
		Avatar = "http://localhost:9527/images/laola.png",
		Id = "xx",
		Name = "xx",
		Sex = Sex.Wuman,
		Groups = new[] {"777", "888", "999"}
	}
};

var viewModel = new VmUser
{
	Avatar = "http://localhost:9527/images/laola.png",
	Id = "pengqian",
	Name = "d",
	Sex = Sex.Man,
	Groups = "666"
};
```

单元测试方法
```csharp
[TestMethod]
public void 生成表达式树单元测试()
{
	var expr = (Expression<Func<UserTest, bool>>) viewModel.GenerateExpressTree();
	var result = list.Where(expr.Compile()).ToList();
	Assert.IsTrue(result.Count > 0);
	result.ForEach(x => Console.WriteLine($"Avatar:{x.Avatar},Id:{x.Id},Name:{x.Name},Sex:{x.Sex},Group:{string.Join("-",x.Group)}"));
}
```

```
output:
Avatar:http://localhost:9527/images/laola.png,Id:pengqian,Name:dpz,Sex:Man,Group:444-555-666
```

结语：当然这只是一个尝试，一个思路，也可以有其它实现方式，比如使用模板之类的实现