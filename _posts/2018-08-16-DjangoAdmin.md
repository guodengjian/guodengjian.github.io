---
layout: post
title: "Django-Admin"
date: 2018-08-16 
description: "Django-Admin"
tag: Django 
--- 
## django-admin组件

>创建超级用户`python manage.py createsuperuser`

### admin组件的定制
>注册modles：`admin.site.register(modles.UserInfo)`

### admin.ModelAdmin方法提供大量的可定制功能
1. list_display,定制显示的列
```python
class Useradmin(admin.ModelAdmin):
    list_display = [nid,name,...,]

admin.site.register(modles.UserInfo,Useradmin)

```
2. list_display_links,定制点击列可以跳转(显示为a标签)
3. list_filter,定制右侧快速筛选(一般将筛选字段设置为一对多，多对多字段)
4. search_fields,定制模糊搜索功能(可以设置多个字段为搜索对象)
5. action,定制action中的操作
```python

class UserAdmin(admin.ModelAdmin):
 
    # 定制Action行为具体方法
    def func(self, request, queryset):
        print(self, request, queryset)
        print(request.POST.getlist('_selected_action'))
 
    func.short_description = "中文显示自定义Actions"
    # 可以在actions中添加多个方法
    actions = [func, ] 
 
    # Action选项都是在页面上方显示
    actions_on_top = True
    # Action选项都是在页面下方显示
    actions_on_bottom = False
 
    # 是否显示选择个数
    actions_selection_counter = True
    
```


### admin组件源码分析
1. 启动
> 执行每一个app下的admin.py文件
```
#源码，通过这个函数自动执行
def autodiscover():
    autodiscover_modules('admin', register_to=site)
```
2. 注册
>admin.site.register(modles.UserInfo,Useradmin)
>源码：`site = AdminSite()`,django中的注册所用到的site都是同一个对象。---具体参考python的单例模式
```
#源码，注册流程

class AdminSite(object):
    def __init__(self, name='admin'):
        self._registry = {}  # model_class class -> admin_class instance
----------------------------------------
    def register(self, model_or_iterable, admin_class=None, **options):
        if not admin_class:
            admin_class = ModelAdmin #这是一个通用的配置类
------------------------------------------
    self._registry[model] = admin_class(model, self)   # registry中的每一个对象有一个 self.model属性，可以直接调用models数据类

-------------------------------------------- 
class ModelStark(): # 配置类对象
    def __init__(self,model):
        self.model=model


```
总结：
    通过源码可以得出，self.registry中存放的是：models中的数据类，以及配置类的子类(指的是models的自定义配置类)的对象
    
3. 设计URL
>储备知识：
    拓展Url的方式：
```
    url(r'^index/',([
        url(r'^add/$',view.xxx),
        url(r'^delete/$',view.xxx),
        url(r'^change/$',view.xxx),
        ],None,None)
```
>可以顺利访问的url：http://index/add/
>如何拿到app的名字和注册到admin的数据表名(modles的类名)
```
model._meta.app_label, model._meta.model_name
```

源码部分：
```python

    @property
    def urls(self):
        return self.get_urls(), 'admin', self.name
    def get_urls(self):
        for model, model_admin in self._registry.items(): # 注意这里的model，model_admin分别指的是什么
            urlpatterns += [
                    url(r'^%s/%s/' % (model._meta.app_label, model._meta.model_name), include(model_admin.urls)), # 此处是二级路由匹配
                ]
        return urlpatterns
```
说明：admin组件会为每个注册的model生成增删改查四个对应的url，上述源码只是生成了一级url。
二级路由生成源码：
```python
class ModelAdmin(BaseModelAdmin):
    def get_urls(self):
        urlpatterns = [
            url(r'^$', wrap(self.changelist_view), name='%s_%s_changelist' % info),
            url(r'^add/$', wrap(self.add_view), name='%s_%s_add' % info),
            url(r'^(.+)/history/$', wrap(self.history_view), name='%s_%s_history' % info),
            url(r'^(.+)/delete/$', wrap(self.delete_view), name='%s_%s_delete' % info),
            url(r'^(.+)/change/$', wrap(self.change_view), name='%s_%s_change' % info),
            # For backwards compatibility (was the change url before 1.9)
            url(r'^(.+)/$', wrap(RedirectView.as_view(
                pattern_name='%s:%s_%s_change' % ((self.admin_site.name,) + info)
            ))),
        ]
        return urlpatterns
    @property
    def urls(self):
        return self.get_urls()

```