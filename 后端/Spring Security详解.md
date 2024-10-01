{% raw %}

# Spring Security 详解

[江南一点雨 Spring Security](http://www.javaboy.org/springsecurity/)

## 1 入门

### 1.1 初始化

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

添加依赖后默认会在启动日志打印

```
Using generated security password: 30abfb1f-36e1-446a-a79b-f70024f589ab
```

在登录页面，默认的用户名就是 user，默认的登录密码则是项目启动时控制台打印出来的密码，输入用户名密码之后，就登录成功了

用户相关的自动化配置类在 `UserDetailsServiceAutoConfiguration` 里边，在该类的 `getOrDeducePassword` 方法中，我们看到如下一行日志

```java
if (user.isPasswordGenerated()) {
	logger.info(String.format("%n%nUsing generated security password: %s%n", user.getPassword()));
}
```

user.getPassword 出现在 SecurityProperties 中，在 SecurityProperties 中我们看到如下定义

```java
/**
 * Default user name.
 */
private String name = "user";
/**
 * Password for the default user name.
 */
private String password = UUID.randomUUID().toString();
private boolean passwordGenerated = true;
```

### 1.2 用户配置

#### 1.2.1 配置文件

我们可以在 application.properties 中配置默认的用户名密码。

默认的用户就定义在`SecurityProperties`里边，是一个静态内部类，我们如果要定义自己的用户名密码，必然是要去覆盖默认配置，我们先来看下 `SecurityProperties `的定义：

```java
@ConfigurationProperties(prefix = "spring.security")
public class SecurityProperties {
```

这就很清晰了，我们只需要以 spring.security.user 为前缀，去定义用户名密码即可：

```properties
spring.security.user.name=javaboy
spring.security.user.password=123
```

在 properties 中定义的用户名密码最终是通过 set 方法注入到属性中去的，这里我们顺便来看下 SecurityProperties.User#setPassword 方法

```java
public void setPassword(String password) {
	if (!StringUtils.hasLength(password)) {
		return;
	}
	this.passwordGenerated = false; //取消自动生成密码
	this.password = password;
}
```

#### 1.2.2 配置类

##### 加密方案

密码加密我们一般会用到散列函数，又称散列算法、哈希函数，这是一种从任何数据中创建数字“指纹”的方法。散列函数把消息或数据压缩成摘要，使得数据量变小，将数据的格式固定下来，然后将数据打乱混合，重新创建一个散列值。散列值通常用一个短的随机字母和数字组成的字符串来代表。好的散列函数在输入域中很少出现散列冲突。在散列表和数据处理中，不抑制冲突来区别数据，会使得数据库记录更难找到。我们常用的散列函数有 MD5 消息摘要算法、安全散列算法

但是仅仅使用散列函数还不够，为了增加密码的安全性，一般在密码加密过程中还需要加盐，所谓的盐可以是一个随机数也可以是用户名，加盐之后，即使密码明文相同的用户生成的密码密文也不相同，这可以极大的提高密码的安全性。但是传统的加盐方式需要在数据库中有专门的字段来记录盐值，这个字段可能是用户名字段（因为用户名唯一），也可能是一个专门记录盐值的字段，这样的配置比较繁琐

Spring Security 提供了多种密码加密方案，官方推荐使用 BCryptPasswordEncoder，BCryptPasswordEncoder 使用 BCrypt 强哈希函数，开发者在使用时可以选择提供 strength 和 SecureRandom 实例。strength 越大，密钥的迭代次数越多，密钥迭代次数为 2^strength。strength 取值在 4~31 之间，默认为 10

##### PasswordEncoder 接口

```java
public interface PasswordEncoder {
	String encode(CharSequence rawPassword);
	boolean matches(CharSequence rawPassword, String encodedPassword);
	default boolean upgradeEncoding(String encodedPassword) {
		return false;
	}
}
```

1. encode 方法用来对明文密码进行加密，返回加密之后的密文。
2. matches 方法是一个密码校对方法，在用户登录的时候，将用户传来的明文密码和数据库中保存的密文密码作为参数，传入到这个方法中去，根据返回的 Boolean 值判断用户密码是否输入正确。
3. upgradeEncoding 是否还要进行再次加密，这个一般来说就不用了

##### 配置

1. 首先我们自定义 SecurityConfig 继承自 WebSecurityConfigurerAdapter，重写里边的 configure 方法。
2. 首先我们提供了一个 PasswordEncoder 的实例，因为目前的案例还比较简单，因此我暂时先不给密码进行加密，所以返回 NoOpPasswordEncoder 的实例即可。
3. configure 方法中，我们通过 inMemoryAuthentication 来开启在内存中定义用户，withUser 中是用户名，password 中则是用户密码，roles 中是用户角色。
4. 如果需要配置多个用户，用 and 相连

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean //这个注解表示这个类不在项目的package里面，所以在config里创建并返回
    PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("javaboy.org")
                .password("123").roles("admin");
    }
}
```

### 1.3 自定义表单登录页

#### 1.3.1 服务端定义

重写 SecurityConfig 类的 `configure(WebSecurity web)` 和 `configure(HttpSecurity http)` 方法

```java
@Override
public void configure(WebSecurity web) throws Exception {
    //忽略掉的 URL
    web.ignoring().antMatchers("/js/**", "/css/**","/images/**");
}
@Override
//用 XML 来配置 Spring Security ，里边会有一个重要的标签 <http>，HttpSecurity 提供的配置方法 都对应了该标签
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests() //authorizeRequests 对应了 <intercept-url>
            .anyRequest().authenticated()
            .and()
            .formLogin() //对应了 <formlogin>
            .loginPage("/login.html")
            .permitAll() //表示登录相关的页面/接口不要被拦截
            .and() //and 方法表示结束当前标签，上下文回到HttpSecurity，开启新一轮的配置
            .csrf().disable();//关闭 csrf
}
```

## 2 定制 Spring Security 中的表单登录

### 2.1 登录接口

在 Spring Security 中，如果我们不做任何配置，默认的登录页面和登录接口的地址都是 `/login`，也就是说，默认会存在如下两个请求：

- GET http://localhost:8080/login
- POST http://localhost:8080/login

```java
.and()
.formLogin()
.loginPage("/login.html")
.loginProcessingUrl("/doLogin")
.permitAll()
.and()
```

新的登录页面和登录接口地址都是 `/login.html`，现在存在如下两个请求：

- GET http://localhost:8080/login.html
- POST http://localhost:8080/login.html

前面的 GET 请求用来获取登录页面，后面的 POST 请求用来提交登录数据。

可以通过 loginProcessingUrl 方法来指定登录接口地址

## 官方文档 5.6.2

### 1 Getting Started

只要在 pom.xml 中加入 security，Spring Boot 就自动化完成以下内容

1. Spring Security 默认配置，创建一个叫做 springSecurityFilterChain 的 servlet Filter Bean。这个 Bean 用于过滤 URL，验证用户名密码，重定向登陆页面
2. 创建了一个 UserDetailService Bean。用户名是 user,随机产生密码并打印到终端
3. 注册了一个叫做 springSecurityFilterChain 的 filter 到 Servlet 容器的每一次请求

默认的 Configuration 做了如下工作

- 要求对应用程序中的每个 URL 进行身份验证
- 为您生成一个登录表单
- 允许具有 **Username** * user *和 **Password** * password *的用户使用基于表单的身份验证进行身份验证
- 允许用户注销
- [CSRF attack](https://en.wikipedia.org/wiki/Cross-site_request_forgery) prevention
- [Session Fixation](https://en.wikipedia.org/wiki/Session_fixation) protection
- 安全标题集成
- [HTTP 严格传输安全](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security)安全请求
    - [X-Content-Type-Options](<https://msdn.microsoft.com/en-us/library/ie/gg622941(v=vs.85).aspx>) integration
    - 缓存控制(以后可以由您的应用程序覆盖以允许缓存您的静态资源)
    - [X-XSS-Protection](<https://msdn.microsoft.com/en-us/library/dd565647(v=vs.85).aspx>) integration
    - X-Frame-Options 集成有助于防止[Clickjacking](https://en.wikipedia.org/wiki/Clickjacking)

### 2 架构

#### 2.1 Servlet 的 Filters

处理一笔 http 请求的 filter chain

![filterchain](../image/Spring Security 详解/filterchain.png)

Spring MVC 里 Servlet 就是一个 DispatcherServlet 的实例。一组 `HttpServletRequest` and `HttpServletResponse`最多一个 Servlet 可以处理。

容器创建了包含多个 Filter 的 FilterChain 处理`HttpServletRequest` 。

FilterChain 处理的代码片段

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	// do something before the rest of the application
    chain.doFilter(request, response); // invoke the rest of the application
    // do something after the rest of the application
}
```

