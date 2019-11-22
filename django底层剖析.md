## 一、对于web服务的理解

`web`服务应该至少包含两个模块：`web`服务器和`web`应用程序，两个模块在功能和代码上解耦。 `web`服务器负责处理`socket`调用、`http`数据解析和封装等底层操作。 `web`应用程序负责业务处理、数据增删改查、页面渲染/生成等高层操作。 `web`服务器一旦接收到`http`请求，经过自身的解析后就会调用`web`应用程序来处理业务逻辑，并得到`web`应用程序的返回值，再经过自身的封装发送给客户端。

## 二、对于wsgi协议的理解

在`web`服务器和`web`应用程序之间需要定义一个接口规则，这也叫协议，用于明确两者之间以什么样的形式交互数据。即：**web服务器应该以什么样的形式调用web应用程序，而web应用程序又应该定义成什么形式。**

`python`下规定的`web`服务的接口规则叫做`wsgi`，`wsgi`协议对于`server`和`application`的接口定义如下：

对于`server`调用规则的定义：

```
response = application(environ, start_response) 
```

对于`application`接口编码的定义：

```
def application(environ, start_response):
    status = '200 OK'
    response_headers = [('Content-Type', 'text/plain'),]
    start_response(status, response_headers)
    
    return [b'hello',]
```

只要是遵从如上形式进一步封装`server`和`application`的，均称为实现了`wsgi`协议的`server/application`。

`python`内置提供了一个`wsigref`模块用于提供`server`，但是只能用于开发测试，`django`框架就是使用此模块作为它的`server`部分，也就说，实际生产中的`server`部分，还需要使用其他模块来实现。

任何`web`框架，可能没有实现`server`部分或者只实现一个简单的`server`，但是，`web`框架肯定实现了`application`部分。**application部分完成了对一次请求的全流程处理**，其中各环节都可以提供丰富的功能，比如请求和响应对象的封装、`model/template`的实现、中间件的实现等，让我们可以更加细粒度的控制请求/响应的流程。

![img](https://img2018.cnblogs.com/blog/1381809/201810/1381809-20181008160101110-1762669968.jpg)

## 三、自定义一个简单的基于wsgi协议的web框架

`django`框架的`server`部分由`python`内置的`wsgiref`模块提供，我们只需要编写`application`应用程序部分。

```
from wsgiref.simple_server import make_server

def app(environ, start_response):  # wsgi协议规定的application部分的编码形式，可在此基础上扩展
    status = '200 OK'
    respones_headers = []
    
    start_response(status, response_headers)
    return [b'hello',]

if __name__ == '__main__':
    httpd = make_server('127.0.0.1', 8080, app)
    httpd.serve_forever()
```

用以下图示表示简单的web请求流程架构（伪代码） ![img](https://img2018.cnblogs.com/blog/1381809/201810/1381809-20181008224931282-2126835772.jpg)

**web服务器就像是一颗心脏不停的跳动，驱动整个web系统为用户提供http访问服务，并调用application返回响应**

## 四、django中的server实现

`django`使用的底层`server`模块是基于`python`内置的`wsgiref`模块中的`simple_server`，每次`django`的启动都会执行如下`run`函数。`run`函数中会执行`serve_forever`，此步骤将会启动`socket_server`的无限循环，此时就可以循环提供请求服务，每次客户端请求到来，服务端就执行`django`提供的`application`模块。

```
django`中`server`的启动----`django.core.servers.basehttp.py
"""
HTTP server that implements the Python WSGI protocol (PEP 333, rev 1.21).

Based on wsgiref.simple_server which is part of the standard library since 2.5.

This is a simple server for use in testing or debugging Django apps. It hasn't
been reviewed for security issues. DON'T USE IT FOR PRODUCTION USE!
"""

def run(addr, port, wsgi_handler, ipv6=False, threading=False, server_cls=WSGIServer):
    server_address = (addr, port)
    if threading:
        httpd_cls = type('WSGIServer', (socketserver.ThreadingMixIn, server_cls), {})
    else:
        httpd_cls = server_cls
    httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
    if threading:
        # ThreadingMixIn.daemon_threads indicates how threads will behave on an
        # abrupt shutdown; like quitting the server by the user or restarting
        # by the auto-reloader. True means the server will not wait for thread
        # termination before it quits. This will make auto-reloader faster
        # and will prevent the need to kill the server manually if a thread
        # isn't terminating correctly.
        httpd.daemon_threads = True
    httpd.set_app(wsgi_handler)
    httpd.serve_forever()
```

底层无限循环将作为`web`服务的主要驱动----`socektserver.py`

```
def serve_forever(self, poll_interval=0.5):
    """Handle one request at a time until shutdown.

    Polls for shutdown every poll_interval seconds. Ignores
    self.timeout. If you need to do periodic tasks, do them in
    another thread.
    """
    self.__is_shut_down.clear()
    try:
        # XXX: Consider using another file descriptor or connecting to the
        # socket to wake this up instead of polling. Polling reduces our
        # responsiveness to a shutdown request and wastes cpu at all other
        # times.
        with _ServerSelector() as selector:
            selector.register(self, selectors.EVENT_READ)

            while not self.__shutdown_request:
                ready = selector.select(poll_interval)
                if ready:
                    self._handle_request_noblock()

                self.service_actions()
    finally:
        self.__shutdown_request = False
        self.__is_shut_down.set()
```

　　

```
server`对于`application`的调用----`wsgiref.handlers.py
def run(self, application):
    """Invoke the application"""
    # Note to self: don't move the close()!  Asynchronous servers shouldn't
    # call close() from finish_response(), so if you close() anywhere but
    # the double-error branch here, you'll break asynchronous servers by
    # prematurely closing.  Async servers must return from 'run()' without
    # closing if there might still be output to iterate over.
    try:
        self.setup_environ()
        self.result = application(self.environ, self.start_response)
        self.finish_response()
    except:
        try:
            self.handle_error()
        except:
            # If we get an error handling an error, just give up already!
            self.close()
            raise   # ...and let the actual server figure it out.
```

## 五、django中的application实现

`django`的`application`模块是通过`WSGIHandler`的一个实例来提供的，此实例可以被`call`，然后根据`wsgi`的接口规则传入`environ`和`start_response`。所以本质上，`django`就是使用的内置`python`提供的`wsgiref.simple_server`再对`application`进行丰富的封装。大部分的`django`编码工作都在`application`部分。

```
application`的编码定义部分----`django.core.handlers.wsgi.py
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        request = self.request_class(environ)
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = list(response.items())
        for c in response.cookies.values():
            response_headers.append(('Set-Cookie', c.output(header='')))
        start_response(status, response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```

## 六、django的底层调用链

![img](https://img2018.cnblogs.com/blog/1381809/201810/1381809-20181006175526955-1852462410.jpg)

## 七、总结

`web`服务是基于`socket`的高层服务，所以`web`服务必须含有`web`服务器这一模块。 `web`服务需要动态渲染数据，需要中间件来丰富功能，需要封装和解析来处理数据，所以`web`服务必须含有`web`应用程序这一模块。 `web`框架是一种工具集，封装了各种功能的底层代码，提供给我们方便开发的接口。但不论是哪一种框架，它们的底层原理基本都是一致的。 应该深入学习、研究一个`web`框架，精通一门框架的实现原理和设计理念。

 