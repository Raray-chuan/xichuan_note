# 手写spring之aop

仓库地址:
- [Raray-chuan/mini-spring](https://github.com/Raray-chuan/mini-spring)

博文列表:
- [导读](https://github.com/Raray-chuan/mini-spring/blob/main/README.md)
- [手写spring之ioc](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之ioc.md)
- [手写spring之aop](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之aop.md)
- [手写spring之简单实现springboot](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之简单实现springboot.md)



## 1.什么是AOP
AOP(Aspect-oriented Programming), AOP翻译过来叫面向切面编程, 核心就是这个切面. 
切面表示从业务逻辑中分离出来的横切逻辑, 比如性能监控, 日志记录, 权限控制等, 这些功能都可以从核心业务逻辑代码中抽离出来. 
也就是说, 通过AOP可以解决代码耦合问题, 让职责更加单一。

**AOP的作用:**
在程序运行期间，在不修改源码的情况下对方法进行功能增强。

**如何对方法进行增强:**
Spring通过动态代理技术动态的生成代理对象，代理对象方法执行时进行增强功能的介入，在去调用目标对象的方法，从而完成功能的增强。

我们此处用的是cglib作为动态代理,项目中也有jdk动态代理的实现,就不一一细说了。对aop的术语和cglib如何使用就不赘述了，下面直接看aop流程与代码吧。

## 2.实现AOP的简单流程
- 创建@Aspect、@PointCut、@Before、@After与AOP相关的注解
- 在SpringContext.loadAllBeanDefinition的时候，会将含有@Aspect注解的类进行特殊处理，将@PointCut中需要被增强的全部类封装成MethodNode
- 在SpringContext.productBean中，在实例化class的时候，会判断该class的方法在切面类中是否要被增强，如果是则创建动态代理对象,并在对应方法前后添加增强方法




## 3.AOP相关的基础类
### 3.1 @Aspect
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 切面类注解
 */
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Aspect {
}
```


### 3.2 @PointCut
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description
 */

/**
 * 切面类中，切入点表达式的方法注解
 *
 * 示例一：@PointCut("com.xichuan.dev")
 * 对类进行切面，传入的可以是类路径或者是包路径
 * 此时在类执行的前后执行@Before与@After操作
 *
 * 示例二:@PointCut("com.xichuan.dev.impl.MethodAspectImpl.methdFunction()")
 *  此对方法进行切面
 *  此时在方法执行前后执行@Before与@After操作
 *
 */
@Target({ElementType.FIELD,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface PointCut {
    String value();
}

```


### 3.3 @Before
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 切面类中，前置通知方法注解
 */
@Target({ElementType.FIELD,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Before {

}
```


### 3.4 @After
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 切面类中，后置通知方法注解
 */
@Target({ElementType.FIELD,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface After {
}
```


### 3.5 MethodNode
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 保存切面类与被切面类的一些信息
 */
public class MethodNode {

    //切面类的方法，加了@After或@Before注解
    private Method method;

    //是否对方法进行切面
    private boolean isFunction;

    //如果是方法切面，被切面的方法名
    private String methodName;

    public String getMethodName() {
        return methodName;
    }

    public void setMethodName(String methodName) {
        this.methodName = methodName;
    }

    public MethodNode(Method method, boolean isFunction) {
        this.method = method;
        this.isFunction = isFunction;
    }

    public Method getMethod() {
        return method;
    }

    public void setMethod(Method method) {
        this.method = method;
    }

    public boolean isFunction() {
        return isFunction;
    }

    public void setFunction(boolean function) {
        isFunction = function;
    }
}
```

## 5.实现AOP的过程

对ioc的实现已经讲过了，请转下面链接进行阅读


- [手写spring之ioc](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之ioc.md)


下面会将到只会对LoadBeanHelper中与aop相关的代码进行说明



### 5.1 LoadBeanHelper.loadAllBeanDefinition()中处理含有@Aspect注解的类 
```java
    public static void loadAllBeanDefinition() {
        for (Class<?> clazz : classesHashSet) {
            if (!(clazz.isAnnotationPresent(Component.class)||
                    clazz.isAnnotationPresent(Controller.class)||
                    clazz.isAnnotationPresent(Service.class) ||
                    clazz.isAnnotationPresent(Repository.class)))
                continue;

            //创建BeanDefinition
            BeanDefinition newBeanDefine = new BeanDefinition();
            newBeanDefine.setClazz(clazz);
            //获取beanName
            String BeanName=getBeanNameByClass(clazz);
            
            ...
            ...
            
            //对@Aspect切面类的处理
            resolveAspect(clazz);

            //将每一个beanDefinition放在map种
            beanDefinitionHashMap.put(newBeanDefine.getBeanName(), newBeanDefine);
        }
    }
```
我们可以看到在此方法中，`resolveAspect()`会对含有@Aspect注解的类进行特殊处理，进入`resolveAspect()`方法

```java
    /**
     * 对@Aspect切面类的处理
     * @param clazz 切面类class
     */
    private static void resolveAspect(Class<?> clazz){
        //添加切面;(方法上或者类上)
        //判断方法是否有@Aspect切面类注解
        if (clazz.isAnnotationPresent(Aspect.class)) {
            for (Method method : clazz.getMethods()) {
                String annotationPath = "";

                //判断哪一个方法是切入点表达式的注解
                //like this:@PointCut("com.xichuan.dev.aop.service.StudentAopServiceImpl.study()")
                if (method.isAnnotationPresent(PointCut.class)) {
                    String delegateString = method.getAnnotation(PointCut.class).value();
                    annotationPath = delegateString;
                    //切点是方法
                    if (delegateString.charAt(delegateString.length() - 1) == ')') {
                        addMethodAspect(clazz,annotationPath);

                     //切点是某个包或者类
                    }else {
                        //判断
                        URL url = BeanContainer.classLoader.getResource(basePackage.replace(".","/"));
                        //切点是class或者package,从file中加载
                        if (url.getProtocol().equals("file")){
                            addClassAndPackageAspectFromFile(clazz,annotationPath);
                        }else if (url.getProtocol().equals("jar")){   //切点是class或者package,从jar中加载
                            addClassAndPackageAspectFromJar(clazz,annotationPath);
                        }
                    }
                }
            }
        }
    }
```
`resolveAspect()`方法会先判断方法是含有`@Aspect`注解的切面类，如果是则找到含有`@PointCut`的方法，根据切点是方法还是class、package进入不同的分支

先看切点是方法的处理逻辑`addMethodAspect()`

#### 5.1.1 addMethodAspect()
```java

    //切面类中，后置通知方法Map(key:被切面(需要被代理)的类;value:Method(参数一:切面类的方法加了@Before;参数二:切面是否事方法；参数三:如果切面是方法，此值是方法名))
    private static HashMap<String, ArrayList<MethodNode>> beforeDelegatedSet = new HashMap<>();
    //切面类中，前置通知方法Map(key:被切面(需要被代理)的类; value:Method(切面类加了@After的方法;参数二:切面是否事方法；参数三:如果切面是方法，此值是方法名))
    private static HashMap<String, ArrayList<MethodNode>> afterDelegatedSet = new HashMap<>();

    /**
     * 获取Aspect中被切面的方法；切点是method
     * @param aspectClass 切面类class
     * @param annotationPath
     */
    private static void addMethodAspect(Class<?> aspectClass,String annotationPath){
        //说明是在某个方法上面的切面
        annotationPath =annotationPath.replace("()","");
        //切掉方法保留到类
        String[] seg = cutName(annotationPath);
        //seg(0) like this: com.xichuan.dev.boot.service.StudentBootServiceImpl (被切面的类) ； seg(1) like this:study
        addToAspects(aspectClass,seg[0],true,seg[1]);
    }

    /**
     * 添加切入面类
     * @param clazz 切面类Class
     * @param key 需要增强的类
     * @param isMethod 是否是对方法切面
     * @param MethodName 需要增强的方法名
     */
    private static void addToAspects(Class<?> clazz,String key,boolean isMethod,String MethodName) {
        for (Method method : clazz.getMethods()) {
            MethodNode methodNode=new MethodNode(method,isMethod);
            methodNode.setMethodName(MethodName);

            if(method.isAnnotationPresent(Before.class)) {
                if(!beforeDelegatedSet.containsKey(key)) {
                    beforeDelegatedSet.put(key, new ArrayList<>());
                }
                beforeDelegatedSet.get(key).add(methodNode);
            }
            if(method.isAnnotationPresent(After.class)) {
                if(!afterDelegatedSet.containsKey(key)) {
                    afterDelegatedSet.put(key, new ArrayList<>());
                }
                afterDelegatedSet.get(key).add(methodNode);
            }
        }

    }
```
处理切入点是方法逻辑很简单，不说复杂的过程了，举个例子来看`afterDelegatedSet`与`beforeDelegatedSet`

**例子:**
```java
package com.xichuan.dev.aop.service;

@Service
public class StudentAopServiceImpl implements StudentAopService {

    public StudentAopServiceImpl(){
        System.out.println("init StudentAopServiceImpl.....");
    }

    @Override
    public void study(String studentName) {
        System.out.println(studentName + " is start study!");
    }
}
```
有一个service叫`StudentAopServiceImpl`，我们需要将将`study()`方法进行增强,那切面类如下:
```java
/**
 * @Author Xichuan
 * @Date 2022/5/10 10:00
 * @Description
 */
@Component
@Aspect
public class MethodAspect {

    @PointCut("com.xichuan.dev.aop.service.StudentAopServiceImpl.study()")
    public void pointcut() {}

    @Before
    public void before() {
        System.out.println("----Student start study time: "+new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
    }
    @After
    public void after() {
        System.out.println("----Student end study time: "+new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
    }
}
```

在`beforeDelegatedSet`存储的数据结构是:
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211161432319.png)

在`aeforeDelegatedSet`存储的数据结构是:
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211161432065.png)

这样的存储的优势是，当在实例化`StudentAopServiceImpl`的时候，我们可以很方便的找到对应的增强方法

我们继续看`resolveAspect`中的`addClassAndPackageAspectFromFile()`方法

#### 5.1.2 addMethodAspect()
```java
    /**
     * 从文件夹中(IED)中，获取Aspect中被切面的类；切点是class或者package
     * @param aspectClass 切面类class
     * @param annotationPath
     */
    private static void addClassAndPackageAspectFromFile(Class<?> aspectClass,String annotationPath){
        //pointcut annotation的path,(是一个包名或者是一个类名<不带.class>)
        // class like this: com.xichuan.dev.boot.service.TeacherBootServiceImpl
        // package like this: com.xichuan.dev.boot.service
        String annotationPathClone = new String(annotationPath);

        //IDE package like this: file:/D:/development/my_test_code/MySpring/xichuan-mvc-test/target/classes/com/xichuan/dev/boot/service
        //IDE class like this: file:/D:/development/my_test_code/MySpring/xichuan-mvc-test/target/classes/com/xichuan/dev/boot/service/TeacherBootServiceImpl
        URL resource = BeanContainer.classLoader.getResource(annotationPathClone.replace(".","/"));
        //resource是null,说明resource是class
        if(resource==null) {
            resource = BeanContainer.classLoader.getResource(annotationPathClone.replace(".", "/") + ".class");
        }
        File file = new File(resource.getFile());

        //如果切点是包名，需要将包下的所有类注册到切面Map中
        if(file.isDirectory()) {
            ArrayList<File> fileArray=new ArrayList<>();
            //获取该路
            DFSGetCurrentDir(file,fileArray);
            String key;
            for(File f:fileArray) {
                key = f.getAbsolutePath().replace(splitOP,".");
                key = key.substring(key.indexOf(annotationPath),key.indexOf(f.getName())+f.getName().length()-6);
                addToAspects(aspectClass,key,false,"");
            }

            //如果切点是类，直接将类添加进去
        }else {
            String key = file.getAbsolutePath().replace(splitOP,".");
            key = key.substring(key.indexOf(annotationPath),key.indexOf(file.getName())+file.getName().length()-6);
            addToAspects(aspectClass,key,false,"");
        }
    }

    /**
     * 添加切入面类
     * @param clazz 切面类Class
     * @param key 被切面的类
     * @param isMethod 是否是对方法切面
     * @param MethodName 被切面的方法名字
     */
    private static void addToAspects(Class<?> clazz,String key,boolean isMethod,String MethodName) {
        for (Method method : clazz.getMethods()) {
            MethodNode methodNode=new MethodNode(method,isMethod);
            methodNode.setMethodName(MethodName);

            if(method.isAnnotationPresent(Before.class)) {
                if(!beforeDelegatedSet.containsKey(key)) {
                    beforeDelegatedSet.put(key, new ArrayList<>());
                }
                beforeDelegatedSet.get(key).add(methodNode);
            }
            if(method.isAnnotationPresent(After.class)) {
                if(!afterDelegatedSet.containsKey(key)) {
                    afterDelegatedSet.put(key, new ArrayList<>());
                }
                afterDelegatedSet.get(key).add(methodNode);
            }
        }

    }
```
我们可以看出，如果切入点如果是包的话，会扫描包下的所有类添加到`beforeDelegatedSet`与`beforeDelegatedSet`中



### 5.2 LoadBeanHelper.productBean：对需要增强的类创建动态代理对象
此方法会实例化类，们进入此方法看如果对需要增强的类创建动态代理对象
```java
    /**
     * 生产单例bean,将需要代理的bean进行代理，放到一级缓存中
     */
    public static void productBean() {
        for (String beanName : beanDefinitionHashMap.keySet()) {
            BeanDefinition beanDefinition = beanDefinitionHashMap.get(beanName);
            //如果是单例变成生产工厂
            if (beanDefinition.getScope().equals(ScopeEnum.SingleTon.getName())) {
                //创建单例bean
                createBean(beanDefinition,true);
            }
        }
    }
  /**
     * 创建bean，并进行代理
     * @param beanDefinition bean的定义信息
     * @param singleton 是否是单例bean
     * @return
     */
    private static Object createBean(BeanDefinition beanDefinition, Boolean singleton) {
        try {
            //如果在一级或者二级直接返回;如果是在三级缓存，则将三级缓存中的bean移到二级缓存中
            if(BeanContainer.singletonObjects.containsKey(beanDefinition.getBeanName())&&singleton)
                return BeanContainer.singletonObjects.get(beanDefinition);
            else if(BeanContainer.earlySingletonObjects.containsKey(beanDefinition.getBeanName())) {
                return BeanContainer.earlySingletonObjects.get(beanDefinition.getBeanName());
            }else if(BeanContainer.singletonFactory.containsKey(beanDefinition.getBeanName())){
                //将此bean放在二级缓存中，并在三级缓存中删除
                BeanContainer.earlySingletonObjects.put(beanDefinition.getBeanName(), BeanContainer.singletonFactory.get(beanDefinition.getBeanName()).getTarget());
                BeanContainer.singletonFactory.remove(beanDefinition.getBeanName());
                return BeanContainer.earlySingletonObjects.get(beanDefinition.getBeanName());

            //此bean不存在
            }else {
                //如果该类是接口，直接返回null
                if(beanDefinition.getClazz().isInterface())
                   return null;

                //将bean对象放到动态代理工厂中
                DynamicBeanFactory dynamicBeanFactory = new DynamicBeanFactory();
                dynamicBeanFactory.setBeanDefinition(beanDefinition);
                dynamicBeanFactory.setClazz(beanDefinition.getClazz());

                //查看是否存在切面并放入工厂中，在工厂中准备代理
                //如果类使用了aop，需要进行动态代理处理
                if(beforeDelegatedSet.containsKey(beanDefinition.getClazz().getName())) {
                    dynamicBeanFactory.setDelegated(true);
                    dynamicBeanFactory.setBeforeMethodCache(beforeDelegatedSet.get(beanDefinition.getClazz().getName()));
                }
                if(afterDelegatedSet.containsKey(beanDefinition.getClazz().getName())) {
                    dynamicBeanFactory.setDelegated(true);
                    dynamicBeanFactory.setAfterMethodCache(afterDelegatedSet.get(beanDefinition.getClazz().getName()));
                }

                //创建代理对象或者实例对象
                dynamicBeanFactory.createInstance();
            
                ...
                ...
            }
        }
    }
```
可以看出，当创建对象之前会判断此类在`beforeDelegatedSet`与`afterDelegatedSet`中存在，如果存在，说明此类需要进行动态代理并需要添加增强方法,此时会将`DynamicBeanFactory.isDelegated`设置为true，那进入`DynamicBeanFactory.createInstance()`方法
```java
public class DynamicBeanFactory {
    //Bean定义信息
    private BeanDefinition beanDefinition;

    //被代理对象的class
    private Class clazz;

    //Bean实例化对象(原始对象)
    private Object instance;

    //代理对象
    private Object proxyInstance;

    //是否动态代理
    private Boolean isDelegated = false;

    //aop的前置处理方法与后置处理方法
    private ArrayList<MethodNode> beforeMethodCache;
    private ArrayList<MethodNode> afterMethodCache;

    //是否使用cglib
    private boolean isCGlib = true;

    /**
     * 创建实例对象或者代理对象
     * @return
     * @throws NoSuchMethodException
     * @throws IllegalAccessException
     * @throws InvocationTargetException
     * @throws InstantiationException
     */
    public Object createInstance() throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        //如果是CGlib，则只需要创建代理类即可
        if (isDelegated && proxyInstance == null) {
            //创建代理工具类
            if (isCGlib) {
                BeanCGlibProxy beanCGlibProxy = new BeanCGlibProxy();
                beanCGlibProxy.setAfterMethodCache(getAfterMethodCache());
                beanCGlibProxy.setBeforeMethodCache(getBeforeMethodCache());
                proxyInstance = beanCGlibProxy.creatCarProxy(clazz);
            } else {
                instance = clazz.getDeclaredConstructor().newInstance();
                BeanJDKProxy beanJDKProxy = new BeanJDKProxy();
                beanJDKProxy.setAfterMethodCache(getAfterMethodCache());
                beanJDKProxy.setBeforeMethodCache(getBeforeMethodCache());
                proxyInstance = beanJDKProxy.creatCarProxy(instance);
            }
            return proxyInstance;
        } else if (isDelegated && proxyInstance != null) {
            return proxyInstance;
        } else {  //不代理的话直接返回
            instance = clazz.getDeclaredConstructor().newInstance();
            return instance;
        }
    }

    /**
     * 返回实例对象或者代理对象
     * @return
     */
    public Object getTarget() throws InvocationTargetException, NoSuchMethodException, InstantiationException, IllegalAccessException {
        //代理的话返回代理对象，不代理的话返回本身对象
        if (isDelegated) {
            return proxyInstance == null ? createInstance() : proxyInstance;
        } else {
            return instance == null ? createInstance() : instance;
        }
    }
    ...
    ...
}
```
`DynamicBeanFactory`在调用CGlib创建对象的时候，是先将`beforeMethod`与`afterMethod`增强方法赋值到`BeanCGlibProxy`中，继续往下看`BeanCGlibProxy`如何创建代理对象

```java
/**
 * @Author Xichuan
 * @Date 2022/5/10 13:33
 * @Description CGlib 动态代理工具类
 */
public class BeanCGlibProxy {

    //有@Aspect注解的@Before前置处理
    private ArrayList<MethodNode> beforeMethodCache;
    //有@Aspect注解的@After后置处理
    private ArrayList<MethodNode> afterMethodCache;

    /**
     * 初始化代理对象class
     * @param targetClass
     * @return
     */
    public Object creatCarProxy(Class targetClass) {
        try {
            Object proxy = this.createProxy(targetClass);
            return proxy;
        }catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 输入一个目标类, 输出一个代理对象
     */
    @SuppressWarnings("unchecked")
    public <T> T createProxy(final Class<?> targetClass) {
        return (T) Enhancer.create(targetClass, new MethodInterceptor() {
            /**
             * 代理方法, 每次调用目标方法时都会先创建一个 ProxyChain 对象, 然后调用该对象的 doProxyChain() 方法.
             */
            @Override
            public Object intercept(Object targetObject, Method targetMethod, Object[] methodParams, MethodProxy methodProxy) throws Throwable {
                return doProxyChain(targetObject,targetMethod,methodParams,methodProxy);
            }
        });
    }

    /**
     * 进行拦截执行
     * @param targetObject 被代理对象
     * @param targetMethod 被代理的方法
     * @param methodParams 方法参数
     * @param methodProxy
     * @return
     * @throws Throwable
     */
    public Object doProxyChain(Object targetObject, Method targetMethod, Object[] methodParams, MethodProxy methodProxy) throws Throwable {
        Object methodResult;

        before(targetMethod.getName());
        //目标方法执行
        methodResult = methodProxy.invokeSuper(targetObject, methodParams);
        after(targetMethod.getName());

        return methodResult;
    }

    /**
     * 方法执行前执行
     * @param methodName
     * @throws Throwable
     */
    public void before(String methodName) throws Throwable{
        //整个类级别的切面Before方法执行
        if(beforeMethodCache!=null) {
            for (MethodNode beforeMethod : beforeMethodCache) {
                if ((!beforeMethod.isFunction())) {
                    String[] name = beforeMethod.getMethod().getDeclaringClass().toString().split("\\.");
                    beforeMethod.getMethod().invoke(BeanContainer.singletonObjects.get(name[name.length - 1]));
                }
            }
        }

        //特定方法Before执行
        if(beforeMethodCache!=null) {
            for (MethodNode beforeMethod : beforeMethodCache) {
                if (beforeMethod.isFunction() && beforeMethod.getMethodName().equals(methodName)) {
                    String[] name = beforeMethod.getMethod().getDeclaringClass().toString().split("\\.");
                    beforeMethod.getMethod().invoke(BeanContainer.singletonObjects.get(name[name.length - 1]));
                }
            }
        }
    }

    /**
     * 方法执行后执行
     * @param methodName
     * @throws Throwable
     */
    public void after(String methodName) throws Throwable{
        //特定方法After执行
        if(afterMethodCache!=null) {
            for (MethodNode afterMethod : afterMethodCache) {
                if (afterMethod.isFunction() && afterMethod.getMethodName().equals(methodName)) {
                    String[] name = afterMethod.getMethod().getDeclaringClass().toString().split("\\.");
                    afterMethod.getMethod().invoke(BeanContainer.singletonObjects.get(name[name.length - 1]));
                }
            }
        }

        //类级别After执行
        if(afterMethodCache!=null) {
            for (MethodNode afterMethod : afterMethodCache) {
                if ((!afterMethod.isFunction())) {
                    String[] name = afterMethod.getMethod().getDeclaringClass().toString().split("\\.");
                    afterMethod.getMethod().invoke(BeanContainer.singletonObjects.get(name[name.length - 1]));
                }
            }
        }
    }

    public ArrayList<MethodNode> getBeforeMethodCache() {
        return beforeMethodCache;
    }

    public void setBeforeMethodCache(ArrayList<MethodNode> beforeMethodCache) {
        this.beforeMethodCache = beforeMethodCache;
    }

    public ArrayList<MethodNode> getAfterMethodCache() {
        return afterMethodCache;
    }

    public void setAfterMethodCache(ArrayList<MethodNode> afterMethodCache) {
        this.afterMethodCache = afterMethodCache;
    }
}
```
可以看出，在调用切入类的方法之前会调用`before()`逻辑，在调用切入类的方法之前会调用`after()`逻辑。`before()`与`after()`中正是增强的方法的调用。


此处就将aop的核心讲完了。