# 手写spring之ioc

仓库地址:
- [Raray-chuan/mini-spring](https://github.com/Raray-chuan/mini-spring)

博文列表:
- [导读](https://github.com/Raray-chuan/mini-spring/blob/main/README.md)
- [手写spring之ioc](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之ioc.md)
- [手写spring之aop](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之aop.md)
- [手写spring之简单实现springboot](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之简单实现springboot.md)



## 1.什么是IOC
   IOC Inversion of Control 即控制反转，是指程序将创建对象的控制权转交给Spring框架进行管理，由Spring通过java的反射机制根据配置文件在运行时动态的创建实例，并管理各个实例之间的依赖关系。

## 2.什么是三级缓存
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211151651233.png)
- 第一级缓存：也叫单例池，存放已经经历了完整生命周期的Bean对象。
- 第二级缓存：存放早期暴露出来的Bean对象，实例化以后，就把对象放到这个Map中。（Bean可能只经过实例化，属性还未填充）。
- 第三级缓存：存放早期暴露的Bean的工厂。
```
注：
只有单例的bean会通过三级缓存提前暴露来解决循环依赖的问题，而非单例的bean，每次从容器中获取都是一个新的对象，都会重新创建，所以非单例的bean是没有缓存的，不会将其放到三级缓存中。

为了解决第二级缓存中AOP生成新对象的问题，Spring就提前AOP，比如在加载b的流程中，如果发送了循环依赖，b依赖了a，就要对a执行AOP，提前获取增强以后的a对象，这样b对象依赖的a对象就是增强以后的a了。

二三级缓存就是为了解决循环依赖，且之所以是二三级缓存而不是二级缓存，主要是可以解决循环依赖对象需要提前被aop代理，以及如果没有循环依赖，早期的bean也不会真正暴露，不用提前执行代理过程，也不用重复执行代理过程。
```


## 3.手动实现的简单流程
- 创建@ComponentScan,@Component,@Controller,@Repository,@Service,@Autowired,@Scope等注解
- 创建三级缓存类
- 在初始化SpringContext的时候扫描基础包下的所有class,存储到一个集合中
- 将类上含有@Component,@Controller,@Repository,@Service注解的class,封装成BeanDefinition,然后存储到 Bean容器中
- 遍历Bean容器中的所有Bean,如果是单例对象,实例化bean并放在三级缓存中。如果该bean上有@Autowired注解,需要将@Autowired的类实例化放在三级缓存中




## 4.IOC相关的基础类

### 4.1 定义IOC相关的注解

#### 4.1.1 @ComponentScan
设置包根路径注解
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description 设置根路径的注解
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface ComponentScan {
    String value();
}
```
#### 4.1.2 @Component
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description Component通用注解
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {
    String value() default "";
}
```
#### 4.1.3 @Controller
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description Controller注解
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Controller {
    String value() default "";
}
```
#### 4.1.4 @Repository
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:43
 * @Description Repository注解
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Repository {
    String value() default "";
}
```
#### 4.1.5 @Service
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Service {
    String value() default "";
}
```
#### 4.1.6 @Autowired
```java
@Target({ElementType.FIELD,ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
    String value() default "";
}
```
#### 4.1.7 @Scope
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Scope {
    String value();
}
```



### 4.2 BeanDefinition
在基础包下的每一个class在被SpringContext扫描到后，都会封装成BeanDefinition进行管理
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description Bean定义信息
 */
public class BeanDefinition {

    //Class
    private Class clazz;

    //Bean名词
    private String beanName;

    //是否是Controller注解
    private  boolean isController = false;

    //作用域1、singleton 2、prototype
    private String scope;

    //根据扩容原理，如果不添加元素，永远是空数组不占用空间
    private ArrayList<BeanPostProcessor> beanPostProcessors=new ArrayList<>();

    ...
    ...
}
```


### 4.3 BeanContainer 存放三级缓存的类
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description bean实例容器
 */
public class BeanContainer {
    //一级缓存，日常实际获取Bean的地方
    public static ConcurrentHashMap<String,Object> singletonObjects = new ConcurrentHashMap<>();
    //二级缓存，临时
    public static ConcurrentHashMap<String,Object> earlySingletonObjects = new ConcurrentHashMap<>();
    //三级缓存，value是一个动态代理对象工厂
    public static ConcurrentHashMap<String, DynamicBeanFactory> singletonFactory = new ConcurrentHashMap<>();
    //Controller对象容器
    public static ConcurrentHashMap<String,Object> controllerMap = new ConcurrentHashMap<>();
    //类加载器,在SpringContext的静态代码块中进行赋值的
    public static ClassLoader classLoader = null;
}
```



## 5.实现IOC的过程
### 5.1 SpringContext初始化过程
```java
/**
 * @Author Xichuan
 * @Date 2022/5/7 11:25
 * @Description
 */
public class SpringContext {

    static {
        ClassLoader classLoader = SpringContext.class.getClassLoader();//拿到应用类加载器
        //给容器类赋值类加载器
        BeanContainer.classLoader=classLoader;
    }

    /**
     * 通过@ComponentScan注解获取根路径，从而加载根路径下的所有class
     * @param config
     */
    public SpringContext(Class<?> config) {
        if(BeanContainer.singletonObjects.size()==0) {
            //获取根路径
            ComponentScan componentScanAnnotation = (ComponentScan) config.getAnnotation(ComponentScan.class);
            String path = componentScanAnnotation.value();
            //获取packagePath下的所有class，注册到classesHashSet
            LoadBeanHelper.loadAllClass(path);
            //将BeanDefinition、BeforeDelegatedSet、AfterDelegatedSet、BeanPostProcessorList进行注册
            LoadBeanHelper.loadAllBeanDefinition();
            //生产bean,将需要代理的bean进行代理，放到一级缓存中
            LoadBeanHelper.productBean();
        }
    }

    /**
     * 通过config.properties配置文件，来加载根路径，从而加载根路径下的所有class
     */
    public SpringContext() {
        if(BeanContainer.singletonObjects.size()==0) {
            //获取packagePath下的所有class，注册到classesHashSet
            LoadBeanHelper.loadAllClass(ConfigHelper.getAppBasePackage());
            //将BeanDefinition、BeforeDelegatedSet、AfterDelegatedSet、BeanPostProcessorList进行注册
            LoadBeanHelper.loadAllBeanDefinition();
            //生产单例bean,将需要代理的bean进行代理，放到一级缓存中
            LoadBeanHelper.productBean();
        }

    }
}
```
我们先看这两个重载构造方法的不同：`SpringContext(Class<?> config)`与`SpringContext()`,
这两个重载方法的区别在于：

`SpringContext(Class<?> config)`构造方法是通过`@ComponentScan`注解来获取包的根路径
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211151824800.png)
`SpringContext()`构造方法是通过`config.properties`配置文件中的`scan.dir=com.xichuan.dev`配置来获取包的根路径
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211151822194.png)

