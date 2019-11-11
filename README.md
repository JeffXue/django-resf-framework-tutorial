# 学习纪要
本仓库及纪要主要用来快速熟悉如何在django上使用REST framework

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

---

## requests and responses：请求及响应

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
