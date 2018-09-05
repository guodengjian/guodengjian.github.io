---
layout: post
title: "Flask源码启动流程"
date: 2017-01-10 
description: "flask"
tag: Flask
--- 
# werkzeug(第三方wsgi)

## 基本原理实现

```python
from werkzeug.wrappers import Request, Response

@Request.application
def hello(request):
    return Response('hello world')

if __name__ == '__main__':
    from werkzeug.serving import run_simple
    run_simple('localhost', 4000, hello)

```

```python
def run(self, host=None, port=None, debug=None,
            load_dotenv=True, **options):
    from werkzeug.serving import run_simple

    try:
        # self就是app对象，这句话会执行app.__call__
        run_simple(host, port, self, **options)
    finally:
        self._got_first_request = False
```

```python
def __call__(self, environ, start_response):
    # environ 是原始的请求数据
    # start_response，用于设置响应相关数据
    return self.wsgi_app(environ, start_response)

```

```python
def wsgi_app(self, environ, start_response):
    # 1, 获取environ并对其进行再次封装
    # 2， 从environ中获取名称为session的cookie，解密，反序列化
    ctx = self.request_context(environ)
    ”“”
    def request_context(self, environ):
        return RequestContext(self, environ)
    “”“
    error = None
    try:
        try:
            # 3 将上面两个东西放到Local中(包含request/session的ctx对象)
            # 3 ctx.push() 内部还有 app_ctx.push,维护了一个Local(包含了app/g的app_ctx) 
            ctx.push()
            # 4， 执行视图函数
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        except:
            error = sys.exc_info()[1]
            raise
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        # 5， 从Local中获取session，加密，序列化，写入cookie
        # 6， 返回给用户，清空Local中对应的数据
        ctx.auto_pop(error)
```

## 上下文管理

### 程序启动:创建Flask全局变量

```python
# 请求上下文相关的变量
_request_ctx_stack = LocalStack()
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))

# app上下文相关的变量
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
g = LocalProxy(partial(_lookup_app_object, 'g'))
```

#### 维护两个LocalStark对象

`_request_ctx_stack = LocalStack()`,
`_app_ctx_stack = LocalStack()`

### 请求到来

#### 对数据进行封装

```python
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ) # 封装(request,session)
    ctx.push()


def push:
    # app_ctx = AppContext(app)
    app_ctx = self.app.app_context()` # 封装(app,g)
    app_ctx.push()
```

#### 保存数据

```python
# _request_ctx_stack 维护的 Local
# 数据结构
__storage__ = {
    ident:{
        stack:[ctx(request,session)]
        }
}

# _app_ctx_stack 维护的Local
# 数据结构
__storage__ = {
    ident:{
        stack:[app_ctx(app,g)]
    }
}
```

### 视图函数处理

```python
# 去请求上下文中取值 _request_ctx_stack
request.methods
session['xxx']

# 去app上下文中取值 _app_ctx_stack
current_app
g
```

### 结束--清除数据

```python
_request_ctx_stack.pop()
_app_ctx_stack.pop()
```

## 关于g

1. Flask中g的生命周期
    >请求到来，创建g，请求结束，销毁g
2. g和session一样吗？
    >不一样，session会保存在cookie中，下次请求会携带
3. g和全局变量一样吗?
    >不一样，全局变量在程序运行时就被创建，一直存在内存中，程序终止才会销毁
4. 多线程环境下g是否安全？
    >安全，__storage__是按照线程唯一标识存储的
