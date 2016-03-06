---
layout: post
title: "Github Pages的Jekyll 集成 简单文章搜索"
description: ""
category: "note"
tags: [jekyll,web]
author: ksh
---
{% include JB/setup %}

在用Github Pages + Jekyll-Bootstrap(JB) 搭建博客的过程中, 发现虽然在 JB 的导航栏上有一个搜索框，但是这个搜索框实际上是装样子的，根本没有实际作用
不过能够搜索博客中的文章也是一个挺有用的功能，所以还想实现这个功能的
查找解决方案后发现，jekyll本身虽然有几个做搜索的plugin，但是因为 github pages限制了plugin的使用，所以这些利用部分后端的搜索都不能使用，只能依靠纯前端做文章搜索。

后来找到了这个库基本符合需求: [https://github.com/christian-fei/Simple-Jekyll-Search](https://github.com/christian-fei/Simple-Jekyll-Search){:target="_blank"}
于是基于这个库做了一些修改实现了这个功能

<!--more-->

## 安装 simple-jekyll-search

这是一个前端插件 所以可以使用 `bower` 来安装

```bash
bower install --save simple-jekyll-search
```

## 开始

将下述代码放到根目录中名为`search.json`的文件中，这个文件的是用来提供数据源给前端搜索插件的

```
---
---
{% raw %}[
  {% for post in site.posts %}
    {
      "title"    : "{{ post.title | escape }}",
      "category" : "{{ post.category }}",
      "tags"     : "{{ post.tags | join: ', ' }}",
      "url"      : "{{ site.baseurl }}{{ post.url }}",
      "date"     : "{{ post.date | date: "%B %e, %Y"}}",
      "author"   : "{{ post.author }}"
    } {% unless forloop.last %},{% endunless %}
  {% endfor %}"
]{% endraw %}
```

在 `_includes/themes/bootstrap-3/default.html` 中添加 script 引用

```html
{% raw %}<script src="{{ site.baseurl }}/bower_components/simple-jekyll-search/dest/jekyll-search.js" type="text/javascript"></script>{% endraw %}
```

修改 `_includes/themes/bootstrap-3/default.html` 的搜索表单

```html
<form class="navbar-form navbar-right" role="search" action="/search" target="_blank">
  <div class="form-group">
    <input type="text" class="form-control" name="query" placeholder="Search">
  </div>
  <button type="submit" class="btn btn-default">Submit</button>
</form>
```

## 添加搜索页面

在根目录下添加名为 `search.html` 的文件，内容如下:

```html
---
layout: default
title: Search
tagline: search my post
---

<div id="search-container">
	<input type="text" id="search-input" placeholder="search...">
</div>

<div id="results-container"></div>


<script type="text/javascript">
	document.onreadystatechange = function () {
		if (document.readyState == 'complete') {  	
	        SimpleJekyllSearch({
	          searchInput: document.getElementById('search-input'),
	          resultsContainer: document.getElementById('results-container'),
	          json: '/search.json',
	          searchResultTemplate: '<h3><a href="{url}">{title}</a></h3><p><small><strong>{date}</strong> . {category} . <span   class="author">@{author}</span><a href="{url}#disqus_thread"></a></small></p>',
	          noResultsText: '<h3>No results found</h3>',
	          limit: 10,
	          fuzzy: false
	        });
		}
	};
</script>
```

因为 `SimpleJekyllSearch` 是在 `search.html` 的内容之后加载的，所以需要通过 `onreadystatechange` 等 `jekyll-search.js` 加载完成之后才能执行 SimpleJekyllSearch配置设置，`searchInput`，`resultsContainer`，`json` 这三个参数是必须设置的，其他为可选

现在，在搜索页面`/search`的输入框中已经可以进行搜索并且查看结果了，但是要怎么在导航栏的搜索框进行搜索呢？
在前面修改 `default.html` 后，如导航栏搜索 "web" 会转跳到 `/search?query=web`，只要能够在加载页面时从URL的search中读取query参数并进行搜索就可以了

将 `onreadystatechange` 绑定的函数修改为如下:

```html
<script type="text/javascript">
	document.onreadystatechange = function () {
		if (document.readyState == 'complete') {
			var urlParam = function(name){
			    var results = new RegExp('[\?&]' + name + '=([^&#]*)').exec(window.location.href);
			    if (results==null){
			       return null;
			    }
			    else{
			       return results[1] || 0;
			    }
			}
			var search_input = $('#search-input');
			var query = urlParam('query');
			if(query){
				search_input.val(query);	
			}

			SimpleJekyllSearch({
			  searchInput: document.getElementById('search-input'),
			  resultsContainer: document.getElementById('results-container'),
			  json: '/search.json',
			  searchResultTemplate: '<h3><a href="{url}">{title}</a></h3><p><small><strong>{date}</strong> . {category} . <span   class="author">@{author}</span><a href="{url}#disqus_thread"></a></small></p>',
			  noResultsText: '<h3>No results found</h3>',
			  limit: 10,
			  fuzzy: false
			});

			search_input.bind('keyup', (function(){
				$this = $(this);
				history.replaceState({}, window.title, window.location.pathname+'?query='+$this.val())
			}))
		}
	};
</script>
```

首先定义了一个 `urlParam` 函数，来获取URL中的参数，然后设置到输入框中，并且绑定输入框的 keyup 事件将输入框中的最新值通过 `history.replaceState` 在reload的情况下更新到地址栏，这样的目的是可以方便在后退时回到之前的搜索关键词
目前虽然在加载时设置了输入框的值，但是并不能够触发搜索，这里需要修改 `jekyll-search.js` 才行

```javascript
function registerInput(){
    options.searchInput.addEventListener('keyup', function(e){
      var key = e.which
      var query = e.target.value
      console.log('query='+query);
      if( isWhitelistedKey(key) && isValidQuery(query) ) {
        emptyResultsContainer();
        render( repository.search(query) );
      }
    })
    var query = options.searchInput.value;
    /* Add start */
    if(query) {
      emptyResultsContainer();
      render( repository.search(query) );
    }
    /* Add end */
  }
```


