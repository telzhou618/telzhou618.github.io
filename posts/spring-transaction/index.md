# Spring 事务源码解析

Spring 事务利用AOP原理实现,主要过程和AOP原理一样，可以分为三步，启用事务、生成代理对对象、执行。

## 从如何使用事务开始

本实例使用Spring的 JdbcTemplate 作为操作数据库的工具，使用 druid 做为数据库连接池。

- 首先导入相关的 jar 包

```xml

&lt;dependency&gt;
    &lt;groupId&gt;mysql&lt;/groupId&gt;
    &lt;artifactId&gt;mysql-connector-java&lt;/artifactId&gt;
    &lt;version&gt;8.0.20&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
&lt;groupId&gt;com.alibaba&lt;/groupId&gt;
&lt;artifactId&gt;druid&lt;/artifactId&gt;
&lt;version&gt;1.2.4&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
&lt;groupId&gt;org.springframework&lt;/groupId&gt;
&lt;artifactId&gt;spring-jdbc&lt;/artifactId&gt;
&lt;version&gt;5.2.12.RELEASE&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
&lt;groupId&gt;org.springframework&lt;/groupId&gt;
&lt;artifactId&gt;spring-tx&lt;/artifactId&gt;
&lt;version&gt;5.2.12.RELEASE&lt;/version&gt;
&lt;/dependency&gt;
```

- 在配置类上加上 @EnableTransactionManagement 注解表示开启Spring事务功能

```java
@ComponentScan(&#34;com.example&#34;)
@EnableTransactionManagement
public class ApplicationMain {}
```

- 配置数据源、配置JdbcTemplate、配置事务管理器

```java
@Configuration
public class JdbcConfig {

    @Bean(initMethod = &#34;init&#34;)
    public DruidDataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(&#34;jdbc:mysql://localhost/test?useUnicode=true&amp;characterEncoding=utf-8&#34;);
        dataSource.setUsername(&#34;root&#34;);
        dataSource.setPassword(&#34;rootroot&#34;);
        return dataSource;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    @Bean
    public TransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

- 创建测试表 blog、创建实体对象Blog

```sql
create table blog
(
    id      int auto_increment comment &#39;id&#39; primary key,
    title   varchar(20) null comment &#39;标题&#39;,
    content varchar(100) null comment &#39;内容&#39;
) comment &#39;blog表&#39;;
```

- 创建实体对象

```java
@Getter
@Setter
@ToString
@Accessors(chain = true)
public class Blog {

    private Integer id;
    private String title;
    private String content;
}
```

- 创建 BlogService和一个 insert 方法插入一条记录，在方法上加上 @Transactional 注解，在执行插入数据成功后故意写个异常情况1/0，测试事务是否回滚。

```java
@Service
@AllArgsConstructor
public class BlogService {

    private final JdbcTemplate jdbcTemplate;

    @Transactional(rollbackFor = Exception.class)
    public int insert(Blog blog) {
        String sql = &#34;insert into blog(title,content) value(?,?)&#34;;
        Object[] args = new Object[]{blog.getTitle(), blog.getContent()};
        int ret = jdbcTemplate.update(sql, args);
        System.out.println(1 / 0);
        return ret;
    }

}
```

- 执行测试方法,看运行结果

```java
@ComponentScan(&#34;com.example&#34;)
@EnableTransactionManagement
public class ApplicationMain {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(ApplicationMain.class);
        BlogService blogService = context.getBean(BlogService.class);
        Blog blog = new Blog().setTitle(&#34;xxx222&#34;).setContent(&#34;xxx2222&#34;);
        System.out.println(blogService.insert(blog));
    }
}
```

- 查询执行结果

![image.png](/_static/aop/img1.png)

&gt; 从执行结果看，后面的代码抛异常后，事务回滚了，那个仅凭一个简单的注解事务是如何开启、提交已经回滚的呢，下面我们从底层源码一探究竟。

## 从事务的入口 开始

@EnableTransactionManagement

点开 @EnableTransactionManagement 注解，我们发现他导入了另一类@Import(TransactionManagementConfigurationSelector.class)，@Import
的作用是它可以导入一个普通类到Spring容器中，使其注册为一个bean。

![image.png](/_static/aop/img2.png)

TransactionManagementConfigurationSelector 类它有实现了 ImportSelector 接口，重写了 selectImports
方法，该方法会在Spring扫描的时候执行，通过返回的类名数组生成bean，最后发现添加EnableTransactionManagement注解的最终目的是注入 AutoProxyRegistrar 和
ProxyTransactionManagementConfiguration 这两个类到Spring容器。

**AutoProxyRegistrar**

```java
AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
```

```java
@Nullable
	public static BeanDefinition registerAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
	}
