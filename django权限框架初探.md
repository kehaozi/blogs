
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
> 那么，通过ContentType和Permissions的一对多的关联，就让Permissions也能记录到权限和哪个app的哪个models的对应关系
> 这些是理解Django的权限实现的要点

**下面是官网文档对Permission这个Models的描述**
> class models.Permission
> **name** 
> Required. 255 characters or fewer. Example: 'Can vote'.
> 
> **content_type** Required. A reference to the django_content_type database table, which contains a record for each installed model.
> 
> **codename** Required. 100 characters or fewer. Example: 'can_vote'.

> 其中, content_type上面已经讲过
> name和codename 的字段值通常一样，用于描述一个权限，比如 'can  add_XXX'



> 下面简单的介绍Django Admin原生页面里给用户授权/取消权限的操作的实现方法
> 这是通过User().user_permissions.add()方法实现的

> django.crontrib.auth.models 里的User类，继承关系是

![Diagram](https://img2018.cnblogs.com/blog/1212342/201810/1212342-20181003165141697-855565892.png)

> 可以看到， AbstractUser分别从djaogo.contrib.auth.models的PermissionMixin类和django.crontrib.auth.base_user的AbstractBaseUser类中继承了 uer_permissions 属性和is_active属性，当然，AbstractUser也分别从PermissionMixin和AbstractBaseUser中继承了其他属性和方法，具体看UML图或源码
> 而 user_permissionss属性，我们看看源码都干了什么

``` python?linenums
class PermissionsMixin(models.Model):
    """
    A mixin class that adds the fields and methods necessary to support
    Django's Group and Permission model using the ModelBackend.
    """
......
......
......
user_permissions = models.ManyToManyField(
        Permission,
        verbose_name=_('user permissions'),
        blank=True,
        help_text=_('Specific permissions for this user.'),
        related_name="user_set",
        related_query_name="user",
    )

    class Meta:
        abstract = True
```

**==可以看到，user_permissions这个属性做的事情是:==**
> 由于PermissionMixin这个类有meta属性，abstract = True,那么实际不会创建者这张表
> 当在实际的代码中，通过user类创建了一个用户，那么这个实例化的user，称之为uerA
> userA具有了user_permissions这个属性
> 而这个user_permissions属性,从上面的源码可以看到，实际上是和Permission类建立了多对多的关系
> 即给userA与权限表建立了多对多的关系
> 因为一种权限可以分配给多个用户，而一个用户也可以具有多种类型的权限
> 具体的授权，就可以用ManyToMany类型的add/remove方法进行某种/某些权限的授权/删除权限

具体支持这些方法

> myuser.groups.set([group_list])
> myuser.groups.add(group, group, ...)
> myuser.groups.remove(group,group, ...) myuser.groups.clear()
> myuser.user_permissions.set([permission_list])
> myuser.user_permissions.add(permission, permission, ...)
> myuser.user_permissions.remove(permission, permission, ...)
> myuser.user_permissions.clear()


#### Django中对于对象权限的相关说明

[Django中对于对象级别权限的处理](https://docs.djangoproject.com/en/2.1/topics/auth/customizing/#handling-object-permissions)

> **==Handling object permissions==**
> 
> Django's permission framework has a foundation for object permissions,
> though there is no implementation for it in the core. That means that
> checking for object permissions will always return False or an empty
> list (depending on the check performed). An authentication backend
> will receive the keyword parameters obj and user_obj for each object
> related authorization method and can return the object level
> permission as appropriate.
> 
> 可见，Django没有为用户具体的去实现处理一个具体的对象权限的逻辑，Django默认对所有传入的obj返回None或者空列表
> 用户想要具体的去处理对象权限，则要自己去编写处理对象权限的方法了

**==为了确认官方文档所说的，我们看看源码的实现==**

**==django.auth.models==**

``` python?linenums
......
......
......
# A few helper functions for common logic between User and AnonymousUser.
def _user_get_all_permissions(user, obj):
    permissions = set()
    for backend in auth.get_backends():
        if hasattr(backend, "get_all_permissions"):
            permissions.update(backend.get_all_permissions(user, obj))
    return permissions


def _user_has_perm(user, perm, obj):
    """
    A backend can raise `PermissionDenied` to short-circuit permission checking.
    """
    for backend in auth.get_backends():
        if not hasattr(backend, 'has_perm'):
            continue
        try:
            if backend.has_perm(user, perm, obj):
                return True
        except PermissionDenied:
            return False
    return False
	......
	......
	......
```
	
**可以看到，django把has_perm()的具体实现交给了backends里的方法
那么，我们接着去看backends里的has_perm方法**


**==django.auth.backends==**

![Diagram](https://img2018.cnblogs.com/blog/1212342/201810/1212342-20181003165720363-2015762920.png)


``` python?linenums

def _get_permissions(self, user_obj, obj, from_name):
"""
        Returns the permissions of `user_obj` from `from_name`. `from_name` can
        be either "group" or "user" to return permissions from
        `_get_group_permissions` or `_get_user_permissions` respectively.
"""
	if not user_obj.is_active or user_obj.is_anonymous or obj is not None:
            return set()

        perm_cache_name = '_%s_perm_cache' % from_name
        if not hasattr(user_obj, perm_cache_name):
            if user_obj.is_superuser:
                perms = Permission.objects.all()
            else:
                perms = getattr(self, '_get_%s_permissions' % from_name)(user_obj)
            perms = perms.values_list('content_type__app_label', 'codename').order_by()
            setattr(user_obj, perm_cache_name, set("%s.%s" % (ct, name) for ct, name in perms))
        return getattr(user_obj, perm_cache_name)
```

**从源码里，可以清晰的看到，当传入的obj不为空的时候，直接返回空的集合	，就是说，django.auth并不具体去处理对象权限**

也就是说，要自己在django的基础上实现对于对象权限的管理的话，需要自己在backends里去具体实现对于对象权限的处理，即重写has_perm()方法。

> 而在Django里，就是利用django.auth提供的方法进行权限的判断(是否具有需要的权限)/权限的设定授予
	
	
**==一个简单的django原生权限的例子==**

```python?linenums
def vote(request):
	if not request.user.is_authenticated():
 		return HttpResponse("You admin is not valid.")
	else:
 		if request.user.has_perm('polls.can_vote')):
  			# 写有关vote访问数据的逻辑
			......
			......
			......
	 	else:
  			return HttpResponse("You can't vote in this poll.")
```

> Django提供了 has_perm() 方法和 @permission_required() 装饰器，都可以用来检查是否具有指定需要的权限，然后，针对检查结果，开发者可以编写自己的进一步的基于权限检查结果的逻辑。