我们可以看出`SpringContext`做了三件事:
- `loadAllClass()`:获取`packagePath`下的所有class,将所有扫描到的class存放到`classesHashSet`中
- `loadAllBeanDefinition()`:遍历`classesHashSet`中的所有class，如果该class上含有`@Component,@Controller,@Repository,@Service`注解，将该class封装成`BeanDefinition`,并存放到`beanDefinitionHashMap`中
- `productBean()`:核心方法,遍历`beanDefinitionHashMap`中的所有`BeanDefinition`,将符合条件的`BeanDefinition`存放到`BeanContainer.singletonObjects`一级缓存中

我们依次看这个三个方法的代码



### 5.2 LoadBeanHelper.loadAllClass 加载所有的class
```java
    /**
     *获取packagePath下的所有class
     *
     * @param packagePath 扫描路径
     * @return 所有类的class对象
     */
    public static Set<Class<?>> loadAllClass(String packagePath) {
        basePackage = packagePath;
        List<String> classNames = PackageUtil.getClassName(packagePath);
        Class<?>clazz;
        for (String className:classNames) {
            try {
                clazz=classLoader.loadClass(className);
                classesHashSet.add(clazz);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        }
        return classesHashSet;
    }
```
我们可以看到，此方法的代码非常简单，只是将`packagePath`包路径下的class扫描出来，并存放到`classesHashSet`中;因为代码比较简单，递归获取class的操作就不细说了。

有一点需要注意的是，因为在idea运行与达成jar运行,扫描包的方式不一样,具体代码看此方法:`com.xichuan.framework.core.helper.PackageUtil.getClassName(java.lang.String, boolean)`
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211151910741.png)



