## 前言
在前面的文章中，我们使用@Bean(或者@Component)注解将Bean注册到了Spring容器；我们创建这些Bean的目的，最终还是为了使用，@Autowired注解可以将bean自动注入到类的属性中。@Autowired注解可以直接标注在属性上，也可以标注在构造器，方法，甚至是传入参数上，其实质都是调用了setter方法注入。
## 源码
```
java
复制代码Source Code: 
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
    boolean required() default true;
}
```
当自动注入的Bean不存在时，Spring会报错；如果希望Bean存在时才注入，可以使用@Autowired(required=false)。
## 使用
标注在属性上：
```

@Autowired
private MyBean myBean;

```
标注在方法上：
```

@Autowired
public void setMyBean(MyBean myBean) {
    this.myBean = myBean;
}
```
标注在构造函数上：
```

@Autowired
public MyClass(MyBean myBean) {
    this.myBean = myBean;
}

```
标注在方法参数上：
```

public void setMyBean(@Autowired MyBean myBean) {
    this.myBean = myBean;
}

```
## 补充说明

1. @Autowired和@Resource注解都是作为bean对象注入时使用的，@Autowired是Spring提供的注解，而@Resource是J2EE本身提供的。
2. @Autowired首先根据类型去寻找注入的对象，如果有多个再根据名字匹配。
3. 当名字也无法区分时可以通过@Qulifier显式指定，如：
```
@Autowired
@Qualifier("userServiceImpl1")
private UserService userService;
```


## 参考资料
[Spring Boot注解全攻略(四)：@Autowired - 掘金](https://juejin.cn/post/6959753065297608711)
