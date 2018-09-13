---
layout: post
title: "djangoCBV源码分析"
date: 2017-10-23 
description: "restful"
tag: restful 
---

>app/views.py
>>class loginView(View):

当django项目加载之后，执行以下步骤
>urls.py
>> url(r"^stark/", views.loginView.as_view()),

**views.loginView.as_view()的执行流程**

```python
@classonlymethod
def as_view(cls, **initkwargs):  # 类的方法，cls指的是loginView(那个cls调用，就把那个cls传入)
    # 此时不需要的关心的代码省略
    def view(request, *args, **kwargs):
        self = cls(**initkwargs) # 这个self指的就是loginView的实例对象(根据cls得出)
        if hasattr(self, 'get') and not hasattr(self, 'head'):
            self.head = self.get
        self.request = request
        self.args = args
        self.kwargs = kwargs
        
        return self.dispatch(request, *args, **kwargs) # 返回dispatch的执行结果,这个地方需要注意的是，self.dispatch查找顺序(先找自己，再找类，再找父类......所有我们可以在类中重写dispatch方法)
    
    # 分发，找到loginView中的请求方法，并返回执行结果
    def dispatch(self, request, *args, **kwargs):
        if request.method.lower() in self.http_method_names:
            handler = getattr(self, request.method.lower())
        return handler(request, *args, **kwargs)
```

从源码分析得出：
>url(r"^stark/",views.view),url在项目启动之后就会执行到这里,直到用户访问`stark/`时,就会执行`views.view`,找到`dispatch方法`并执行,`dispatch方法`返回的结果就是用户看到的结果。