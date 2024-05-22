**一、SpringMVC工作流程图**
![](https://raw.githubusercontent.com/danmuking/image/main/205e6196508af6c5c6f054ae32df115c.webp)
**二、SpringMVC执行流程介绍**
1、用户通过客户端发送请求到服务端，请求会被前端控制器DispatcherServlet进行拦截。
2、DispatcherServlet收到请求之后，调用处理器映射器HandlerMapping。
3、HandlerMapping处理器映射器根据请求url找到具体的处理器，生成处理器执行链HandlerExecutionChain(包括处理器对象和处理器拦截器)一并返回给DispatcherServlet。
4、DispatcherServlet根据处理器Handler获取处理器适配器HandlerAdapter，执行HandlerAdapter处理一系列的操作，如：参数封装，数据格式转换，数据验证等操作。
5、执行处理器Handler(Controller，也叫页面控制器)。
6、Handler执行完成返回ModelAndView。
7、HandlerAdapter将Handler执行结果ModelAndView返回到DispatcherServlet。
8、DispatcherServlet将ModelAndView对象选择一个合适ViewReslover视图解析器。
9、ViewReslover解析后，向DispatcherServlet返回具体View（视图）。
10、DispatcherServlet对View进行渲染视（即将模型数据model填充至视图中）。
11、DispatcherServlet响应用户（将渲染结果返回给客户端进行展示）。
**三、SpringMVC工作原理**
1、 客户端发送请求到 DispatcherServlet
2、DispatcherServlet 查询 handlerMapping 找到处理请求的 Controller
3、Controller 调用业务逻辑后，返回 ModelAndView
4、DispatcherServlet 查询 ModelAndView，找到指定视图
5、视图将结果返回到客户端

#### 参考资料
[https://juejin.cn/post/7327227246619181095](https://juejin.cn/post/7327227246619181095)
