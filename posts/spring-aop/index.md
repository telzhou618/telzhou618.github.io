# Spring AOP 从入门到源码解析


## 先学会如何使用

### Spring-boot 或 Spring-cloud  中使用AOP

在Spring-boot 工程中使用AOP功能，先引入相关的依赖。

pom.xml

```xml
 &lt;dependency&gt;
    &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
    &lt;artifactId&gt;spring-boot-starter-web&lt;/artifactId&gt;
&lt;/dependency&gt;

&lt;dependency&gt;
  &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
  &lt;artifactId&gt;spring-boot-starter-aop&lt;/artifactId&gt;
&lt;/dependency&gt;

&lt;dependency&gt;
  &lt;groupId&gt;org.projectlombok&lt;/groupId&gt;
  &lt;artifactId&gt;lombok&lt;/artifactId&gt;
  &lt;version&gt;1.18.22&lt;/version&gt;
&lt;/dependency&gt;

&lt;dependency&gt;
  &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
  &lt;artifactId&gt;spring-boot-starter-test&lt;/artifactId&gt;
  &lt;scope&gt;test&lt;/scope&gt;
  &lt;exclusions&gt;
    &lt;exclusion&gt;
      &lt;groupId&gt;org.junit.vintage&lt;/groupId&gt;
      &lt;artifactId&gt;junit-vintage-engine&lt;/artifactId&gt;
    &lt;/exclusion&gt;
  &lt;/exclusions&gt;
&lt;/dependency&gt;
```

测试业务接口  UserService.java

```java
public interface UserService {
    void test();
}
```

测试业务接口实现类 UserServiceImp.java

```java
@Slf4j
@Service
public class UserServiceImpl implements UserService {
    @Override
    public void test() {
        log.info(&#34;执行业务方法test...&#34;);
    }
}
```

切面类 LogAspect.java

```java
@Slf4j
@Aspect
@Component
public class LogAspect {

    @Pointcut(&#34;execution(public * com.example.service.impl.*.*(..))&#34;)
    public void pointcut() {
    }

    @Before(&#34;pointcut()&#34;)
    public void before() {
        log.info(&#34;Before通知 -&gt; 业务方法执行前调用...&#34;);
    }

    @After(&#34;pointcut()&#34;)
    public void after() {
        log.info(&#34;After通知 -&gt; 业务方法执行后调用...&#34;);
    }

}
```

启动类 Application.java

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

执行测试类 UserServiceTest.java

```java

@SpringBootTest
public class SpringTests {

}

class UserServiceTest extends SpringTests {

    @Autowired
    private UserService userService;
    @Test
    void test() {
        userService.test();
    }

}
```

```
INFO 53837 --- [           main] com.example.aspect.LogAspect             : Before通知 -&gt; 业务方法执行前调用...
INFO 53837 --- [           main] c.example.service.impl.UserServiceImpl   : 执行业务方法test...
INFO 53837 --- [           main] com.example.aspect.LogAspect             : After通知 -&gt; 业务方法执行后调用...
```

可以看到，在springboot项目中只要引入  spring-boot-starter-aop 包，默认会自动开启AOP功能。