### 5.3 LoadBeanHelper.loadAllBeanDefinition,将class封装成BeanDefinition
```java
    /**
     * 将所有的BeanDefinition放入map中，（只有@Component、@Controller、@Service、@Repository才会放入）
     */
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

            //判断是否是Controller类
            if(clazz.isAnnotationPresent(Controller.class))
                newBeanDefine.setIsController(true);
            newBeanDefine.setBeanName(BeanName);

            //加载后置处理器
            resolveBeanPostProcessor(clazz);

            //scope作用域注解
            if (clazz.isAnnotationPresent(Scope.class)) {
                String scope = clazz.getDeclaredAnnotation(Scope.class).value();
                newBeanDefine.setScope(scope);
            } else {
                //默认为单例模式
                newBeanDefine.setScope(ScopeEnum.SingleTon.getName());
            }

            //对@Aspect切面类的处理
            resolveAspect(clazz);

            //将每一个beanDefinition放在map种
            beanDefinitionHashMap.put(newBeanDefine.getBeanName(), newBeanDefine);
        }
    }
```
我们可以看出，此方法也是很简单，
此方法遍历`classesHashSet`中的所有class，如果该class上含有`@Component,@Controller,@Repository,@Service`注解，将该class封装成`BeanDefinition`,并存放到`beanDefinitionHashMap`中

`BeanDefinition`的属性有:
- `clazz`:此bean的clazz
- `beanName`:此bean的名称
- `isController`:此bean是否还有`@Controller`注解
- `scope`:作用域,`singleton`或`prototype`

此方法还有对aop相关的处理，此处先不过多介绍，下一个文章会专门介绍aop相关的代码



### 5.4 LoadBeanHelper.productBean() 将所有BeanDefinition实例化成对象
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
```

我们可以看出，如果是单例对象的话，在初始化SpringContext的时候，就会创建bean并放到一级缓存中

```java
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

            //此bean不存在，或者在二级缓存中时的逻辑代码
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

                //扔到三级缓存
                BeanContainer.singletonFactory.put(beanDefinition.getBeanName(), dynamicBeanFactory);

                //将此bean上的@Autowired注解的类都进行注入(DI注入)
                Object targetBean = populate(beanDefinition.getBeanName());

                //将对象从三级缓存与二级缓存中清除
                if(BeanContainer.earlySingletonObjects.containsKey(beanDefinition.getBeanName()))
                    BeanContainer.earlySingletonObjects.remove(beanDefinition.getBeanName());
                if(BeanContainer.singletonFactory.containsKey(beanDefinition.getBeanName()))
                    BeanContainer.singletonFactory.remove(beanDefinition.getBeanName());
                //将bean对象存放到一级缓存中
                BeanContainer.singletonObjects.put(beanDefinition.getBeanName(),targetBean);


                //加入ControllerMap引用
                if(beanDefinition.isController()) {
                    BeanContainer.controllerMap.put(beanDefinition.getBeanName(), BeanContainer.singletonObjects.get(beanDefinition.getBeanName()));
                }

                //处理BeanNameAware的setBeanName
                if(targetBean instanceof BeanNameAware) {
                    ((BeanNameAware)targetBean).setBeanName(beanDefinition.getBeanName());
                }

                //Spring容器中完成bean实例化、配置以及其他初始化方法前添加一些自己逻辑处理
                for(BeanPostProcessor processor:beanPostProcessorList) {
                    BeanContainer.singletonObjects.put(beanDefinition.getBeanName(),processor.postProcessBeforeInitialization(targetBean, beanDefinition.getBeanName()));
                }

                //InitializingBean接口为bean提供了初始化方法的方式，它只包括afterPropertiesSet方法，凡是继承该接口的类，在初始化bean的时候会执行该方法。
                if(targetBean instanceof InitializingBean) {
                   ((InitializingBean) targetBean).afterPropertiesSet();
                }

                //Spring容器中完成bean实例化、配置以及其他初始化方法后添加一些自己逻辑处理
                for(BeanPostProcessor processor:beanPostProcessorList) {
                    BeanContainer.singletonObjects.put(beanDefinition.getBeanName(),processor.postProcessAfterInitialization(targetBean, beanDefinition.getBeanName()));
                }
            }

        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }

        Object rs = BeanContainer.singletonObjects.get(beanDefinition.getBeanName());
        //如果不是单例，要将上面处理的一级缓存中的单例bean清除，并返回bean对象
        if(!singleton) {
            //不是单例删除
            BeanContainer.singletonObjects.remove(beanDefinition.getBeanName());
            return rs;
        }
        return rs;
    }
