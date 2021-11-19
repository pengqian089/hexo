---
title: 过滤html标签、属性，防止XSS攻击
date: 2021-11-20 04:15:57
tags: "asp.net core"
categories: "asp.net core"
summary: "在asp.net core中使用NLog记录所有访问记录"
---

在我网站中，使用的是sqlite保存的日志记录，所以NLog需要sqlite相关的配置

访问记录表的DDL
``` sql
create table AccessRecord (
  id integer not null constraint AccessRecord_pk primary key autoincrement, 
  AccessUrl nvarchar(1000), 
  ClientIp nvarchar(300), 
  ContentType nvarchar(500), 
  Duration integer, 
  StartTime datetime, 
  Ticks integer, 
  Method nvarchar(50), 
  Referrer nvarchar(1000), 
  UserAgent nvarchar(500), 
  Identity nvarchar(500), 
  Controller nvarchar(100), 
  Action nvarchar(100)
);
create unique index AccessRecord_id_uindex on AccessRecord (id);
```

然后在`nlog.config`中配置
``` xml
<?xml version="1.0" encoding="utf-8"?>

<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
      throwConfigExceptions="true"
      autoReload="true"
>
	<extensions>
        <add assembly="NLog.Web.AspNetCore" />
    </extensions>
	 <targets async="true">
	 	<!-- name:目标名称，规则中配置写向哪个目标 -->
	 	<target xsi:type="Database"
                name="AccessRecord"
                dbProvider="System.Data.SQLite.SQLiteConnection, System.Data.SQLite"
                connectionString="这里配置sqlite的连接字符串">
            <install-command commandType="text">
				<!--安装表的DDL-->
                <text>
                    create table AccessRecord
                    (
                    id integer not null
                    constraint AccessRecord_pk
                    primary key autoincrement,
                    AccessUrl nvarchar(1000),
                    ClientIp nvarchar(300),
                    ContentType nvarchar(500),
                    Duration integer,
                    StartTime datetime,
                    Ticks integer,
                    Method nvarchar(50),
                    Referrer nvarchar(1000),
                    UserAgent nvarchar(500),
                    Identity nvarchar(500),
                    Controller nvarchar(100),
                    Action nvarchar(100)
                    );

                    create unique index AccessRecord_id_uindex
                    on AccessRecord (id);
                </text>
                <ignoreFailures>false</ignoreFailures>
            </install-command>
			<!--添加日志的sql语句-->
            <commandText>
                INSERT INTO AccessRecord (AccessUrl, ClientIp, ContentType, Duration, StartTime, Ticks, Method, Referrer, UserAgent, Identity, Controller, Action)
                VALUES (@AccessUrl, @ClientIp, @ContentType, @Duration, @StartTime, @Ticks, @Method, @Referrer, @UserAgent, @Identity, @Controller, @Action);
            </commandText>
			<!--sql语句中的相关参数-->
			
			<!--访问地址-->
            <parameter name="@AccessUrl" layout="${aspnet-request-url:IncludeHost=true:IncludePort=true:IncludeQueryString=true:IncludeScheme=true}" />
            <!--客户端IP，因为网站启用了CDN，所以根据Http Headers中的X-Forwarede-For获取IP-->
			<parameter name="@ClientIp" layout="${aspnet-request-headers:HeaderNames=X-Forwarded-For:ValuesOnly=true}" />
            <parameter name="@ContentType" layout="${aspnet-request-contenttype}" />
			<!--消耗时间，将会在中间件中计算，并传递给message，所以这里获取message-->
            <parameter name="@Duration" layout="${message}" />
            <!--开始请求时间-->
			<parameter name="@StartTime" layout="${longdate:universalTime=False}" />
            <!--Ticks-->
			<parameter name="@Ticks" layout="${ticks}" />
            <!--请求方法：GET/POST/PUT/DELETE 等-->
			<parameter name="@Method" layout="${aspnet-request-method}" />
			<!--Http Headers中的Referrer-->
            <parameter name="@Referrer" layout="${aspnet-request-referrer}" />
            <!--Http Heasers中的UserAgent-->
			<parameter name="@UserAgent" layout="${aspnet-request-useragent}" />
            <!--请求用户的身份信息，没有身份验证的为空-->
			<parameter name="@Identity" layout="${aspnet-user-identity}" />
			<!--该请求的Controller-->
            <parameter name="@Controller" layout="${aspnet-mvc-controller}" />
            <!--该请求的Action-->
			<parameter name="@Action" layout="${aspnet-mvc-action}" />
        </target>
	 </targets>
	 <!--配置规则-->
	 <rules>
	 	<!--name:looger名称 ，writeTo要写向哪个目标-->
        <logger name="http access record" minlevel="Debug" writeTo="AccessRecord" />
    </rules>
</nlog>
```

然后使用中间件来记录日志，具体实现：
``` csharp
public class HttpRequestRecord
{
	private readonly RequestDelegate _next;

	public HttpRequestRecord(RequestDelegate next)
	{
		_next = next;
	}

	public async Task Invoke(HttpContext httpContext, ILoggerFactory loggerFactory)
	{
		// CheckAuthorizeAttribute 是身份认证的特性，所有访问标注了该特性的action都不记录 。
		// runtask 是Hangfire仪表盘的url，也不记录
		var path = httpContext.Request.Path.Value;
		var endpoint = httpContext.Features.Get<IEndpointFeature>()?.Endpoint;
		var attribute = endpoint?.Metadata.GetMetadata<CheckAuthorizeAttribute>();
		if (attribute != null
			|| (path != null && path.StartsWith("/runtask",StringComparison.CurrentCultureIgnoreCase))
		)
		{
		   
	#if DEBUG
			var temp = attribute == null ? "no authorize" : "permissions:" + attribute.Permissions;
			Console.WriteLine(httpContext.Request.GetEncodedUrl() + "\n\t" + temp);
	#endif
			await _next.Invoke(httpContext);
			return;
		}
		// 创建名为 http access record 的Logger
		var logger = loggerFactory.CreateLogger("http access record");
		//记录访问时长
		var watch = Stopwatch.StartNew();
		await _next.Invoke(httpContext);
		watch.Stop();
		//写入日志
		logger.LogInformation(watch.ElapsedMilliseconds.ToString());
	}
}
```

然后在`Startup.cs`中调用中间件
`app.UseMiddleware<HttpRequestRecord>()`