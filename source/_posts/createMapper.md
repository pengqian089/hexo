---
title: "asp.net core中使用AutoMapper自动创建Mapper"
date: 2021-11-19 16:11:45
tags: "CShharp"
categories: "CSharp"
top: true
summary: "每次在ViewModel转换成Entity类型或者Entity转到ViewModel的时候都要手动创建Mapper，所以有一个想法，在asp.net core中使用AutoMapper自动创建Mapper。"
---


在我的项目中约定，类型、属性名称一致的ViewModel都实现`IMapFrom<T>`接口
``` csharp
public interface IMapFrom<T>
{

}
```
`IMapForm<T>`是个泛型接口，T就是Entity的实际类型

但是ViewModel总会有和Entity不一致的地方，所以当有这种情况时，应该实现`IHaveCustomMapping`接口
```csharp
public interface IHaveCustomMapping
{
    void CreateMappings(MapperConfigurationExpression cfg);
}
```

其中`MapperConfigurationExpression`为AutoMapper提供的配置表达式

关于Entity，约定都必须实现`IBaseEntiy`接口,而且有一个默认的实现
``` csharp
public interface IBaseEntity
{

}

public class BaseEntity : IBaseEntity
{
	[JsonConverter(typeof(ObjectIdConverter))]
	public ObjectId Id { get; set; }
}
```
这里拿时间轴来举例子，关于时间轴的Entity定义：
```csharp
public class Timeline:BaseEntity
{
	public string Title { get; set; }
	
	public string Content { get; set; }
	
	public DateTime Date { get; set; }
	
	public string More { get; set; }
	
	public UserInfo Author { get; set; }	
	
	public DateTime CreateTime { get; set; }
	
	public DateTime LastUpdateTime { get; set; }
}
```
继承了`BaseEntity`，有Id属性。而ViewModel没有特别的需求，所以适用第一种情况(既**约定好的类型、属性名称一致**)，ViewModel直接实现`IMapFrom<Timeline>`接口：
```csharp
public class VmTimeline:IMapFrom<Timeline>
{
	public string Id { get; set; }
	
	public string Title { get; set; }
	
	public string Content { get; set; }
	
	public DateTime Date { get; set; }
	
	public string More { get; set; }
	
	public VmUserInfo Author { get; set; }
	
	public DateTime CreateTime { get; set; }
	
	public DateTime LastUpdateTime { get; set; }
}
```
这里看到Id属性并不一致，`ObjectId`类型和`string`类型，这里也属于约定，会在后续默认处理
其它补充（注意：UserInfo不属于严格意义上的Entity，所以没有实现IBaseEntity接口，[详情查看](/code/Dpz.Core.Public.Entity/User.cs "详情查看")）：
```csharp
//UserInfo type
public class UserInfo
{
	public UserInfo()
	{
		LastAccessTime = DateTime.Now;
	}

	/// <summary>
	/// 账号
	/// </summary>
	public string Id { get; set; }

	/// <summary>
	/// 昵称
	/// </summary>
	public string Name { get; set; }
	
	/// <summary>
	/// 是否启用
	/// </summary>
	public bool? Enable { get; set; }

	/// <summary>
	/// 最后访问时间
	/// </summary>
	public DateTime? LastAccessTime { get; set; }

	/// <summary>
	/// 个性签名
	/// </summary>
	public string Sign { get; set; }

	/// <summary>
	/// 头像
	/// </summary>
	public string Avatar { get; set; }

	/// <summary>
	/// 性别
	/// </summary>
	public Sex Sex { get; set; }

	public string Key { get; set; }
	
	public Permissions? Permissions { get; set; }
}

// UserInfo view model
public class VmUserInfo:IMapFrom<UserInfo>
{

	public VmUserInfo()
	{
		LastAccessTime = DateTime.Now;
	}

	/// <summary>
	/// 账号
	/// </summary>
	public string Id { get; set; }

	/// <summary>
	/// 昵称
	/// </summary>
	public string Name { get; set; }

	/// <summary>
	/// 最后访问时间
	/// </summary>
	public DateTime? LastAccessTime { get; set; }

	/// <summary>
	/// 个性签名
	/// </summary>
	public string Sign { get; set; }

	/// <summary>
	/// 头像
	/// </summary>
	public string Avatar { get; set; }

	/// <summary>
	/// 性别
	/// </summary>
	public Sex Sex { get; set; }

	public Permissions? Permissions { get; set; }
	
	/// <summary>
	/// 是否启用
	/// </summary>
	public bool? Enable { get; set; }
	
	public string Key { get; set; }
}
```
项目中所有的view model都在命名空间(namespace)`Dpz.Core.Public.ViewModel`下，所以根据该命名空间反射出所有view model的类型，并创建Mapper
```csharp
//获取所有view model类型
var types = Assembly.Load("Dpz.Core.Public.ViewModel").GetExportedTypes();
var config = new MapperConfigurationExpression();
//从view model中过滤出默认转换规则的类型，即实现了IMapFrom<>接口的类型
var maps = from x in types
	from y in x.GetInterfaces()
	let faceType = y.GetTypeInfo()
	where faceType.IsGenericType && y.GetGenericTypeDefinition() == typeof(IMapFrom<>) &&
		  !x.IsAbstract && !x.IsInterface
	select new
	{
		Source = y.GetGenericArguments()[0],
		Destination = x
	};
//创建默认规则的Mapper
foreach (var item in maps)
{
	//entity to view model
	config.CreateMap(item.Source, item.Destination);
	//view model to entity
	config.CreateMap(item.Destination, item.Source);
}
```

