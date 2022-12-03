CreateTime:2019-11-07 17:34:16.0

# No active profile set, falling back to default profiles: default

未找到依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

但是仍然会报
![](https://oscimg.oschina.net/oscnet/573023dfb0b005777f7a89852aae6d0ee9c.jpg)

但是运行成功了，因为采取了默认配置

# Whitelabel Error Page

## 问题分析

什么叫Whitelabel Error Page（也叫白页），就是SpringBoot中HTTP请求出现异常的说明页，如下图

![](https://oscimg.oschina.net/oscnet/4c7a437db13af2d65b9624b91da8a20f5ce.jpg)

白页内容会展示状态码、path、以及错误原因等情况，正式环境更多的是自定义的404页面或者500页面等。

那么现在我们就来了解下什么情况会产生白页的情况，以及如何解决这种情况。我们就以404的情况去了解其原因。

直接来到DispatcherServlet类的protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception方法，其中包含的代码片段

```
  mappedHandler = this.getHandler(processedRequest);
                    if (mappedHandler == null) {
                        this.noHandlerFound(processedRequest, response);
                        return;
                    }
```

```
   protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
        if (pageNotFoundLogger.isWarnEnabled()) {
            pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
        }

        if (this.throwExceptionIfNoHandlerFound) {
            throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request), (new ServletServerHttpRequest(request)).getHeaders());
        } else {
            response.sendError(404);
        }
    }
```

是的，没错，sendError404
![](https://oscimg.oschina.net/oscnet/31372efad06d9b2c17c4f6117f49a25fbe2.jpg)

# 

