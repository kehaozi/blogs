,---
title: django权限系统之一:Django自带的权限系统的机制
tags: django,auth
grammar_cjkRuby: true
---

# 认证和授权

**==认证==**
> 鉴别用户是否是有效的合法用户

**==授权==**
> 给合法用户授予需要的权限，非法用户给与相应的重定向登陆或其他提示等操作

**==django中的认证和授权==**
> django.auth模块统一提供了认证和授权的功能


## Django中的认证和授权

### 配置Django的认证和授权生效

``` python?linenums
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',   # line1
    'django.contrib.contenttypes',  # line2
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'app01',
]
```

> 其中line1 line2就是配置django内置的权限系统需要的配置项
> line2 用于将模型和权限关联

> 同时，还需要在middleware中配置

``` python?linenums
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware', # line1
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware', # line 2
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```
> line1 是用于管理跨请求的会话的中间件
> line2 是用于授权管理的中间件

### User对象

#### User对象的默认属性

 - username 
 - password 
 - email first_name 
 - last_name
 
 #### 创建User实例的方法
 

``` python?linenums
from django.contrib.auth.models import User
user = User.objects.create_user('john', 'lennon@thebeatles.com', 'johnpassword')
user.last_name = 'Lennon'
user.save()
```

> 此时，user是User对象，已经保存到到数据库。 你可以继续改变其属性. 如果你想改变其他字段。

#### 验证用户

**==is_authenticated() 方法是鉴别用户的方法,不是鉴定授权的方法==**

> 用django.auth的authenticate方法进行用户的验证

验证用户
authenticate(request=None, **credentials)[source]

使用authenticate()验证一组凭据。 它将用户凭据作为关键字参数（username和password为其默认值）来针对每个配置的认证后端进行检查，并为用户凭据在后台验证为合法时返回一个User对象。 如果传入的用户凭据在所有后端验证为无效，或者后端抛出PermissionDenied异常，则返回None。 例如：

``` python?linenums
from django.contrib.auth import authenticate
user = authenticate(username='john', password='secret')
if user is not None:
    # A backend authenticated the credentials
else:
    # No backend authenticated the credentials
```

#### Django中授权的实现

> 对于一个权限系统，某个用户/用户组/角色的权限，其实就是用一个字符串去描述
> 权限系统对于别的系统提供的功能: 告诉别的系统, 某个用户/用户组/角色有某个/某些权限
> 比如，can_add这个权限描述告诉别的系统是具有can_add这个权限
> 至于具有can_add这个权限的用户/用户组/角色究竟能实现什么操作，不是权限系统关注的事情
> 这是具体的业务逻辑去做的
> 权限系统提供接口告诉别的系统:在别的系统里将要进行某种操作的用户/用户组/角色拥有该操作的(进一步还包括该操作对应的资源)权限吗?
> 至于有，是干什么，没有，怎么给用户返回，是业务系统逻辑去实现的事情了
> 权限系统关注于:提供方式查询某个用户/用户组/角色是否具有某种操作/某种资源的权限，并提供可以给用户/用户组/角色进行授予某种权限/资源的功能

**==Django中如何实现权限功能的==**

> 在Django中，使用一张表记录这些权限的
>  代码位置: django.contrib.auth.models
>  其中的Permissions类，定义了权限

> 下面是Permissions类的部分主要代码
``` python?linenums
@python_2_unicode_compatible
class Permission(models.Model):
    """
    The permissions system provides a way to assign permissions to specific
    users and groups of users.

    The permission system is used by the Django admin site, but may also be
    useful in your own code. The Django admin site uses permissions as follows:

        - The "add" permission limits the user's ability to view the "add" form
          and add an object.
        - The "change" permission limits a user's ability to view the change
          list, view the "change" form and change an object.
        - The "delete" permission limits the ability to delete an object.

    Permissions are set globally per type of object, not per specific object
    instance. It is possible to say "Mary may change news stories," but it's
    not currently possible to say "Mary may change news stories, but only the
    ones she created herself" or "Mary may only change news stories that have a
    certain status or publication date."

    Three basic permissions -- add, change and delete -- are automatically
    created for each Django model.
    """
    name = models.CharField(_('name'), max_length=255)
    content_type = models.ForeignKey(
        ContentType,
        models.CASCADE,
        verbose_name=_('content type'),
    )
    codename = models.CharField(_('codename'), max_length=100)
    objects = PermissionManager()

    class Meta:
        verbose_name = _('permission')
        verbose_name_plural = _('permissions')
        unique_together = (('content_type', 'codename'),)
        ordering = ('content_type__app_label', 'content_type__model',
                    'codename')

    def __str__(self):
        return "%s | %s | %s" % (
            six.text_type(self.content_type.app_label),
            six.text_type(self.content_type),
            six.text_type(self.name))

    ......
	......
	......
```

**==关键点==**

> 从源码的注释部分可以了解到
> Django的权限是针对models级别的，做不到object级别的权限
> ContentType这张表和Permissions这张表做了一对多的关联
> 因为一个models可以被授予多种权限
> 而ContentType这张表记录了app和models的对应关系
> 那么，通过ContentType和Permissions的一对多的关联，就让
