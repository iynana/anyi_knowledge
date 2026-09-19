![[Pasted image 20260520143416.png]]
1.Tomcat监听到用户发送的HTTP请求（本质Tomcat在处理HTTP请求）（Servlet容器还有Jetty、Undertow）

2.经过过滤器链过滤

**SpringMVC请求流程**：
3.用户发出请求到前端控制器

4.前端控制器请求处理映射器，处理器映射器根据 URL 找到Controller方法

5.经过拦截器链后由处理器适配器调用 Controller 方法

6.最后通过HttpMessageConverter或者视图解析来把返回结果转换为JSON并响应