# 初识 Django

Django 最初被设计用于具有快速开发需求的新闻类站点，目的是要实现简单快捷的网站开发。以下内容简要介绍了如何使用 Django 实现一个数据库驱动的 Web 应用。

为了让您充分理解 Django 的工作原理，这份文档为您详细描述了相关的技术细节，不过这并不是一份入门教程或者是参考文档（我们当然也为您准备了这些）。如果您想要马上开始一个项目，可以从[实例教程（zh）](part1.md)开始入手，或者直接开始阅读详细的[参考文档](https://docs.djangoproject.com/en/1.8/topics/)。

## 设计模型

Django 无需数据库就可以使用，它提供了[对象关系映射器](https://en.wikipedia.org/wiki/Object-relational_mapping)。通过此技术，你可以使用 Python 代码来描述数据库结构。

你可以使用强大的[数据-模型语句](https://docs.djangoproject.com/en/1.8/topics/db/models/)来描述你的数据模型，这解决了数年以来在数据库模式中的难题。以下是一个简明的例子：

```python3
# mysite/news/models.py

from django.db import models

class Reporter(models.Model):
    full_name = models.CharField(max_length=70)

    def __str__(self):              # Python 2 下请使用 __unicode__
        return self.full_name

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter)

    def __str__(self):              # Python 2 下请使用 __unicode__
        return self.headline
```

## 应用数据模型

然后，运行 Django 命令行工具来创建数据库表。

```bash
$ python manage.py migrate
```

[**migrate**](https://docs.djangoproject.com/en/1.8/ref/django-admin/#django-admin-migrate) 命令会查找所有可用的模型，如果数据库中没有与之对应的表，则会为其自动创建。Django 也提供了其他[更丰富的控制方式](https://docs.djangoproject.com/en/1.8/topics/migrations/)。

## 享用便捷的API

接下来，你就可以使用一套便捷而丰富的 [Python API](https://docs.djangoproject.com/en/1.8/topics/db/queries/) 用于访问你的数据。这些 API 是即时创建的，而不用显式的生成代码。

```pycon
# 从我们的 news 应用里导入模型（译注：记者和文章模型）。
>>> from news.models import Reporter, Article

# 现在系统中还没有记者。
>>> Reporter.objects.all()
[]

# 建立一个 Reporter 对象。
>>> r = Reporter(full_name='John Smith')

# 将对象保存到数据库。save()方法需要被显式调用。
>>> r.save()

# 现在它有了id。
>>> r.id
1

# 现在这个新人记者已经在数据库里了。
>>> Reporter.objects.all()
[<Reporter: John Smith>]

# 数据字段被表示成 Python 对象中的属性。
>>> r.full_name
'John Smith'

# Django 提供了一套丰富的数据库查找 API。
>>> Reporter.objects.get(id=1)
<Reporter: John Smith>
>>> Reporter.objects.get(full_name__startswith='John')
<Reporter: John Smith>
>>> Reporter.objects.get(full_name__contains='mith')
<Reporter: John Smith>
>>> Reporter.objects.get(id=2)
Traceback (most recent call last):
  ...
DoesNotExist: Reporter matching query does not exist.

# 创建一篇文章。
>>> from datetime import date
>>> a = Article(pub_date=date.today(), headline='Django is cool',
...     content='Yeah.', reporter=r)
>>> a.save()

# 现在文章在数据库里了。
>>> Article.objects.all()
[<Article: Django is cool>]

# 通过 Article 对象可以访问与其关联的 Reporter 对象。
>>> r = a.reporter
>>> r.full_name
'John Smith'

# 反之亦然：Reporter 对象也可以访问到与其关联的 Article 对象。
>>> r.article_set.all()
[<Article: Django is cool>]

# 这个 API 接受你所需的查询条件，并在后台高效地执行 JOIN 数据库操作。
# 这个操作会查找所有名为 "John" 的记者发表的文章。
>>> Article.objects.filter(reporter__full_name__startswith='John')
[<Article: Django is cool>]

# 通过变更对象的属性值来修改对象，然后调用save()。
>>> r.full_name = 'Billy Goat'
>>> r.save()

# 用 delete() 来删除一个对象。
>>> r.delete()
```

## 一个动态管理接口：并非徒有其表

当你的模型完成定义，Django 就会自动生成一个专业的生产级[管理接口](https://docs.djangoproject.com/en/1.8/ref/contrib/admin/) - 一个可以认证用户添加、更改和删除对象的 Web 站点。你只需简单的在 admin 站点上注册你的模型即可。

```	python3
# mysite/news/models.py

from django.db import models

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter)
```

```python3
# mysite/news/admin.py

from django.contrib import admin

from . import models

admin.site.register(models.Article)
```

这样设计所遵循的理念是，站点编辑人员可以是你的员工、你的客户、或者就是你自己——而你大概不会乐意去废半天劲创建一个只有内容管理功能的后台管理界面。

创建 Django 应用的典型流程是，先建立数据模型，然后搭建管理站点，之后你的员工（或者客户）就可以向网站里填充数据了。后面我们会谈到如何展示这些数据。

## 规划 URL

简洁优雅的 URL 规划对于一个高质量 Web 应用来说至关重要。Django 推崇优美的 URL 设计，所以不要把诸如 **.php** 和 **.asp** 之类的冗余的后缀放到 URL 里。

为了设计你自己的 URL，你需要创建一个叫做 [URLconf](https://docs.djangoproject.com/en/1.8/topics/http/urls/) 的 Python 模块。这是网站的目录，它包含了一张 URL 和 Python 回调函数之间的映射表。URLconf 也有利于将 Python 代码与 URL 进行解耦（译注：使各个模块分离，独立）。

下面这个 URLconf 适用于前面 **Reporter/Article** 的例子：

```python3
# mysite/news/urls.py

from django.conf.urls import url

from . import views

urlpatterns = [
    url(r'^articles/([0-9]{4})/$', views.year_archive),
    url(r'^articles/([0-9]{4})/([0-9]{2})/$', views.month_archive),
    url(r'^articles/([0-9]{4})/([0-9]{2})/([0-9]+)/$', views.article_detail),
]
```

上面的代码将 URL 的[正则表达式](https://docs.python.org/howto/regex.html)映射到 views 里的回调函数。正则表达式通过括号来提取 URL 中的参数值。当一个用户请求页面时，Django 会遍历这些匹配模式，直至请求的 URL 被成功匹配。（如果 URL 无法匹配，Django 会返回一个404视图。）这个过程会在瞬间完成，因为这些正则表达式在启动时会被自动编译。

一旦正则表达式匹配成功，Django 会导入并调用指定的视图——一个简单地 Python 函数。每个视图都会接收到一个请求（requeset）对象——其中包含了请求元数据——同时还有正则表达式匹配到的那些值。

比如，如果用户请求了“/articles/2005/05/39323/”这样的 URL，Django 就会这样调用函数：**news.views.article_detail(request, '2005', '05', '39323')**。

## 编写视图

视图函数的执行结果只可能有两种：返回一个包含请求页面元素的 [**HttpResponse**](https://docs.djangoproject.com/en/1.8/ref/request-response/#django.http.HttpResponse) 对象，或者是抛出 [**Http404**](https://docs.djangoproject.com/en/1.8/topics/http/views/#django.http.Http404) 这类异常。至于执行过程中的其他的动作则由你决定。

通常来说，一个视图的工作就是：从参数获取数据，装载一个模板，然后将根据获取的数据对模板进行渲染。下面是一个 **year_archive** 的视图样例：

```python3
# mysite/news/views.py

from django.shortcuts import render

from .models import Article

def year_archive(request, year):
    a_list = Article.objects.filter(pub_date__year=year)
    context = {'year': year, 'article_list': a_list}
    return render(request, 'news/year_archive.html', context)
```

这个例子使用了 Django 的[模板系统](https://docs.djangoproject.com/en/1.8/topics/templates/)，它有着很多强大的功能，而且使用起来足够简单，即使不是程序员也可轻松使用。

## 设计模板

上面的代码加载了 **news/year_archive.html** 这个模板。

Django 允许设置搜索模板路径，这样可以最小化模板之间的冗余。在Django设置中，你可以通过 [**DIRS**](https://docs.djangoproject.com/en/1.8/ref/settings/#std:setting-TEMPLATES-DIRS) 参数指定一个路径列表用于检索模板。如果第一个路径中不包含任何模板，就继续检查第二个，以此类推。

**news/year_archive.html** 模板可能是这样的：

```html+django
{% extends "base.html" %}

{% block title %}Articles for {{ year }}{% endblock %}

{% block content %}
<h1>Articles for {{ year }}</h1>

{% for article in article_list %}
    <p>{{ article.headline }}</p>
    <p>By {{ article.reporter.full_name }}</p>
    <p>Published {{ article.pub_date|date:"F j, Y" }}</p>
{% endfor %}
{% endblock %}
```

我们看到变量都被双大括号括起来了。 **{{ article.headline }}** 的意思是“输出 **article** 的 **headline** 属性值”。这个“点”还有更多的用途，比如查找字典键值、查找索引和函数调用。

我们注意到 **{{ article.pub_date|date:"F j, Y" }}** 使用了 Unix 风格的“管道符”（“|”字符）。这是一个模板过滤器，用于过滤变量值。在这里过滤器将一个 Python **datetime** 对象转化为指定的格式（就像 PHP 中的日期函数那样）。

你可以将多个过滤器连在一起使用。你还可以使用你[自定义的模板过滤器](https://docs.djangoproject.com/en/1.8/howto/custom-template-tags/#howto-writing-custom-template-filters)。你甚至可以自己编写[自定义的模板标签](https://docs.djangoproject.com/en/1.8/howto/custom-template-tags/)，相关的 Python 代码会在使用标签时在后台运行。

Django 使用了“模板继承”的概念。这就是 **{% extends "base.html" %}** 的作用。它的含义是“先加载名为 base 的模板，并且用下面的标记块对模板中定义的标记块进行填充”。简而言之，模板继承可以使模板间的冗余内容最小化：每个模板只需包含与其他文档有区别的内容。

下面是 **base.html** 可能的样子，它使用了[静态文件](https://docs.djangoproject.com/en/1.8/howto/static-files/)：

```html+django
# mysite/templates/base.html

{% load staticfiles %}
<html>
<head>
    <title>{% block title %}{% endblock %}</title>
</head>
<body>
    <img src="{% static "images/sitelogo.png" %}" alt="Logo" />
    {% block content %}{% endblock %}
</body>
</html>
```

简而言之，它定义了这个网站的外观（还有网站的 logo），并且给子模板们挖好了可以填的坑。这也让网站的改版变得简单无比——你只需更改这个根模板文件即可。

它也可以用来创建网站的多个版本，多个根模板可以重用同一套子模板。Django 的创始人就用这种技术建立了网站的移动端适配版——只需建立一个新的根模板。

注意，你并不是非得使用 Django 的模板系统，你可以使用其他你喜欢的模板系统。尽管 Django 的模板系统与其模型层能够集成得很好，但这不意味着你必须使用它。同样，你可以不使用 Django 的数据库 API。你可以用其他的数据库抽象层，像是直接读取 XML 文件，亦或直接读取磁盘文件，你可以使用任何方式。Django 的任何组成——模型、视图和模板——都是独立的。

## 这只是冰山一角

以上只是 Django 的功能性概述。Django 还有更多实用的特性：

 - [缓存框架](https://docs.djangoproject.com/en/1.8/topics/cache/)可以与 memcached 或其他后端集成。
 - [聚合器框架](https://docs.djangoproject.com/en/1.8/ref/contrib/syndication/)可以通过简单编写一个 Python 类来推送 RRS 和 Atom。
 - 更多令人心动的自动化管理功能：概述里面仅仅浅尝辄止。

接下来您可以[下载 Django](https://www.djangoproject.com/download/)，阅读[实例教程（zh）][part1.md]，然后加入[我们的社区](https://www.djangoproject.com/community/)！感谢您的关注！
