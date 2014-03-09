---
layout: post
title: Flask Ajax简记，实现一个类似人人，Fancy的下拉滚动自动加载
description: "Flask ajax技术简记"
category: articles
tags: [Python, Flask]
---

#SQLAlchemy 查询对象转换为JSON

这是一个悲伤的话题，如果要实现一个特别完整的转换实现，需要花点功夫。但是简单的来看，我们只需要把一个查询对象，转换为字典即可，所以我们可以为每一个类定义一个方法，将SQLAlchemy实例转换为字典。  
<br>
这是来自Stack Overflow网友的一个实现，能够满足一般的需求，不能处理datetime.datetime。  
<br>
下面的这个类，是我在一个瞎写的应用中写的一个类，存储豆瓣的影评。  
<br>
{% highlight py %}
class Review_of_Movie(db.Model):
    __tablename__ = 'review_of_movie'
    id = db.Column(db.Integer, primary_key = True)
    title = db.Column(db.String(200))
    author = db.Column(db.String(100))
    author_link = db.Column(db.String(100))
    zan = db.Column(db.Integer)
    link = db.Column(db.String(100), unique = True)
    description = db.Column(db.Text)
    douban_useful = db.Column(db.Integer)

    #one to many
    movie_id = db.Column(db.Integer, db.ForeignKey('movie.id'))

    def __init__(self, title, author, author_link, link, movie_id, description, douban_useful, zan = 0):
        self.title = title
        self.author = author
        self.author_link = author_link
        self.link = link
        self.movie_id = movie_id
        self.zan = zan
        self.description = description
        self.douban_useful = douban_useful

    def __repr__(self):
        return '<review %r>' %self.link

    #将SQLAlchemy实例转换为字典
    def as_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}
{% endhighlight %}
Flask 封装了 json函数，使用flask.jsonify可以将一个字典转换为json格式的数据。  

#JQuery Ajax  

{% highlight html %}
<body>
    <div class = "list">
    {% if reviews|length != 0 %}
        {% for review in reviews %}
        <div class = "item">
            <a href="{{ review.link }}" target = '_Blank'><h3>{{ review.title }}</h3></a>
            <a href="{{ review.author_link }}" target = '_Blank'><pre>{{ review.author }}</pre></a>
            <p>{{ review.description }}
            <hr width=100 size=1 color=#FFCCCC align=left>
            <p><i>Douban total useful : {{ review.douban_useful }}</i></p>
        </div>
        {% endfor %}
    {% endif %}
    </div>
    <div class = "loading"><img src="{{ url_for('static', filename = "loading.gif") }}", width = "80px"></div>


<script type="text/javascript">
    var page = 2;
    var lock = false;
    
    $(function(){
        function getData(i){
            page++;
            $.getJSON($SCRIPT_ROOT + '/aja_' + '{{ kind }}' + '/page' ,{page : i} ,function(data){
                if(data){
                    insertDiv(data);
                    lock = false;
                }
                else{
                    html = "<div class = info><p>No more comments</p></div>";
                    $(".list").append(html);
                    $(".loading").fadeOut('fast');
                }
            })
        }

        function insertDiv(json){
            var html = '';
            //alert(json.reviews.length);
            for(var i = 0; i < json.reviews.length; i++){
                html += "<div class = \"item\">";
                html += "<a href = " + json.reviews[i].link + " target = '_Blank'><h3>" + json.reviews[i].title + "</h3></a>";
                html += "<a href= " + json.reviews[i].author_link + " target = '_Blank'><pre>" + json.reviews[i].author + "</pre></a>";
                html += "<p>" + json.reviews[i].description;
                html += "<hr width=100 size=1 color=#FFCCCC align=left>";
                html += "<p><i>Douban total useful :" +  json.reviews[i].douban_useful + "</i></p>";
                html += "</div>";
            }
            $('.list').append(html);    
        }

        var WindowHeight = $(window).height();

        $(window).scroll(function(){
            var pageHeight = $(document).height();
            var scrollTop = $(window).scrollTop();
            
            if(pageHeight - scrollTop -WindowHeight < WindowHeight/2 && !lock){
                getData(page);
                lock = true;
            }
        });
    });


</script>
</body>
{% endhighlight %}

与PHP， ASP一样，我们使用getJSON函数请求一个url， 并得到一个JSON对象，在脚本中判断鼠标滚动位置，在达到合适的位置时，调用AJAX函数，请求数据即可。  
<br>
可以访问[http://douping.sinaapp.com/movie](http://douping.sinaapp.com/movie)来查看实际效果, 慢慢滚到页底，就可以了。  
<br>
注意请求的url:$SCRIPT_ROOT/aja_movie/page ,这里kind是movie，并且附加一个参数page。放在本例，请求的url就是 http://douping.sinaapp.com/aja_movie/page?2 ...
<br>
最后，只需要一个响应这个url的函数，返回json数据：
{% highlight py %}
from flask import request, abort, jsonify
 
@app.route('/aja_movie/page')
def aja_movie():
    page = request.args.get('page', 0, type=int)
    pages = models.Review_of_Movie.query.paginate(app.config['PAGE']).pages
    if  pages < page:
        abort(404)
    review = models.Review_of_Movie.query.order_by('douban_useful' + ' DESC').paginate(page, app.config['PAGE']).items
    reviews = [i.as_dict() for i in review]
    return jsonify(reviews = reviews, pages = pages, kind = 'movie')
{% endhighlight %}

flask-sqlalchemy提供了*分页*（SQLAlchemy中是不提供的)，所以我们的一切都变得简单