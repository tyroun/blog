{% raw %}

# 廖雪峰 Java 教程

## 1 Web 开发基础

### 1.1 Servlet

Servlet 是 JavaEE 定义的一套接口规范

JavaEE 提供了 Servlet API，我们使用 Servlet API 编写自己的 Servlet 来处理 HTTP 请求，Web 服务器实现 Servlet API 接口，实现底层功能：

```scheme
                 ┌───────────┐
                 │My Servlet │
                 ├───────────┤
                 │Servlet API│
┌───────┐  HTTP  ├───────────┤
│Browser│<──────>│Web Server │
└───────┘        └───────────┘
```

通过 Maven 打包编译出 war(Java Web Application Archive)文件

运行时，先运行 Tomcat, Tomcat 中的 main 函数运行，最后调用到自定义的 Servlet 接口

一点简单的 Servlet 例子

```java
// WebServlet注解表示这是一个Servlet，并映射到地址/:
@WebServlet(urlPatterns = "/")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        // 设置响应类型:
        resp.setContentType("text/html");
        // 获取输出流:
        PrintWriter pw = resp.getWriter();
        // 写入响应:
        pw.write("<h1>Hello, world!</h1>");
        // 最后不要忘记flush强制输出:
        pw.flush();
    }
}
```

### 1.2 MVC 开发

#### 1 先定义一些 JavaBean 的类

JavaBeam 是一种实体类，类中只有私有成员变量，和外部读写的函数

#### 2 定义 controller 类

该类的接口只需要返回 Model 和 View 给 MVC 框架即可

```java
public class UserController {
    @GetMapping("/signin")
    public ModelAndView signin() {
        ...
    }

    @PostMapping("/signin")
    public ModelAndView doSignin(SignInBean bean) {
        ...
    }

    @GetMapping("/signout")
    public ModelAndView signout(HttpSession session) {
        ...
    }
}
```

#### 3 设计 MVC 框架

框架架构如下

```scheme
   HTTP Request    ┌─────────────────┐
──────────────────>│DispatcherServlet│
                   └─────────────────┘
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
         ┌───────────┐┌───────────┐┌───────────┐
         │Controller1││Controller2││Controller3│
         └───────────┘└───────────┘└───────────┘
               │            │            │
               └────────────┼────────────┘
                            ▼
   HTTP Response ┌────────────────────┐
<────────────────│render(ModelAndView)│
                 └────────────────────┘
```

DispatcherServlet 用于接收 Servlet 并分发给不同的 controller

render 用于渲染，需要调用模板引擎 ViewEngine。常用的有以下几种