这是因为SpringBoot自动配置的特性，如下是AOP的配置类，默认是开启。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(prefix = &#34;spring.aop&#34;, name = &#34;auto&#34;, havingValue = &#34;true&#34;, matchIfMissing = true)
public class AopAutoConfiguration {
  //
  @Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(Advice.class)
	static class AspectJAutoProxyingConfiguration {

		@Configuration(proxyBeanMethods = false)
		@EnableAspectJAutoProxy(proxyTargetClass = false)
		@ConditionalOnProperty(prefix = &#34;spring.aop&#34;, name = &#34;proxy-target-class&#34;, havingValue = &#34;false&#34;,
				matchIfMissing = false)
		static class JdkDynamicAutoProxyConfiguration {

		}

		@Configuration(proxyBeanMethods = false)
		@EnableAspectJAutoProxy(proxyTargetClass = true)
		@ConditionalOnProperty(prefix = &#34;spring.aop&#34;, name = &#34;proxy-target-class&#34;, havingValue = &#34;true&#34;,
				matchIfMissing = true)
		static class CglibAutoProxyConfiguration {

		}

	}
}
```

- 根据ConditionalOnProperty条件注解，当前环境中存在 spring.aop.auto，且值为true会启用该配置类， 而matchIfMissing的值为 true, 表明当没有手动配置 spring.aop.auto 时其默认值为true, 即默认开启AOP自动配置。
- havingValue 表示手动配置的值和havingValue的值相等时才生效。

- 通过 AspectJAutoProxyingConfiguration 内部类注入 JdkDynamicAutoProxyConfiguration 和  CglibAutoProxyConfiguration 配置类型，到底哪一个会生效呢 ? 
- CglibAutoProxyConfiguration会生效，因为没有配置spring.aop.proxy-target-class时CglibAutoProxyConfiguration上的默认值为true, 所以CglibAutoProxyConfiguration配置生效。
- matchIfMissing 表示没有手动配置时 spring.aop.proxy-target-class 配置的默认值。
- 通过 @EnableAspectJAutoProxy(proxyTargetClass = true)  开启AOP功能，这是实现AOP的关键。

## 原理分析

Spring-AOP 的原理可以划分为三个步骤，总共做了三件事，分别是解析切面、创建动态代理、调用代理方法。

![image-20211208172327744](https://raw.githubusercontent.com/telzhou618/images/main/img01/image-20211208172327744.png)

解析切面：

在Spring容器启动的过程中会扫描所有的切面类，即标注@Aspect的bean,解析该类中的所有通知方法，生成Advisor，每个含有 @Before、@After... 的类都是一个通知，都会生成一个Advisor，最后将所有解析好的Advisor保存在缓存中。



创建代理对象：

根据切点表达式匹配所有bean和方法，如果匹配成功，实现了接口的bean使用JDK动态代理，没有实现接口使用Cglib动态代理。



调用代理方法：

当调用被代理的类的方法是，AOP会在执行目标方法前后执行被植入的切面逻辑。



## 解析切面

从入口 @EnableAspectJAutoProxy 开始。

导入了另一个类 AspectJAutoProxyRegistrar，@Import注解可以将一个普通类注入到spring容器，注册成bean。

```java
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {}
```

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

   @Override
   public void registerBeanDefinitions(
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

      // 注入AOP核心类型
      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

     // 获取 EnableAspectJAutoProxy 注解中的 proxyTargetClass 和  exposeProxy 执行。
     // proxyTargetClass = true 表示都使用cglib代理。
     // exposeProxy = true 暴露代理对象到AOP的上下文，可以通过AopContext获取。
      AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
      if (enableAspectJAutoProxy != null) {
         if (enableAspectJAutoProxy.getBoolean(&#34;proxyTargetClass&#34;)) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
         }
         if (enableAspectJAutoProxy.getBoolean(&#34;exposeProxy&#34;)) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
         }
      }
   }
}
```

这里导入的是 ImportBeanDefinitionRegistrar 类型的bean, 通过实现 registerBeanDefinitions 方法可以注入其他任何类型的bean。

关于ImportBeanDefinitionRegistrar 的用法参考另一篇文章：https://juejin.cn/post/7039308644768284679



关键是注册了另一个bean，AnnotationAwareAspectJAutoProxyCreator 。

![image-20211208170900734](https://raw.githubusercontent.com/telzhou618/images/main/img01/image-20211208170900734.png)



AnnotationAwareAspectJAutoProxyCreator 实现了 BeenPostProcessor接口 , 是bean的后置处理器。

BeenPostProcessor 中有两个重要方法如下，他们分别在bean初始化前后执行。

```java
default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  return bean;
}

default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
  return bean;
}
```

- postProcessBeforeInitialization 在bean 初始化前执行。
- postProcessAfterInitialization 在 bean 初始化后执行。



另外还实现了 InstantiationAwareBeanPostProcessor接口，该类也有两个方法，分别在bean实例化前后执行。

```java
default Object postProcessBeforeInstantiation(Class&lt;?&gt; beanClass, String beanName) throws BeansException {
  return null;
}
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
  return true;
}
```

- postProcessBeforeInstantiation 在bean实例化前执行。
- postProcessAfterInstantiation 在bean实例化后执行。

以上四个方法是AOP解析切面、创建代理的关键，在抽象类AbstractAutoProxyCreator中分别实现了这四个方法。

```java
@Override
public Object postProcessBeforeInstantiation(Class&lt;?&gt; beanClass, String beanName) {
   Object cacheKey = getCacheKey(beanClass, beanName);

   if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
      if (this.advisedBeans.containsKey(cacheKey)) {
         return null;
      }
      // 解析切面
      if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
         this.advisedBeans.put(cacheKey, Boolean.FALSE);
         return null;
      }
   }
   TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
   if (targetSource != null) {
      if (StringUtils.hasLength(beanName)) {
         this.targetSourcedBeans.add(beanName);
      }
      Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
      Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   return null;
}

@Override
public boolean postProcessAfterInstantiation(Object bean, String beanName) {
   return true;
}

@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
   return bean;
}

@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (this.earlyProxyReferences.remove(cacheKey) != bean) {
        // 创建代理
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

在 postProcessBeforeInstantiation 方法中实现了切面解析的逻辑，会在第一个bean实例化前执行。

在 postProcessAfterInitialization 方法中实现创建代理，在bean初始化后执行。

### 切面解析入口方法 postProcessBeforeInstantiation

```java
// 切面bean缓存
private final Map&lt;Object, Boolean&gt; advisedBeans = new ConcurrentHashMap&lt;&gt;(256);

@Override
public Object postProcessBeforeInstantiation(Class&lt;?&gt; beanClass, String beanName) {
  Object cacheKey = getCacheKey(beanClass, beanName);

  if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
    // 如果已经解析返回
    if (this.advisedBeans.containsKey(cacheKey)) {
      return null;
    }
		// 解析切面
    if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return null;
    }
  }
  return null;
}
```

postProcessBeforeInstantiation 在bean实例化前执行。

shouldSkip 方法中实现注解方式AOP的解析过程。

解析过的切面会把beanName存入 advisedBeans 缓存中, 不再重复解析。

### 解析切面核心方法 shouldSkip

```java
@Override
protected boolean shouldSkip(Class&lt;?&gt; beanClass, String beanName) {
  // TODO: Consider optimization by caching the list of the aspect names
  List&lt;Advisor&gt; candidateAdvisors = findCandidateAdvisors();
  for (Advisor advisor : candidateAdvisors) {
    if (advisor instanceof AspectJPointcutAdvisor &amp;&amp;
        ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
      return true;
    }
  }
  return super.shouldSkip(beanClass, beanName);
}
```

```java
@Override
protected List&lt;Advisor&gt; findCandidateAdvisors() {
  // Add all the Spring advisors found according to superclass rules.
  List&lt;Advisor&gt; advisors = super.findCandidateAdvisors();
  // Build Advisors for all AspectJ aspects in the bean factory.
  if (this.aspectJAdvisorsBuilder != null) {
    advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
  }
  return advisors;
}
```

shouldSkip 中调用 findCandidateAdvisors() 解析所有候选的切面，切面有两种切面，一种是基于接口的切面，一种是基于注解的切面。

### 解析接口方式的切面

一种是调用 super.findCandidateAdvisors() 解析所有实现Advisor接口的切面bean。

```java
protected List&lt;Advisor&gt; findCandidateAdvisors() {
  return this.advisorRetrievalHelper.findAdvisorBeans();
}

