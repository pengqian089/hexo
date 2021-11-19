---
title: 过滤html标签、属性，防止XSS攻击
date: 2021-11-19 23:21:57
tags: "CShharp"
categories: "CSharp"
summary: "使用HtmlSanitizer过滤<b>html</b><b><mark>标签</mark></b> <b><mark>属性</mark></b>，以防止<b><mark>XSS</mark></b>攻击"
---

虽然我网站中编辑的内容大部分都是由Markdown语法生成的，但是还是有一小部分功能Markdown无法满足，使用了原始的Html，所以考虑需要过滤一些Html的标签、属性来保证访问的安全性，防止XSS攻击。

有关XSS攻击，可查看[StackExChange](https://security.stackexchange.com/questions/1368/how-does-xss-work "StackExChange")。

首先从Nuget从下载过滤组件
``` bash
Install-Package HtmlSanitizer -Version 6.0.441
```

[HtmlSanitizer](https://github.com/mganss/HtmlSanitizer "HtmlSanitizer")中设定了一些默认允许的标签
`a, abbr, acronym, address, area, article, aside, b, bdi, big, blockquote, br, button, caption, center, cite, code, col, colgroup, data, datalist, dd, del, details, dfn, dir, div, dl, dt, em, fieldset, figcaption, figure, font, footer, form, h1, h2, h3, h4, h5, h6, header, hr, i, img, input, ins, kbd, keygen, label, legend, li, main, map, mark, menu, menuitem, meter, nav, ol, optgroup, option, output, p, pre, progress, q, rp, rt, ruby, s, samp, section, select, small, span, strike, strong, sub, summary, sup, table, tbody, td, textarea, tfoot, th, thead, time, tr, tt, u, ul, var, wbr`

默认允许的属性
`abbr, accept, accept-charset, accesskey, action, align, alt, autocomplete, autosave, axis, bgcolor, border, cellpadding, cellspacing, challenge, char, charoff, charset, checked, cite, clear, color, cols, colspan, compact, contenteditable, coords, datetime, dir, disabled, draggable, dropzone, enctype, for, frame, headers, height, high, href, hreflang, hspace, ismap, keytype, label, lang, list, longdesc, low, max, maxlength, media, method, min, multiple, name, nohref, noshade, novalidate, nowrap, open, optimum, pattern, placeholder, prompt, pubdate, radiogroup, readonly, rel, required, rev, reversed, rows, rowspan, rules, scope, selected, shape, size, span, spellcheck, src, start, step, style, summary, tabindex, target, title, type, usemap, valign, value, vspace, width, wrap`

默认允许的CSS属性
`background, background-attachment, background-clip, background-color, background-image, background-origin, background-position, background-repeat, background-repeat-x, background-repeat-y, background-size, border, border-bottom, border-bottom-color, border-bottom-left-radius, border-bottom-right-radius, border-bottom-style, border-bottom-width, border-collapse, border-color, border-image, border-image-outset, border-image-repeat, border-image-slice, border-image-source, border-image-width, border-left, border-left-color, border-left-style, border-left-width, border-radius, border-right, border-right-color, border-right-style, border-right-width, border-spacing, border-style, border-top, border-top-color, border-top-left-radius, border-top-right-radius, border-top-style, border-top-width, border-width, bottom, caption-side, clear, clip, color, content, counter-increment, counter-reset, cursor, direction, display, empty-cells, float, font, font-family, font-feature-settings, font-kerning, font-language-override, font-size, font-size-adjust, font-stretch, font-style, font-synthesis, font-variant, font-variant-alternates, font-variant-caps, font-variant-east-asian, font-variant-ligatures, font-variant-numeric, font-variant-position, font-weight, height, left, letter-spacing, line-height, list-style, list-style-image, list-style-position, list-style-type, margin, margin-bottom, margin-left, margin-right, margin-top, max-height, max-width, min-height, min-width, opacity, orphans, outline, outline-color, outline-offset, outline-style, outline-width, overflow, overflow-wrap, overflow-x, overflow-y, padding, padding-bottom, padding-left, padding-right, padding-top, page-break-after, page-break-before, page-break-inside, quotes, right, table-layout, text-align, text-decoration, text-decoration-color, text-decoration-line, text-decoration-skip, text-decoration-style, text-indent, text-transform, top, unicode-bidi, vertical-align, visibility, white-space, widows, width, word-spacing, z-index`

默认的URI方案
`http, https`

URI默认包含的属性
`action, background, dynsrc, href, lowsrc, src`

某些文本内容也将会进行变更，比如：
+ `4 < 5` 变成 `4 &lt; 5`
+ `<SPAN>test</p>` 变成 `<span>test<p></p></span>`
+ `<span title='test'>test</span>` 变成 `<span title="test">test</span>`

如果默认允许的标签、属性不满足自己的需求，可以自行添加或者移除
``` csharp
var sanitizer = new HtmlSanitizer();
//从默认的Html标签的属性中添加额外的属性
sanitizer.AllowedAttributes.Add("id");
//从默认的Html标签的属性中移除额外的属性
sanitizer.AllowedAttributes.Remove("color");

//从默认的标签中添加额外的标签
sanitizer.AllowedTags.Add("audio");
//从默认的标签中移除额外的标签
sanitizer.AllowedTags.Remove("textarea");

//添加默认css属性
sanitizer.AllowedCssProperties.Add("outline")；
//移除默认css属性
sanitizer.AllowedCssProperties.Remove("widows")；


//允许 mailto 超链接
sanitizer.AllowedSchemes.Add("mailto");
```

因为录音，文章区的图片延迟加载，以及不允许阅读区不允许出现表单控件，所以我在`appsetting.json`自定义了一些规则：
``` json
  "XSS": {
    "Tags": {
      "Add": [
        "audio"
      ],
      "Remove": [
        "textarea",
        "input",
        "select",
        "button",
        "form",
        "optgroup",
        "option"
      ]
    },
    "Attributes": {
      "Add": [
        "loading",
        "controls",
        "preload",
        "id",
        "name"
      ],
      "Remove": []
    },
    "CSSProperties": {
      "Add": [],
      "Remove": []
    }
  }
```

定义C#类型
``` csharp
public class CrossSiteRule
{
	public Allowed Tags { get; set; }
	
	public Allowed Attributes { get; set; }
	
	public Allowed CssProperties { get; set; }
}

public class Allowed
{
	public string [] Add { get; set; }
	
	public string [] Remove { get; set; }
}
```

写一个`IConfiguration`的扩展方法，读取配置，进行添加或移除默认规则的操作
``` csharp
public static IHtmlSanitizer CustomHtmlSanitizer(this IConfiguration configuration)
{
	var sanitizer = new HtmlSanitizer();

	var rule = configuration.GetSection("XSS").Get<CrossSiteRule>();
	if (rule != null)
	{
		if (rule.Attributes?.Add?.Any() == true)
		{
			rule.Attributes.Add.ForEach(x => sanitizer.AllowedAttributes.Add(x));
		}

		if (rule.Attributes?.Remove?.Any() == true)
		{
			rule.Attributes.Remove.ForEach(x => sanitizer.AllowedAttributes.Remove(x));
		}

		if (rule.Tags?.Add?.Any() == true)
		{
			rule.Tags.Add.ForEach(x => sanitizer.AllowedTags.Add(x));
		}

		if (rule.Tags?.Remove?.Any() == true)
		{
			rule.Tags.Remove.ForEach(x => sanitizer.AllowedTags.Remove(x));
		}

		if (rule.CssProperties?.Add?.Any() == true)
		{
			rule.CssProperties.Add.ForEach(x => sanitizer.AllowedCssProperties.Add(x));
		}

		if (rule.CssProperties?.Remove?.Any() == true)
		{
			rule.CssProperties.Remove.ForEach(x => sanitizer.AllowedAttributes.Remove(x));
		}
	}

	return sanitizer;
}
```

接下来在显示编辑区域的位置过滤原始的Html
``` csharp
var sanitizer = _configuration.CustomHtmlSanitizer();
var context = BrowsingContext.New(Configuration.Default);
var html = await context.OpenAsync(x => x.Content(blog.BlogContents));
var links = html.GetElementsByTagName("a");
links.ForEach(x =>
{
	var href = x.GetAttribute("href");
	if (href != null && !href.StartsWith("javascript", StringComparison.CurrentCultureIgnoreCase))
	{
		//把所有超链接设置为以新标签页的形式打开
		x.SetAttribute("target","_blank");
	}
});
blog.BlogContents = sanitizer.Sanitize(html.Body?.InnerHtml ?? "");
```