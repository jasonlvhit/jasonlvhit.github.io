---
layout: post
title: SAE Flask坏点简记 
description: "SAE Flask开发中碰到的一些好玩的东西"
category: articles
tags: [Python, Flask]
---

SAE预装了Flask和Flask-SQLAlchemy，我们只需要在config.yaml中配置即可，同样，我在app中使用了lxml，在文件中也需要指明，否则系统会提示no module named lxml  

###静态文件

不同于传统的Flask应用，SAE有自己的静态文件搜索系统。我们需要将静态文件文件夹static放在与config.yaml文件平级的位置。所以文件的结构如下：

{% highlight py %}
/app
    /static
        style.css
        ...
    /app
        /templates
            layout.html
            index.html
            list.html
            ...
        __init__.py
        views.py
        forms,py
        models.py
        ...
    index.wsgi
    config.yaml
    config.py

{% endhighlight %}

在我们引用静态文件时，直接通过http://appname.sinaapp.com/static/style.css 就可以引用我们的文件了 。

###数据库连接

SAE有着自己的一个应用常量系统，在连接数据库时，我们需要引用这些常量，这里面就包括了数据库的所有常量。
{% highlight py %}
import sae.const
 
SQLALCHEMY_DATABASE_URI = 'mysql://' + sae.const.MYSQL_USER + ':' +sae.const.MYSQL_PASS + '@' +sae.const.MYSQL_HOST+ ':' +sae.const.MYSQL_PORT+ '/' + sae.const.MYSQL_DB
{% endhighlight %}

值得注意的是，SAE Database不能从外部接入。

###SAE Flask-SQLAlchemy

大坏点，倒入模块时少一个“点”。  
<br>
如下：
{% highlight py %}
#app/app/__init__.py
#from flask.ext.sqlalchemy import SQLAlchemy
from flaskext.sqlalchemy import SQLAlchemy
{% endhighlight %}
另外一定要在每次请求结束后shutdown数据库连接。

###抓取外网资源

在SAE中，可以直接使用Python中的urllib，urllib2中的函数，但是不能使用代理。也就是说，在Python的接口之下，SAE在背后monkey了自己的实现。但是，SAE抓取速度非常之快。文档中也提及了一种disable SAE实现的方式:
{% highlight py %}
import os
os.environ['disable_fetchurl'] = "1"
{% endhighlight %}