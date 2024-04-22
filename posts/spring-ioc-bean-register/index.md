# Spring容器注册bean有哪些方式？

常见注册bean对象到spring容器的方式:

- @Component、@Controller、@Service、@Repository 方式
- @Bean 工厂方式
- @mport 普通类
- @Import ImportSelector
- @Import ImportBeanDefinitionRegistrar
- 实现 BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry

## 方式一：@注解方式

最常见的可以用 @Component、@Controller、@Service、@Repository 注解方式注入对象到spring容器中。

```java
@Component
public class RequestTemplate{
  
}
```

```java
@Service
public class UserServiceImpl implements UserService {
  
}
```

## 方式二：Bean 工厂方式

```java
@Configuration
public class AppConfig {
    /**
     * 注册 user对象，默认方法名为beanName
    */
    @Bean
    public User user(){
        return new User();
    }
}
```

## 方式三：@Import  普通类

```java
@Import(Cat.class)
@Configuration
public class AppConfig {
    
}
```

## 方式四：@Import ImportSelector 

可以通过 import 实现ImportSelector接口的类，重写selectImports方法可以再注册其他的类。

```java
public class UserImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
       // 注册User类型到spring容器中
        return new String[]{User.class.getName()};
    }
}
```

```java

@Import(UserImportSelector.class)
public class ImportSelectorSpringBeanDemo {

    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(ImportSelectorSpringBeanDemo.class);
        System.out.println(ctx.getBean(User.class));
    }
}
```

## 方式五：@Import ImportBeanDefinitionRegistrar

通过import实现 ImportBeanDefinitionRegistrar 接口的类，重写registerBeanDefinitions方法可以再注册其他类。

```java
public class UserImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
				
       // 注册User的ben定义
        BeanDefinition beanDefinition = new RootBeanDefinition(User.class);
        registry.registerBeanDefinition(User.class.getName(), beanDefinition);
    }
}
```

```java
@Import(UserImportBeanDefinitionRegistrar.class)
public class ImportBeanDefinitionRegistrarSpringBeanDemo {

    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(ImportBeanDefinitionRegistrarSpringBeanDemo.class);
        System.out.println(ctx.getBean(User.class));
    }
}
```

## 方式六：实现 BeanDefinitionRegistryPostProcessor 接口

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
				// 注册其他bean
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Dog.class);
        registry.registerBeanDefinition(&#34;dog&#34;, rootBeanDefinition);
    }

```

前提先注册 MyBeanDefinitionRegistryPostProcessor对象, spring容器才能触发 postProcessBeanDefinitionRegistry 方法。

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/spring-ioc-bean-register/  

