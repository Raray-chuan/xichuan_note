# 手写Spring系列
手写Spring,支持ioc(三级缓存)、aop(cglib)、内嵌tomcat.
目前实现的注解有@Autowired、@Aspect、@PointCut、@After、@Before、@ComponentScan、 
@Scope、@Controller、@Service、@Repository、@Component、@RequestBody、@RequestParam、@RequestMapping.

**仓库地址:**[Raray-chuan/mini-spring](https://github.com/Raray-chuan/mini-spring)

手写spring的目的是为了更好了解ioc、aop以及springmvc、springboot的工作机制，也是为了更好了解spring常用注解的是如何实现的，
如果你对ioc与aop以及servlet、tomcat有一些了解，可以根据下面文章了解整个代码是如何实现的.

阅读之前如果对以下知识有所了解，会更好的读懂代码:
- Java自定义注解
- Java反射机制
- 静态代理与动态代理
- servlet的使用
- 通过Java代码启动Tomcat
- 常用的设计模式


博文列表:
- [手写spring之ioc](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之ioc.md)
- [手写spring之aop](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之aop.md)
- [手写spring之简单实现springboot](https://github.com/Raray-chuan/mini-spring/tree/main/doc/手写spring之简单实现springboot.md)