#### 2.2 DelegatingFilterProxy

Spring 实现 Filter
的类是 [`DelegatingFilterProxy`](https://docs.spring.io/spring-framework/docs/5.3.16/javadoc-api/org/springframework/web/filter/DelegatingFilterProxy.html)

Servlet 容器注册 filter 的接口和 Spring 定义的 Beans 不一样，所有有`DelegatingFilterProxy`用户转换

![delegatingfilterproxy](../image/Spring Security 详解/delegatingfilterproxy.png)

#### 2.3 FilterChainProxy

Spring Securtiy 使用了 FilterChainProxy 来实现 SecurityFilterChain 的。

[Security Filters](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-security-filters)
是一系列标准的 Bean，朝 FilterChainProxy 注册

![securityfilterchain](../image/Spring Security 详解/securityfilterchain.png)

对于不同的路径，FilterChainProxy 可以注册不同的 SecurityFilterChain

![multi securityfilterchain](../image/Spring Security 详解/multi-securityfilterchain.png)

每个 SecurityFilterChain 有不同的 Configuration

#### 2.4 Security Filters

常用的 Filter 顺序

- ChannelProcessingFilter
- WebAsyncManagerIntegrationFilter
- SecurityContextPersistenceFilter
- HeaderWriterFilter
- CorsFilter
- CsrfFilter
- LogoutFilter
- OAuth2AuthorizationRequestRedirectFilter
- Saml2WebSsoAuthenticationRequestFilter
- X509AuthenticationFilter
- AbstractPreAuthenticatedProcessingFilter
- CasAuthenticationFilter
- OAuth2LoginAuthenticationFilter
- Saml2WebSsoAuthenticationFilter
- [`UsernamePasswordAuthenticationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html#servlet-authentication-usernamepasswordauthenticationfilter)
- OpenIDAuthenticationFilter
- DefaultLoginPageGeneratingFilter
- DefaultLogoutPageGeneratingFilter
- ConcurrentSessionFilter
- [`DigestAuthenticationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/digest.html#servlet-authentication-digest)
- BearerTokenAuthenticationFilter
- [`BasicAuthenticationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/basic.html#servlet-authentication-basic)
- RequestCacheAwareFilter
- SecurityContextHolderAwareRequestFilter
- JaasApiIntegrationFilter
- RememberMeAuthenticationFilter
- AnonymousAuthenticationFilter
- OAuth2AuthorizationCodeGrantFilter
- SessionManagementFilter
- [`ExceptionTranslationFilter`](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-exceptiontranslationfilter)
- [`FilterSecurityInterceptor`](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-requests.html#servlet-authorization-filtersecurityinterceptor)
- SwitchUserFilter

#### 2.5 处理 Security Exception

[`ExceptionTranslationFilter`](https://docs.spring.io/spring-security/site/docs/5.6.2/api/org/springframework/security/web/access/ExceptionTranslationFilter.html)

可以把 [`AccessDeniedException`](https://docs.spring.io/spring-security/site/docs/5.6.2/api/org/springframework/security/access/AccessDeniedException.html)
and [`AuthenticationException`](https://docs.spring.io/spring-security/site/docs/5.6.2/api//org/springframework/security/core/AuthenticationException.html)

这两种异常转换成 HTTP 响应

`ExceptionTranslationFilter`也作为 Security Filter 的一种注册到 FilterChainProxy

![exceptiontranslationfilter](../image/Spring Security 详解/exceptiontranslationfilter.png)

1. ExceptionTranslationFilter 调用 FilterChain.doFilter(request, response)
2. 如果用户是未认证的，开始做验证(authentication)
    1. 清空 SecurityContextHolder
    2. 把 HttpServletRequest 存到 RequestCache 里面。如果用户验证成功，RequestCache 被用来应答原始请求
    3. AuthenticationEntryPoint 用于向客户端请求证书，比如重定向到登陆页面或者送一个 www-authenticate 的 header
3. 如果是客户已登陆，但是访问权限不对。进入 AccessDeniedHandler

### 3 Authentication 验证

验证就是怎么校验登录信息

#### 3.1 Authentication 架构

- [SecurityContextHolder](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontextholder)
  - `SecurityContextHolder` 保存通过验证的用户信息
- [SecurityContext](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontext)
  - 从 SecurityContextHolder 获得，保存当前授权用户的 `Authentication`
- [Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authentication)
  - 作为 `AuthenticationManager` 的输入，表示一个授权用户的证书
- [GrantedAuthority](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-granted-authority)
  - `Authentication` 中以获取的授权(i.e. roles, scopes, etc.)
- [AuthenticationManager](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authenticationmanager)
  - 定义了一组 API,提供给 Spring Security’s Filters
  处理[authentication](https://docs.spring.io/spring-security/reference/features/authentication/index.html#authentication)
  .
- [ProviderManager](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-providermanager)
  - `AuthenticationManager`最常用的实现
- [AuthenticationProvider](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authenticationprovider) - 在`ProviderManager`里使用，用于定义一种特别的验证方式
- [用`AuthenticationEntryPoint`请求证书](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authenticationentrypoint)
  - 从客户端获取证书(i.e. redirecting to a log in page, sending a `WWW-Authenticate` response, etc.)
- [AbstractAuthenticationProcessingFilter](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-abstractprocessingfilter)
  - 用于 authentication 的 Filter 基类

##### 3.1.1 SecurityContextHolder

![securitycontextholder](../image/Spring Security 详解/securitycontextholder.png)

SecurityContextHolder 用于存储被认证用户信息

最简单的方法表示用户被认证是直接设置 SecurityContextHolder

```java
SecurityContext context = SecurityContextHolder.createEmptyContext(); //1
Authentication authentication =
    new TestingAuthenticationToken("username", "password", "ROLE_USER"); //2
context.setAuthentication(authentication);

SecurityContextHolder.setContext(context); //3
```

1. 不能用 SecurityContextHolder.getContext().setAuthentication(authentication)这样的方法，避免 race condition
2. 创建 Authentication 对象，更常用的方法创建用 `UsernamePasswordAuthenticationToken(userDetails, password, authorities)`
3. 最终设置 SecurityContextHolder

通过 SecurityContextHolder 获取经认证的用户信息

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();

String username = authentication.getName();
Object principal = authentication.getPrincipal();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```

SecurityContextHolder 是用 ThreadLocal 来存储数据的，这意味着 SecurityContext 在同一线程里永远有效，即使 SecurityContext 不是被显式地传参。

###### ThreadLocal

Java 标准库提供了一个特殊的`ThreadLocal`，它可以在一个线程中传递同一个对象

典型用法

```java
void processUser(user) {
    try {
        threadLocalUser.set(user);
        step1();
        step2();
    } finally {
        threadLocalUser.remove();
    }
}

void step1() {
    User u = threadLocalUser.get();
    log();
    printUser();
}
```

##### 3.1.2 Authentication

这个类有两个用途:

- [`AuthenticationManager`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authenticationmanager)
  的输入，提供认证用的用户证书。在这种场景下， `isAuthenticated()`返回 false
- 表示当前已认证的用户

Authentication 包含 3 个部分

- principal - 表示用户，当有了 username/password 认证，就是一个 UserDetails 的实例
- credentials - password. 用户认证完之后就会 clear
- authrities - roles/scops

##### 3.1.3 GrantedAuthority

用户获取的高级权限

通过 [`Authentication.getAuthorities()`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authentication)
获取。这个方法获取一个 GrantedAuthority 的集合。roles: `ROLE_ADMINISTRATOR` or `ROLE_HR_SUPERVISOR`

当用 username/password 来认证时，GrantedAuthority
被 [`UserDetailsService`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/user-details-service.html#servlet-authentication-userdetailsservice)
加载

##### 3.1.4 AuthenticationManager

定义了一组 API,让 filter 用来做认证。

如果直接设置 SecurityContextHolder，而不集成 Filters，但就不用 AuthenticationManager

最常用的实现就是[`ProviderManager`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-providermanager)

##### 3.1.5 ProviderManager

ProviderManger 里面放了一个 AuthenticationProvider 的 list。每个 AuthenticationProvider 表示一阵认证方式。如果没有一个 AuthenticationProvider
认证成功，抛出 ProviderNotFoundException 异常

![providermanager](../image/Spring Security 详解/providermanager.png)

当有多个[`SecurityFilterChain`](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-securityfilterchain)
实例的时候，每个实例都会有一个不同的 ProviderManager 实例。这个时候可以声明一个父类，让各个子实例继承自父类

![providermanagers parent](../image/Spring Security 详解/providermanagers-parent.png)

##### 3.1.6 AuthenticationProvider

1 个 ProviderManager
可以注入多个[`AuthenticationProvider`s](https://docs.spring.io/spring-security/site/docs/5.6.2/api/org/springframework/security/authentication/AuthenticationProvider.html)
。每一种都是一个不同的认证方式。比如 [`DaoAuthenticationProvider`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html#servlet-authentication-daoauthenticationprovider)
支持 username/password 认证。 `JwtAuthenticationProvider`支持 JWT token

##### 3.1.7 AuthenticationEntryPoint

`AuthenticationEntryPoint`用于发送一个 HTTP 应答给客户端，要求客户端发送证书。一般操作就是重定向到 login 页或者带 WWW-Authenticate 的 head

##### 3.1.8 AbstractAuthenticationProcessingFilter

AbstractAuthenticationProcessingFilter 是个认证用的基类。Spring Security 用 AuthenticationEntryPoint
发送认证请求以后，AbstractAuthenticationProcessingFilter 用于认证

![abstractauthenticationprocessingfilter](../image/Spring Security 详解/abstractauthenticationprocessingfilter.png)

1. AbstractAuthenticationProcessingFilter 从 HttpServletRequest 里创建 Authentication。Authentication 类型取决于
   AbstractAuthenticationProcessingFilter
   的子类。比如[`UsernamePasswordAuthenticationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html#servlet-authentication-usernamepasswordauthenticationfilter)
   创建了一个 UsernamePasswordAuthenticationToken
2. Authentication
   传给[`AuthenticationManager`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authenticationmanager)
   认证
3. 如果认证失败
    1. [SecurityContextHolder](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontextholder)
       clear
    2. 调用 RememberMeServices.loginFail
    3. 调用 AuthenticationFailureHandler
4. 如果认证成功
    1. 通知 SessionAuthenticationStrategy
    2. [Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authentication)
       设置到 SecurityContextHolder 里。SecurityContextPersistenceFilter 把 SecurityContext 存到 HttpSession 里
    3. 调用 RememberMeServices.loginSuccess
    4. ApplicationEventPublisher 触发 InteractiveAuthenticationSuccessEvent 事件
    5. AuthenticationSuccessHandler 被调用

#### 3.2 Username/Password Authentication

##### 3.2.1 获取用户名密码

用户访问重定向到 login form 的流程

![loginurlauthenticationentrypoint](../image/Spring Security 详解/loginurlauthenticationentrypoint.png)

UsernamePasswordAuthenticationFilter
是一个 [AbstractAuthenticationProcessingFilter](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-abstractprocessingfilter)
的子类。认证用户名和密码流程如下

![usernamepasswordauthenticationfilter](../image/Spring Security 详解/usernamepasswordauthenticationfilter.png)

Spring Security 默认提供 form log。但是如果定义了 Configuration，那么就要显示打开支持

```java
//默认登陆页面
protected void configure(HttpSecurity http) {
	http
		// ...
		.formLogin(withDefaults());
}
//自定义登陆页面
protected void configure(HttpSecurity http) throws Exception {
	http
		// ...
		.formLogin(form -> form
			.loginPage("/login")
			.permitAll()
		);
}
```

##### 3.2.2 密码存储

1. 简单存储 In-Memory Authentication
2. 数据库 JDBC Authentication
3. 自定义格式 UserDetailsService
4. LDAP Authentication

###### UserDetailsService

UserDetailsService 返回
UserDetails。 [`DaoAuthenticationProvider`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html#servlet-authentication-daoauthenticationprovider)
验证 UserDetails
然后返回[`Authentication`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authentication)

```java
Class UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException
}
```

###### DaoAuthenticationProvider

工作流程

![daoauthenticationprovider](../image/Spring Security 详解/daoauthenticationprovider.png)

1. Filter 传递 1 个 UsernamePasswordAuthenticationToken 给 AuthenticationManager
2. ProviderManger 被配置成调用用 DaoAuthenticationProvider
3. DaoAuthenticationProvider 从 UserDetailsService 获取 UserDetails
4. DaoAuthenticationProvider
   用 [`PasswordEncoder`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/password-encoder.html#servlet-authentication-password-storage)
   验证密码
5. 验证成功，以 UsernamePasswordAuthenticationToken
   的格式返回 [`Authentication`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authentication)
   。最终这个 UsernamePasswordAuthenticationToken
   会被设置到[`SecurityContextHolder`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontextholder)
   里

###### LDAP 认证

通过创建 LDAP ContextSource 来配置 Spring Security 指向 LDAP server

```java
ContextSource contextSource(UnboundIdContainer container) {
	return new DefaultSpringSecurityContextSource("ldap://localhost:53389/dc=springframework,dc=org");
}
```

LDAP 不用 UserDetailsService。因为 LDAP 服务器不返回 password 用于验证密码

LDAP 支持使用 LdapAuthenticator 接口。LdapAuthenticator 支持获取各种用户属性。有 2 种 LdapAuthenticator 接口实现

- BindAuthenticator
- PasswordComparisonAuthenticator

**_1 使用 Bind 认证_**

用户名密码直接发送给 LDAP server 来认证

```java
@Bean
BindAuthenticator authenticator(BaseLdapPathContextSource contextSource) {
	BindAuthenticator authenticator = new BindAuthenticator(contextSource);
	authenticator.setUserDnPatterns(new String[] { "uid={0},ou=people" });
	return authenticator;
}

@Bean
LdapAuthenticationProvider authenticationProvider(LdapAuthenticator authenticator) {
	return new LdapAuthenticationProvider(authenticator);
}
```

如果需要配置 LDAP 的 search filter

```java
@Bean
BindAuthenticator authenticator(BaseLdapPathContextSource contextSource) {
	String searchBase = "ou=people";
	String filter = "(uid={0})";
	FilterBasedLdapUserSearch search =
		new FilterBasedLdapUserSearch(searchBase, filter, contextSource);
	BindAuthenticator authenticator = new BindAuthenticator(contextSource);
	authenticator.setUserSearch(search);
	return authenticator;
}

@Bean
LdapAuthenticationProvider authenticationProvider(LdapAuthenticator authenticator) {
	return new LdapAuthenticationProvider(authenticator);
}
```

**_2 使用密码认证_**

从 LDAP server 获取 password 属性校验

直接执行 LDAP compare 操作

```java
@Bean
PasswordComparisonAuthenticator authenticator(BaseLdapPathContextSource contextSource) {
	return new PasswordComparisonAuthenticator(contextSource);
}

@Bean
LdapAuthenticationProvider authenticationProvider(LdapAuthenticator authenticator) {
	return new LdapAuthenticationProvider(authenticator);
}
```

更复杂一点的操作

```java
@Bean
PasswordComparisonAuthenticator authenticator(BaseLdapPathContextSource contextSource) {
	PasswordComparisonAuthenticator authenticator =
		new PasswordComparisonAuthenticator(contextSource);
	authenticator.setPasswordAttributeName("pwd"); //定义password属性的名字
	authenticator.setPasswordEncoder(new BCryptPasswordEncoder()); //用BCrypt
	return authenticator;
}

@Bean
LdapAuthenticationProvider authenticationProvider(LdapAuthenticator authenticator) {
	return new LdapAuthenticationProvider(authenticator);
}
```

**_3 LdapAuthoritiesPopulator_**

LdapAuthoritiesPopulator 定义了返回给 user 的权限

```java
@Bean
LdapAuthoritiesPopulator authorities(BaseLdapPathContextSource contextSource) {
	String groupSearchBase = "";
	DefaultLdapAuthoritiesPopulator authorities =
		new DefaultLdapAuthoritiesPopulator(contextSource, groupSearchBase);
	authorities.setGroupSearchFilter("member={0}");
	return authorities;
}

@Bean
LdapAuthenticationProvider authenticationProvider(LdapAuthenticator authenticator, LdapAuthoritiesPopulator authorities) {
	return new LdapAuthenticationProvider(authenticator, authorities);
}
```

#### 3.3 Session 管理

##### 3.3.1 超时检测

检测错误的 session id 并且重定向

```java
@Override
protected void configure(HttpSecurity http) throws Exception{
    http
        .sessionManagement(session -> session
            .invalidSessionUrl("/invalidSession.htm")
        );
}
```

如果用户先登出，在不管浏览器直接登录，会进入 invalidate。因为客户端 cookie 里的 seesion 未被删除。需要配置 logout handdler

```java
@Override
protected void configure(HttpSecurity http) throws Exception{
    http
        .logout(logout -> logout
            .deleteCookies("JSESSIONID")
        );
}
```

##### 3.3.2 单一用户管理

要限制一个用户同时多地登录

```java
@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}

@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .maximumSessions(1) //第二次登录，会导致第一次退出
            .maxSessionsPreventsLogin(true) //第二次无法登录
        );
}
```

#### 3.4 Remember Me

实现原理： 服务器发送一个 cookie 给浏览器。浏览器在不同的 session 里都用这个 cookie 来自动登录

Spring Security 有两种形式：

1 用 Hash 实现一个基于 cookie 的 token

2 用数据库持久化一个 token

##### 3.4.1 基于 Hash 的 Token 实现

cookie 的格式

```
base64(username + ":" + expirationTime + ":" +
md5Hex(username + ":" + expirationTime + ":" password + ":" + key))

username:          As identifiable to the UserDetailsService
password:          That matches the one in the retrieved UserDetails
expirationTime:    The date and time when the remember-me token expires, expressed in milliseconds
key:               A private key to prevent modification of the remember-me token
```

### 4 Authorization 授权

#### 4.1 架构

`GrantedAuthority`对象通过`AuthenticationManager`注入到`Authentication`对象。然后被`AuthorizationManager`读取来做权限控制

```java
public interface GrantedAuthority extends Serializable {
	String getAuthority();
}
```

如果是要回一个复杂的权限控制，getAuthority()要返回 null。

#### 4.2 鉴权处理

##### 4.2.1 AuthorizationManager

这个是用来取代[`AccessDecisionManager` and `AccessDecisionVoter`](https://docs.spring.io/spring-security/reference/servlet/authorization/architecture.html#authz-legacy-note)

有以下两个接口

```java
//这个Object可以是一个方法调用，然后根据authentication鉴权
AuthorizationDecision check(Supplier<Authentication> authentication, Object secureObject);

//
default AuthorizationDecision verify(Supplier<Authentication> authentication, Object secureObject)
    //调用check
    AuthorizationDecision decision = check(authentication, object);
    if (decision != null && !decision.isGranted()) {
        throw new AccessDeniedException("Access Denied");
    }
}
```

##### 4.2.2 AuthorizationManager 的实现

![authorizationhierarchy](../image/Spring Security 详解/authorizationhierarchy.png)

#### 4.3 分层的 Roles

配置方法

```java
@Bean
AccessDecisionVoter hierarchyVoter() {
    RoleHierarchy hierarchy = new RoleHierarchyImpl();
    hierarchy.setHierarchy("ROLE_ADMIN > ROLE_STAFF\n" +
            "ROLE_STAFF > ROLE_USER\n" +
            "ROLE_USER > ROLE_GUEST");
    return new RoleHierarcyVoter(hierarchy);
}
```

#### 4.4 鉴权 http 访问

## Spring Security 实践

### 1 简单原理 - 比文档讲的好多了

1. 用户登陆

   会被`AuthenticationProcessingFilter`拦截，调用`AuthenticationManager`的实现，而且 AuthenticationManager 会调用`ProviderManager`
   来获取用户验证信息（不同的 Provider 调用的服务不同，因为这些信息可以是在数据库上，可以是在 LDAP 服务器上，可以是 xml 配置文件上等），如果验证通过后会将用户的权限信息封装一个 User 放到 spring
   的全局缓存`SecurityContextHolder`中，以备后面访问资源时使用。

   AuthenticationProcessingFilter -> AuthenticationManager -> ProviderManager -> 把 user 放入 spring 的全局缓存
   SecurityContextHolder

2. 访问资源（即授权管理）

   访问 url 时，会通过`AbstractSecurityInterceptor`拦截器拦截，其中会调用`FilterInvocationSecurityMetadataSource`的方法来获取被拦截 url
   所需的全部权限，在调用授权管理器`AccessDecisionManager`，这个授权管理器会通过 spring 的全局缓存 SecurityContextHolder 获取用户的权限信息，还会获取被拦截的 url 和被拦截 url
   所需的全部权限，然后根据所配的策略（有：一票决定，一票否定，少数服从多数等），如果权限足够，则返回，权限不够则报错并调用权限不足页面。

   AbstractSecurityInterceptor 注入 MetaSource 和 DecisionManager，调用 doFilter->
   FilterInvocationSecurityMetadataSource.getAttributes() -> AccessDecisionManager.decide()

### 2 如何配置多个 AuthenticationProvider

#### 2.1 创建 2 个 Provider

ImMemoryAuthenticationProvider

```java
@Component
public class InMemoryAuthenticationProvider implements AuthenticationProvider {
  private final String adminName = "root";
  private final String adminPassword = "root";

  //根用户拥有全部的权限
  private final List<GrantedAuthority> authorities = Arrays.asList(new SimpleGrantedAuthority("CAN_SEARCH"),
      new SimpleGrantedAuthority("CAN_SEARCH"),
      new SimpleGrantedAuthority("CAN_EXPORT"),
      new SimpleGrantedAuthority("CAN_IMPORT"),
      new SimpleGrantedAuthority("CAN_BORROW"),
      new SimpleGrantedAuthority("CAN_RETURN"),
      new SimpleGrantedAuthority("CAN_REPAIR"),
      new SimpleGrantedAuthority("CAN_DISCARD"),
      new SimpleGrantedAuthority("CAN_EMPOWERMENT"),
      new SimpleGrantedAuthority("CAN_BREED"));

  //验证过程
  //如果AuthenticationProvider返回了null，AuthenticationManager会交给下一个支持authentication类型的AuthenticationProvider处理
  @Override
  public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    if(isMatch(authentication)){
      User user = new User(authentication.getName(),authentication.getCredentials().toString(),authorities);
      return new UsernamePasswordAuthenticationToken(user,authentication.getCredentials(),authorities);
    }
    return null;
  }

  @Override
  public boolean supports(Class<?> authentication) {
    return true;
  }

  private boolean isMatch(Authentication authentication){
    if(authentication.getName().equals(adminName)&&authentication.getCredentials().equals(adminPassword))
      return true;
    else
      return false;
  }
}
```

用数据库的 DaoAuthenticationProvider

```java
@Bean
DaoAuthenticationProvider daoAuthenticationProvider(){
    DaoAuthenticationProvider daoAuthenticationProvider = new DaoAuthenticationProvider();
    daoAuthenticationProvider.setPasswordEncoder(new BCryptPasswordEncoder());
    daoAuthenticationProvider.setUserDetailsService(userServiceDetails);
    return daoAuthenticationProvider;
}
```

#### 2.2 WebSecurityConfigurerAdapter 配置

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    UserDetailsService userServiceDetails;

    @Autowired
    InMemoryAuthenticationProvider inMemoryAuthenticationProvider;

    @Bean
    DaoAuthenticationProvider daoAuthenticationProvider(){
        DaoAuthenticationProvider daoAuthenticationProvider = new DaoAuthenticationProvider();
        daoAuthenticationProvider.setPasswordEncoder(new BCryptPasswordEncoder());
        daoAuthenticationProvider.setUserDetailsService(userServiceDetails);
        return daoAuthenticationProvider;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .rememberMe().alwaysRemember(true).tokenValiditySeconds(86400).and()
            .authorizeRequests()
            .antMatchers("/","/*swagger*/**", "/v2/api-docs").permitAll()
            .anyRequest().authenticated().and()
            .formLogin()
            .loginPage("/")
            .loginProcessingUrl("/login")
            .successHandler(new AjaxLoginSuccessHandler())
            .failureHandler(new AjaxLoginFailureHandler()).and()
            .logout().logoutUrl("/logout").logoutSuccessUrl("/");
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/public/**", "/webjars/**", "/v2/**", "/swagger**");
    }

    @Override
    protected AuthenticationManager authenticationManager() throws Exception {
        ProviderManager authenticationManager = new ProviderManager(Arrays.asList(inMemoryAuthenticationProvider,daoAuthenticationProvider()));
        //不擦除认证密码，擦除会导致TokenBasedRememberMeServices因为找不到Credentials再调用UserDetailsService而抛出UsernameNotFoundException
        authenticationManager.setEraseCredentialsAfterAuthentication(false);
        return authenticationManager;
    }

    /**
   * 这里需要提供UserDetailsService的原因是RememberMeServices需要用到
   * @return
   */
    @Override
    protected UserDetailsService userDetailsService() {
        return userServiceDetails;
    }
}
```

### 3 Security 中的 3 种权限控制方法

- 表达式控制 URL 路径权限
- 表达式控制方法权限
- 使用过滤注解
- 动态权限

#### 3.1 配置里控制 URL 路径权限

```java
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .antMatchers("/admin/**").hasRole("admin")
            .antMatchers("/user/**").hasAnyRole("admin", "user")
            .anyRequest().authenticated()
            .and()
            ...
}
```

#### 3.2 注解控制权限

先开启注解使用

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true,securedEnabled = true,jsr250Enabled=true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    ...
    ...
}
```

##### JSR-250 注解

主要注解

- @DenyAll
- @RolesAllowed
- @PermitAll

```java
@RolesAllowed({"USER", "ADMIN"})
//代表标注的方法只要具有USER, ADMIN任意一种权限就可以访问。这里可以省略前缀ROLE_，实际的权限可能是ROLE_ADMIN
```

##### securedEnabled 注解

主要注解

@Secured @Secured 在方法上指定安全性要求

可以使用@Secured 在方法上指定安全性要求 角色/权限等 只有对应 角色/权限 的用户才可以调用这些方法。 如果有人试图调用一个方法，但是不拥有所需的 角色/权限，那会将会拒绝访问将引发异常。

@Secured 使用示例

```java
@Secured("ROLE_ADMIN")
@Secured({ "ROLE_DBA", "ROLE_ADMIN" })
```

##### prePostEnabled 注解

这个开启后支持 Spring EL 表达式 算是蛮厉害的。如果没有访问方法的权限，会抛出`AccessDeniedException`。

主要注解

@PreAuthorize --适合进入方法之前验证授权

@PostAuthorize --检查授权方法之后才被执行

@PostFilter --在方法执行之后执行，而且这里可以调用方法的返回值，然后对返回值进行过滤或处理或修改并返回

@PreFilter --在方法执行之前执行，而且这里可以调用方法的参数，然后对参数值进行过滤或处理或修改

```java
@Service
public class HelloService {
    @PreAuthorize("principal.username.equals('javaboy')")
    public String hello() {
        return "hello";
    }

