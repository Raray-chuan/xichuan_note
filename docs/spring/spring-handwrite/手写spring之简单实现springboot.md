# 手写spring之简单实现springboot

仓库地址:
- [Raray-chuan/mini-spring](https://github.com/Raray-chuan/mini-spring)

博文列表:
- [导读](https://github.com/Raray-chuan/mini-spring/blob/main/README.md)
- [手写spring之ioc](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之ioc.md)
- [手写spring之aop](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之aop.md)
- [手写spring之简单实现springboot](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之简单实现springboot.md)



## 1.springmvc框架的理解

**什么是MVC:**
```
MVC就是一个分层架构模式:
MVC 设计模式一般指 MVC 框架，M（Model）指数据模型层，V（View）指视图层，C（Controller）指控制层。使用 MVC 的目的是将 M 和 V 的实现代码分离，使同一个程序可以有不同的表现形式。其中，View 的定义比较清晰，就是用户界面。

在 Web 项目的开发中，能够及时、正确地响应用户的请求是非常重要的。用户在网页上单击一个 URL 路径，这对 Web 服务器来说，相当于用户发送了一个请求。而获取请求后如何解析用户的输入，并执行相关处理逻辑，最终跳转至正确的页面显示反馈结果，这些工作往往是控制层（Controller）来完成的。

在请求的过程中，用户的信息被封装在 User 实体类中，该实体类在 Web 项目中属于数据模型层（Model）。

在请求显示阶段，跳转的结果网页就属于视图层（View）。
```


**Spring MVC核心架构:**
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211171154436.jpg)
核心架构的具体流程步骤如下：

1、  首先用户发送请求——>DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控制；

2、  DispatcherServlet——>HandlerMapping， HandlerMapping将会把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象，通过这种策略模式，很容易添加新的映射策略；

3、  DispatcherServlet——>HandlerAdapter，HandlerAdapter将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；

4、  HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理；并返回一个ModelAndView对象（包含模型数据、逻辑视图名）；

5、  ModelAndView的逻辑视图名——> ViewResolver， ViewResolver将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；

6、  View——>渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术；

7、返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束



**Spring MVC 最核心部分的就是前端控制器DispatcherServlet, 而DispatcherServlet其实就是一个Servlet，我们再在内部加入一个轻量的tomcat，简单实现springboot的功能**




## 2.实现springboot的思路
- 定义一个SpringBootApplication做为启动类，其作用为:1.初始化SpringContext,2.映射Request与RequestHandler(将请求的uri封装成Request,将Controller以及对应的方法封装成RequestHandler),
  3.启动tomcat,此时tomcat会扫描templates/与static/静态资源
- 当tomcat启动，然后http请求过来后，tomcat会找对应的servlet进行处理,我们先定义三个servlet:HtmlRequestServlet、JspRequestServlet、RestRequestServlet分别处理静态资源的请求、jsp页面的请求、rest的请求
- HtmlRequestServlet会根据静态资源的uri，将静态资源返回
- JspRequestServlet会根据`.jsp`结尾的请求返回jsp页面
- RestRequestServlet请求就比较复杂，下面会说思路：
    - RestRequestServlet在初始化的时候，需要先将所有的ArgumentResolver加载进去，现在已经实现的ArgumentResolver有:HttpServletRequestArgumentResolver、
      HttpServletResponseArgumentResolver、HttpSessionArgumentResolver、RequestBodyArgumentResolver、RequestParamArgumentResolver，
      分别处理HttpServletRequest参数、HttpServletResponse参数、HttpSession参数、含有@RequestBody注解的参数、含有@RequestParam注解的参数
    - 根据请求的uri以及与请求的类型(GET、POST...)找到对应的RequestHandler
    - 根据不同的ArgumentResolver，将hhtp请求的参数转换为Java的类型
    - 调用RequestHandler中的method处理逻辑
    - 将处理后的数据，根据不通的类型返回json、String或者html






## 3.基础类
### 3.1 RequestBody
`@RequestBody`注解
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description
 */
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestBody {
    String value() default "";
}

```


### 3.2 RequestMapping
`@RequestMapping`注解
```java
/**
* @Author Xichuan
* @Date 2022/5/7 11:25
* @Description RequestMapping注解
  */
@Target({ElementType.TYPE,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {
  String value() default "";

  RequestMethod method() default RequestMethod.GET;
}
```


### 3.3 RequestParam
`@RequestParam`注解
```java
/**
* @Author Xichuan
* @Date 2022/5/7 11:25
* @Description
*/
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestParam {
    String value() default "";
}
```


### 3.4 ResponseBody
`@ResponseBody`注解
```java
/**
* @Author Xichuan
* @Date 2022/5/7 11:25
* @Description
  */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ResponseBody {
}
```


### 3.5 RequestMethod
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description
 */
public enum RequestMethod {
    GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE
}
```


### 3.6 Request类
将请求的uri以及请求的类型封装成`Request`
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description Controller中含有@RequestMapping的都封装成一个Request，主要存放uri以及请求类型
 */
public class Request {

    //请求的uri
    private String requestPath;

    //请求的类型，GET、POST....
    private String requestMethod;

    public Request(String requestPath, String requestMethod) {
        this.requestPath = requestPath;
        this.requestMethod = requestMethod;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Request request = (Request) o;

        return this.requestPath.equals(request.requestPath) && this.requestMethod.equals(request.requestMethod);
    }

    @Override
    public int hashCode() {
        return requestPath.hashCode()&requestMethod.hashCode()&21;
    }


    public String getRequestPath() {
        return requestPath;
    }

    public void setRequestPath(String requestPath) {
        this.requestPath = requestPath;
    }

    public String getRequestMethod() {
        return requestMethod;
    }

    public void setRequestMethod(String requestMethod) {
        this.requestMethod = requestMethod;
    }
}
```


### 3.7 RequestHandler
Controller中一个方法封装一个方法; 此类作用是通过uri找到对应的`RequestHandler`
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 
 */
public class RequestHandler {

    //Controller类
    private Object controller;

    //该请求对应Controller的方法
    private Method controllerMethod;

    public RequestHandler(Object controller, Method controllerMethod) {
        this.controller = controller;
        this.controllerMethod = controllerMethod;
    }

    public Object getController() {
        return controller;
    }

    public void setController(Object controller) {
        this.controller = controller;
    }

    public Method getControllerMethod() {
        return controllerMethod;
    }

    public void setControllerMethod(Method controllerMethod) {
        this.controllerMethod = controllerMethod;
    }
}
```


### 3.8 RequestParam
每一个Http请求的参数，封装成一个类

`creatParam()`作用时将http请求的参数根据不同的`ArgumentResolver`转换为Java中的类型
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 每一个Http请求的参数，封装成一个类
 */
public class RequestParam {
    //将请求的参数封装成为map
    private Object[] params;

    /**
     * 将HttpServletRequest中的请求参数放置到Object[]
     * @param request
     */
    public void creatParam(List<ArgumentResolver> argumentResolvers,RequestHandler requestHandler,HttpServletRequest request, HttpServletResponse response) throws IOException {
        Class<?>[] paramClazzs = requestHandler.getControllerMethod().getParameterTypes();

        params = new Object[paramClazzs.length];
        int paramIndex = 0;
        int i = 0;
        for (Class<?> paramClazz : paramClazzs) {
            for (ArgumentResolver resolver : argumentResolvers) {
                if (resolver.support(paramClazz, paramIndex, requestHandler.getControllerMethod())) {
                    params[i++] = resolver.argumentResolver(request,
                            response,
                            paramClazz,
                            paramIndex,
                            requestHandler.getControllerMethod());
                }
            }
            paramIndex++;
        }

    }

    public Object[] getParams() {
        return params;
    }

    public void setParams(Object[] params) {
        this.params = params;
    }
}
```


### 3.9 View
http请求返回的View封装类
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 
 */
public class View {
    //跳转的请求路径
    private String path= null ;
    //跳转的请求参数
    private Map<String, Object> model;

    public View(String path) {
        this.path = path;
        model = new HashMap<String, Object>();
    }

    public View addModel(String key, Object value) {
        model.put(key, value);
        return this;
    }

    public String getPath() {
        return path;
    }

    public Map<String, Object> getModel() {
        return model;
    }
}

```



## 4.实现的流程
### 4.1 SpringBootApplication
```java
/**
 * @Author Xichuan
 * @Date 2022/5/11 9:16
 * @Description SpringBoot入口类
 */
public class SpringBootApplication {
    public static void run(Class<?> cls,String[] args){
        TomcatServer tomcatServer = new TomcatServer();
        //加载bean
        SpringContext context = new SpringContext(cls,null);
        //对Controller类的一些处理
        HandlerMappingHelper.getAllHandler();

        try {
            //执行tomcatServer处理类
            tomcatServer.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
SpringBootApplication启动的时候做了三件事:
- new SpringContext(cls,null): 初始化SpringContext,加载所有的bean
- HandlerMappingHelper.getAllHandler(): 将Request与RequestHandler进行映射
- 启动tomcat



### 4.2 SpringContext初始化
关于SpringContext的前面已经讲过了:
- [手写spring之ioc](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之ioc.md)
- [手写spring之aop](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之aop.md)

SpringContext的代码中有一块是对@Controler注解类的处理，会将实例化后的Controller对象放在`BeanContainer.controllerMap`中

`com.xichuan.framework.core.helper.LoadBeanHelper.createBean`：
```java
    /**
     * 创建bean，并进行代理
     * @param beanDefinition bean的定义信息
     * @param singleton 是否是单例bean
     * @return
     */
    private static Object createBean(BeanDefinition beanDefinition, Boolean singleton) {
        ...
        ...
        
        //加入ControllerMap引用
        if(beanDefinition.isController()) {
            BeanContainer.controllerMap.put(beanDefinition.getBeanName(), BeanContainer.singletonObjects.get(beanDefinition.getBeanName()));
        }

        ...
        ...
    }
```

### 4.3 HandlerMappingHelper.getAllHandler()获取所有的Request与RequestHandler的映射

```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description
 */
public class HandlerMappingHelper {

    //Request与RequestHandler的映射
    private static HashMap<Request, RequestHandler> requestHandlerMap =new HashMap<>();

    /**
     * 将含有@RequestMapping的方法，<Request, RequestHandler>并放在HandlerMap中
     */
    public static void getAllHandler() {
        //获取所有的Controller
        for(Object controllerName: BeanContainer.controllerMap.keySet()) {
            Object controllerInstance = BeanContainer.controllerMap.get(controllerName);

            //获取类上的@RequestMapping的value值
            String classUri = null;
            if (controllerInstance.getClass().isAnnotationPresent(RequestMapping.class)){
                classUri = controllerInstance.getClass().getDeclaredAnnotation(RequestMapping.class).value();
            }

            //遍历所有的Controller的方法
            String methodUri;
            String requestMethod;
            Request request;
            for(Method method:controllerInstance.getClass().getDeclaredMethods()) {
                //获取controller中所有带@RequestMapping的方法
                if(!method.isAnnotationPresent(RequestMapping.class)) {
                    return;
                }

                RequestMapping requestMapping = method.getAnnotation(RequestMapping.class);
                methodUri = requestMapping.value();
                requestMethod = requestMapping.method().name();
                //将适用的方法封装成请求体
                request = new Request(joinFullUri(classUri,methodUri),requestMethod);

                //将handler封装成对象
                RequestHandler requestHandler =new RequestHandler(controllerInstance,method);
                requestHandlerMap.put(request, requestHandler);
            }


        }
    }

    /**
     * 对server.base.path、类上的@RequestMapping的value、方法上的@RequestMapping的value进行拼接
     * @param classUri 类上的@RequestMapping的value
     * @param methodUri 方法上的@RequestMapping的value
     * @return
     */
    public static String joinFullUri(String classUri,String methodUri){
        String basePath = UrlUtil.formatUrl(ConfigHelper.getString(ConfigConstant.SERVER_BASE_PATH));
        String classPath = UrlUtil.formatUrl(classUri);
        String methodPath = UrlUtil.formatUrl(methodUri);
        return UrlUtil.formatUrl(basePath + classPath + methodPath);
    }

    /**
     * 通过request(uri与请求类型)获取对应的Handler
     * @param request
     * @return
     */
    public static RequestHandler getRequestHandler(Request request) {
        if(requestHandlerMap.containsKey(request))
            return requestHandlerMap.get(request);
        else
          return null;
    }
}

```
代码很简单，遍历`BeanContainer.controllerMap`中的所有Controller类
- RequestHandler是有controller类与对应的方法拼接而来
- RequestHandler对应的uri是并根据`config.properties中的server.base.path` + `类上的@RequestMapping中的uri` + `方法上的@RequestMapping的uri`拼接而来



### 4.4 TomcatServer的启动
```java
/**
 * @Author Xichuan
 * @Date 2022/5/11 9:18
 * @Description 集成Tomcat服务器，将请求转发至DispatcherServlet
 */
public class TomcatServer {

    //tomcat
    private Tomcat tomcat;

    /**
     * 执行tomcat初始化操作
     * @throws LifecycleException
     */
    public void start() throws LifecycleException, ServletException, IOException {
        String tomcatBaseDir = createTempDir("tomcat", ConfigHelper.getInt(ConfigConstant.SERVER_PORT)).getAbsolutePath();
        String contextDocBase = createTempDir("tomcat-docBase", ConfigHelper.getInt(ConfigConstant.SERVER_PORT)).getAbsolutePath();
        String hostName = CommonUtils.getHostName();
        String contextPath = "";

        //实例化tomcat
        tomcat = new Tomcat();
        tomcat.setBaseDir(tomcatBaseDir);
        tomcat.setPort(ConfigHelper.getInt(ConfigConstant.SERVER_PORT));
        tomcat.setHostname(hostName);

        //添加静态资源映射
        //扫描templates/ 与 static/ 目录下的静态资源
        Host host = tomcat.getHost();
        Context context = tomcat.addWebapp(host, contextPath, contextDocBase, new EmbededContextConfig());

        //标准的web项目会一个WEB-INF/lib文件加存放额外的jar包，此处就是为了加载里面的jar（因为我们这个不是标准的web项目，此处可以不使用）
        //如果是在IED中，扫描WEB-INF/lib路径;如果是在打包好的jar，扫描WEB-INF/classes
        context.setJarScanner(new EmbededStandardJarScanner());

        //设置类加载器
        ClassLoader classLoader = TomcatServer.class.getClassLoader();
        context.setParentClassLoader(classLoader);

        //context load WEB-INF/web.xml from classpath() (此处可以不使用)
        context.addLifecycleListener(new WebXmlMountListener());

        //启动tomcat
        tomcat.start();

        //设置常驻线程防止tomcat中途退出
        Thread awaitThread = new Thread("tomcat-await-thread"){
            @Override
            public void run() {
                TomcatServer.this.tomcat.getServer().await();
            }
        };
        //设置为非守护线程
        awaitThread.setDaemon(false);
        awaitThread.start();
    }


    /**
     * 创建临时文件
     * @param prefix
     * @param port
     * @return
     * @throws IOException
     */
    public static File createTempDir(String prefix, int port) throws IOException {
        File tempDir = File.createTempFile(prefix + ".", "." + port);
        tempDir.delete();
        tempDir.mkdir();
        tempDir.deleteOnExit();
        return tempDir;
    }

}
```

tomcat启动就是扫描静态资源，然后开启一个守护进程防止tomcat中途退出。其中:
- `EmbededContextConfig`:扫描`templates/`与`static/`目录下的静态资源(已经适配了将项目打成jar包后仍可以扫到静态资源)
- `EmbededStandardJarScanner`:扫描`WEB-INF/classes`与`WEB-INF/lib`,此类是由`org.apache.tomcat.util.scan.StandardJarScanner`改造而来
- `WebXmlMountListener`: 扫描`WEB-INF/web.xml`并进行加载



tomcat启动完，当有http请求到来，tomcat会将请求转到到相应的servlet进行处理，下面看`HtmlRequestServlet`、`JspRequestServlet`、`RestRequestServlet`这三个servlet



### 4.5 HtmlRequestServlet的介绍

`HtmlRequestServlet`会将`*.html","*.ico","*.js","*.css","*.png","*.gif","*.jpg","*.ttf","*.woff","*.woff2`结尾的静态资源请求进行拦截处理

如果请求的页面的页面是`404.html`，会将返回的页面状态设置为404
```java
/**
 * @Author Xichuan
 * @Date 2022/5/12 12:48
 * @Description 此Servlet对静态资源的请求进行拦截
 */
@WebServlet(name = "html_request",urlPatterns = {"*.html","*.ico","*.js","*.css","*.png","*.gif","*.jpg","*.ttf","*.woff","*.woff2"},loadOnStartup = 1)
public class HtmlRequestServlet extends DefaultServlet {
    private Logger logger = LoggerFactory.getLogger(HtmlRequestServlet.class);

    @Override
    public void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        logger.debug("html request,request uri:" + req.getRequestURI());
        if (req.getRequestURI().endsWith("404.html")) {
            resp.setStatus(404);
        }
        super.service(req, resp);
    }
}
```


### 4.6 JspRequestServlet介绍
`JspRequestServlet`会将`*.jps`结尾的静态资源请求进行拦截处理

```java
/**
 * @Author Xichuan
 * @Date 2022/5/12 16:17
 * @Description 此Servlet对jsp的请求进行拦截
 */
@WebServlet(name = "jsp_request",urlPatterns = "*.jps",loadOnStartup = 2)
public class JspRequestServlet extends JspServlet {
    private Logger logger = LoggerFactory.getLogger(JspRequestServlet.class);

    @Override
    public void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        logger.debug("jsp request,request uri:+" + request.getRequestURI());
        super.service(request, response);
    }
}
```


### 4.7 RestRequestServlet介绍
`RestRequestServlet` 会拦截rest请求
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description
 */

/**
 * 每一个http请求进行拦截一次（此Servlet对Rest的请求进行拦截）
 * (通过的注解的方式加载Servlet，tomcat会将请求转发此Servlet)
 */
@WebServlet(name = "rest_request",urlPatterns = "/",loadOnStartup = 3)
public class RestRequestServlet extends HttpServlet {
    private static Logger logger = LoggerFactory.getLogger(RestRequestServlet.class);

    //参数处理集合
    private List<ArgumentResolver> argumentResolvers;

    /**
     * 1.初始化方法,加载所有的ArgumentResolver
     *
     * @param config
     * @throws ServletException
     */
    @Override
    public void init(ServletConfig config) {
        try {
            //初始化ArgumentResolver
            argumentResolvers = findImplementObject(ArgumentResolver.class);

        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }

    /**
     * 请求处理逻辑
     *
     * @param request
     * @param response
     * @throws ServletException
     * @throws IOException
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        logger.debug("rest request,request uri:" + request.getRequestURI());

        String requestMethod = request.getMethod(); //请求类型，GET/POST...
        String requestPath = request.getRequestURI();//获取到的路径类似 /aa=xxx

        //2.找到对应的RequestHandler
        RequestHandler requestHandler = HandlerMappingHelper.getRequestHandler(new Request(UrlUtil.formatUrl(requestPath), requestMethod));
        if (requestHandler == null) {
            ViewResolver.handle404Result(request, response);
            return;
        }

        //3.封装RequestParam,将HttpServletRequest中的数据根据不同的argumentResolver转换为Java数据类型
        RequestParam param = new RequestParam();
        param.creatParam(argumentResolvers,requestHandler,request, response);

        //4.通过反射调用RequestHandler中的Controller方法进行处理逻辑
        Object result = HandlerAdapter.adapterForRequest(param, requestHandler);

        //5.根据上一步返回的数据，将数据进行View或者Data进行处理
        ViewAdapter.adapter(result,requestHandler,request,response);
    }


    /**
     * 获取接口对应目录下的，对应的实现类对象
     */
    private static <T> List<T> findImplementObject(Class<?> interfaceClass) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException, IOException, ClassNotFoundException {
        List<T> result = new ArrayList<>();
        Set<Class<?>> classSet = loadAllClass(interfaceClass.getPackage().getName());
        //接口对应的所有实现类
        List<Class<?>> argumentClass = getClassSetBySuper(classSet, interfaceClass);
        for (Class<?> clazz : argumentClass) {
            logger.debug("init argument resolver: " + clazz.getName());
            result.add((T)clazz.getDeclaredConstructor().newInstance());
        }


        return result;
    }

    /**
     * 获取基础包名下某父类的所有子类 或某接口的所有实现类
     *
     * @param classesHashSet 说扫描的class
     * @param superClass     父类class
     * @return
     */
    public static List<Class<?>> getClassSetBySuper(Set<Class<?>> classesHashSet, Class<?> superClass) {
        List<Class<?>> classList = new ArrayList<>();
        for (Class<?> cls : classesHashSet) {
            //isAssignableFrom() 指 superClass 和 cls 是否相同或 superClass 是否是 cls 的父类/接口
            if (superClass.isAssignableFrom(cls) && !superClass.equals(cls)) {
                classList.add(cls);
            }
        }
        return classList;
    }

    /**
     * 获取某包下的所有类
     *
     * @param packagePath
     * @return
     */
    public static Set<Class<?>> loadAllClass(String packagePath) throws IOException, ClassNotFoundException {
        URL url = Thread.currentThread().getContextClassLoader().getResource(packagePath.replace(".","/"));
        HashSet<Class<?>> classesHashSet = new HashSet<>();
        Class<?> clazz;
        if (url != null){
            if (url.getProtocol().equals("file")){
                List<String> classNames = PackageUtil.getClassName(packagePath,false);

                for (String className : classNames) {
                    try {
                        clazz = BeanContainer.classLoader.loadClass(className);
                        classesHashSet.add(clazz);
                    } catch (ClassNotFoundException e) {
                        e.printStackTrace();
                    }
                }
            } else if (url.getProtocol().equals("jar")) {
                List<String> packageClass = new ArrayList<>();
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.DefaultArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.HttpServletRequestArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.HttpServletResponseArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.HttpSessionArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.RequestBodyArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.RequestParamArgumentResolver");
                for (String className : packageClass){
                    clazz = BeanContainer.classLoader.loadClass(className);
                    classesHashSet.add(clazz);
                }
            }
        }
        return classesHashSet;
    }
}
```

通过代码可以看出，在`RestRequestServlet`流程大致如下:
- 1.`RestRequestServlet`在初始化的时候，需要先将所有的`ArgumentResolver`加载进去，现在已经实现的`ArgumentResolver`有:`HttpServletRequestArgumentResolver`、
  `HttpServletResponseArgumentResolver`、`HttpSessionArgumentResolver`、`RequestBodyArgumentResolver`、`RequestParamArgumentResolver`，
  分别处理`HttpServletRequest`参数、`HttpServletResponse`参数、`HttpSession`参数、含有`@RequestBody`注解的参数、含有`@RequestParam`注解的参数
- 2.根据请求的uri以及与请求的类型(GET、POST...)找到对应的`RequestHandler`
- 3.根据不同的`ArgumentResolver`，将hhtp请求的参数转换为Java的类型
- 4.调用`RequestHandler`中的method处理逻辑
- 5.将处理后的数据，根据不通的类型返回json、String或者html



#### 4.7.1 RestRequestServlet在init的时候加载所有的ArgumentResolver

##### 7.7.1.1 了解ArgumentResolver的作用
我们只看下`RequestBodyArgumentResolver`与`RequestParamArgumentResolver`参数处理类
**RequestBodyArgumentResolver:**
```java
/**
 * @Author Xichuan
 * @Date 2022/5/13 20:25
 * @Description 含有@RequestBody注解的Argument处理，将String参数映射成一个对象
 */
public class RequestBodyArgumentResolver implements ArgumentResolver {
    @Override
    public boolean support(Class<?> type, int paramIndex, Method method) {
        Annotation[][] an = method.getParameterAnnotations();

        Annotation[] paramAns = an[paramIndex];

        for (Annotation paramAn : paramAns) {
            if (RequestBody.class.isAssignableFrom(paramAn.getClass())) {
                return true;
            }
        }
        return false;
    }

    @Override
    public Object argumentResolver(HttpServletRequest request, HttpServletResponse response, Class<?> type, int paramIndex, Method method) throws IOException {
        final Parameter[] parameters = method.getParameters();
        //读取流对象，需要根据req getInputStream 得到一个流对象，从这个流对象中去获取
        InputStream inputStream=request.getInputStream();
        //利用 contentLength 拿到请求中的body的字节数
        int length=request.getContentLength();
        byte[] bytes=new byte[length];
        inputStream.read(bytes);
        return JSON.parseObject(new String(bytes,"utf-8"),parameters[paramIndex].getType());
    }


}

```
`RequestBodyArgumentResolvers`是从`HttpServletRequest`获取`InputStream`,并从`InputStream`获取数据并转换为fastjson,
显而易见,此处理类处理的对应的注解是@RequestBody


**RequestParamArgumentResolver:**
```java
/**
 * @Author Xichuan
 * @Date 2022/5/13 20:25
 * @Description 含有@RequestParam注解的Argument处理，参数名词可以重命名
 */
public class RequestParamArgumentResolver implements ArgumentResolver {
    
    public boolean support(Class<?> type, int paramIndex, Method method) {
        
        Annotation[][] an = method.getParameterAnnotations();
        Annotation[] paramAns = an[paramIndex];
        for (Annotation paramAn : paramAns) {
            if (RequestParam.class.isAssignableFrom(paramAn.getClass())) {
                return true;
            }
        }
        return false;
    }
    
    public Object argumentResolver(HttpServletRequest request,
                                   HttpServletResponse response, Class<?> type, int paramIndex,
                                   Method method) {
        
        Annotation[][] an = method.getParameterAnnotations();
        Annotation[] paramAns = an[paramIndex];
        
        for (Annotation paramAn : paramAns) {
            if (RequestParam.class.isAssignableFrom(paramAn.getClass())) {
                RequestParam rp = (RequestParam)paramAn;

                final Parameter[] parameters = method.getParameters();
                String requestStr = request.getParameter(rp.value());
                return transDataType(requestStr,parameters[paramIndex]);
            }
        }
        
        return null;
    }


    /**
     * 对String进行类型转换
     * @param requestStr
     * @param parameter
     * @return
     */
    private Object transDataType(String requestStr,Parameter parameter){
        Object result = null;
        if (CommonUtils.isNotBlack(requestStr)){
            //进行类型转换
            if (String.class ==parameter.getType()) {
                result = requestStr;
            } else if (Integer.class == parameter.getType() || int.class == parameter.getType()) {
                result = Integer.valueOf(requestStr);
            } else if (Double.class == parameter.getType() || double.class == parameter.getType()) {
                result = Double.valueOf(requestStr);
            } else if(Long.class == parameter.getType() || long.class == parameter.getType()) {
                result = Long.valueOf(requestStr);
            } else if(Short.class == parameter.getType() || short.class == parameter.getType()) {
                result = Short.valueOf(requestStr);
            } else if(Boolean.class == parameter.getType() || boolean.class == parameter.getType()) {
                result = Boolean.valueOf(requestStr);
            } else if(Float.class == parameter.getType() || float.class == parameter.getType()) {
                result = Float.valueOf(requestStr);
            } else if(Byte.class == parameter.getType() || byte.class == parameter.getType()) {
                result = Byte.valueOf(requestStr);
            } else if(Map.class == parameter.getType() || HashMap.class == parameter.getType()) {
                result = JSON.parseObject(requestStr,Map.class);
            }else if (List.class == parameter.getType() || ArrayList.class == parameter.getType() || LinkedList.class == parameter.getType()){
                result = JSON.parseArray(requestStr,String.class);
            }
        }
        return result;
    }
    
}
```
`RequestParamArgumentResolver`是将`HttpServletRequest`中每一个数据转换为Controller处理方法中对应的参数定义类型，此处理类处理的对应注解是`@RequestParam`



##### 7.7.1.2 RestRequestServlet如何加载所有的ArgumentResolver
```java
/**
 * 每一个http请求进行拦截一次（此Servlet对Rest的请求进行拦截）
 * (通过的注解的方式加载Servlet，tomcat会将请求转发此Servlet)
 */
@WebServlet(name = "rest_request",urlPatterns = "/",loadOnStartup = 3)
public class RestRequestServlet extends HttpServlet {
    private static Logger logger = LoggerFactory.getLogger(RestRequestServlet.class);

    //参数处理集合
    private List<ArgumentResolver> argumentResolvers;

    /**
     * 1.初始化方法,加载所有的ArgumentResolver
     *
     * @param config
     * @throws ServletException
     */
    @Override
    public void init(ServletConfig config) {
        try {
            //初始化ArgumentResolver
            argumentResolvers = findImplementObject(ArgumentResolver.class);

        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }

    ...
    ...
    ...

    /**
     * 获取接口对应目录下的，对应的实现类对象
     */
    private static <T> List<T> findImplementObject(Class<?> interfaceClass) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException, IOException, ClassNotFoundException {
        List<T> result = new ArrayList<>();
        Set<Class<?>> classSet = loadAllClass(interfaceClass.getPackage().getName());
        //接口对应的所有实现类
        List<Class<?>> argumentClass = getClassSetBySuper(classSet, interfaceClass);
        for (Class<?> clazz : argumentClass) {
            logger.debug("init argument resolver: " + clazz.getName());
            result.add((T)clazz.getDeclaredConstructor().newInstance());
        }


        return result;
    }

    /**
     * 获取基础包名下某父类的所有子类 或某接口的所有实现类
     *
     * @param classesHashSet 说扫描的class
     * @param superClass     父类class
     * @return
     */
    public static List<Class<?>> getClassSetBySuper(Set<Class<?>> classesHashSet, Class<?> superClass) {
        List<Class<?>> classList = new ArrayList<>();
        for (Class<?> cls : classesHashSet) {
            //isAssignableFrom() 指 superClass 和 cls 是否相同或 superClass 是否是 cls 的父类/接口
            if (superClass.isAssignableFrom(cls) && !superClass.equals(cls)) {
                classList.add(cls);
            }
        }
        return classList;
    }

    /**
     * 获取某包下的所有类
     *
     * @param packagePath
     * @return
     */
    public static Set<Class<?>> loadAllClass(String packagePath) throws IOException, ClassNotFoundException {
        URL url = Thread.currentThread().getContextClassLoader().getResource(packagePath.replace(".","/"));
        HashSet<Class<?>> classesHashSet = new HashSet<>();
        Class<?> clazz;
        if (url != null){
            if (url.getProtocol().equals("file")){
                List<String> classNames = PackageUtil.getClassName(packagePath,false);

                for (String className : classNames) {
                    try {
                        clazz = BeanContainer.classLoader.loadClass(className);
                        classesHashSet.add(clazz);
                    } catch (ClassNotFoundException e) {
                        e.printStackTrace();
                    }
                }
            } else if (url.getProtocol().equals("jar")) {
                //此处先将写死所有的ArgumentResolver路径
                List<String> packageClass = new ArrayList<>();
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.DefaultArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.HttpServletRequestArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.HttpServletResponseArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.HttpSessionArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.RequestBodyArgumentResolver");
                packageClass.add("com.xichuan.framework.web.helper.argumentHelper.RequestParamArgumentResolver");
                for (String className : packageClass){
                    clazz = BeanContainer.classLoader.loadClass(className);
                    classesHashSet.add(clazz);
                }
            }
        }




        return classesHashSet;
    }
}

```
我们可以看出是在init()初始化的时候，通过ArgumentResolver接口，获取所有的实现类进行加载的




#### 4.7.2 根据请求的uri以及与请求的类型(GET、POST...)找到对应的`RequestHandler`
**com.xichuan.framework.web.tomcat.RestRequestServlet.service()**
```java
       //2.找到对应的RequestHandler
        RequestHandler requestHandler = HandlerMappingHelper.getRequestHandler(new Request(UrlUtil.formatUrl(requestPath), requestMethod));
        if (requestHandler == null) {
            ViewResolver.handle404Result(request, response);
            return;
        }
```
此处很简单,通过uri，直接在`HandlerMappingHelper`中的map中获取的



#### 4.7.3 根据不同的ArgumentResolver，将http请求的参数转换为Java的类型
**com.xichuan.framework.web.tomcat.RestRequestServlet.service():**
```java
        //3.封装RequestParam,将HttpServletRequest中的数据根据不同的argumentResolver转换为Java数据类型
        RequestParam param = new RequestParam();
        param.creatParam(argumentResolvers,requestHandler,request, response);
```

**com.xichuan.framework.web.data.RequestParam.creatParam():**
```java
/**
 * 将HttpServletRequest中的请求参数放置到Object[]
 * @param request
 */
public void creatParam(List<ArgumentResolver> argumentResolvers,RequestHandler requestHandler,HttpServletRequest request, HttpServletResponse response) throws IOException {
    Class<?>[] paramClazzs = requestHandler.getControllerMethod().getParameterTypes();

    params = new Object[paramClazzs.length];
    int paramIndex = 0;
    int i = 0;
    for (Class<?> paramClazz : paramClazzs) {
        for (ArgumentResolver resolver : argumentResolvers) {
            if (resolver.support(paramClazz, paramIndex, requestHandler.getControllerMethod())) {
                params[i++] = resolver.argumentResolver(request,
                        response,
                        paramClazz,
                        paramIndex,
                        requestHandler.getControllerMethod());
            }
        }
        paramIndex++;
    }

}
```
这个方法也很简单，遍历方法中的每一个参数，然后每一个参数调用`ArgumentResolver`的每一个实现类的`resolver.support()`方法来判断是否可以由此`ArgumentResolver`进行处理



#### 4.7.4 调用`RequestHandler`中的method处理逻辑
**com.xichuan.framework.web.tomcat.RestRequestServlet.service():**
```java
    //4.通过反射调用RequestHandler中的Controller方法进行处理逻辑
    Object result = HandlerAdapter.adapterForRequest(param, requestHandler);
```

**HandlerAdapter.adapterForRequest:**
```java
public class HandlerAdapter {

    /**
     * 调用controller的对应的方法
     * @param requestParam
     * @param requestHandler
     * @return
     */
        public static Object adapterForRequest(RequestParam requestParam, RequestHandler requestHandler) {
            if(requestParam.getParams() != null && requestParam.getParams().length>0) {
                return MvcProxy.invokeMethod(requestHandler.getController(), requestHandler.getControllerMethod(),requestParam.getParams());
            }else {
                return MvcProxy.invokeMethod(requestHandler.getController(), requestHandler.getControllerMethod(),null);
            }
        }
}
```

**MvcProxy.invokeMethod():**
```java
public class MvcProxy {
    /**
     * 代理调用调用Controller目标方法，并返回
     * @param target 目标Controller类
     * @param method 目标类的方法
     * @param params 请求参数
     * @return
     */
    public static Object invokeMethod(Object target, Method method, Object[] params) {
            Object res = null;

        try {
            method.setAccessible(true);
            if(params!=null && params.length > 0) {
                res = method.invoke(target, params);
            }else
                res = method.invoke(target);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }

        return res;
    }
}
```
可以看出此方法也很简单，通过反射，调用Controller中的对应方法，并传入参数进行业务逻辑处理的



#### 4.7.5 将处理后的数据，根据不通的类型返回json、String或者html
**com.xichuan.framework.web.tomcat.RestRequestServlet.service():**
```java
        //5.根据上一步返回的数据，将数据进行View或者Data进行处理
        ViewAdapter.adapter(result,requestHandler,request,response);
```



**ViewAdapter.adapter():**

```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 对View处理的适配
 */
public class ViewAdapter {
    public static void adapter(Object result, RequestHandler requestHandler, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        //对对返回的View进行处理
        if (result instanceof View) {
            ViewResolver.handleViewResult((View) result, request, response);
        } else {
            //判断class或者method上是否有@ResponseBody;如果有则返回json,否则返回String
            if(requestHandler.getController().getClass().isAnnotationPresent(ResponseBody.class) || requestHandler.getControllerMethod().isAnnotationPresent(ResponseBody.class)){
                ViewResolver.handleJsonResult(result,response);
            }else{
                ViewResolver.handleStingResult(result,response);
            }
        }
    }


}
```

我们可以看出，根据返回数据的不同，或者是否@ResponseBody注解，会进行不通的处理方法:
- 如果返回数据的数据类型是View,则按页面进行处理:ViewResolver.handleViewResult()
- 如果Controller的class或者对应的method上是否有@ResponseBody,如果由则返回json:ViewResolver.handleJsonResult()
- 其他的话就返回String

```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description View处理类
 */
public class ViewResolver {
    private static Logger logger = LoggerFactory.getLogger(ViewResolver.class);

    //用户自定义404页面是否存在
    private static boolean html404IsExists = false;
    //是否搜寻过用户自定义的404页面
    private static boolean hadFind404Html = false;


    /**
     * 处理404
     * @param response
     */
    public static void handle404Result(HttpServletRequest request,HttpServletResponse response) throws IOException, ServletException {
        //获取404页面是否存在
        is404HtmlExist();

        //如果存在自定义404页面跳转到404页面，否则自动生成一个
        if (html404IsExists){
            request.getRequestDispatcher("/404.html").forward(request, response);
        }else{
            final ServletOutputStream out = response.getOutputStream();
            response.setContentType("text/html");
            response.setCharacterEncoding("UTF-8");
            response.setStatus(404);
            String html404 =
                    "<!DOCTYPE html>\n" +
                            "<html>\n" +
                            "    <head>\n" +
                            "        <meta charset=\"UTF-8\">\n" +
                            "        <title>404</title>\n" +
                            "    </head>\n" +
                            "    <body>\n" +
                            "        <p>\n" +
                            "            <h1>Whitelabel Error Page</h1> \n  " +
                            "This application has no explicit mapping for /error, so you are seeing this as a fallback.</br>\n" +
                            "</br>\n" +
                            new Date().toString() + "</br>\n" +
                            "There was an unexpected error (type=Not Found, status=404).</br>\n" +
                            "No message available</br>\n" +
                            "        </p>\n" +
                            "    </body>\n" +
                            "</html>";
            out.write(html404.getBytes());
        }


    }


    /**
     * 如果Controller返回的是View，在此处处理
     * @param view
     * @param request
     * @param response
     * @throws IOException
     * @throws ServletException
     */
    public static void handleViewResult(View view, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        //如果以"/"开头，重定向；否则请求转发
        String path = view.getPath();
        String directPath = null;
        if (!(path.endsWith(".jsp")
                || path.endsWith(".html")
                || path.endsWith(".ico")
                || path.endsWith(".js")
                || path.endsWith(".css")
                || path.endsWith(".png")
                || path.endsWith(".gif")
                || path.endsWith(".jpg")
                || path.endsWith(".ttf")
                || path.endsWith(".woff")
                || path.endsWith(".woff2"))){
            String basePath = UrlUtil.formatUrl(ConfigHelper.getString(ConfigConstant.SERVER_BASE_PATH));
            String fullPath = UrlUtil.formatUrl(basePath + path);;
            directPath = fullPath.substring(0,fullPath.length() - 1);
        }else{
            String tempPath = UrlUtil.formatUrl(path);
            directPath = tempPath.substring(0,tempPath.length()-1);
        }


        if (CommonUtils.isNotBlack(path)){
            if (path.startsWith("/")) { //重定向
                response.setCharacterEncoding("UTF-8");
                response.sendRedirect( directPath);
            } else { //请求转发
                Map<String, Object> model = view.getModel();
                String urlParam = "?";
                for (Map.Entry<String, Object> entry : model.entrySet()) {
                    request.setAttribute(entry.getKey(), entry.getValue());
                    urlParam += entry.getKey() + "=" + entry.getValue() + "&";
                }
                urlParam = urlParam.substring(0,urlParam.length()-1);
                if (!(path.endsWith(".jsp")
                        || path.endsWith(".html")
                        || path.endsWith(".ico")
                        || path.endsWith(".js")
                        || path.endsWith(".css")
                        || path.endsWith(".png")
                        || path.endsWith(".gif")
                        || path.endsWith(".jpg")
                        || path.endsWith(".ttf")
                        || path.endsWith(".woff")
                        || path.endsWith(".woff2"))) {
                    directPath = directPath + urlParam;
                }
                request.setCharacterEncoding("UTF-8");
                request.getRequestDispatcher(directPath).forward(request, response);
            }
        }
    }

    /**
     * 如果Controller返回的是对象(不是view),且含有@ResponseBody注解，返回json
     * @param data
     * @param response
     * @throws IOException
     */
    public static void handleJsonResult(Object data, HttpServletResponse response) throws IOException {
        if (data != null) {
            response.setContentType("application/json");
            response.setCharacterEncoding("UTF-8");
            PrintWriter writer = response.getWriter();
            String json = JSON.toJSON(data).toString();
            writer.write(json);
            writer.flush();
            writer.close();
        }
    }

    /**
     * 如果Controller返回的是对象(不是view),且不含有@ResponseBody注解，返回String
     * @param data
     * @param response
     * @throws IOException
     */
    public static void handleStingResult(Object data, HttpServletResponse response) throws IOException {
        // text/html ： HTML格式
        // text/plain ：纯文本格式
        if (data != null) {
            response.setContentType("text/plain");
            response.setCharacterEncoding("UTF-8");
            PrintWriter writer = response.getWriter();
            String json = JSON.toJSON(data).toString();
            writer.write(json);
            writer.flush();
            writer.close();
        }
    }




    /**
     * 判断是否存在用户自定义的404页面
     * @return
     * @throws IOException
     */
    private static boolean is404HtmlExist() throws IOException{
        if (!hadFind404Html){
            //判断是否存在自定义的404页面
            URL url = Thread.currentThread().getContextClassLoader().getResource("templates");

            if (url != null){
                if (url.getProtocol().equals("file")){
                    for (File childFile : new File(url.getPath()).listFiles()) {
                        if (!childFile.isDirectory() && childFile.getName().equals("404.html")) {
                            html404IsExists = true;
                            break;
                        }
                    }
                } else if (url.getProtocol().equals("jar")) {
                    String[] jarInfo = url.getPath().split("!");
                    String jarFilePath = jarInfo[0].substring(jarInfo[0].indexOf("/"));
                    Enumeration<JarEntry> entrys = new JarFile(jarFilePath).entries();
                    JarEntry jarEntry;
                    while (entrys.hasMoreElements()) {
                        jarEntry = entrys.nextElement();
                        if (!jarEntry.isDirectory() && jarEntry.getName().equals("templates/404.html")){
                            html404IsExists = true;
                            break;
                        }
                    }
                }
            }
            hadFind404Html = true;
        }
        return html404IsExists;
    }
}
```

ViewResolver的每一个处理方法就不介绍了，可以根据注解看下



这里已经所有代码实现已经讲完了，如果有不明白的地方，代码中有一个模块`xichuan-spring-test`是测试项目，可以按照里面的示例进行打断点进行阅读，希望能帮助到你