```

```java
public class InfrastructureAdvisorAutoProxyCreator extends AbstractAdvisorAutoProxyCreator {

	@Nullable
	private ConfigurableListableBeanFactory beanFactory;


	@Override
	protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		super.initBeanFactory(beanFactory);
		this.beanFactory = beanFactory;
	}

	@Override
	protected boolean isEligibleAdvisorBean(String beanName) {
		return (this.beanFactory != null &amp;&amp; this.beanFactory.containsBeanDefinition(beanName) &amp;&amp;
				this.beanFactory.getBeanDefinition(beanName).getRole() == BeanDefinition.ROLE_INFRASTRUCTURE);
	}

}
```

AutoProxyRegistrar 实现了 ImportBeanDefinitionRegistrar，重写了 registerBeanDefinitions
方法，该方法在Spring容器启动的时候回执行，可以注册bean定义，在这里核心是注册了另一个 bean
InfrastructureAdvisorAutoProxyCreator,InfrastructureAdvisorAutoProxyCreator方法中几乎没有什么核心代码，它的全部功能来源于 AOP 解析类
AbstractAdvisorAutoProxyCreator。

**ProxyTransactionManagementConfiguration**

```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		advisor.setAdvice(transactionInterceptor());
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.&lt;Integer&gt;getNumber(&#34;order&#34;));
		}
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}
}
```

ProxyTransactionManagementConfiguration 类是一个配置类，他注册了三个bean。

- BeanFactoryTransactionAttributeSourceAdvisor 它是一个Advisor，实现了Advisor 接口，它是是AOP的其中一种方式和 @Aspect 注解效果是一样的，都会被AOP解析识别。
- AnnotationTransactionAttributeSource 是事务属性源，他主要负责判断一个bean及其方法是不是有@Transactional注解，如果有就要生成动态代理。
- TransactionInterceptor 该类是代理类具体实现逻辑，生成代理代理对象时会用到，在具体执行事务时会调用其中的 invock 方法。

另外 AnnotationTransactionAttributeSource 和 TransactionInterceptor 都是 BeanFactoryTransactionAttributeSourceAdvisor
的属性，具体的调用在BeanFactoryTransactionAttributeSourceAdvisor 完成。

以上就是启用事务时所作的事情。

## 何时生成代理对象

什么时候生成代理对象 ？

为那些bean生成代理对象？

**InfrastructureAdvisorAutoProxyCreator**

还记得它吗，该对象时具体扫描 @Transactional 标注的类和方法，再生成代理对象的，但是它的核心功能都是继承自AOP的对象AbstractAdvisorAutoProxyCreator
的，所以说事务的实现在于创建了符合AOP规则的 Advisor(BeanFactoryTransactionAttributeSourceAdvisor) ，然后交给AOP去生成代理对象,

AOP 在哪里找到 BeanFactoryTransactionAttributeSourceAdvisor ?

![](https://raw.gitmirror.com/telzhou618/images/main/img02/img3.png)
![](https://raw.gitmirror.com/telzhou618/images/main/img02/img4.png)
![](https://raw.gitmirror.com/telzhou618/images/main/img02/img5.png)

## 何时调用事务方法

未完

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/spring-transaction/  