    @PreAuthorize("hasRole('admin')")
    public String admin() {
        return "admin";
    }

    @Secured({"ROLE_user"})
    public String user() {
        return "user";
    }

    @PreAuthorize("#age>98")
    public String getAge(Integer age) {
        return String.valueOf(age);
    }
}
```

Spring Security 中还有两个过滤函数 @PreFilter 和 @PostFilter，可以根据给出的条件，自动移除集合中的元素

```java
@PostFilter("filterObject.lastIndexOf('2')!=-1")
public List<String> getAllUser() {
    List<String> users = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        users.add("javaboy:" + i);
    }
    return users;
}
@PreFilter(filterTarget = "ages",value = "filterObject%2==0")
public void getAllAge(List<Integer> ages,List<String> users) {
    System.out.println("ages = " + ages);
    System.out.println("users = " + users);
}
```

##### 自定义匹配器

另外说一点就是`@PreAuthorize` 和 `@PostAuthorize` 中 除了支持原有的权限表达式之外，也是可以支持自定义的

定义一个自己的权限匹配器

```java
interface TestPermissionEvaluator {
    boolean check(Authentication authentication);
}

@Service("testPermissionEvaluator")
public class TestPermissionEvaluatorImpl implements TestPermissionEvaluator {

    public boolean check(Authentication authentication) {
        System.out.println("进入了自定义的匹配器" + authentication);
        return false;
    }
}
```

返回 true 就是有权限 false 则是无权限

```java
@PreAuthorize("@testPermissionEvaluator.check(authentication)")
public String test0() {
	return "说明你有自定义权限";
}
```

##### 异常处理

```java
@Component
@Slf4j
public class AccessDeniedAuthenticationHandler implements AccessDeniedHandler {
    private final ObjectMapper objectMapper;

    public AccessDeniedAuthenticationHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AccessDeniedException e) throws IOException {
        log.info("没有权限");
        httpServletResponse.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        httpServletResponse.setContentType("application/json;charset=UTF-8");
        httpServletResponse.getWriter().write(objectMapper.writeValueAsString(e.getMessage()));
    }

}

```

然后在`WebSecurityConfig` 中的 `configure` 中配置即可

```java
http.anyRequest()
    .authenticated()
    .and().exceptionHandling()
    .accessDeniedHandler(accessDeniedAuthenticationHandler);
```

#### 3.3 动态权限

就是可以让用户随时修改不同模块的权限

##### 3.3.1 自定义 filter

```java
@Service
public class MyFilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {

    @Autowired
    private FilterInvocationSecurityMetadataSource securityMetadataSource;

    @Autowired
    public void setMyAccessDecisionManager(MyAccessDecisionManager myAccessDecisionManager) {
        super.setAccessDecisionManager(myAccessDecisionManager);
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

        FilterInvocation fi = new FilterInvocation(request, response, chain);
        invoke(fi);
    }

    public void invoke(FilterInvocation fi) throws IOException, ServletException {
        //fi里面有一个被拦截的url
        //里面调用MyInvocationSecurityMetadataSource的getAttributes(Object object)这个方法获取fi对应的所有权限
        //再调用MyAccessDecisionManager的decide方法来校验用户的权限是否足够
        InterceptorStatusToken token = super.beforeInvocation(fi);
        try {
            //执行下一个拦截器
            fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
        } finally {
            super.afterInvocation(token, null);
        }
    }