- [Thymeleaf](https://www.thymeleaf.org/)
- [FreeMarker](https://freemarker.apache.org/)
- [Velocity](https://velocity.apache.org/)

## 2 Spring 开发

### 2.1 IoC 容器

Tomcat 就是一个 Servlet 容器，它可以为 Servlet 的运行提供运行环境。类似 Docker 这样的软件也是一个容器，它提供了必要的 Linux 环境以便运行一个特定的 Linux 进程

Spring 的核心就是提供了一个 IoC 容器，它可以管理所有轻量级的 JavaBean 组件，提供的底层服务包括组件的生命周期管理、配置和组装服务、AOP 支持，以及建立在 AOP 基础上的声明式事务服务等

#### 2.1.1 IoC 原理

IoC 全称 Inversion of Control，直译为控制反转，就是**依赖注入**

把如下的类改成依赖注入方式，由 IoC 容器决定内部实例的生命周期

```java
public class BookService {
    private DataSource dataSource = new HikariDataSource(config);
    public Book getBook(long bookId) {
			...
    }
}
//改成如下依赖注入的形式
public class BookService {
    private DataSource dataSource; //dataSource由外部输入
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

如何告诉 IoC 容器怎么创建组件和依赖关系，最简单的方法是通过 xml 配置

```xml
<beans>
    <bean id="dataSource" class="HikariDataSource" />
    <bean id="bookService" class="BookService">
        <property name="dataSource" ref="dataSource" />
    </bean>
    <bean id="userService" class="UserService">
        <property name="dataSource" ref="dataSource" />
    </bean>
</beans>
```

在 Spring 的 IoC 容器中，我们把所有组件统称为 JavaBean，即配置一个组件就是配置一个 Bean。

依赖注入可以通过设置属性注入，也可以直接在构造函数时初始化注入

#### 2.1.2 使用 xml 注入 Bean

```java
public class UserService {
    private MailService mailService;

    public void setMailService(MailService mailService) {
        this.mailService = mailService;
    }
}
```

注入配置在 resource/application.xml

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
...>

    <bean id="userService" class="com.itranswarp.learnjava.service.UserService">
        <property name="mailService" ref="mailService" />
    </bean>

    <bean id="mailService" class="com.itranswarp.learnjava.service.MailService" />
</beans>
```

如果注入的不是 Bean，而是`boolean`、`int`、`String`这样的数据类型，则通过`value`注入

```xml
<bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
    <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/test" />
    <property name="username" value="root" />
    <property name="password" value="password" />
    <property name="maximumPoolSize" value="10" />
    <property name="autoCommit" value="true" />
</bean>
```

最后一步，我们需要创建一个 Spring 的 IoC 容器实例，然后加载配置文件，让 Spring 容器为我们创建并装配好配置文件中指定的所有 Bean，这只需要一行代码：

```java
ApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
```

接下来，我们就可以从 Spring 容器中“取出”装配好的 Bean 然后使用它：

```java
// 获取Bean:
UserService userService = context.getBean(UserService.class);
// 正常调用:
User user = userService.login("bob@example.com", "password");
```

#### 2.1.3 使用注解注入

给`MailService`添加一个`@Component`注解：

```java
@Component
public class MailService {
    ...
}
```

给`UserService`添加一个`@Component`注解和一个`@Autowired`注解：

```java
@Component
public class UserService {
    @Autowired		//使用@Autowired就相当于把指定类型的Bean注入到指定的字段中
    MailService mailService;

    //也可以加在构造函数上
    public UserService(@Autowired MailService mailService) {
        this.mailService = mailService;
    }
}
```

编写一个`AppConfig`类启动容器：

```java
@Configuration	//表示它是一个配置类
@ComponentScan //告诉容器，自动搜索当前类所在的包以及子包，把所有标注为@Component的Bean自动创建出来，并根据@Autowired进行装配
public class AppConfig {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        UserService userService = context.getBean(UserService.class);
        User user = userService.login("bob@example.com", "password");
        System.out.println(user.getName());
    }
}
```

#### 2.1.4 定制 Bean 注入

##### Scope

对于 Spring 容器来说，当我们把一个 Bean 标记为`@Component`后，它就会自动为我们创建一个单例（Singleton）。如果想每次都返回一个新的实例，用 Prototype

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) // @Scope("prototype")
public class MailSession {
    ...
}
```

##### 注入 List

```java
//接口
public interface Validator {
    void validate(String email, String password, String name);
}
//实现
@Component
@Order(1)  //次序
public class EmailValidator implements Validator {
    public void validate(String email, String password, String name) {
        if (!email.matches("^[a-z0-9]+\\@[a-z0-9]+\\.[a-z]{2,10}$")) {
            throw new IllegalArgumentException("invalid email: " + email);
        }
    }
}

@Component
@Order(2)  //次序
public class PasswordValidator implements Validator {
    public void validate(String email, String password, String name) {
        if (!password.matches("^.{6,20}$")) {
            throw new IllegalArgumentException("invalid password");
        }
    }
}

@Component
public class Validators {
    @Autowired
    List<Validator> validators; //注入的位置

    public void validate(String email, String password, String name) {
        for (var validator : this.validators) {
            validator.validate(email, password, name);
        }
    }
}
```

##### 可选注入

默认情况下，当我们标记了一个`@Autowired`后，Spring 如果没有找到对应类型的 Bean，它会抛出`NoSuchBeanDefinitionException`异常。

可以给`@Autowired`增加一个`required = false`的参数：

```java
@Component
public class MailService {
    @Autowired(required = false)
    ZoneId zoneId = ZoneId.systemDefault();
    ...
}
```

##### 创建第三方 Bean

如果一个 Bean 不在我们自己的 package 管理之内, 在`@Configuration`类中编写一个 Java 方法创建并返回它，注意给方法标记一个`@Bean`注解：

```java
@Configuration
@ComponentScan
public class AppConfig {
    // 创建一个Bean:
    @Bean
    ZoneId createZoneId() {
        return ZoneId.of("Z");
    }
}
```

##### 初始化和销毁

有些时候，一个 Bean 在注入必要的依赖后，需要进行初始化（监听消息等）。在容器关闭时，有时候还需要清理资源（关闭连接池等）。我们通常会定义一个`init()`方法进行初始化，定义一个`shutdown()`方法进行清理

引入 JSR-250 定义的 Annotation：

```xml
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>
```

在 Bean 的初始化和清理方法上标记`@PostConstruct`和`@PreDestroy`：

```java
@Component
public class MailService {
    @Autowired(required = false)
    ZoneId zoneId = ZoneId.systemDefault();

    @PostConstruct
    public void init() {
        System.out.println("Init mail service with zoneId = " + this.zoneId);
    }

    @PreDestroy
    public void shutdown() {
        System.out.println("Shutdown mail service");
    }
}
```

##### 使用别名

一种类型的 Bean 创建多个实例

```java
@Configuration
@ComponentScan
public class AppConfig {
    @Bean("z") //指定别名
    @Primary // 指定为主要Bean 在注入时，如果没有指出Bean的名字，Spring会注入标记有@Primary的Bean
    ZoneId createZoneOfZ() {
        return ZoneId.of("Z");
    }

    @Bean
    @Qualifier("utc8")
    ZoneId createZoneOfUTC8() {
        return ZoneId.of("UTC+08:00");
    }
}
```

##### 使用 FactoryBean

用工厂模式创建 Bean 需要实现`FactoryBean`接口。我们观察下面的代码：

```java
@Component
public class ZoneIdFactoryBean implements FactoryBean<ZoneId> {

    String zone = "Z";

    @Override
    public ZoneId getObject() throws Exception {
        return ZoneId.of(zone);
    }

    @Override
    public Class<?> getObjectType() {
        return ZoneId.class;
    }
}
```

当一个 Bean 实现了`FactoryBean`接口后，Spring 会先实例化这个工厂，然后调用`getObject()`创建真正的 Bean。`getObjectType()`可以指定创建的 Bean
的类型，因为指定类型不一定与实际类型一致，可以是接口或抽象类

#### 2.1.5 使用 Resource

使用 Spring 容器时，我们可以把“文件”注入进来，方便程序读取

Spring 提供了一个`org.springframework.core.io.Resource`（注意不是`javax.annotation.Resource`），它可以像`String`、`int`一样使用`@Value`注入：

```java
@Component
public class AppService {
    @Value("classpath:/logo.txt") //classpath就是在/src/main/resource里面
    private Resource resource;

    private String logo;

    @PostConstruct
    public void init() throws IOException {
        try (var reader = new BufferedReader(
                new InputStreamReader(resource.getInputStream(), StandardCharsets.UTF_8))) {
            this.logo = reader.lines().collect(Collectors.joining("\n"));
        }
    }
}
```

#### 2.1.6 配置文件注入

在开发应用程序时，经常需要读取配置文件。最常用的配置方法是以`key=value`的形式写在`.properties`文件中。

Spring 容器还提供了一个更简单的`@PropertySource`来自动读取配置文件。我们只需要在`@Configuration`配置类上再添加一个注解：

```java
@Configuration
@ComponentScan
@PropertySource("app.properties") // 表示读取classpath的app.properties
public class AppConfig {
    @Value("${app.zone:Z}")
    String zoneId;

    @Bean
    ZoneId createZoneId() {
        return ZoneId.of(zoneId);
    }
}
```

Spring 容器看到`@PropertySource("app.properties")`注解后，自动读取这个配置文件，然后，我们使用`@Value`正常注入

注意注入的字符串语法，它的格式如下：

- `"${app.zone}"`表示读取 key 为`app.zone`的 value，如果 key 不存在，启动将报错；
- `"${app.zone:Z}"`表示读取 key 为`app.zone`的 value，但如果 key 不存在，就使用默认值`Z`。

这样一来，我们就可以根据`app.zone`的配置来创建`ZoneId`。

还可以把注入的注解写到方法参数中：

```java
@Bean
ZoneId createZoneId(@Value("${app.zone:Z}") String zoneId) {
    return ZoneId.of(zoneId);
}
```

另一种注入配置的方式是先通过一个简单的 JavaBean 持有所有的配置，例如，一个`SmtpConfig`：

```java
@Component
public class SmtpConfig {
    @Value("${smtp.host}")
    private String host;

    @Value("${smtp.port:25}")
    private int port;

    public String getHost() {
        return host;
    }

    public int getPort() {
        return port;
    }
}
```

然后，在需要读取的地方，使用`#{smtpConfig.host}`注入：

```java
@Component
public class MailService {
    @Value("#{smtpConfig.host}")
    private String smtpHost;

    @Value("#{smtpConfig.port}")
    private int smtpPort;
}
```

#### 2.1.7 条件注入

##### 使用 profile

创建某个 Bean 时，Spring 容器可以根据注解`@Profile`来决定是否创建。例如，以下配置：

```java
@Configuration
@ComponentScan
public class AppConfig {
    @Bean
    @Profile("!test")
    ZoneId createZoneId() {
        return ZoneId.systemDefault();
    }

    @Bean
    @Profile("test")
    ZoneId createZoneIdForTest() {
        return ZoneId.of("America/New_York");
    }
}
```

在运行程序时，加上 JVM 参数`-Dspring.profiles.active=test`就可以指定以`test`环境启动。

实际上，Spring 允许指定多个 Profile，例如：

```shell
-Dspring.profiles.active=test,master
```

可以表示`test`环境，并使用`master`分支代码。

要满足多个 Profile 条件，可以这样写：

```java
@Bean
@Profile({ "test", "master" }) // 同时满足test和master
ZoneId createZoneId() {
    ...
}
```

##### 使用 Conditional

除了根据`@Profile`条件来决定是否创建某个 Bean 外，Spring 还可以根据`@Conditional`决定是否创建某个 Bean。

例如，我们对`SmtpMailService`添加如下注解：

```java
@Component
@Conditional(OnSmtpEnvCondition.class)
public class SmtpMailService implements MailService {
    ...
}
```

它的意思是，如果满足`OnSmtpEnvCondition`的条件，才会创建`SmtpMailService`这个 Bean。`OnSmtpEnvCondition`的条件是什么呢？我们看一下代码：

```java
public class OnSmtpEnvCondition implements Condition {
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return "true".equalsIgnoreCase(System.getenv("smtp"));
    }
}
```

### 2.2 使用 AOP

AOP 是 Aspect Oriented Programming，即面向切面编程

如果有一些业务逻辑，不同的类都需要调用。就可以看出一个 Aspect。框架可以通过 proxy 的模式把这些逻辑加入类中

在 Java 平台上，对于 AOP 的织入，有 3 种方式：

1. 编译期：在编译时，由编译器把切面调用编译进字节码，这种方式需要定义新的关键字并扩展编译器，AspectJ 就扩展了 Java 编译器，使用关键字 aspect 来实现织入；
2. 类加载器：在目标类被装载到 JVM 时，通过一个特殊的类加载器，对目标类的字节码重新“增强”；
3. 运行期：目标对象和切面都是普通 Java 类，通过 JVM 的动态代理功能或者第三方库实现运行期动态织入。

Spring 的 AOP 实现就是基于 JVM 的动态代理

#### 2.2.1 使用 AOP

1. 通过 Maven 引入 Spring 对 AOP 的支持

   ```xml
   <dependency>
       <groupId>org.springframework</groupId>
       <artifactId>spring-aspects</artifactId>
       <version>${spring.version}</version>
   </dependency>
   ```

2. 定义一个`LoggingAspect`

   ```java
   @Aspect
   @Component
   public class LoggingAspect {
       // 在执行UserService的每个方法前执行:
       @Before("execution(public * xxx.UserService.*(..))")
       public void doAccessCheck() {
           System.err.println("[Before] do access check...");
       }

       // 在执行MailService的每个方法前后执行:
       @Around("execution(public * xxx.MailService.*(..))")
       public Object doLogging(ProceedingJoinPoint pjp) throws Throwable {
           System.err.println("[Around] start " + pjp.getSignature());
           Object retVal = pjp.proceed();
           System.err.println("[Around] done " + pjp.getSignature());
           return retVal;
       }
   }
   ```

3. 我们需要给`@Configuration`类加上一个`@EnableAspectJAutoProxy`注解：

   ```java
   @Configuration
   @ComponentScan
   @EnableAspectJAutoProxy
   //自动查找带有@Aspect的Bean，然后根据每个方法的@Before、@Around等注解把AOP注入到特定的Bean中
   public class AppConfig {
       ...
   }
   ```

##### 拦截器类型

顾名思义，拦截器有以下类型：

- @Before：这种拦截器先执行拦截代码，再执行目标代码。如果拦截器抛异常，那么目标代码就不执行了；
- @After：这种拦截器先执行目标代码，再执行拦截器代码。无论目标代码是否抛异常，拦截器代码都会执行；
- @AfterReturning：和@After 不同的是，只有当目标代码正常返回时，才执行拦截器代码；
- @AfterThrowing：和@After 不同的是，只有当目标代码抛出了异常时，才执行拦截器代码；
- @Around：能完全控制目标代码是否执行，并可以在执行前后、抛异常后执行任意拦截代码，可以说是包含了上面所有功能。

#### 2.2.2 使用注解装配 AOP

用 AOP 时，被装配的 Bean 最好自己能清清楚楚地知道自己被安排了

1. 定义一个性能监控的注解

   ```java
   @Target(METHOD)
   @Retention(RUNTIME)
   public @interface MetricTime {
       String value();
   }
   ```

2. 在需要被监控的关键方法上标注该注解

   ```java
   @Component
   public class UserService {
       // 监控register()方法性能:
       @MetricTime("register")
       public User register(String email, String password, String name) {
           ...
       }
       ...
   }
   ```

3. 定义`MetricAspect`

   ```java
   @Aspect
   @Component
   public class MetricAspect {
       @Around("@annotation(metricTime)")
       //符合条件的目标方法是带有@MetricTime注解的方法
       public Object metric(ProceedingJoinPoint joinPoint, MetricTime metricTime) throws Throwable {
           String name = metricTime.value();
           long start = System.currentTimeMillis();
           try {
               return joinPoint.proceed();
           } finally {
               long t = System.currentTimeMillis() - start;
               // 写入日志或发送至JMX:
               System.err.println("[Metrics] " + name + ": " + t + "ms");
           }
       }
   }
   ```

### 2.3 访问数据库

#### 2.3.1 使用 JDBC

Java 程序使用 JDBC 接口访问关系数据库的时候，需要以下几步：

- 创建全局`DataSource`实例，表示数据库连接池；
- 在需要读写数据库的方法内部，按如下步骤访问数据库：
  - 从全局`DataSource`实例获取`Connection`实例；
  - 通过`Connection`实例创建`PreparedStatement`实例；
  - 执行 SQL 语句，如果是查询，则通过`ResultSet`读取结果集，如果是修改，则获得`int`结果。

正确编写 JDBC 代码的关键是使用`try ... finally`释放资源，涉及到事务的代码需要正确提交或回滚事务。

**_使用步骤_**

1. 创建并管理 DataSource
2. 实例化 JdbcTemplate

```java
@Configuration
@ComponentScan
//通过@PropertySource("jdbc.properties")读取数据库配置文件
@PropertySource("jdbc.properties")
public class AppConfig {

    //通过@Value("${jdbc.url}")注入配置文件的相关配置；
    @Value("${jdbc.url}")
    String jdbcUrl;

    @Value("${jdbc.username}")
    String jdbcUsername;

    @Value("${jdbc.password}")
    String jdbcPassword;

    //创建一个DataSource实例，它的实际类型是HikariDataSource，
    //创建时需要用到注入的配置
    @Bean
    DataSource createDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(jdbcUsername);
        config.setPassword(jdbcPassword);
        config.addDataSourceProperty("autoCommit", "true");
        config.addDataSourceProperty("connectionTimeout", "5");
        config.addDataSourceProperty("idleTimeout", "60");
        return new HikariDataSource(config);
    }

    //创建一个JdbcTemplate实例，它需要注入DataSource，这是通过方法参数完成注入的
    @Bean
    JdbcTemplate createJdbcTemplate(@Autowired DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

3. 初始化表

```java
@Component
public class DatabaseInitializer {
    @Autowired
    JdbcTemplate jdbcTemplate;

    @PostConstruct
    public void init() {
        //在jdbcTemplate里加入SQL
        jdbcTemplate.update("CREATE TABLE IF NOT EXISTS users (" //
                + "id BIGINT IDENTITY NOT NULL PRIMARY KEY, " //
                + "email VARCHAR(100) NOT NULL, " //
                + "password VARCHAR(100) NOT NULL, " //
                + "name VARCHAR(100) NOT NULL, " //
                + "UNIQUE (email))");
    }
}
```

4. JBDCTemplate 在 Service 中使用

`用法1` T execute(ConnectionCallback<T> action)方法

提供了 Jdbc 的 Connection 供我们使用

```java
@Component
public class UserService {
    @Autowired
    JdbcTemplate jdbcTemplate;

    public User getUserById(long id) {
        // 注意传入的是ConnectionCallback:
        return jdbcTemplate.execute((Connection conn) -> {
            // 可以直接使用conn实例，不要释放它，回调结束后JdbcTemplate自动释放:
            // 在内部手动创建的PreparedStatement、ResultSet必须用try(...)释放:
            try (var ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
                ps.setObject(1, id);
                try (var rs = ps.executeQuery()) {
                    if (rs.next()) {
                        return new User( // new User object:
                                rs.getLong("id"), // id
                                rs.getString("email"), // email
                                rs.getString("password"), // password
                                rs.getString("name")); // name
                    }
                    throw new RuntimeException("user not found by id.");
                }
            }
        });
    }

}
```

`用法2` T execute(String sql, PreparedStatementCallback<T> action)

```java
public User getUserByName(String name) {
    // 需要传入SQL语句，以及PreparedStatementCallback:
    return jdbcTemplate.execute("SELECT * FROM users WHERE name = ?", (PreparedStatement ps) -> {
        // PreparedStatement实例已经由JdbcTemplate创建，并在回调后自动释放:
        ps.setObject(1, name);
        try (var rs = ps.executeQuery()) {
            if (rs.next()) {
                return new User( // new User object:
                        rs.getLong("id"), // id
                        rs.getString("email"), // email
                        rs.getString("password"), // password
                        rs.getString("name")); // name
            }
            throw new RuntimeException("user not found by id.");
        }
    });
}
```

`用法3`

T queryForObject(String sql, Object[] args, RowMapper<T> rowMapper)方法

```java
public User getUserByEmail(String email) {
    // 传入SQL，参数和RowMapper实例:
    return jdbcTemplate.queryForObject("SELECT * FROM users WHERE email = ?", new Object[] { email },
            (ResultSet rs, int rowNum) -> {
                // 将ResultSet的当前行映射为一个JavaBean:
                return new User( // new User object:
                        rs.getLong("id"), // id
                        rs.getString("email"), // email
                        rs.getString("password"), // password
                        rs.getString("name")); // name
            });
}
```

rowMapper 是个匿名函数，queryForObject()会自动生成 ps，并调用 ps.excute 执行 sql 语句，执行完以后返回 rs 和 rowNum 作为输入，调用 rowMapper 这个匿名函数

Spring 也提供了 BeanPropertyRowMapper 这样的函数，可以直接按列名把 record 转成 JavaBean

#### 2.3.2 使用声明式事务

手写事务代码，使用`try...catch`如下

```java
TransactionStatus tx = null;
try {
    // 开启事务:
    tx = txManager.getTransaction(new DefaultTransactionDefinition());
    // 相关JDBC操作:
    jdbcTemplate.update("...");
    jdbcTemplate.update("...");
    // 提交事务:
    txManager.commit(tx);
} catch (RuntimeException e) {
    // 回滚事务:
    txManager.rollback(tx);
    throw e;
}
```

##### 使用方法

如果要在 Spring 中操作事务，没必要手写 JDBC 事务，可以使用 Spring 提供的高级接口来操作事务。Spring 提供了一个`PlatformTransactionManager`
来表示事务管理器，所有的事务都由它负责管理。而事务由`TransactionStatus`表示。

Spring 同时支持 JDBC 和分布式事务 JTA(Java Transaction API)两种事务模型

可以在 Config 里面定义 Bean

```java
@Configuration
@ComponentScan
@PropertySource("jdbc.properties")
public class AppConfig {
    ...
    @Bean
    PlatformTransactionManager createTxManager(@Autowired DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

也可以用注解

```java
@Configuration
@ComponentScan
@EnableTransactionManagement // 启用声明式
@PropertySource("jdbc.properties")
public class AppConfig {
    ...
}
```

对需要事务支持的方法，加一个`@Transactional`注解

```java
@Component
public class UserService {
    // 此public方法自动具有事务支持:
    @Transactional
    public User register(String email, String password, String name) {
       ...
    }
}

//或者直接在Bean的class处加上，表示所有public方法都具有事务支持
@Component
@Transactional
public class UserService {
    ...
}


```

##### 回滚事务

在一个事务方法中，如果程序判断需要回滚事务，只需抛出`RuntimeException`

```java
@Transactional(rollbackFor = {RuntimeException.class, IOException.class})
public buyProducts(long productId, int num) {
    ...
    if (store < num) {
        // 库存不够，购买失败:
        throw new IllegalArgumentException("No enough products");
    }
    ...
}
```

##### 事务边界和事务传播

简单情况，以函数定义为边界

```java
@Component
public class UserService {
    @Transactional
    public User register(String email, String password, String name) { // 事务开始
       ...
    } // 事务结束
}

@Component
public class BonusService {
    @Transactional
    public void addBonus(long userId, int bonus) { // 事务开始
       ...
    } // 事务结束
}
```

如果事务嵌套调用的情况

```java
@Component
public class UserService {
    @Autowired
    BonusService bonusService;

    @Transactional
    public User register(String email, String password, String name) {
        // 插入用户记录:
        User user = jdbcTemplate.insert("...");
        // 增加100积分:
        bonusService.addBonus(user.id, 100); //里面还有一个事务
    }
}
```

以上情况有好几种策略，成为**_事务传播级别_**

默认是`REQUIRED` ： 表示多个事务合成 1 个事务

`SUPPORTS`：表示如果有事务，就加入到当前事务，如果没有，那也不开启事务执行。这种传播级别可用于查询方法，因为 SELECT 语句既可以在事务内执行，也可以不需要事务；

`MANDATORY`：表示必须要存在当前事务并加入执行，否则将抛出异常。这种传播级别可用于核心更新逻辑，比如用户余额变更，它总是被其他事务方法调用，不能直接由非事务方法调用；

`REQUIRES_NEW`：表示不管当前有没有事务，都必须开启一个新的事务执行。如果当前已经有事务，那么当前事务会挂起，等新事务完成后，再恢复执行；

`NOT_SUPPORTED`：表示不支持事务，如果当前有事务，那么当前事务会挂起，等这个方法执行完成后，再恢复执行；

`NEVER`：和`NOT_SUPPORTED`相比，它不但不支持事务，而且在监测到当前有事务时，会抛出异常拒绝执行；

`NESTED`：表示如果当前有事务，则开启一个嵌套级别事务，如果当前没有事务，则开启一个新事务。

在注解里定义事务传播级别

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public Product createProduct() {
    ...
}
```

#### 2.3.2 使用 DAO

Controller -> Service Service -> Dao

写数据访问层的时候，可以使用 DAO 模式。DAO 即 Data Access Object 的缩写

```java
public class UserDao {

    @Autowired
    JdbcTemplate jdbcTemplate;

    User getById(long id) {
        ...
    }

    List<User> getUsers(int page) {
        ...
    }

    User createUser(User user) {
        ...
    }

    User updateUser(User user) {
        ...
    }

    void deleteUser(User user) {
        ...
    }
}
```

是否使用 DAO，根据实际情况决定，因为很多时候，直接在 Service 层操作数据库也是完全没有问题的

#### 2.3.4 集成 Hibernate

在 jdbcTemplate 种，用 RowMapper 函数把 ResultSet 转成 Java Bean

这种把关系数据库的表记录映射为 Java 对象的过程就是 ORM：Object-Relational Mapping。ORM 既可以把记录转换成 Java 对象，也可以把 Java 对象转换为行记录

Hibernate 作为 ORM 框架，它可以替代`JdbcTemplate`，但 Hibernate 仍然需要 JDBC 驱动，所以，我们需要引入 JDBC 驱动、连接池，以及 Hibernate 本身

##### 配置

```java
@Configuration
@ComponentScan
@EnableTransactionManagement
@PropertySource("jdbc.properties")
public class AppConfig {
    @Bean
    DataSource createDataSource() {
        ...
    }

    //用了上面注入的DataSource,创建一个LocalSessionFactoryBean
     @Bean
    LocalSessionFactoryBean createSessionFactory(@Autowired DataSource dataSource) {
        var props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "update"); // 生产环境不要使用
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.HSQLDialect");
        props.setProperty("hibernate.show_sql", "true");
        var sessionFactoryBean = new LocalSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource);
        // 扫描指定的package获取所有entity class:
        sessionFactoryBean.setPackagesToScan("com.itranswarp.learnjava.entity");
        sessionFactoryBean.setHibernateProperties(props);
        return sessionFactoryBean;
    }

    //创建Template和TransactionManager
     @Bean
    HibernateTemplate createHibernateTemplate(@Autowired SessionFactory sessionFactory) {
        return new HibernateTemplate(sessionFactory);
    }

    @Bean
    PlatformTransactionManager createTxManager(@Autowired SessionFactory sessionFactory) {
        return new HibernateTransactionManager(sessionFactory);
    }
}
```

`LocalSessionFactoryBean`是一个`FactoryBean`，它会再自动创建一个`SessionFactory`

在 Hibernate 中，`Session`是封装了一个 JDBC `Connection`的实例，而`SessionFactory`是封装了 JDBC `DataSource`的实例，即`SessionFactory`
持有连接池，每次需要操作数据库的时候，`SessionFactory`创建一个新的`Session`，相当于从连接池获取到一个新的`Connection`。`SessionFactory`就是 Hibernate
提供的最核心的一个对象，但`LocalSessionFactoryBean`是 Spring 提供的为了让我们方便创建`SessionFactory`的类

##### 用注解定义 Entity

```java
@Entity
@Table(name="users)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(nullable = false, updatable = false)
    public Long getId() { ... }

    @Column(nullable = false, unique = true, length = 100)
    public String getEmail() { ... }

    @Column(nullable = false, length = 100)
    public String getPassword() { ... }

    @Column(nullable = false, length = 100)
    public String getName() { ... }

    @Column(nullable = false, updatable = false)
    public Long getCreatedAt() { ... }
}
```

使用 Hibernate 时，不要使用基本类型的属性，总是使用包装类型，如 Long 或 Integer。

可以把不同的 entity 中相同的属性抽象出来

```java
@MappedSuperclass //这个注解表示用于继承
public abstract class AbstractEntity {

    private Long id;
    private Long createdAt;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(nullable = false, updatable = false)
    public Long getId() { ... }

    @Column(nullable = false, updatable = false)
    public Long getCreatedAt() { ... }

    @Transient //表示不是查表得到的属性
    public ZonedDateTime getCreatedDateTime() {
        return Instant.ofEpochMilli(this.createdAt).atZone(ZoneId.systemDefault());
    }

    @PrePersist //insert之前会执行该操作
    public void preInsert() {
        setCreatedAt(System.currentTimeMillis());
    }
}
```

注意到使用的所有注解均来自`javax.persistence`，它是 JPA 规范的一部分

##### 表的增删改操作

```java
@Component
@Transactional
public class UserService {
    @Autowired
    HibernateTemplate hibernateTemplate;

    //Insert操作
    public User register(String email, String password, String name) {
        // 创建一个User对象:
        User user = new User();
        // 设置好各个属性:
        user.setEmail(email);
        user.setPassword(password);
        user.setName(name);
        // 不要设置id，因为使用了自增主键
        // 保存到数据库:
        hibernateTemplate.save(user);
        // 现在已经自动获得了id:
        System.out.println(user.getId());
        return user;
    }

    //Delete操作
    public boolean deleteUser(Long id) {
        User user = hibernateTemplate.get(User.class, id);
        if (user != null) {
            hibernateTemplate.delete(user);
            return true;
        }
        return false;
    }

    //Update操作
    public void updateUser(Long id, String name) {
        User user = hibernateTemplate.load(User.class, id);
        user.setName(name);
        hibernateTemplate.update(user);
    }

}
```

##### 表的查询

`方法1`使用 Example

使用`findByExample()`，给出一个`User`实例，Hibernate 把该实例所有非`null`的属性拼成`WHERE`条件

```java
public User login(String email, String password) {
    User example = new User();
    example.setEmail(email);
    example.setPassword(password);
    List<User> list = hibernateTemplate.findByExample(example);
    return list.isEmpty() ? null : list.get(0);
}
```

`方法2` 使用 Criteria 查询

```java
public User login(String email, String password) {
    DetachedCriteria criteria = DetachedCriteria.forClass(User.class);
    criteria.add(Restrictions.eq("email", email))
            .add(Restrictions.eq("password", password));
    List<User> list = (List<User>) hibernateTemplate.findByCriteria(criteria);
    return list.isEmpty() ? null : list.get(0);
}
```

`方法3` 使用 HQL 查询

```java
List<User> list = (List<User>) hibernateTemplate.find("FROM User WHERE email=? AND password=?", email, password);
```

#### 2.3.5 集成 JPA

JPA 就是 JavaEE 的一个 ORM 标准

Hibernate 是 JPA 的一个实现

##### 配置

```java
@Configuration
@ComponentScan
@EnableTransactionManagement
@PropertySource("jdbc.properties")
public class AppConfig {
    @Bean
    DataSource createDataSource() { ... }

    //类比Hibernate的LocalSessionFactoryBean 和 SessionFactory
    @Bean
	LocalContainerEntityManagerFactoryBean createEntityManagerFactory(@Autowired DataSource dataSource) {
        var entityManagerFactoryBean = new LocalContainerEntityManagerFactoryBean();
        // 设置DataSource:
        entityManagerFactoryBean.setDataSource(dataSource);
        // 扫描指定的package获取所有entity class:
        entityManagerFactoryBean.setPackagesToScan("com.itranswarp.learnjava.entity");
        // 指定JPA的提供商是Hibernate:
        JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        entityManagerFactoryBean.setJpaVendorAdapter(vendorAdapter);
        // 设定特定提供商自己的配置:
        var props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "update");
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.HSQLDialect");
        props.setProperty("hibernate.show_sql", "true");
        entityManagerFactoryBean.setJpaProperties(props);
        return entityManagerFactoryBean;
    }

    //实例化一个JpaTransactionManager，以实现声明式事务
    @Bean
    PlatformTransactionManager createTxManager(@Autowired EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

##### Service 中使用

```java
@Component
@Transactional
public class UserService {
    @PersistenceContext //不要使用Autowired，而是@PersistenceContext,表示多个线程共享
    EntityManager em;

    //业务使用
    public User getUserById(long id) {
        User user = this.em.find(User.class, id);
        if (user == null) {
            throw new RuntimeException("User not found by id: " + id);
        }
        return user;
    }
}
```

这里注入的并不是真正的`EntityManager`，而是一个`EntityManager`的代理类

Spring 遇到标注了`@PersistenceContext`的`EntityManager`会自动注入代理，该代理会在必要的时候自动打开`EntityManager`。换句话说，多线程引用的`EntityManager`
虽然是同一个代理类，但该代理类内部针对不同线程会创建不同的`EntityManager`实例

#### 2.3.6 MyBatis

Hibernate/JPA 被称作全自动 ORM，对比 jdbcTemplate，主要自动化了以下 2 点

1. 增删改的参数不需要手动传入，不需要去对应 Entity 的哪个属性
2. 结果不需要用 RowMapper 函数把 ResultSet 转成 JavaBean

Mybatis 被称为半自动化 ORM，只实现了 RowMapper 的功能，还是需要输入 SQL 语句

##### 配置 - 几种方法的配置区别表

```java
@Configuration
@ComponentScan
@EnableTransactionManagement
@PropertySource("jdbc.properties")
public class AppConfig {
    @Bean
    DataSource createDataSource() { ... }

    //创建SqlSessionFactory
    @Bean
    SqlSessionFactoryBean createSqlSessionFactoryBean(@Autowired DataSource dataSource) {
        var sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        return sqlSessionFactoryBean;
    }

    //事务管理器
    @Bean
    PlatformTransactionManager createTxManager(@Autowired DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    //不需要Template
}
```

| JDBC       | Hibernate      | JPA                  | MyBatis           |
| :--------- | :------------- | :------------------- | :---------------- |
| DataSource | SessionFactory | EntityManagerFactory | SqlSessionFactory |
| Connection | Session        | EntityManager        | SqlSession        |

##### MyBatis Mapper

和 Hibernate 不同的是，MyBatis 使用 Mapper 来实现映射，而且 Mapper 必须是接口

```java
public interface UserMapper {
	@Select("SELECT * FROM users WHERE id = #{id}")
	User getById(@Param("id") long id);

    //如果是多个参数，用#占位符
    @Select("SELECT * FROM users LIMIT #{offset}, #{maxResults}")
	List<User> getAll(@Param("offset") int offset, @Param("maxResults") int maxResults);

    //别名返回，用AS
    //SELECT id, name, email, created_time AS createdAt FROM users

    //INSERT 以#{obj.property}的方式写占位符
    @Insert("INSERT INTO users (email, password, name, createdAt) VALUES (#{user.email}, #{user.password}, #{user.name}, #{user.createdAt})")
    void insert(@Param("user") User user);

    //如果users表的id是自增主键，那么，我们在SQL中不传入id，但希望获取插入后的主键，需要再加一个@Options注解
    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
    @Insert("INSERT INTO users (email, password, name, createdAt) VALUES (#{user.email}, #{user.password}, #{user.name}, #{user.createdAt})")
    void insert(@Param("user") User user);

    //Update和Delete
    @Update("UPDATE users SET name = #{user.name}, createdAt = #{user.createdAt} WHERE id = #{user.id}")
    void update(@Param("user") User user);

    @Delete("DELETE FROM users WHERE id = #{id}")
    void deleteById(@Param("id") long id);

}
```

使用@MapperScan 来为所有 Interface 类型的 Mapper，自动创建实现类

```java
@MapperScan("com.itranswarp.learnjava.mapper")
...其他注解...
public class AppConfig {
    ...
}
```

##### 注入 Sevice

```java
@Component
@Transactional
public class UserService {
    // 注入UserMapper:
    @Autowired
    UserMapper userMapper;

    public User getUserById(long id) {
        // 调用Mapper方法:
        User user = userMapper.getById(id);
        if (user == null) {
            throw new RuntimeException("User not found by id.");
        }
        return user;
    }
}
```

## 3 Maven 基础

### 3.1 Maven 介绍

一个使用 Maven 管理的普通的 Java 项目，它的目录结构默认如下：

```scheme
a-maven-project				//项目名
├── pom.xml					//项目描述文件
├── src
│   ├── main
│   │   ├── java			//源码
│   │   └── resources		//资源文件
│   └── test
│       ├── java			//测试源码
│       └── resources		//测试资源
└── target					//编译结果
```

pom.xml

```xml
<project ...>
	<modelVersion>4.0.0</modelVersion>
<groupId>com.itranswarp.learnjava</groupId>  // 包名
<artifactId>hello</artifactId>                //  类名
<version>1.0</version>    //每个maven工程=groupId+artifactId+version作为唯一标识
	<packaging>jar</packaging>
	<properties>
        ...
	</properties>
	<dependencies>
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>
	</dependencies>
</project>
```

### 3.2 依赖管理

#### 依赖关系

Maven 定义了几种依赖关系，分别是`compile`、`test`、`runtime`和`provided`：

| scope    | 说明                                            | 示例            |
| :------- | :---------------------------------------------- | :-------------- |
| compile  | 编译时需要用到该 jar 包（默认）                 | commons-logging |
| test     | 编译 Test 时需要用到该 jar 包                   | junit           |
| runtime  | 编译时不需要，但运行时需要用到                  | mysql           |
| provided | 编译时需要用到，但运行时由 JDK 或某个服务器提供 | servlet-api     |

示例

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.3.2</version>
    <scope>test</scope>
</dependency>
```

Maven 的中央仓库（[repo1.maven.org](https://repo1.maven.org/)）

一个 jar 包一旦被下载过，就会被 Maven 自动缓存在本地目录（用户主目录的`.m2`目录）

#### Maven 镜像

除了可以从 Maven 的中央仓库下载外，还可以从 Maven 的镜像仓库下载

中国区用户可以使用阿里云提供的 Maven 镜像仓库。使用 Maven 镜像仓库需要一个配置，在用户主目录下进入`.m2`目录，创建一个`settings.xml`配置文件，内容如下：

```xml
<settings>
    <mirrors>
        <mirror>
            <id>aliyun</id>
            <name>aliyun</name>
            <mirrorOf>central</mirrorOf>
            <!-- 国内推荐阿里云的Maven镜像 -->
            <url>https://maven.aliyun.com/repository/central</url>
        </mirror>
    </mirrors>
</settings>
```

也可以在 pom.xml 中设置仓库地址

```xml
<repositories>
    <repository>
        <id>nexus-aliyun</id>
        <name>Nexus aliyun</name>
        <url>https://maven.aliyun.com/repository/public</url>
        <layout>default</layout>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <releases>
            <enabled>true</enabled>
        </releases>
    </repository>
</repositories>
<pluginRepositories>
    <pluginRepository>
        <id>nexus-aliyun</id>
        <name>Nexus aliyun</name>
        <url>https://maven.aliyun.com/repository/public</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <releases>
            <enabled>true</enabled>
        </releases>
    </pluginRepository>
</pluginRepositories>
```

#### 搜索第三方组件

如果我们要引用一个第三方组件，比如`okhttp`，如何确切地获得它的`groupId`、`artifactId`和`version`？方法是通过[search.maven.org](https://search.maven.org/)搜索关键字，找到对应的组件后，直接复制

### 3.3 构建流程

Maven 有不同的 lifecycle，每种 lifecycle 有不同的 phase 组成，每个 phase 又有不同的 Goal

default lifecycle 的 phase 如下

- validate
- initialize
- generate-sources
- process-sources
- generate-resources
- process-resources
- compile
- process-classes
- generate-test-sources
- process-test-sources
- generate-test-resources
- process-test-resources
- test-compile
- process-test-classes
- test
- prepare-package
- package
- pre-integration-test
- integration-test
- post-integration-test
- verify
- install
- deploy

在实际开发过程中，经常使用的命令有：

`mvn clean`：清理所有生成的 class 和 jar；

`mvn clean compile`：先清理，再执行到`compile`；

`mvn clean test`：先清理，再执行到`test`，因为执行`test`前必须执行`compile`，所以这里不必指定`compile`；

`mvn clean package`：先清理，再执行到`package`。

#### Goal

执行一个 phase 又会触发一个或多个 goal：

| 执行的 Phase | 对应执行的 Goal                    |
| :----------- | :--------------------------------- |
| compile      | compiler:compile                   |
| test         | compiler:testCompile surefire:test |

{% endraw %}