public List&lt;Advisor&gt; findAdvisorBeans() {
  String[] advisorNames = this.cachedAdvisorBeanNames;
		if (advisorNames == null) {
			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the auto-proxy creator apply to them!
			advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
					this.beanFactory, Advisor.class, true, false);
			this.cachedAdvisorBeanNames = advisorNames;
		}
		if (advisorNames.length == 0) {
			return new ArrayList&lt;&gt;();
		}
    // ... 略
}
```

这里的核心操作是解析所有实现 Advisor 接口的bean, 缓存起来。

### 解析注解方式的切面

另一种调用 aspectJAdvisorsBuilder.buildAspectJAdvisors() 解析所有注解方式的切面。

```java
// Advisor缓存，key是aspectNeam
private final Map&lt;String, List&lt;Advisor&gt;&gt; advisorsCache = new ConcurrentHashMap&lt;&gt;();

public List&lt;Advisor&gt; buildAspectJAdvisors() {}
```

buildAspectJAdvisors 的代码很长，这里略过，只说一下核心逻辑，它会解析Spring容器中取出所有的切面，构建成Advisor放入到advisorsCache缓存起来。

现在所有的切面都解析好并且缓存起来了，下面就是创建动态代理。



## 创建动态代理

AOP创建动态代理有两种方式，如果代理目标实现了接口，则采用JDK动态代理，如果没有实现接口，则采用Cglib动态代理。

![image-20211209164425900](https://raw.githubusercontent.com/telzhou618/images/main/img01/image-20211209164425900.png)

### 代理创建入口方法 postProcessAfterInitialization 

前面说过在bean初始化后执行。

```java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
  if (bean != null) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (this.earlyProxyReferences.remove(cacheKey) != bean) {
      // 创建代理
      return wrapIfNecessary(bean, beanName, cacheKey); // ①
    }
  }
  return bean;
}
```

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  if (StringUtils.hasLength(beanName) &amp;&amp; this.targetSourcedBeans.contains(beanName)) {
    return bean;
  }
  if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
    return bean;
  }
  if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
  }

  // 根据当前bean获取候选的Advisor，这里做第一次筛选，初步筛选，只根据beanClass匹配。
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  if (specificInterceptors != DO_NOT_PROXY) {
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    // 创建代理
    Object proxy = createProxy(
      bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }

  this.advisedBeans.put(cacheKey, Boolean.FALSE);
  return bean;
}
```