那么出现属性名称、类型不一致的情况怎么办呢?这里就需要自定义转换规则了。
比如音乐Entity，音乐时长是`long?`类型：
```csharp
public class Music:BaseEntity
{
	/// <summary>
	/// 音乐时长
	/// </summary>
	public long? Duration { get; set; }
}
```
而音乐ViewModel中，音乐时长是`TimeSpan?`类型，那么就要继承`IHaveCustomMapping`接口，实现`CreateMappings()`方法了
```csharp
public class VmMusic:IHaveCustomMapping
{
	public string Id { get; set; }
	/// <summary>
	/// 音乐时长
	/// </summary>
	public TimeSpan? Duration { get; set; }
	
	public void CreateMappings(MapperConfigurationExpression cfg)
	{
		var mapper = new Mapper(new MapperConfiguration(cfg));
		//view model to entity
		cfg.CreateMap<VmMusic, Music>().ConvertUsing((viewModel, _) =>
		{
			if (viewModel == null) return null;
			var success = ObjectId.TryParse(viewModel.Id,out var oid);
			var entity = new Music
			{
				Id = success ? oid : ObjectId.Empty,
				Duration = viewModel.Duration?.Ticks
			};
			return entity;
		});
		//entity to view model
		cfg.CreateMap<Music,VmMusic>().ConvertUsing((entity, _) =>
		{
			if (entity == null) return null;
			var viewModel = new VmMusic
			{
				Id = entity.Id.ToString(),
				Duration = entity.Duration.HasValue ? new TimeSpan(entity.Duration.Value) : null
			};
			return viewModel;
		});
	}
}
```
然后从view model中过滤出实现了`IHaveCustomMapping`接口的类型：
```csharp
//筛选实现了IHaveCustomMapping接口的view model
var customMaps = from x in types
	from y in x.GetInterfaces()
	where typeof(IHaveCustomMapping).IsAssignableFrom(x) &&
		  !x.GetTypeInfo().IsAbstract &&
		  !x.GetTypeInfo().IsInterface
	select (IHaveCustomMapping) Activator.CreateInstance(x);
//再执行IHaveCustomMapping接口的CreateMappings实现方法
foreach (var item in customMaps)
{
	item.CreateMappings(config);
}
```
最后处理一下前文提到的`ObjectId`和`string`类型的问题，还有mongodb的时间类型存储时UTC时间，取值的时候转为本地时间：
```csharp
config.CreateMap<string, ObjectId>().ConstructUsing((x, y) =>
	!string.IsNullOrEmpty(x) && ObjectId.TryParse(x, out var oid) ? oid : ObjectId.Empty);
config.CreateMap<DateTime, DateTime>().ConvertUsing((utc, local, context) =>
{
	if (local == new DateTime())
	{
		if (utc.Kind == DateTimeKind.Utc)
			return utc.ToLocalTime();
		return utc;
	}
	if (local.Kind == DateTimeKind.Local)
		return utc.ToUniversalTime();
	return local;
});

//创建Mapper
var cfg = new MapperConfiguration(config);
IMapper mapper = new Mapper(cfg);
```

这样就不用每次view model to entity或者entity to view model的时候，都需要CreateMapper了，
直接调用`mapper.Map<Entity>(viewModel)`||`mapper.Map<ViewModel>(entity)`