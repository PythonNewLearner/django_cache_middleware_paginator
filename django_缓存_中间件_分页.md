**缓存**：

​	提升服务器响应速度

​	将执行的操作数据存贮下来，在一定的时间内，再次获取数据的时候，直接从缓存中获取

​	比较理想的方案，缓存使用内存级缓存

​	Django内置缓存框架

​	

创建数据库缓存table:    python manage.py createcachetable my_cache_table





pip install django-redis

pip install django-redis-cache



安装redis-server : sudo apt install redis-server



setting.py

```python
CACHES = {
    'default':{    
        'BACKEND':'django.core.cache.backends.db.DatabaseCache', #数据库缓存
        'LOCATION': 'my_cache_table',
        'TIMEOUT': 60*5,
    },
    'redis_backend':{
        "BACKEND":"django_redis.cache.RedisCache", #redis缓存
        "LOCATION":"redis://127.0.0.1:6379/1",
        "OPTION":{
            "CLIENT_CLASS":"django_redis.client.DefaultClient",
        }
    }
}
```



views.py

```python
def news(request):
    cache = caches['redis_backend']   #选用redis缓存
    result = cache.get('news')
    if result:
        return HttpResponse(result)

    news_ls = []
    for i in range(10):
        news_ls.append("病毒蔓延{}".format(i))
    sleep(5)
    context = {'news_ls':news_ls}

    resp = render(request,'news.html',context=context)

    cache.set('news', resp.content,timeout=60)
    return resp

@cache_page(60,cache='default')  #选用default缓存
def jokes(request):
    sleep(5)
    return HttpResponse("Jokes")
```





**中间件**

​	可以实现统计功能

​		比如统计IP，统计浏览器等

​	可以实现权重控制

​		黑名单

​		白名单

​	可以实现反爬虫

​		频率控制



```python
from django.utils.deprecation import MiddlewareMixin
from django.http import HttpResponse
from django.core.cache import cache
import time
class HelloMiddle(MiddlewareMixin):

    def process_request(self,request):
        ip = request.META.get('REMOTE_ADDR')
        # if request.path == '/app/getphone/':
        #     if ip == '127.0.0.1':
        #         return HttpResponse("恭喜你免费获得手机")
        #
        # if request.path == '/app/search/':
        #     result = cache.get(ip)
        #     if result:
        #         return HttpResponse("访问过于频繁")
        #     cache.set(ip,ip,timeout=10)

        #需求：60秒内只限制访问10次;60秒内访问超过30次，列入黑名单
        black_list = cache.get('black',[])
        if ip in black_list:
            return HttpResponse("臭傻x你被封杀了")

        requests = cache.get(ip,[])
        if requests and time.time() - requests[-1] > 60:
            requests.pop()

        requests.insert(0, time.time())
        cache.set(ip, requests, timeout=60)

        if len(requests) > 30:
            black_list.append(ip)
            cache.set('black',black_list,timeout=60*2)
            return HttpResponse("臭傻x,你被封杀了")

        if len(requests) > 10:
            return HttpResponse("傻x不要刷网页")



```



​	

界面友好化

有报错直接redirect 到home

```python
    def process_exception(self,requests,exception):
        print(requests,exception)
        return redirect('app:home')
```







中间件调用顺序

​	在中间件列表中， 如果有返回的中间件就返回，下面的中间件不会执行

中间件切点：

​	process_request

​	process_view

​	process_template-response

​	process_response

​	process_exception





**分页**

可以自己写：

​	

```python
def getstudents(request):
    students = Student.objects.all()
    page = int(request.GET.get('page',1))
    per_page = int(request.GET.get('per_page',10))
    context = {'students':students[(page-1)*per_page:page*per_page]}
    return render(request,'student_list.html',context=context)
```





使用django自带的分页器:

```python
def studentpage(request):
    students = Student.objects.all()
    page = int(request.GET.get('page',1))
    per_page = int(request.GET.get('per_page',10))

    paginator = Paginator(students,per_page)
    page_object = paginator.page(page)
    context = {
        'page_object':page_object,
        'page_range': paginator.page_range,
                }

    return render(request,'page.html',context=context)
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Page</title>
</head>
<body>

{%for student in page_object.object_list %}
    <li>{{ student.s_name }}</li>
{% endfor %}


{% for page_index in page_range %}
<a href="{% url 'app:studentpage' %}?page={{ page_index }}">{{ page_index }}</a>
{% endfor %}

</body>
</html>
```



作业：

​	分页

​		页码超过10个的时候，中间的页面使用...代替

​		显示的时候只显示前五页和后五页