```java
protected Object createProxy(Class&lt;?&gt; beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
    // 实例化代理工厂并设置创建代理所需的原料
		ProxyFactory proxyFactory = new ProxyFactory(); 
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}
    // 构建切面
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    // 设置切面
		proxyFactory.addAdvisors(advisors); 
   // 设置代理目标对象
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}
```

- 创建代理工厂，为代理工程设置原料，一个是要创建代理的代理目标，表明为哪些类创建代理对象，这里是当前bean对象。
- 另一个是要植入的切面，为代理目标增强的那些功能。

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
   // 创建AOP代理工厂，创建代理对象
   return createAopProxy().getProxy(classLoader);
}
```



```java
protected final synchronized AopProxy createAopProxy() {
   if (!this.active) {
      activate();
   }
   // 创建AOP代理工厂
   return getAopProxyFactory().createAopProxy(this);
}
```



```java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
   if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
      Class&lt;?&gt; targetClass = config.getTargetClass();
      if (targetClass == null) {
         throw new AopConfigException(&#34;TargetSource cannot determine target class: &#34; &#43;
               &#34;Either an interface or a target is required for proxy creation.&#34;);
      }
      // 如果接口或已经是jdk代理，则走JDK动态代理。
      if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
         return new JdkDynamicAopProxy(config);
      }
      // CGLIB 动态代理
      return new ObjenesisCglibAopProxy(config);
   }
   else {
      // JDK 动态代理
      return new JdkDynamicAopProxy(config);
   }
}
```

- 首先根据当前bean获取符合条件的签名通知列表。
- 实例化代理工厂，设置代理目标，设置代理切面通知。
- 在代理工厂中又创建AOP代理工厂，根据代理目标选择Cglib或jdk动态代理。
- 最后调用代理工厂中 getProxy 方法生成代理对象。

###  CglibAopProxy cglib动态代理

Cglib可以为普通类和接口创建代理类，原理是在应用启动是通过asm技术生成新的字节码文件，新的字节码都统一的签注$$。

实例：

代理目标类 Person.java

```java
public class Person {

    public void sayHello() {
        System.out.println(&#34;hello word!&#34;);
    }
}
```

代理工厂类 CglibProxyFactory.java

```java
public class CglibProxyFactory implements MethodInterceptor {

    /**
     * 代理目标
     */
    private final Object target;
    private final Enhancer enhancer;
  
    public CglibProxyFactory(Object target) {
        this.target = target;
        this.enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
    }


