# 学习纪要
本仓库及纪要主要用来快速熟悉如何在django上使用REST framework

---

## quick start：快速开始

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

### 序列化
新建`tutorial/quickstart/serializers.py的文件`，用于序列化数据
```python
from django.contrib.auth.models import User, Group
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ('url', 'username', 'email', 'groups')


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ('url', 'name')
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
