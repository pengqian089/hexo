---
title: "知乎问题答案收集"
date: 2021-11-19 16:30:21
tags: "CShharp"
categories: "CSharp"
top: true
cover: true
summary: "C#实现知乎问题答案收集"
---
知乎答案收集，首先使用`RestSharp`组件访问，从Nuget获取
``` bash
Install-Package RestSharp -Version 106.12.0
```
### 获取知乎Html
```csharp
IRestSharp client = new RestSharp
{
	// 设置UA 为一个正常的桌面端用户
	UserAgent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36";
};
var request = new RestRequest("知乎提问地址");
var response = await client.ExecuteGetAsync(request);
var html = response.Content;
```
这样提问页的Html就获取到了
### 解析Html和Javascript
接下来准备解析知乎Html和Javascript
使用`AngleSharp`组件解析Html和Javascript
``` bash
Install-Package AngleSharp -Version 1.0.0-alpha-844
```
使用`Newtonsoft.Json`进行json解析
```bash
Install-Package Newtonsoft.Json -Version 13.0.1
```
将获取到的整页Html解析
![](/Home/Image/612ca8426334bcb72527a0e7)
这里保存着一些基本数据，将会解析
``` csharp
// 使用默认的配置和js解析
var config = Configuration.Default.WithJs();
var context = BrowsingContext.New(config);
// 获取到的Html进行解析
var document = await context.OpenAsync(x => x.Content(html));
// 知乎页内有一个id为js-initialData的script标签保存着一些基本数据
var data = document.GetElementById("js-initialData");
if(data != null)
{
	//每页大小，知乎默认每页5条答案
	const int limst = 5;
	//currentPage 当前页默认为1；answerCount 当前问题的答案总数， totalPage 当前问题的总页数
	int currentPage = 1, answerCount = 0, totalPage = 0;
	var source = JObject.Parse(data.InnerHtml);
	var entities = source["initialState"]?["entities"];
	if (entities != null)
	{
		//获取分页基本信息
		if (entities["questions"]?.Children().FirstOrDefault() is JProperty question)
		{
			//从json中获取该问题答案总数
			answerCount = question.Value["answerCount"]?.Value<int>() ?? 0;
			//从json中获取该问题总页数
			totalPage = answerCount % limit == 0 ? answerCount / limit : answerCount / limit + 1;
		}
	}
}
```
### 遍历所有答案
从浏览器的开发者工具中得到分页的API
![](/Home/Image/612cac1a6334bcb72527a171)
从中可以得知API为 `https://www.zhihu.com/api/v4/questions/{问题ID}/answers`

其中有5个参数，抛开`include`参数外,其余参数分别为：

+ `limit` 每页答案数量
+ `limit` 偏移量
+ `platform` 平台，默认为桌面端，即 `desktop`
+ `sort_by` 排序，默认排序规则，即 `default`

接下来遍历答案信息，实现如下：
``` csharp
for (; currentPage <= totalPage; currentPage++)
{
	var pageRequest = new RestRequest("https://www.zhihu.com/api/v4/questions/{知乎问题ID}/answers")
		.AddParameter("include", "data[*].is_normal,admin_closed_comment,reward_info,is_collapsed,annotation_action,annotation_detail,collapse_reason,is_sticky,collapsed_by,suggest_edit,comment_count,can_comment,content,editable_content,attachment,voteup_count,reshipment_settings,comment_permission,created_time,updated_time,review_info,relevant_info,question,excerpt,is_labeled,paid_info,paid_info_content,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp,is_recognized;data[*].mark_infos[*].url;data[*].author.follower_count,vip_info,badge[*].topics;data[*].settings.table_of_content.enabled")
		.AddParameter("limit", limit)
		.AddParameter("offset", limit * currentPage)
		.AddParameter("platform", "desktop")
		.AddParameter("sort_by", "default");
	//使用RestSharp请求API获取信息
	var pageResponse = await client.ExecuteGetAsync(pageRequest);
	//解析json
	var pageData = JObject.Parse(pageResponse.Content);
	var pageItems = pageData["data"] as JArray;
	if (pageItems == null) continue;
	//遍历每页的答案
	foreach (var item in pageItems)
	{
		if (item is JObject answer)
		{
			//获取答案内容
			var answerHtml = answer["content"]?.Value<string>();
		}
	}
}
```
这样，一个知乎提问的所有回答都获取到了

### 也许可以解析答案，获得你想要的
#### 比如获取所有答案图片并下载下来
``` csharp
var answerDocument = await context.OpenAsync(x => x.Content(answerHtml));
// AngleSharp 可以像使用JavaScript的API一样去获取html的Element、Attribute等
var images = answerDocument.QuerySelectorAll("img")；
foreach(var item in images)
{
	var src = item.GetAttribute("data-actualsrc");
	if(!string.IsNullOrEmpty(src))
	{
		var imageRequest = new RestRequest(src);
		var imageResponse = await client.ExecuteGetAsync(pageRequest);
		var stream = new MemoryStream(imageResponse.RawBytes);
		await using var fileStream = new FileStream("要保存的路径的完整限定名", FileMode.Create, FileAccess.Write);
		await stream.CopyToAsync(fileStream);
	}
}
```

或者其它实现 `Todo`