    @Override
    public void destroy() {

    }

    @Override
    public Class<?> getSecureObjectClass() {
        return FilterInvocation.class;
    }

    @Override
    public SecurityMetadataSource obtainSecurityMetadataSource() {
        return this.securityMetadataSource;
    }
}
```

##### 3.3.2 自定义 MetadataSource

```java
@Service
public class MyInvocationSecurityMetadataSourceService  implements
        FilterInvocationSecurityMetadataSource {

    @Autowired
    private PermissionDao permissionDao;

    private HashMap<String, Collection<ConfigAttribute>> map =null;

    /**
     * 加载权限表中所有权限
     */
    public void loadResourceDefine(){
        map = new HashMap<>();
        Collection<ConfigAttribute> array;
        ConfigAttribute cfg;
        List<Permission> permissions = permissionDao.findAll();
        for(Permission permission : permissions) {
            array = new ArrayList<>();
            cfg = new SecurityConfig(permission.getName());
            //此处只添加了用户的名字，其实还可以添加更多权限的信息
            //例如请求方法到ConfigAttribute的集合中去。此处添加的信息将会作为MyAccessDecisionManager类的decide的第三个参数。
            array.add(cfg);
            //用权限的getUrl() 作为map的key，用ConfigAttribute的集合作为 value，
            map.put(permission.getUrl(), array);
        }

    }

	//此方法是为了判定用户请求的url 是否在权限表中，如果在权限表中，则返回给 decide 方法，用来判定用户是否有此权限。如果不在权限表中则放行。
    @Override
    public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
        if(map ==null) loadResourceDefine();
        //object 中包含用户请求的request 信息
        HttpServletRequest request = ((FilterInvocation) object).getHttpRequest();
        AntPathRequestMatcher matcher;
        String resUrl;
        for(Iterator<String> iter = map.keySet().iterator(); iter.hasNext(); ) {
            resUrl = iter.next();
            matcher = new AntPathRequestMatcher(resUrl);
            if(matcher.matches(request)) {
                return map.get(resUrl);
            }
        }
        return null;
    }

    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return true;
    }
}
```

##### 3.3.3 自定义 DecisionManager

```java
@Service
public class MyAccessDecisionManager implements AccessDecisionManager {

