# 学习纪要
本仓库及纪要主要用来快速熟悉如何在django上使用REST framework

[TOC]

---

## quick start：快速开始
本章节主要从一个极小的应用去讲解所涉及到的相关内容
- 初始化django应用
- 使用`serializers.HyperlinkedModelSerializer`设置序列化类
- 使用`viewsets.ModelViewSet`设置视图集
- 使用`routers.DefaultRouter()`设置路由
- 配置`REST_FRAMEWORK`全局参数

### 初始化项目
```shell
# 新建项目及应用
django-admin.py startproject tutorial
cd tutorial
django-admin.py startapp quickstart

# 同步数据库，默认为db.sqllite3
python manage.py migrate

# 创建超级用户，即管理员用户
python manage.py createsuperuser

```

配置对应的app，打开`tutorial/settings.py`，添加对应的quickstart app
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'quickstart',
]
```

### 序列化
新建`tutorial/quickstart/serializers.py的文件`，用于序列化数据
```python
from django.contrib.auth.models import User, Group
from rest_framework import serializers


# 继承HyperlinkedModelSerializer
class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        # 指定model
        model = User
        # 指定字段
        fields = ('url', 'username', 'email', 'groups')


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ('url', 'name')
```

### 配置视图
打开`tutorial/quickstart/views.py`文件

```python
from django.contrib.auth.models import User, Group
from rest_framework import viewsets
from tutorial.quickstart.serializers import UserSerializer, GroupSerializer

# 集成ModelViewSet
class UserViewSet(viewsets.ModelViewSet):
    """
    允许用户查看或编辑的API路径。
    """
    # 指定queryset数据
    queryset = User.objects.all().order_by('-date_joined')
    # 指定序列化类
    serializer_class = UserSerializer


class GroupViewSet(viewsets.ModelViewSet):
    """
    允许组查看或编辑的API路径。
    """
    queryset = Group.objects.all()
    serializer_class = GroupSerializer
```

### 设置URL
打开`tutorial/urls.py`

```python
from django.conf.urls import url, include
from rest_framework import routers
from tutorial.quickstart import views