```
我们看下这个方法的流程(关于aop的处理，此处先不详细介绍):
- 1.如果在三级缓存中没有这个bean的话，需要先创建`DynamicBeanFactory`动态代理工厂类
- 2.如果该类的方法中还有@Before与@After注解，需要通过CGlib创建代理对象,如果不含有这两个注解，则直接通过反射创建对象
- 3.将创建的对象存放到第三级缓存中
- 4.此bean中含有@Autowired注解的Field都进行注入(DI注入)
- 5.将对象从三级缓存与二级缓存中清除,并存放到一级缓存中
- 6.如果此bean含有`@Controller`注解，将此bean放到`ControllerMap`中,在springmvc中会用到
- 7.对`BeanNameAware`、`InitializingBean`、`BeanPostProcessor`接口的处理

其实这个方法并不复杂，主要关系的是如何通过动态代理创建对象，以及此bean中含有@Autowired注解的Field都进行注入(DI注入)。
动态代理我们在aop篇章会讲到，现在我们看下`com.xichuan.framework.core.helper.LoadBeanHelper.populate()`方法

```java
    /**
     * 获取属性，填充属性；对@Autowired的处理
     * @param beanName
     * @return
     */
    private static Object populate(String beanName) {
        try {
            //获取bean的class
            Class<?> beanClass = null;
            if(BeanContainer.singletonFactory.containsKey(beanName))
                beanClass = BeanContainer.singletonFactory.get(beanName).getClazz();
            else if(BeanContainer.earlySingletonObjects.containsKey(beanName))
                beanClass = BeanContainer.earlySingletonObjects.get(beanName).getClass();


            //遍历bean的方法
            for (Field declaredField : beanClass.getDeclaredFields()) {
                if(!declaredField.isAnnotationPresent(Autowired.class))
                    continue;

                String methodBenName = null;
                //如果此类是接口的话，获取此方法的实现类；如果没有实现类，则获取类本身
                Class<?> implementClass = findImplementClass(declaredField.getType());
                //如果实现类为null，那么就用本身
                if (implementClass == null){
                    implementClass = declaredField.getDeclaringClass();
                }
                //通过class获取beanName
                methodBenName = getBeanNameByClass(implementClass);

                //如果@Autowired的value不为"",那么beanName就是value的值
                if(!declaredField.getAnnotation(Autowired.class).value().equals(""))
                    methodBenName=declaredField.getAnnotation(Autowired.class).value();

                //获取此方法上的bean
                Object methodBean = getBean(methodBenName);
                declaredField.setAccessible(true);

                //重新设置该方法属性值（即：对接口注入子类对象）
                //declaredField.set(bean,methodBean);
                if(BeanContainer.singletonFactory.containsKey(beanName)) {

                    //如果是CGlib设置代理对象属性，如果是jdk Proxy设置原始对象的属性；否则报错
                    if (BeanContainer.singletonFactory.get(beanName).isCGlib()){
                        //Field.set(该Field所属的类对象，该对象的新值)
                        declaredField.set(BeanContainer.singletonFactory.get(beanName).getTarget(), methodBean);
                    }else{
                        declaredField.set(BeanContainer.singletonFactory.get(beanName).getInstance(), methodBean);
                    }
                } else if(BeanContainer.earlySingletonObjects.containsKey(beanName))
                    declaredField.set(BeanContainer.earlySingletonObjects.get(beanName),methodBean);
            }

            //返回此类的bean
            if(BeanContainer.singletonFactory.containsKey(beanName)) {
                Object res = BeanContainer.singletonFactory.get(beanName).getTarget();
                return res;
            } else if(BeanContainer.earlySingletonObjects.containsKey(beanName)) {
                Object res = BeanContainer.earlySingletonObjects.get(beanName);
                return  res;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
```
我们可以看出`populate()`方法是通过遍历该class的所有Field，如果此Filed含有`@Autowired`注解，会继续调用`getBean()`方法获取实例后的bean,并赋值给该Field。



上面又是IOC的核心代码了，当`SpringContext`初始化完成后，会将含有`@Component,@Controller,@Repository,@Service`的bean进行初始化，放到一级缓存中,并bean中含有`@Autowired`注解的Field都进行注入。
这样我们就可以通过`SpringContext.getBean(String beanName)`方式,通过beanName获取到该bean的实例。