 	// decide 方法是判定是否拥有权限的决策方法，
    //authentication 是释CustomUserService中循环添加到 GrantedAuthority 对象中的权限信息集合.
    //object 包含客户端发起的请求的requset信息，可转换为 HttpServletRequest request = ((FilterInvocation) object).getHttpRequest();
    //configAttributes 为MyInvocationSecurityMetadataSource的getAttributes(Object object)这个方法返回的结果
    //此方法是为了判定用户请求的url 是否在权限表中，如果在权限表中，则返回给 decide 方法，用来判定用户是否有此权限。如果不在权限表中则放行。
    @Override
    public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException {

        if(null== configAttributes || configAttributes.size() <=0) {
            return;
        }
        ConfigAttribute c;
        String needRole;
        for(Iterator<ConfigAttribute> iter = configAttributes.iterator(); iter.hasNext(); ) {
            c = iter.next();
            needRole = c.getAttribute();
            for(GrantedAuthority ga : authentication.getAuthorities()) {//authentication 为在注释1 中循环添加到 GrantedAuthority 对象中的权限信息集合
                if(needRole.trim().equals(ga.getAuthority())) {
                    return;
                }
            }
        }
        throw new AccessDeniedException("no right");
    }

    @Override
    public boolean supports(ConfigAttribute attribute) {
        return true;
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return true;
    }
}
```

##### 3.3.4 修改 configure

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .anyRequest().authenticated() //任何请求,登录后可以访问
        .and()
        .formLogin()
        .loginPage("/login")
        .failureUrl("/login?error")
        .permitAll() //登录页面用户任意访问
        .and()
        .logout().permitAll(); //注销行为任意访问
    http.addFilterBefore(myFilterSecurityInterceptor, FilterSecurityInterceptor.class);
}
```

{% endraw %}