# 使用路由器类注册视图来自动生成API的URL conf
router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# 使用自动URL路由连接我们的API。
# 另外，我们还包括支持浏览器浏览API的登录URL。
urlpatterns = [
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

### 设置

```python
INSTALLED_APPS = (
    ...
    'rest_framework',
)

REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAdminUser',
    ],
    'PAGE_SIZE': 10
}
```

---


## serialization：序列化
本教程将介绍如何创建一个简单的收录代码高亮展示的Web API。这个过程中, 将会介绍组成Rest框架的各个组件，并让你全面了解各个组件是如何一起工作的。
- 使用models.Model实现model类
- 使用serializers.ModelSerializer实现序列化类
- 使用序列化类编写常规views

### 新建snippet应用
```shell
python manage.py startapp snippets
```

添加app应用
```
INSTALLED_APPS = (
    ...
    'snippets',
)
```

### 创建model

```python
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```


```shell
python manage.py makemigrations snippets
python manage.py migrate
```

## 创建序列化类

为web api提供一种代码片段实例序列化和反序列化为`json`之类的表示形式的方式
在`snippets`目录下创建一个名为`serializers.py`

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        根据提供的验证过的数据创建并返回一个新的`Snippet`实例。
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        根据提供的验证过的数据更新和返回一个已经存在的`Snippet`实例。
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```


### 使用ModelSerializers

SnippetSerializer类中重复了很多包含在Snippet模型类（model）中的信息。如果能保证我们的代码整洁，那就更好了。
就像Django提供了Form类和ModelForm类一样，REST framework包括Serializer类和ModelSerializer类，
使用ModelSerializer类重构我们的序列化类

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')

# 序列一个非常棒的属性就是可以通过打印序列化器类实例的结构(representation)查看它的所有字段。
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
# SnippetSerializer():
#    id = IntegerField(label='ID', read_only=True)
#    title = CharField(allow_blank=True, max_length=100, required=False)
#    code = CharField(style={'base_template': 'textarea.html'})
#    linenos = BooleanField(required=False)
#    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
#    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```

ModelSerializer类并不会做任何特别神奇的事情，它们只是创建序列化器类的快捷方式：
- 一组自动确定的字段。
- 默认简单实现的create()和update()方法。

### 使用序列化实现views
目标不会使用任何REST框架的其他功能，只需将视图作为常规django视图编写,`snippets/views.py`
```python
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer

class JSONResponse(HttpResponse):
    """
    An HttpResponse that renders its content into JSON.
    """
    def __init__(self, data, **kwargs):
        content = JSONRenderer().render(data)
        kwargs['content_type'] = 'application/json'
        super(JSONResponse, self).__init__(content, **kwargs)


@csrf_exempt
def snippet_list(request):
    """
    列出所有的code snippet，或创建一个新的snippet。
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JSONResponse(serializer.data)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data, status=201)
        return JSONResponse(serializer.errors, status=400)


@csrf_exempt
def snippet_detail(request, pk):
    """
    获取，更新或删除一个 code snippet。
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JSONResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data)
        return JSONResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)

```

### 添加URL
`snippets/urls.py`
```python
from snippets import views

urlpatterns = [
    url(r'^/$', views.snippet_list),
    url(r'^/(?P<pk>[0-9]+)/$', views.snippet_detail),
]
```

`tutorial/urls.py`
```python
from django.conf.urls import url, include

urlpatterns = [
    url(r'^', include('snippets.urls')),
]
```

到目前为止做得不错，我们有一个与Django的Forms API非常相似序列化API，和一些常规的Django视图

---

## requests and responses：请求及响应

现在开始，将真正开始接触REST框架的核心

### 请求对象（Request objects）
REST框架引入了一个扩展了常规`HttpRequest`的`Request`对象，并提供了更灵活的请求解析。`Request`对象的核心功能是`request.data`属性，它与`request.POST`类似，但对于使用Web API更为有用
```python
request.POST  # 只处理表单数据  只适用于'POST'方法
request.data  # 处理任意数据  适用于'POST'，'PUT'和'PATCH'方法
```

### 响应对象 （Response objects）
REST框架还引入了一个`Response`对象，这是一种获取未渲染（unrendered）内容的`TemplateResponse`类型，并使用内容协商来确定返回给客户端的正确内容类型。
```python
return Response(data)  # 渲染成客户端请求的内容类型。
```

### 状态码（Status）
视图（views）中使用纯数字的HTTP 状态码并不总是那么容易被理解。而且如果错误代码出错，很容易被忽略。REST框架为`statu`s模块中的每个状态代码（如`HTTP_400_BAD_REQUEST`）提供更明确的标识符。使用它们来代替纯数字的HTTP状态码是个很好的主意。


### 包装（wrapping）API视图
REST框架提供了两个可用于编写API视图的包装器（warppers）
- 基于函数视图的`@api_view`装饰器
- 基于类视图的`APIView`类
这些包装器提供了一些功能，例如确保你在视图中接收到`Request`实例，并将上下文添加到`Response`，以便可以执行内容协商。
包装器还提供了诸如在适当时候返回`405 Method Not Allowed`响应，并处理在使用格式错误的输入来访问`request.data`时发生的任何`ParseError`异常

### 组合在一起
使用新的组件重构之前的`snippets views`，之前的`view.py`中不再需要`JSONResponse`
```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    列出所有的snippets，或者创建一个新的snippet。
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    获取，更新或删除一个snippet实例。
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)

```


### 添加可选的格式后缀
为了充分利用的响应不再与单一内容类型连接，可以为API路径添加对格式后缀的支持。
使用格式后缀给我们明确指定了给定格式的URL，这意味着API将能够处理诸如`http://example.com/api/items/4.json`之类的URL

视图中添加一个format的关键字参数
```python
def snippet_list(request, format=None):
    ...

def snippet_detail(request, pk, format=None):
    ...

```

`urls.py`文件，给现在的URL后面添加一组`format_suffix_patterns`

```python
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```


---

## class based views：基于类的视图

---

## authentication and permissions：认证和权限

---

## relationships and hyperlinked apis：关联与超链接API

---

## viewsets and routers：视图集和路由器

---

## schemas and client libraries：概要和客户端库

---