    public Object getProxy() {
        return enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println(&#34;业务方法执行前...&#34;);
        Object result = methodProxy.invokeSuper(o, objects);
        System.out.println(&#34;业务方法执行后...&#34;);
        return result;
    }
}
```

测试 &amp; 执行结果

```java
public static void main(String[] args) {
    CglibProxyFactory proxyFactory = new CglibProxyFactory(new Person());
    Person person = (Person) proxyFactory.getProxy();
    person.sayHello();
}
```

```sh
业务方法执行前...
hello word!
业务方法执行后...
```



### JdkDynamicAopProxy JDK动态代理

jdk 动态代理只能为实现接口的类创建代理。

jdk 动态代理底层通过生成字节码实现，生成的代理类都会继承一个模板类Proxy, 生成的类名一般为 $Proxy0这样，这也是什么jdk动态代理不能为普通类创建代理的原因，它已经继承Proxy类了，没发再继承另一个了。

实例：创建代理目标和接口

```java
public interface Animal {
    void test();
}
```

```java
public class Dog implements Animal {
    @Override
    public void test() {
        System.out.println(&#34;dog 会跑!&#34;);
    }
}
```

代理工厂  JdkDynamicProxyFactory.java

```java
public class JdkDynamicProxyFactory implements InvocationHandler {

    /**
     * 代理目标
     */
    private final Object target;

    public JdkDynamicProxyFactory(Object target) {
        this.target = target;
    }

    public Object getProxy() {
        return  Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(&#34;业务方法执行前...&#34;);
        Object result = method.invoke(target, args);
        System.out.println(&#34;业务方法执行后...&#34;);
        return result;
    }
}
```

测试 &amp; 执行结果

```java
public static void main(String[] args) {
  JdkDynamicProxyFactory proxyFactory = new JdkDynamicProxyFactory(new Dog());
  Animal animal = (Animal) proxyFactory.getProxy();
  animal.test();
}
```

```sh
业务方法执行前...
dog 会跑!
业务方法执行后...
```



## 调用代理方法

### JDK 动态代理调用切面方法

jdk动态代理调用是通过实现 InvocationHandler 接口的 invoke 方法调用切面方法。

```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
  @Override
	@Nullable
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
      // 如果是 equals、hashCode 等方法不用调用切面方法，最终调用原方法。
			if (!this.equalsDefined &amp;&amp; AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
  
			else if (!this.hashCodeDefined &amp;&amp; AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
    
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -&gt; dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
  
			else if (!this.advised.opaque &amp;&amp; method.getDeclaringClass().isInterface() &amp;&amp;
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// Get as late as possible to minimize the time we &#34;own&#34; the target,
			// in case it comes from a pool.
			target = targetSource.getTarget();
			Class&lt;?&gt; targetClass = (target != null ? target.getClass() : null);

			// 获取方法的连拦截链，这里会根据方法名最进一步的筛选
			List&lt;Object&gt; chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			if (chain.isEmpty()) {
			  // 如果拦截链为空，调用原方法
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// 构建调用链，使用责任链模式，执行切面方法。
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class&lt;?&gt; returnType = method.getReturnType();
			if (retVal != null &amp;&amp; retVal == target &amp;&amp;
					returnType != Object.class &amp;&amp; returnType.isInstance(proxy) &amp;&amp;
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned &#34;this&#34; and the return type of the method
				// is type-compatible. Note that we can&#39;t help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null &amp;&amp; returnType != Void.TYPE &amp;&amp; returnType.isPrimitive()) {
				throw new AopInvocationException(
						&#34;Null return value from advice does not match primitive return type for: &#34; &#43; method);
			}
			return retVal;
		}
		finally {
			if (target != null &amp;&amp; !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
}
```

### Cglib 动态代理调用切面方法

Cglib 通过实现 MethodInterceptor 接口的 intercept 方法调用切面方法。

```java
public interface MethodInterceptor extends Callback {
    Object intercept(Object var1, Method var2, Object[] var3, MethodProxy var4) throws Throwable;
}
```

```java
private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {
  
  	@Override
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// Get as late as possible to minimize the time we &#34;own&#34; the target, in case it comes from a pool...
				target = targetSource.getTarget();
				Class&lt;?&gt; targetClass = (target != null ? target.getClass() : null);
        // 获取拦截链
				List&lt;Object&gt; chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// Check whether we only have one InvokerInterceptor: that is,
				// no real advice, but just reflective invocation of the target.
				if (chain.isEmpty() &amp;&amp; Modifier.isPublic(method.getModifiers())) {
					// We can skip creating a MethodInvocation: just invoke the target directly.
					// Note that the final invoker must be an InvokerInterceptor, so we know
					// it does nothing but a reflective operation on the target, and no hot
					// swapping or fancy proxying.
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// 责任连调用，切面方法。
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null &amp;&amp; !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}

}
```

cglib的调用类 CglibMethodInvocation 继承 ReflectiveMethodInvocation。

```java
private static class CglibMethodInvocation extends ReflectiveMethodInvocation {

		private final MethodProxy methodProxy;

		private final boolean publicMethod;

		public CglibMethodInvocation(Object proxy, @Nullable Object target, Method method,
				Object[] arguments, @Nullable Class&lt;?&gt; targetClass,
				List&lt;Object&gt; interceptorsAndDynamicMethodMatchers, MethodProxy methodProxy) {

			super(proxy, target, method, arguments, targetClass, interceptorsAndDynamicMethodMatchers);
			this.methodProxy = methodProxy;
			this.publicMethod = Modifier.isPublic(method.getModifiers());
		}
	}
```

- jdk动态代理调用是通过实现 InvocationHandler 接口的 invoke 方法调用切面方法。
- Cglib 通过实现 MethodInterceptor 接口的 intercept 方法调用切面方法。
- ReflectiveMethodInvocation 通责任连模式对象切面的调用。



## 附：AOP启动流程图



![](https://raw.githubusercontent.com/telzhou618/images/main/img01/Spring%20AOP%20%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B.png)



---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/spring-aop/  

