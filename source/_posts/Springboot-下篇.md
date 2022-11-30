---
title: Springboot(下篇)
date: 2022-11-30 19:00:46
tags:
---
# Springboot底层原理（2）

[Spring Framework Documentation](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/)

## 使用原生的Servlet

### 使用注解声明为Servlet组件

```java
@WebServlet("/my")
public class MyServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("1212");
    }
}
```

可以在Spring中使用原生的Servlet组件，重写里面的doGet，doPost等方法实现具体的逻辑，并加上@WebServlet("/my")添加路由映射，但是只是这样还不能生效，因为它并不是Spring框架下的组件，所以需要在启动类上加上@ServletComponentScan(basePackages = "com.demo")设置包扫描路径，用于扫描原生的Servlet组件。

```java
@ServletComponentScan(basePackages = "com.demo")
@SpringBootApplication
public class MydemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(MydemoApplication.class, args);
    }
}
```

因为不是Spring框架下的组件，所以Spring注册的拦截器不会生效，想要进行拦截需要使用Servlet组件中的拦截器：

```java
@WebFilter(urlPatterns = "/*")
@Slf4j
public class MyFilter implements Filter {

    //Spring容器启动的时候执行
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        log.info("Filter init");
    }
	//如果路由和设置的路由匹配，则先执行这个过滤器，然后再执行具体的业务逻辑
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("do Filter");
    }
	//Spring容器销毁（也就是Servlet销毁时）执行的方法
    @Override
    public void destroy() {
        log.info("destroy");
    }
}
```

@WebFilter(urlPatterns = "/*") 注意路由的写法，Spring组件中的url是`/**` 而Servlet组件的写法是`/*`

监听器：

在项目初始化完成，开始监听之前可以执行contextInitialized方法，项目关闭的时候会执行contextDestroyed方法

```java
@WebListener
public class MyListener implements ServletContextListener {
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("项目结束");
    }

    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("检测到初始化完成");
    }
}
```

![image-20220507193721185](pictures/4855e5cc40ab39b4cc76d71effdace90.png)

### 向Spring容器中添加Servlet组件

```java
@Configuration
public class MyServletConfiger {
    //注册Servlet
    @Bean
    public ServletRegistrationBean myRegistrationBean(){
        return new ServletRegistrationBean(new MyServlet(),"/my","/my1");
    }
    //注册过滤器
    @Bean
    public FilterRegistrationBean filterRegistrationBean(){
        return new FilterRegistrationBean(new MyFilter(),myRegistrationBean());
    }
    //注册监听器
    @Bean
    public ServletListenerRegistrationBean listenerRegistrationBean(){
        return new ServletListenerRegistrationBean(new MyListener());
    }
}
```

注册过滤器也可以使用：

```java
    @Bean
    public FilterRegistrationBean filterRegistrationBean(){
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
        filterRegistrationBean.setFilter(new MyFilter());
        filterRegistrationBean.setUrlPatterns(Arrays.asList("/*","/css/*"));
        return filterRegistrationBean;
    }
```

注意：

这里的@Configuration注解不能将proxyBeanMethods 属性设置为 false，我们前面提到过，如果将这个属性设置为true，在调用里面带有@Bean的方法时，会在Spring容器中找有没有相同的bean，如果有就返回Spring容器中的bean，如果没有会创建一个bean。而设置为false后，会不会生产代理对象，因而会生成很多多余的bean。所以这里需要将proxyBeanMethods 设置为true，也就是它的默认值，来保证依赖的组件始终的单实例的。

### 原生的Servlet的作用原理

前面提到使用原生的Servlet不会触发Spring的拦截器，下面解释这个的原因。

Springboot Web处理请求的核心是DispatcherServlet类，而这个Servlet是在DispatcherServletAutoConfiguration这个自动配置类中注册进Spring容器中的。

```java
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
			DispatcherServlet dispatcherServlet = new DispatcherServlet();
			dispatcherServlet.setDispatchOptionsRequest(webMvcProperties.isDispatchOptionsRequest());
			dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
			dispatcherServlet.setThrowExceptionIfNoHandlerFound(webMvcProperties.isThrowExceptionIfNoHandlerFound());
			dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
			dispatcherServlet.setEnableLoggingRequestDetails(webMvcProperties.isLogRequestDetails());
			return dispatcherServlet;
		}
```

之前介绍介绍的很多组件，比如各种解析器都是在这个类中注册进Spring容器中的

其中的参数：WebMvcProperties webMvcProperties，对应配置文件中spring.mvc下的配置项

```
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
```

然后通过dispatcherServletRegistration这个方法将DispatcherServlet注册进Servlet框架中

```java
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
				WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
			DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
					webMvcProperties.getServlet().getPath());
			registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
			registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
			multipartConfig.ifAvailable(registration::setMultipartConfig);
			return registration;
		}
```

既然是Servlet，就有需要由它来处理的URL

```java
			DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
					webMvcProperties.getServlet().getPath());
```

通过这个方法向服务器中添加Servlet，而它的请求路径是webMvcProperties.getServlet().getPath())，而这个方法的 值就是我们配置的spring.mvc.servlet.path，这个值默认是`/`，也就默认情况下，所有请求都由dispatcherServlet来处理（也就是由Springboot的Web框架来处理）

所以我们用Spring处理请求的时候，实际上用的是一个Servlet：DispatcherServlet，在这个Servlet中处理所有的请求。

tomcat在一个请求有多个Servlet可以处理时，使用精确优先原则，它会在所有能处理的Servlet中，选择前缀匹配程度最长的Servlet进行处理。

例如

如果有两个Servlet，A对应路由`/my`，B对应路由`/my/1`，此时如果收到了`/my/1/2`的请求，则会交给B来处理，而如果收到`/my/2`的请求，则会由A来处理。

我们自定义的原生Servlet组件和Spring的DispatcherServlet也是上述这种关系。DispatcherServlet默认处理的URL是`/`也就是所有的请求，而我们自定义的Servlet对应的URL是`/my/`，所以我们发送/my请求后，根据精确匹配原则会交付给我们自定义的MyServlet，由Tomcat直接来处理，而如果不是/my/开头的请求，就会和DispatcherServlet匹配，然后走Spring的流程后再交给Tomcat来处理。

![image-20220507224859818](pictures/38456e2bfda4fea76779e78a77c816e1.png)

所以我们发送的/my请求没有被Spring拦截的原因就是它是由我们定义的MyServlet处理的，而不是由Spring里的DispatcherServlet来处理，自然不会触发DispatcherServlet中定义的拦截器。

## Spring嵌入式Servlet容器

### 底层原理

Springboot如果发现当前是Web应用，就会自动导入Tomcat服务器所需的依赖，并且会创建一个Web类型的IOC容器ServletWebServerApplicationContext

ServletWebServerApplicationContext 启动的时候需要用到 ServletWebServerFactory 来创建服务器（Servlet 的web服务器工厂——>Servlet 的web服务器）。而SpringBoot底层默认有很多的WebServer工厂（ServletWebServerFactoryConfiguration内创建Bean），如：TomcatServletWebServerFactory，JettyServletWebServerFactory，UndertowServletWebServerFactory，对应三种不同的服务器（Tomcat，Jetty，Undertow）。这几个服务器工厂是在ServletWebServerFactoryAutoConfiguration这个自动配置类中放入Spring容器的，而这个自动配置需要使用使用ServletWebServerFactoryConfiguration这个配置类。

```java
@Configuration(proxyBeanMethods = false)
class ServletWebServerFactoryConfiguration {

	@Configuration(proxyBeanMethods = false)
    //需要tomcat依赖才会放入TomcatServletWebServerFactory
	@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedTomcat {

		@Bean
		TomcatServletWebServerFactory tomcatServletWebServerFactory(
				ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
				ObjectProvider<TomcatContextCustomizer> contextCustomizers,
				ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
			TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
			factory.getTomcatConnectorCustomizers()
					.addAll(connectorCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getTomcatContextCustomizers()
					.addAll(contextCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getTomcatProtocolHandlerCustomizers()
					.addAll(protocolHandlerCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

	}

	/**
	 * Nested configuration if Jetty is being used.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Server.class, Loader.class, WebAppContext.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedJetty {

		@Bean
		JettyServletWebServerFactory JettyServletWebServerFactory(
				ObjectProvider<JettyServerCustomizer> serverCustomizers) {
			JettyServletWebServerFactory factory = new JettyServletWebServerFactory();
			factory.getServerCustomizers().addAll(serverCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

	}

	/**
	 * Nested configuration if Undertow is being used.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedUndertow {

		@Bean
		UndertowServletWebServerFactory undertowServletWebServerFactory(
				ObjectProvider<UndertowDeploymentInfoCustomizer> deploymentInfoCustomizers,
				ObjectProvider<UndertowBuilderCustomizer> builderCustomizers) {
			UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
			factory.getDeploymentInfoCustomizers()
					.addAll(deploymentInfoCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getBuilderCustomizers().addAll(builderCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

		@Bean
		UndertowServletWebServerFactoryCustomizer undertowServletWebServerFactoryCustomizer(
				ServerProperties serverProperties) {
			return new UndertowServletWebServerFactoryCustomizer(serverProperties);
		}
	}
}
```

可以看到这个配置类用于向Spring容器中添加三种服务器工厂，利用条件装配判断放入哪些服务器工厂，只有在导入了所依赖的jar包后，相关的配置才能生效。

```java
//导入tomcat依赖才会放入TomcatServletWebServerFactory
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
//导入Jetty依赖才会引入JettyServletWebServerFactory
@ConditionalOnClass({ Servlet.class, Server.class, Loader.class, WebAppContext.class })
//导入Undertow的依赖才会放入UndertowServletWebServerFactory
@ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
```

而我们在pom文件导入的spring-boot-starter-web依赖会默认导入tomcat的依赖，所以默认会放入导入tomcat依赖才会放入TomcatServletWebServerFactory这个服务器工厂，得到Tomcat服务器

```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

三种服务器工厂的都是ServletWebServerFactory的子类，在查找服务器工厂时会从Spring容器中拿到所有ServletWebServerFactory类型的bean，如果数量是0个或者多个都会抛出异常，因而Spring容器中只能有一个服务器工厂（默认是Tomcat）

```java
	protected ServletWebServerFactory getWebServerFactory() {
		// Use bean names so that we don't consider the hierarchy
		String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
		if (beanNames.length == 0) {
			throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to missing "
					+ "ServletWebServerFactory bean.");
		}
		if (beanNames.length > 1) {
			throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to multiple "
					+ "ServletWebServerFactory beans : " + StringUtils.arrayToCommaDelimitedString(beanNames));
		}
		return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
	}
```

Spring容器启动的时候会调用ServletWebServerApplicationContext类的onRefresh方法

```java
	@Override
	protected void onRefresh() {
		super.onRefresh();
		try {
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
	}
```

在这个方法中调用createWebServer()方法创建服务器

```java
	private void createWebServer() {
		WebServer webServer = this.webServer;
        //尝试获取IOC容器，默认是空
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
            //从Spring容器中获取服务器工厂，如果有0个或者多个会抛出异常，默认是Tomcat
			ServletWebServerFactory factory = getWebServerFactory();
            //使用服务器工厂创建服务器
			this.webServer = factory.getWebServer(getSelfInitializer());
			getBeanFactory().registerSingleton("webServerGracefulShutdown",
					new WebServerGracefulShutdownLifecycle(this.webServer));
			getBeanFactory().registerSingleton("webServerStartStop",
					new WebServerStartStopLifecycle(this, this.webServer));
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
		initPropertySources();
	}
```

创建服务器的方法getWebServer：

```java
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
   if (this.disableMBeanRegistry) {
      Registry.disableRegistry();
   }
   //获取一个tomcat服务器对象
   Tomcat tomcat = new Tomcat();
   //下面是配置tomcat的一些参数
   File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
   tomcat.setBaseDir(baseDir.getAbsolutePath());
   Connector connector = new Connector(this.protocol);
   connector.setThrowOnFailure(true);
   tomcat.getService().addConnector(connector);
   customizeConnector(connector);
   tomcat.setConnector(connector);
   tomcat.getHost().setAutoDeploy(false);
   configureEngine(tomcat.getEngine());
   for (Connector additionalConnector : this.additionalTomcatConnectors) {
      tomcat.getService().addConnector(additionalConnector);
   }
   prepareContext(tomcat.getHost(), initializers);
   return getTomcatWebServer(tomcat);
}
```

所以实际上内嵌服务器就是调用封装好的服务器对象，以前启动Tomcat服务器的时候，是以服务器为顶层调用SpringMVC的逻辑，而在调用之前也会设置这些参数。而Springboot内嵌的Tomcat服务器则是以Springboot为顶层，调用Tomcat对象。如下图所示，tomcat对象中有main方法可以直接运行。

![image-20220508004519983](pictures/7586ecf7941b2ea8d5e4867c9eeb1414.png)

通过tomcat服务器对象会得到一个WebServer对象来操作Tomcat服务器

```java
public interface WebServer {
	//启动服务器
	void start() throws WebServerException;
	//关闭服务器
	void stop() throws WebServerException;
	//获得监听的端口
	int getPort();

	default void shutDownGracefully(GracefulShutdownCallback callback) {
		callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
	}

}
```

创建TomcatWebServer时，会在构造器中调用initialize()方法，这个方法中会调用this.tomcat.start()来启动服务器

```java
	public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
		Assert.notNull(tomcat, "Tomcat Server must not be null");
		this.tomcat = tomcat;
		this.autoStart = autoStart;
		this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL) ? new GracefulShutdown(tomcat) : null;
		initialize();
	}
```

### 切换服务器（一般使用Tomcat即可）

如果想要切换服务器的类型，我们只需要将tomcat服务器的依赖排除，然后导入我们需要的服务器的依赖即可，然后根据上面所说的自动装配原理就会自动帮我们向Spring容器中添加对应的服务器工厂。

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- 排除tomcat依赖 -->
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
		<!-- 引入undertow依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
        </dependency>
```

根据我们之前的分析，Spring容器中只能有一个服务器工厂，所以需要排除tomcat依赖，防止Spring将tomcat的服务器工厂注册进Spring容器中

### 定制服务器

1.修改配置文件

```java
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration
```

ServletWebServerFactoryAutoConfiguration这个自动配置类需要使用ServerProperties这个类

```
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties 
```

这个类和以server开头的配置项绑定在一起，所以配置项在server开头的配置项下

2.直接向Spring容器中添加一个我们定制的服务器工厂

3.可以实现一个定制化器：

```java
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.stereotype.Component;

@Component
public class CustomizationBean implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {

    @Override
    public void customize(ConfigurableServletWebServerFactory server) {
        server.setPort(9000);
    }
}
```

### 定制化原理

根据前面的总结，我们可以得到Spring配置的原理

导入场景的starter包-->相关的AutoConfigration自动配置生效-->自动配置类会引入对应的Properties配置类-->配置类会绑定配置文件的参数

所以一般情况下，我们想要修改Springbooot的功能只需要导入对应场景的包，然后修改配置文件即可

总结起来，常用的定制化方式有：

1.修改配置文件

2.@Confugration+@Bean注解根据Springboot的执行逻辑添加组件

3.xxxCustomizer

4.高级配置：修改Springboot的底层组件，比如RequestMappingHandlerMapping，可以通过以下方式来实现

```java
    @Bean
    public WebMvcRegistrations registrations(){
        return new WebMvcRegistrations() {
            @Override
            public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
                return WebMvcRegistrations.super.getRequestMappingHandlerMapping();
            }
        };
    }
```

5.高级配置：全面接管SpringMVC：@EnableWebMvc+WebMvcConfigurer,加上这个注解后，Springboot一些相关的自动配置就会失效，需要我们进行手动配置。

如果我们不加@EnableWebMvc这个注解，则会在原先配置的基础上添加（修改）成我们需要的配置，如果我们注册了多个WebMvcConfigurer类型的组件，Springboot会让所有的WebMvcConfigurer生效，这个过程发生在DelegatingWebMvcConfiguration类中：

```java
	@Autowired(required = false)
	public void setConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.configurers.addWebMvcConfigurers(configurers);
		}
	}
```

Tips：@Autowired作用在普通方法上，会在注入的时候调用一次该方法，如果方法中有实体参数，会对方法里面的参数进行装配，并调用一次该方法。这个可以用来在自动注入的时候做一些初始化操作。

DelegatingWebMvcConfiguration这个类保证了SpringMVC最基本的使用（即使我们进行了全面接管，但是一些底层的一定要有的组件还是会放入Spring容器）

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
```

SpringMVC的自动装配原理集中在WebMvcAutoConfiguration这个配置类中，而这个配置类生效的条件之一是@ConditionalOnMissingBean(WebMvcConfigurationSupport.class) 也就是Spring容器中不能有WebMvcConfigurationSupport类型的组件，否则自动配置就不会生效。

而@EnableWebMvc注解的定义如下：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

因而加上这个注解后会自动帮我们导入DelegatingWebMvcConfiguration这个类的一个组件，而这个类是WebMvcConfigurationSupport这个类的子类，所以会导致自动配置类失效（也同时提醒我们不要往Spring容器中添加功能时不要继承WebMvcConfigurationSupport，而应该用WebMvcConfigurer），所以DelegatingWebMvcConfiguration在WebMvcAutoConfiguration生效前，默认是不在Spring容器中的，会在我们全面接管SpringMvc的时候提供一些基础的功能，而在WebMvcAutoConfiguration里面继承了DelegatingWebMvcConfiguration实现了更多的功能，并保留了让所有WebMvcConfigurer生效的方法，所以无论是全面接管SpringMVC还是使用默认配置，容器启动的时候会让所有的WebMvcConfigurer生效

```
@Configuration(proxyBeanMethods = false)
	public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
```

## 数据操作

### 依赖引入

使用jdbc操作数据库：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jdbc</artifactId>
        </dependency>
```

![image-20220508140938283](pictures/9d915490f9a0ab4ef17c92d2c88236bc.png)

spring-boot-starter-data-jdbc中为我们整合了数据库连接池，jdbc编程和数据库事务，但是没有数据库连接驱动，这是因为Spring并不知道我们要使用哪种数据库，因而只导入了通用的依赖

引入连接器依赖：

```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
        </dependency>
```

Spring会帮我们进行版本仲裁，但是默认的版本是最新的数据库的版本，也就是8.0以上的版本。实际上这里的数据库连接器的配置应当与本地数据库的版本相匹配，如果本地数据库是5.x的数据库就不要用8.0.x的连接器，而应该用5.x的连接器

修改版本方法：

1.直接引入具体版本（maven的就近依赖原则，优先使用我们设置的版本）

2.修改properties，也就修改了Spring默认配置的数据库版本（属性就近优先原则，优先使用我们配置的属性）

```xml
    <properties>
        <java.version>1.8</java.version>
        <mysql.version>8.0.11</mysql.version>
    </properties>
```

### 自动配置

#### DataSourceAutoConfiguration

自动配置数据源和连接池（默认使用HikariDataSource连接池）

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {
```

`@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")` 如果没有使用响应式编程框架则自动配置这个类

`@EnableConfigurationProperties(DataSourceProperties.class)`绑定配置类DataSourceProperties

DataSourceProperties绑定的配置为：spring.datasource下的所有配置

例如数据库的账号，密码，URL等信息都会绑定到这个配置类中

```
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
```

如果我们没有配置数据库连接池，Spring会帮我们配置一个数据库连接池：

```java
	@Configuration(proxyBeanMethods = false)
	@Conditional(PooledDataSourceCondition.class)
	//如果没有配置数据库连接池，这个类才会生效
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	//引入数据库连接池相关的依赖
	@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
			DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
			DataSourceJmxConfiguration.class })
	protected static class PooledDataSourceConfiguration {

	}
```

而数据库连接池是如何创建的，我们可以来到DataSourceConfiguration配置类：

在有相关的依赖的时候这个类才会生效，然后才会创建HikariDataSource的数据源（其他的还有Tomcat数据源等，但是默认是HikariDataSource数据源）

```java
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}

	}
```

数据源配置（Mysql8.0以上）：

```yml
spring:
  datasource:
    name: document
    url: jdbc:mysql://localhost:3306/document?characterEncoding=utf8&useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true&useJDBCCompliantTimezoneShift=true
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
#    type: com.zaxxer.hikari.HikariDataSource #默认是HikariDataSource数据库连接池
```

数据源配置（Mysql5.x）：

```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/document
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
```

#### DataSourceTransactionManagerAutoConfiguration

事务管理器自动配置

#### JdbcTemplateAutoConfiguration

自动配置JdbcTemplate，可以用于增删改查

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })
@ConditionalOnSingleCandidate(DataSource.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@EnableConfigurationProperties(JdbcProperties.class)
@Import({ JdbcTemplateConfiguration.class, NamedParameterJdbcTemplateConfiguration.class })
public class JdbcTemplateAutoConfiguration {

}
```

其中@EnableConfigurationProperties(JdbcProperties.class)代表与JdbcProperties类绑定，而这个类与@ConfigurationProperties(prefix = "spring.jdbc")绑定，也就是可以通过修改spring.jdbc下面的配置来配置JdbcTemplate的功能

```yml
spring:
  jdbc:
    template:
      query-timeout: 3
```

#### 扩展

JndiDataSourceAutoConfiguration

JDNI自动配置

XADataSourceAutoConfiguration

分布式事务自动配置

### 整合Druid数据源

HikariDataSource是目前市面上性能最好的数据源，而Druid对性能监控，防止sql注入攻击有整套的解决方案

```java
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.17</version>
        </dependency>
```

配置HikariDataSource的代码块：

```java
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}

	}
```

@ConditionalOnMissingBean(DataSource.class)表示如果Spring容器中没有DataSource数据源来回帮我们配置HikariDataSource数据源，如果我们配置了DataSource就用我们自己的数据源。向Spring容器添加我们自己的数据源即可。

方式一：用户名密码直接在配置类中设置用户名密码

```java
@Configuration
public class DataSourceConfig {
    @Bean
    DataSource druidDataSource(){
        DruidDataSource druidDataSource=new DruidDataSource();
        druidDataSource.setUrl();
        druidDataSource.setUsername();
        druidDataSource.setPassword();
        return druidDataSource;
    }
}
```

但是这样不方便修改，所以我们可以使用配置文件中配置的参数

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties("spring.datasource")
    DataSource druidDataSource(){
        DruidDataSource druidDataSource=new DruidDataSource();
        return druidDataSource;
    }
}
```

@ConfigurationProperties("spring.datasource") 这个注解我们在研究源码的时候看了很多回了，用于将返回值中对应的名称的参数和配置文件中对应的名称的参数绑定在一起。

Tips：Spring中的测试环节可以直接在Test目录下进行，这样就不用使用postman发请求了

```java
@SpringBootTest
class MydemoApplicationTests {

    @Resource
    JdbcTemplate jdbcTemplate;
    @Test
    void contextLoads() {
        List<Usert> userts = jdbcTemplate.query("select * from usert",new BeanPropertyRowMapper<>(Usert.class));
        userts.forEach((System.out::println));
    }

}
```

### Druid数据监控

#### 监控SQL

整合Druid数据源后，我们就可以通过配置Druid监控页来监控数据库的状态

想要达成监控功能就需要配置一个给Druid使用的Servlet

```java
    @Bean
    public ServletRegistrationBean servletRegistrationBean(){
        return new ServletRegistrationBean<>(new StatViewServlet(),"/druid/*");
    }
```

这样/druid/*的请求就会交给Druid中的StatViewServlet来处理，而不会走Spring的流程，如下图所示，获得成功

![image-20220508171446887](pictures/22fcaaf1be52e5d0800f4db8995570bf.png)

但是这样只能显示界面，要统计SQL语句执行的各种信息还需要在配置数据源时加上druidDataSource.setFilters("stat");

```java
    @SneakyThrows
    @Bean
    @ConfigurationProperties("spring.datasource")
    DataSource druidDataSource(){
        DruidDataSource druidDataSource=new DruidDataSource();
        druidDataSource.setFilters("stat");
        return druidDataSource;
    }
```

#### 监控请求

配置这个后监控页的URI请求就有数据来源

```java
    @Bean
    public FilterRegistrationBean webStatFilter(){
        WebStatFilter webStatFilter = new WebStatFilter();

        FilterRegistrationBean<WebStatFilter> filterRegistrationBean = new FilterRegistrationBean<>(webStatFilter);
        filterRegistrationBean.setUrlPatterns(Arrays.asList("/*"));
        filterRegistrationBean.addInitParameter("exclusions","*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");

        return filterRegistrationBean;
    }
```

![image-20220508175213844](pictures/3be6aa060673577be3bf992760f16089.png)

![image-20220508211828211](pictures/65a129f1a71eac87e2c80c5fce3facc1.png)

#### 开启防火墙

```java
    @SneakyThrows
    @Bean
    @ConfigurationProperties("spring.datasource")
    DataSource druidDataSource(){
        DruidDataSource druidDataSource=new DruidDataSource();
        druidDataSource.setFilters("stat,wall");
        return druidDataSource;
    }
```

而我们上面也用到过，在@ConfigurationProperties("spring.datasource")注解下的方法中，使用set方法配置的属性，在配置文件中配置同样有效：

```yml
  datasource:
    name: document
    url: jdbc:mysql://localhost:3306/document?characterEncoding=utf8&useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true&useJDBCCompliantTimezoneShift=true
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
    Filters: stat,wall
```

stat代表状态监控

wall代表防火墙

不过Filters会变黄，因为这个并不是Spring的配置

XML配置->配置类配置：看到bean标签就向Spring容器中通过@Bean注解添加一个bean，下面的其他标签就只是它的属性值

https://github.com/alibaba/druid/wiki/%E9%85%8D%E7%BD%AE_StatViewServlet%E9%85%8D%E7%BD%AE

#### 设置访问的账号和密码

```java
    @Bean
	//用于设置监控页的访问路径
    public ServletRegistrationBean servletRegistrationBean(){
        ServletRegistrationBean<StatViewServlet> registrationBean = new ServletRegistrationBean<>(new StatViewServlet(), "/druid/*");
        //监控页账号密码：
        registrationBean.addInitParameter("loginUsername","admin");
        registrationBean.addInitParameter("loginPassword","123456");
        return registrationBean;
    }
```

完整配置：

```java
package com.demo.config;

import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.support.http.StatViewServlet;
import com.alibaba.druid.support.http.WebStatFilter;
import lombok.SneakyThrows;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;
import java.util.Arrays;

/**
 * @author 李天航
 */
@Configuration
public class DataSourceConfig {
    @SneakyThrows
    @Bean
    @ConfigurationProperties("spring.datasource")
    DataSource druidDataSource(){
        DruidDataSource druidDataSource=new DruidDataSource();
        druidDataSource.setFilters("stat,wall");
        return druidDataSource;
    }

    @Bean
    public ServletRegistrationBean servletRegistrationBean(){
        ServletRegistrationBean<StatViewServlet> registrationBean = new ServletRegistrationBean<>(new StatViewServlet(), "/druid/*");
        //监控页账号密码：
        registrationBean.addInitParameter("loginUsername","admin");
        registrationBean.addInitParameter("loginPassword","123456");
        return registrationBean;
    }

    @Bean
    public FilterRegistrationBean webStatFilter(){
        WebStatFilter webStatFilter = new WebStatFilter();

        FilterRegistrationBean<WebStatFilter> filterRegistrationBean = new FilterRegistrationBean<>(webStatFilter);
        filterRegistrationBean.setUrlPatterns(Arrays.asList("/*"));
        filterRegistrationBean.addInitParameter("exclusions","*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");

        return filterRegistrationBean;
    }
}
```



### Druid Starter配置连接池

上述的配置过程显得过去麻烦了，如果有一个自动配置类能像其他组件一样自动帮我们把上述组件配置好，然后用一个配置类绑定配置文件，然后我们直接修改配置文件就会方便很多，这个starter就是druid-spring-boot-starter

```xml
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.17</version>
        </dependency>
```

我们来看一下starter源码：

```java
@Configuration
//必须导入DruidDataSource的依赖
@ConditionalOnClass(DruidDataSource.class)
//必须在DataSourceAutoConfiguration之前配置
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
@EnableConfigurationProperties({DruidStatProperties.class, DataSourceProperties.class})
//引入下面四种依赖
@Import({DruidSpringAopConfiguration.class,
    DruidStatViewServletConfiguration.class,
    DruidWebStatFilterConfiguration.class,
    DruidFilterConfiguration.class})
public class DruidDataSourceAutoConfigure {

    private static final Logger LOGGER = LoggerFactory.getLogger(DruidDataSourceAutoConfigure.class);

    @Bean(initMethod = "init")
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        LOGGER.info("Init DruidDataSource");
        return new DruidDataSourceWrapper();
    }
}
```

我们可以看到注解@AutoConfigureBefore(DataSourceAutoConfiguration.class) ，申明了要在DataSourceAutoConfiguration这个配置类生效之前，让当前这个配置类生效（因为如果DataSourceAutoConfiguration先生效就会像Spring容器放入HikariDataSource），这样我们想要的DruidDataSource就不会被放进去，所以必须要在DataSourceAutoConfiguration之前装配DruidDataSource）

其中引入了四种依赖：

DruidSpringAopConfiguration.class	用于监控各种指标

对应的配置项是spring.datasource.druid.aop-patterns

DruidStatViewServletConfiguration.class

这个类用于向Spring中注册一个用于监控的Servlet，用于开启监控页（和我们前面自己的配置的大致一样，只是这里配置的参数更详细一些）

对应的配置项是spring.datasource.druid.stat-view-servlet

```java
@ConditionalOnWebApplication
@ConditionalOnProperty(name = "spring.datasource.druid.stat-view-servlet.enabled", havingValue = "true")
public class DruidStatViewServletConfiguration {
    private static final String DEFAULT_ALLOW_IP = "127.0.0.1";

    @Bean
    public ServletRegistrationBean statViewServletRegistrationBean(DruidStatProperties properties) {
        DruidStatProperties.StatViewServlet config = properties.getStatViewServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean();
        registrationBean.setServlet(new StatViewServlet());
        registrationBean.addUrlMappings(config.getUrlPattern() != null ? config.getUrlPattern() : "/druid/*");
        if (config.getAllow() != null) {
            registrationBean.addInitParameter("allow", config.getAllow());
        } else {
            registrationBean.addInitParameter("allow", DEFAULT_ALLOW_IP);
        }
        if (config.getDeny() != null) {
            registrationBean.addInitParameter("deny", config.getDeny());
        }
        if (config.getLoginUsername() != null) {
            registrationBean.addInitParameter("loginUsername", config.getLoginUsername());
        }
        if (config.getLoginPassword() != null) {
            registrationBean.addInitParameter("loginPassword", config.getLoginPassword());
        }
        if (config.getResetEnable() != null) {
            registrationBean.addInitParameter("resetEnable", config.getResetEnable());
        }
        return registrationBean;
    }
}
```

DruidWebStatFilterConfiguration.class

这个类用于开启过滤器，统计各种请求的数据，这也是监控页的数据来源

```java
@ConditionalOnWebApplication
@ConditionalOnProperty(name = "spring.datasource.druid.web-stat-filter.enabled", havingValue = "true")
public class DruidWebStatFilterConfiguration {
    @Bean
    public FilterRegistrationBean webStatFilterRegistrationBean(DruidStatProperties properties) {
        DruidStatProperties.WebStatFilter config = properties.getWebStatFilter();
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        WebStatFilter filter = new WebStatFilter();
        registrationBean.setFilter(filter);
        registrationBean.addUrlPatterns(config.getUrlPattern() != null ? config.getUrlPattern() : "/*");
        registrationBean.addInitParameter("exclusions", config.getExclusions() != null ? config.getExclusions() : "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        if (config.getSessionStatEnable() != null) {
            registrationBean.addInitParameter("sessionStatEnable", config.getSessionStatEnable());
        }
        if (config.getSessionStatMaxCount() != null) {
            registrationBean.addInitParameter("sessionStatMaxCount", config.getSessionStatMaxCount());
        }
        if (config.getPrincipalSessionName() != null) {
            registrationBean.addInitParameter("principalSessionName", config.getPrincipalSessionName());
        }
        if (config.getPrincipalCookieName() != null) {
            registrationBean.addInitParameter("principalCookieName", config.getPrincipalCookieName());
        }
        if (config.getProfileEnable() != null) {
            registrationBean.addInitParameter("profileEnable", config.getProfileEnable());
        }
        return registrationBean;
    }
}
```

DruidFilterConfiguration.class

用于设置Druid自己的一些配置项，开启一些功能（比如stat：状态监控，wall防火墙）

```java
    private static final String FILTER_STAT_PREFIX = "spring.datasource.druid.filter.stat";
    private static final String FILTER_CONFIG_PREFIX = "spring.datasource.druid.filter.config";
    private static final String FILTER_ENCODING_PREFIX = "spring.datasource.druid.filter.encoding";
    private static final String FILTER_SLF4J_PREFIX = "spring.datasource.druid.filter.slf4j";
    private static final String FILTER_LOG4J_PREFIX = "spring.datasource.druid.filter.log4j";
    private static final String FILTER_LOG4J2_PREFIX = "spring.datasource.druid.filter.log4j2";
    private static final String FILTER_COMMONS_LOG_PREFIX = "spring.datasource.druid.filter.commons-log";
    private static final String FILTER_WALL_PREFIX = "spring.datasource.druid.filter.wall";
    private static final String FILTER_WALL_CONFIG_PREFIX = FILTER_WALL_PREFIX + ".config";
```

然后我们根据上述配置中的规则配置我们想要的功能即可：

```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db_account
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver

    druid:
      aop-patterns: com.atguigu.admin.*  #监控的范围
      filters: stat,wall,slf4j     # 底层开启功能，stat（sql监控），wall（防火墙），slf4j打印SQL日志

      stat-view-servlet:   # 配置监控页功能
        enabled: true	#开启监控页，默认是false不开启，所以这里需要配置成true
        login-username: admin	#登录用户名
        login-password: admin	#登录密码
        resetEnable: false	#是否开启重置按钮

      web-stat-filter:  # 监控web
        enabled: true	#默认不开启，所以需要配置成true
        urlPattern: /*	#匹配的URL
        exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'	#不监控的URI


      filter:
        stat:    # 对上面filters里面的stat的详细配置
          slow-sql-millis: 1000 #慢查询的阈值
          logSlowSql: true #是否统计慢查询
          enabled: true	#是否开启这个功能
        wall:
          enabled: true	#是否开启防火墙
          config:
            drop-table-allow: false	#拦截哪些操作

```

### 整合MyBatis

#### 完全配置方式

整合框架前我们应当优先寻找这个框架对应的starter，导入这个starter依赖

```xml
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.2.0</version>
        </dependency>
```

查看源码的时候我们先查看它的META-INF中的spring.factories中指定了哪些自定配置类需要加载，然后查看这些自动配置类，然后再查看它引入的配置类绑定了哪些属性，这样就知道再配置文件中有哪些需要配置的属性

Mybatis的自动配置类：MybatisAutoConfiguration

```java
@org.springframework.context.annotation.Configuration
//必须引入这些jar包
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
//容器中有且仅有一个数据源DataSource
@ConditionalOnSingleCandidate(DataSource.class)
//使用Mybatis配置绑定类
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, MybatisLanguageDriverAutoConfiguration.class })
public class MybatisAutoConfiguration implements InitializingBean {
```

我们可以看到这个自动配置类需要使用MybatisProperties这个配置类，并且前缀是mybatis

```java
@ConfigurationProperties(prefix = MybatisProperties.MYBATIS_PREFIX)
public class MybatisProperties {

  public static final String MYBATIS_PREFIX = "mybatis";

```

在自动配置类中自动帮我们配置好的SqlSessionFactory，也就是SQL会话工厂

```java
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
```

装配了sqlSessionTemplate，这个里面含有sqlSession

```java
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
      return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
      return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```

@Import(AutoConfiguredMapperScannerRegistrar.class) 引入包的扫描规则

Mapper：只要我们写的mybatis接口标注了@Mapper注解就会会被自动扫描进来

Mybatis所需要的配置

```yaml
spring:
  datasource:
    username: root
    password: 1234
    url: jdbc:mysql://localhost:3306/my
    driver-class-name: com.mysql.jdbc.Driver

# 配置mybatis规则
mybatis:
  config-location: classpath:mybatis/mybatis-config.xml  #全局配置文件位置
  mapper-locations: classpath:mybatis/*.xml  #Mapper接口的sql映射文件位置

```

对应这个包结构：

![image-20220508232429677](pictures/6529dfcdabd63991f971352fc625b8b5.png)

**mybatis-config.xml**:

这里可以配置一些mybatis的额外功能，可以参照官方文档

https://mybatis.org/mybatis-3/zh/configuration.html#settings

例如配置命名规则

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
<!--    开启将下滑线命名法转换为驼峰命名法-->
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
</configuration>

```

**Mapper接口**：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- 这里需要指定对应的接口 -->
<mapper namespace="com.lun.boot.mapper.UserMapper">

    <select id="getUser" resultType="com.lun.boot.bean.User">
        select * from user where id=#{id}
    </select>
</mapper>

```

java目录下的Mapper接口

```java
import com.lun.boot.bean.User;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface UserMapper {
    public User getUser(Integer id);
}

```

注意这两个文件的文件名的前缀要相同，同时接口函数要加上@Mapper注解来申明这是Mybatis的Mapper层接口。

如果使用@Repository注解，还需要在配置 类加上@MapperScan注解指定Mapper接口所在路径

我们关于Mybatis的配置除了可以在xml里面配置外，也可以直接在yml里面配置

```yml
mybatis:
#  config-location: classpath:mybatis/mybatis-config.xml  #全局配置文件位置
  mapper-locations: classpath:mybatis/*.xml  #Mapper接口的sql映射文件位置
  configuration: #指定Mybatis的全局配置
    map-underscore-to-camel-case: true
```

但是注意config-location配置和configuration配置不能同时存在，要么我使用config-location指定xml配置文件的位置，然后在xml文件中配置，要么就直接在configuration下面配置

使用步骤：

1. 导入mybatis官方starter
2. 编写mapper接口
3. 编写sql映射文件并绑定mapper接口
4. 在application.yml中指定配置文件的位置，以及指定全局配置文件的信息（建议直接在mybatis.configuration下面的配置）

#### 完全注解方式

```java
@Mapper
public interface UserMapper2 {

    @Select("select * from usert")
    List<User> getUsers();
}
```

直接在注解上写上sql语句，即可完成对应的功能，这样就无需编写xml文件

#### 混合使用

上面两种方式可以同时使用，也就是一个接口中可以既有使用注解的方式，也可以有在xml文件中配置的方式

xml中可以编写复杂的sql，而简单的sql直接使用注解即可

```java
@Mapper
public interface UserMapper {
    public User getUser(Integer id);

    @Select("select * from user where id=#{id}")
    public User getUser2(Integer id);

    public void saveUser(User user);

    @Insert("insert into user(`name`) values(#{name})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    public void saveUser2(User user);
}
```

得到自增的主键：

xml：

```xml
    <insert id="saveUser" useGeneratedKeys="true" keyProperty="id">
        insert into user(`name`) values(#{name})
    </insert>
```

注解：

```java
    @Insert("insert into user(`name`) values(#{name})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    public void saveUser2(User user);
```

useGeneratedKeys="true"表示开启主键自增，keyProperty="id"表示自增的主键是id

开启这个后会把自增得到的主键放入User中的id字段中（面向对象，传入的User内部被修改后，外面显然还能拿到）

### 整合Mybatis Plus

Mybatis可以帮我们生成代码，简化开发

```xml
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.4.1</version>
        </dependency>
```

这个依赖帮我们引入了jdbc和基础的mybatis和一些扩展包，所以引入这个包后就不用再引入mybatis和jdbc

- `MybatisPlusAutoConfiguration`配置类，`MybatisPlusProperties`配置项绑定，对应着mybatis-plus为前缀的配置项

- `SqlSessionFactory`自动配置好，底层是容器中默认的数据源。

- `mapperLocations`自动配置好的，有默认值`classpath*:/mapper/**/*.xml`，这表示mapper文件夹下任意路径下的所有xml都是sql映射文件。 建议以后sql映射文件放在 mapper下。
- 容器中也自动配置好了`SqlSessionTemplate`。
- `@Mapper` 标注的接口也会被自动扫描，也可以用MapperScan批量扫描

使用方法：

接口直接继承BaseMapper<User>，泛型是我们要操作的数据库的表

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {

}
```

表名必须和泛型的名称一致，数组库字段要和属性字段一致，并且出现的字段对应数据库中对应名称的字段，如果没有出现可以用加上@TableField(exist = false) 来表示这个字段不存在

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {

    private Integer id;
    private String name;
    private String password;
    private String email;
    private Date birthday;
    private Float money;

    @TableField(exist = false)
    private String uid;

}
```

查询测试

```java
@SpringBootTest
class MydemoApplicationTests {

    @Resource
    UserMapper userMapper;
    @Test
    void contextLoads() {
        System.out.println(userMapper.selectById(57));
    }
}
```

上述严格的对应关系会让开发变得有些麻烦，mybatis-plus提供了一些好用的注解来解决这些问题

```
@TableName("usert") //设置对应的表名
```

Mybatis Plus不仅提供了Mapper层的通用功能接口，也提供了Service层的通用实现接口

```java
public interface UserService extends IService<User> {
}
```

接口类继承IService<User> User是对应的实体类

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper,User> implements UserService {

}
```

编写实现类，规范如下：

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper,User> implements UserService {

}
```

需要继承ServiceImpl，传入两个泛型：

UserMapper是我们继承了BaseMapper的接口

User是对应的实体类

ServiceImpl为我们实现了很多方法：

list()	查询所有的数据

page(Page,Wrapper) 分页查询

removeById() 根据主键删除

Page：

getPages：查询总页数

getRecordes：获取查询的数据

测试：

```java
@SpringBootTest
class MydemoApplicationTests {

    @Resource
    UserService userService;
    @Test
    void contextLoads() {
        Page<User> page1 = userService.page(new Page<>(0,5),null);
        page1.getRecords().forEach(System.out::println);
    }
}
```

但是此时，分页功能会失效，Mybatis会查到所有数据，需要加上一个配置插件才能开启分页功能：

```java
@Configuration
public class MyBatisConfig {
    /**
     * MybatisPlusInterceptor
     */
    @Bean
    public MybatisPlusInterceptor paginationInterceptor() {
        MybatisPlusInterceptor mybatisPlusInterceptor = new MybatisPlusInterceptor();
        // 设置请求的页面大于最大页后操作， true调回到首页，false 继续请求  默认false
        // paginationInterceptor.setOverflow(false);
        // 设置最大单页限制数量，默认 500 条，-1 不受限制
        // paginationInterceptor.setLimit(500);
        // 开启 count 的 join 优化,只针对部分 left join

        //设置一个分页拦截器
        PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();
        paginationInnerInterceptor.setOverflow(true);
        paginationInnerInterceptor.setMaxLimit(500L);
        //添加拦截器
        mybatisPlusInterceptor.addInnerInterceptor(paginationInnerInterceptor);

        return mybatisPlusInterceptor;
    }
}
```

这样分页就能成功使用了：

```java
@SpringBootTest
class MydemoApplicationTests {

    @Resource
    UserService userService;
    @Test
    void contextLoads() {
        Page<User> page1 = userService.page(new Page<>(2,5),null);
        page1.getRecords().forEach(System.out::println);
    }

}
```

注意Spring的分页是从1开始的，0和1都会返回第一页

分页前端表格示例：

```html
<table class="display table table-bordered table-striped" id="dynamic-table">
    <thead>
        <tr>
            <th>#</th>
            <th>name</th>
            <th>age</th>
            <th>email</th>
            <th>操作</th>
        </tr>
    </thead>
    <tbody>
        <tr class="gradeX" th:each="user: ${users.records}">
            <td th:text="${user.id}"></td>
            <td>[[${user.name}]]</td>
            <td th:text="${user.age}">Win 95+</td>
            <td th:text="${user.email}">4</td>
            <td>
                <a th:href="@{/user/delete/{id}(id=${user.id},pn=${users.current})}" 
                   class="btn btn-danger btn-sm" type="button">删除</a>
            </td>
        </tr>
    </tfoot>
</table>

<div class="row-fluid">
    <div class="span6">
        <div class="dataTables_info" id="dynamic-table_info">
            当前第[[${users.current}]]页  总计 [[${users.pages}]]页  共[[${users.total}]]条记录
        </div>
    </div>
    <div class="span6">
        <div class="dataTables_paginate paging_bootstrap pagination">
            <ul>
                <li class="prev disabled"><a href="#">← 前一页</a></li>
                <li th:class="${num == users.current?'active':''}" 
                    th:each="num:${#numbers.sequence(1,users.pages)}" >
                    <a th:href="@{/dynamic_table(pn=${num})}">[[${num}]]</a>
                </li>
                <li class="next disabled"><a href="#">下一页 → </a></li>
            </ul>
        </div>
    </div>
</div>
```

#### Mybatis-Plus使用手册

https://blog.csdn.net/weixin_43811057/article/details/123449767

实际上Mybatis-Plus用于处理基本的增删改成即可，复杂的业务逻辑我们使用xml文件即可，稍简单的逻辑我们可以使用注解来实现

### 整合Redis

引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

我们先来看Redis的自动配置类RedisAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {
```

这个自动配置类绑定了配置类：RedisProperties

这个配置类绑定的配置是@ConfigurationProperties(prefix = "spring.redis")

内部封装了jedis和letture

也就是我们需要配置redis就在spring.redis下配置

并且帮我们准备了两种客户端的连接配置：LettuceConnectionConfiguration，JedisConnectionConfiguration

和两种操作redis的接口：redisTemplate，stringRedisTemplate

redisTemplate<Object,Object>

stringRedisTemplate，kv都是String

RedisProperties中的默认配置：

```java
	/**
	 * Database index used by the connection factory.
	 */
	private int database = 0;

	/**
	 * Connection URL. Overrides host, port, and password. User is ignored. Example:
	 * redis://user:password@example.com:6379
	 */
	private String url;

	/**
	 * Redis server host.
	 */
	private String host = "localhost";

	/**
	 * Login password of the redis server.
	 */
	private String password;

	/**
	 * Redis server port.
	 */
	private int port = 6379;
```

在yml中配置Redis的相关信息：

可以设置Redis的相关属性来连接（推荐）：

```yml
spring:
  redis:
    host: localhost
    port: 6379
    password: 123456
```

也可以直接设置url代替上述参数：

```yml
spring:
  redis:
    url: redis://root:123456@127.0.0.1:6379
```

RedisTemplate默认使用letture来操作redis，我们也可以切换客户端至jedis切换客户端

导入jedis：

```xml
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
```

jedis也是可以直接使用的

```java
spring:
  redis:
#   url: redis://lfy:Lfy123456@r-bp1nc7reqesxisgxpipd.redis.rds.aliyuncs.com:6379
    host: r-bp1nc7reqesxisgxpipd.redis.rds.aliyuncs.com
    port: 6379
    password: lfy:Lfy123456
    client-type: jedis
    jedis:
      pool:
        max-active: 10
#   lettuce:# 另一个用来连接redis的java框架
#      pool:
#        max-active: 10
#        min-idle: 5
```

小功能：

编写一个拦截器类，这个类加上@Component申明为一个组件，这样就可以使用Spring容器中的组件的各种功能。

```java
@Component
public class UriInterceptor implements HandlerInterceptor {

    @Resource
    RedisTemplate redisTemplate;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        redisTemplate.opsForValue().increment(request.getRequestURI());
        return true;
    }
}
```

添加拦截器：

拦截器要从Spring容器中拿才能实现我们想要的功能

```java
            @Resource
            UriInterceptor uriInterceptor;
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                registry.addInterceptor(new LoginIntercepter())
                        .addPathPatterns("/**")
                        .excludePathPatterns("/login","/","/css/**","/js/**","/img/**");
                registry.addInterceptor(uriInterceptor)
                        .addPathPatterns("/**")
                        .excludePathPatterns("/","/css/**","/js/**","/img/**");
            }
```

过滤器和拦截器的区别（Filter和Interceptor的区别）

1.过滤器Filter是Servlet的原生组件，脱离了Spring也能使用，并且被拦截后不能直接回到原来的方法中

2.拦截器Interceptor是Spring处理请求的一个流程，可以使用Spring容器中的组件

![image-20220509162643868](pictures/949df1adea40b3db10776a4d65c3bd53.png)

## 单元测试

### 依赖引入

Junit4用@SpringbootTest+@RunWith(SpringTest.class)来进行单元测试

**Spring Boot 2.2.0 版本开始引入 JUnit 5 作为单元测试默认库**

SpringBoot 2.4 以上版本移除了默认对 Vintage 的依赖。如果需要兼容JUnit4需要自行引入（不能使用JUnit4的功能 @Test）

JUnit 5’s Vintage已经从spring-boot-starter-test从移除。如果需要继续兼容Junit4需要自行引入Vintage依赖：

```xml
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
    <exclusions>
        <exclusion>
            <groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>

```

但是其实我们也没有必要兼容Junit4，直接使用Junit5的功能即可，以org.junit.jupiter开头的就是Junit5下面的框架

单元测试其实之前我们也用过：

```java
@SpringBootTest
class MydemoApplicationTests {

    @Resource
    RedisTemplate redisTemplate;
    @Autowired
    RedisConnectionFactory redisConnectionFactory;

    @Test
    void contextLoads() {
        redisTemplate.opsForValue().set("lth","lth");
        System.out.println(redisTemplate.opsForValue().get("lth"));
        System.out.println(redisConnectionFactory.getClass());
    }
}
```

在Test目录下，人家以及自动帮我们配置了一个测试类，我们直接在这个里面测试即可，要引入什么框架也可以直接注入

### 常见注解使用

官方文档：

https://junit.org/junit5/docs/current/user-guide/#writing-tests-annotations

- @Test：表示方法是测试方法。
- @ParameterizedTest：表示方法是参数化测试。
- @RepeatedTest：表示方法可重复执行，括号中可以写出重复次数。
- @DisplayName：为测试类或者测试方法设置展示名称，展示的名称会在控制台显示出来。
- @BeforeEach：表示在每个单元测试之前执行。
- @AfterEach：表示在每个单元测试之后执行。
- @BeforeAll：表示在所有单元测试之前执行，使用这个注解的方法必须是静态方法。
- @AfterAll：表示在所有单元测试之后执行，使用这个注解的方法必须是静态方法。
- @Tag：表示单元测试类别，类似于JUnit4中的@Categories。
- @Disabled：表示测试类或测试方法不执行，整体测试时会忽略这个方法。
- @Timeout：表示测试方法运行如果超过了指定时间将会返回错误，括号中可以设置超时时间和时间单位。
- @ExtendWith：为测试类或测试方法提供扩展类引用，例如@ExtendWith(SpringExtension.class)申明是使用Spring提供的测试组件，申明这个后就可以进行依赖注入，可以使用@SpringBootTest代替。

```java
import org.junit.jupiter.api.*;

@DisplayName("junit5功能测试类")
public class Junit5Test {


    @DisplayName("测试displayname注解")
    @Test
    void testDisplayName() {
        System.out.println(1);
        System.out.println(jdbcTemplate);
    }
    
    @ParameterizedTest
    @ValueSource(strings = { "racecar", "radar", "able was I ere I saw elba" })
    void palindromes(String candidate) {
        assertTrue(StringUtils.isPalindrome(candidate));
    }
    

    @Disabled
    @DisplayName("测试方法2")
    @Test
    void test2() {
        System.out.println(2);
    }

    @RepeatedTest(5)
    @Test
    void test3() {
        System.out.println(5);
    }

    /**
     * 规定方法超时时间。超出时间测试出异常
     *
     * @throws InterruptedException
     */
    @Timeout(value = 500, unit = TimeUnit.MILLISECONDS)
    @Test
    void testTimeout() throws InterruptedException {
        Thread.sleep(600);
    }


    @BeforeEach
    void testBeforeEach() {
        System.out.println("测试就要开始了...");
    }

    @AfterEach
    void testAfterEach() {
        System.out.println("测试结束了...");
    }

    @BeforeAll
    static void testBeforeAll() {
        System.out.println("所有测试就要开始了...");
    }

    @AfterAll
    static void testAfterAll() {
        System.out.println("所有测试以及结束了...");

    }

}
```

### 断言

如果满足我们给定的条件就无事发生，否则就会抛出异常，后面的代码都不会执行

#### 简单断言

方法	说明
assertEquals	判断两个对象或两个原始类型是否相等（调用equal方法）
assertNotEquals	判断两个对象或两个原始类型是否不相等
assertSame	判断两个对象引用是否指向同一个对象（调用==）
assertNotSame	判断两个对象引用是否指向不同的对象
assertTrue	判断给定的布尔值是否为 true
assertFalse	判断给定的布尔值是否为 false
assertNull	判断给定的对象引用是否为 null
assertNotNull	判断给定的对象引用是否不为 null

#### 数组断言

通过 assertArrayEquals 方法来判断两个对象或原始类型的数组是否相等。

```java
@Test
@DisplayName("array assertion")
public void array() {
	assertArrayEquals(new int[]{1, 2}, new int[] {1, 2});
}
```

#### 组合断言

`assertAll()`方法接受多个 `org.junit.jupiter.api.Executable` 函数式接口的实例作为要验证的断言，可以通过 lambda 表达式很容易的提供这些断言。所有这些断言都通过了才算这个断言通过，有一个不通过就视为这个断言不通过。

```java
@Test
@DisplayName("assert all")
public void all() {
 assertAll("Math",
    () -> assertEquals(2, 1 + 1),
    () -> assertTrue(1 > 0)
 );
}
```

#### 异常断言

如果不抛出指定异常则断言失败

```java
@Test
@DisplayName("异常测试")
public void exceptionTest() {
    ArithmeticException exception = Assertions.assertThrows(
           //扔出断言异常
            ArithmeticException.class, () -> System.out.println(1 % 0));
}

```

#### 超时断言

```java
@Test
@DisplayName("超时测试")
public void timeoutTest() {
    //如果测试方法时间超过1s将会异常
    Assertions.assertTimeout(Duration.ofMillis(1000), () -> Thread.sleep(500));
}

```

#### 快速失败

```java
@Test
@DisplayName("fail")
public void shouldFail() {
	fail("This should fail");
}
```

我们使用maven的Test功能对测试类进行测试，测试完成后会生成一个汇总的报告

#### 前置条件

使用方法和断言一样，但是如果前置条件实现了，这个方法会显示被忽略而不是错误

```java
@DisplayName("前置条件")
public class AssumptionsTest {
    private final String environment = "DEV";

    @Test
    @DisplayName("simple")
    public void simpleAssume() {
        assumeTrue(Objects.equals(this.environment, "DEV"));
        assumeFalse(() -> Objects.equals(this.environment, "PROD"));
    }

    @Test
    @DisplayName("assume then do")
    public void assumeThenDo() {
        assumingThat(
            Objects.equals(this.environment, "DEV"),
            () -> System.out.println("In DEV")
        );
    }
}
```

### 嵌套测试

使用@Nested注解可以在测试类的内部定义一个新的测试类，外层的测试类的@AfterEach等注解可以驱动内部的测试生效，而内部的这些注解不会驱动外部的测试类生效。

```java
@DisplayName("A stack")
class TestingAStackDemo {

    Stack<Object> stack;

    @Test
    @DisplayName("is instantiated with new Stack()")
    void isInstantiatedWithNew() {
        new Stack<>();
    }

    @Nested
    @DisplayName("when new")
    class WhenNew {

        @BeforeEach
        void createNewStack() {
            stack = new Stack<>();
        }

        @Test
        @DisplayName("is empty")
        void isEmpty() {
            assertTrue(stack.isEmpty());
        }

        @Test
        @DisplayName("throws EmptyStackException when popped")
        void throwsExceptionWhenPopped() {
            assertThrows(EmptyStackException.class, stack::pop);
        }

        @Test
        @DisplayName("throws EmptyStackException when peeked")
        void throwsExceptionWhenPeeked() {
            assertThrows(EmptyStackException.class, stack::peek);
        }

        @Nested
        @DisplayName("after pushing an element")
        class AfterPushing {

            String anElement = "an element";

            @BeforeEach
            void pushAnElement() {
                stack.push(anElement);
            }

            @Test
            @DisplayName("it is no longer empty")
            void isNotEmpty() {
                assertFalse(stack.isEmpty());
            }

            @Test
            @DisplayName("returns the element when popped and is empty")
            void returnElementWhenPopped() {
                assertEquals(anElement, stack.pop());
                assertTrue(stack.isEmpty());
            }

            @Test
            @DisplayName("returns the element when peeked but remains not empty")
            void returnElementWhenPeeked() {
                assertEquals(anElement, stack.peek());
                assertFalse(stack.isEmpty());
            }
        }
    }
}
```

### 指定参数来源

```
@ValueSource: 为参数化测试指定入参来源，支持八大基础类以及String类型,Class类型
@NullSource: 表示为参数化测试提供一个null的入参
@EnumSource: 表示为参数化测试提供一个枚举入参
@CsvFileSource：表示读取指定CSV文件内容作为参数化测试入参
@MethodSource：表示读取指定方法的返回值作为参数化测试入参(注意方法返回需要是一个流)
```

```java
@ParameterizedTest
@ValueSource(strings = {"one", "two", "three"})
@DisplayName("参数化测试1")
public void parameterizedTest1(String string) {
    System.out.println(string);
    Assertions.assertTrue(StringUtils.isNotBlank(string));
}


@ParameterizedTest
@MethodSource("method")    //指定方法名
@DisplayName("方法来源参数")
public void testWithExplicitLocalMethodSource(String name) {
    System.out.println(name);
    Assertions.assertNotNull(name);
}

static Stream<String> method() {
    return Stream.of("apple", "banana");
}
```

## 指标监控

Springboot-actuator可以帮我们监控各个微服务的运行状态

引入依赖：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

引入依赖后就可以直接通过http://localhost:8080/actuator来获取可以拿到的信息的列表

```json
{
    "_links": {
        "self": {
            "href": "http://localhost:8080/actuator",
            "templated": false
        },
        "health-path": {
            "href": "http://localhost:8080/actuator/health/{*path}",
            "templated": true
        },
        "health": {
            "href": "http://localhost:8080/actuator/health",
            "templated": false
        },
        "info": {
            "href": "http://localhost:8080/actuator/info",
            "templated": false
        }
    }
}
```

然后再根据其中的网址获取我们想要的信息

self代表当前访问的网址：

```json
    "self": {
        "href": "http://localhost:8080/actuator",
        "templated": false
    }
```

health代表当前服务的运行状态：

```json
        "health": {
            "href": "http://localhost:8080/actuator/health",
            "templated": false
        }
```

```
{
    "status": "UP"
}
```

UP代表正在运行状态，DOWN代表宕机

info代表当前服务的信息（默认没有信息）

```json
        "info": {
            "href": "http://localhost:8080/actuator/info",
            "templated": false
        }
```

Spring默认给我密文提供了info和health两个监控端点（EndPoint），但其实还有很多我们可以监控的端点，需要我们手动开启

https://docs.spring.io/spring-boot/docs/2.4.2/reference/htmlsingle/#production-ready

以web的方式暴露所有端点

```yml
management:
  endpoints:
    enabled-by-default: true #暴露所有端点信息
    web:
      exposure:
        include: '*'  #以web方式暴露
```

查询信息的格式是：http://localhost:8080/actuator/{端点名称}/{具体的路径名称}

会返回JSON格式的数据

常用的端点信息：

auditevents	暴露当前应用程序的审核事件信息。需要一个AuditEventRepository组件。
beans	显示应用程序中所有Spring Bean的完整列表。
caches	暴露可用的缓存。
conditions	显示自动配置的所有条件信息，包括匹配或不匹配的原因。
configprops	显示所有@ConfigurationProperties。
env	暴露Spring的属性ConfigurableEnvironment
flyway	显示已应用的所有Flyway数据库迁移。 需要一个或多个Flyway组件。
health	显示应用程序运行状况信息。
httptrace	显示HTTP跟踪信息（默认情况下，最近100个HTTP请求-响应）。需要一个HttpTraceRepository组件。
info	显示应用程序信息。
integrationgraph	显示Spring integrationgraph 。需要依赖spring-integration-core。
loggers	显示和修改应用程序中日志的配置。
liquibase	显示已应用的所有Liquibase数据库迁移。需要一个或多个Liquibase组件。
metrics	显示当前应用程序的“指标”信息。
mappings	显示所有@RequestMapping路径列表。
scheduledtasks	显示应用程序中的计划任务。
sessions	允许从Spring Session支持的会话存储中检索和删除用户会话。需要使用Spring Session的基于Servlet的Web应用程序。
shutdown	使应用程序正常关闭。默认禁用。
startup	显示由ApplicationStartup收集的启动步骤数据。需要使用SpringApplication进行配置BufferingApplicationStartup。
threaddump	执行线程转储。

- **Health：监控状况**
- **Metrics：运行时指标**
- **Loggers：日志记录**



```yml
management:
  endpoints:
    enabled-by-default: true #暴露所有端点信息
    web:
      exposure:
        include: '*'  #以web方式暴露
  endpoint:
    health: #对某个端点的具体配置
      show-details: always #显示详细信息
```

我们也可以或者禁用所有的Endpoint然后手动开启指定的Endpoint：

```yml
management:
  endpoints:
    enabled-by-default: false
  endpoint:
    beans:
      enabled: true
    health:
      enabled: true
```

### 定制健康信息

```java
@Component
public class MyComHealthIndicator extends AbstractHealthIndicator {

    /**
     * 真实的检查方法
     * @param builder
     * @throws Exception
     */
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        //mongodb。  获取连接进行测试
        Map<String,Object> map = new HashMap<>();
        // 检查完成
        if(1 == 2){
//            builder.up(); //健康
            builder.status(Status.UP);
            map.put("count",1);
            map.put("ms",100);
        }else {
//            builder.down();
            builder.status(Status.OUT_OF_SERVICE);
            map.put("err","连接超时");
            map.put("ms",3000);
        }

        builder.withDetail("code",100)
                .withDetails(map);

    }
}
```

builder.down() 表示不健康

builde.up() 表示健康

也可以用 builder.status(Status.UP);

```java
        builder.withDetail("code",100)
                .withDetails(map);
```

可以往detail中添加一些信息

注意，这个组件的名字是根据类的名称来的，必须实现AbstractHealthIndicator，而且必须以HealthIndicator结尾，前面的就是组件的名称

查询health：

```json
        "myCom": {
            "status": "OUT_OF_SERVICE",
            "details": {
                "code": 100,
                "err": "连接超时",
                "ms": 3000
            }
        }
```

### 定值info信息

可以在yml里定值，获取pom文件的值，可以使用@@来获取

```yml
info:
  appName: boot-admin
  version: 2.0.1
  mavenProjectName: @project.artifactId@  #使用@@可以获取maven的pom文件值
  mavenProjectVersion: @project.version@

```

可以定义一个Controller：

```java
import java.util.Collections;

import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

@Component
public class ExampleInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("example",
                Collections.singletonMap("key", "value"));
    }

}
```

这个controller的名字就没有限制了，只要继承InfoContributor并注入Spring容器中即可

### 定制Metrics

这样在Metrics端点就会有myservice.method.running.counter的相关信息

```java
class MyService{
    Counter counter;
    public MyService(MeterRegistry meterRegistry){
         counter = meterRegistry.counter("myservice.method.running.counter");
    }

    public void hello() {
        counter.increment();
    }
}
```

### 自定义Endpoint

```java
@Component
//Endpoint叫container
@Endpoint(id = "container")
public class DockerEndpoint {

    //可读。不能有参数，显示的信息从这里获取
    @ReadOperation
    public Map getDockerInfo(){
        return Collections.singletonMap("info","docker started...");
    }
	可写
    @WriteOperation
    private void restartDocker(){
        System.out.println("docker restarted....");
    }

}
```

### 整合图形界面

引入依赖：

```xml
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
            <version>2.3.1</version>
        </dependency>
```

在启动类加上@EnableAdminServer表示这是一个监控服务器

```java
@SpringBootApplication
@EnableAdminServer
public class ActuatorApplication {

    public static void main(String[] args) {
        SpringApplication.run(ActuatorApplication.class, args);
    }
}
```

修改一下server.port确保端口不冲突，例如修改为8888

然后访问localhost:8888，即可看到监控页面，但是此时还没有数据，因为监控服务器也不知道要监控什么服务器，所以我们需要配置需要监控的服务器（客户端）

在客户端加上：

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>2.3.1</version>
</dependency>
```



然后设置一下配置文件：

```yml
spring:
  application:
    name: mydemo
  boot:
    admin:
      client:
        url: http://localhost:8888
        instance:
          prefer-ip: true
```

和spring-cloud配置注册中心的过程很像

点开配置文件如下：

```java
	/**
	 * Name to register with. Defaults to ${spring.application.name}
	 */
	@Value("${spring.application.name:spring-boot-application}")
	private String name = "spring-boot-application";

	/**
	 * Should the registered urls be built with server.address or with hostname.
	 */
	private boolean preferIp = false;

	/**
	 * Metadata that should be associated with this application
	 */
	private Map<String, String> metadata = new LinkedHashMap<>();
```

注意到配置：

```java
@Value("${spring.application.name:spring-boot-application}")
	private String name = "spring-boot-application";
```

我们也发现可以使用@Value注解获取配置文件中的值

@Value("${spring.application.name:spring-boot-application}") 表示获取spring.application.name这个配置项的值，如果没有就叫spring-boot-application

配置完成后可以有很好看的图形界面：

![image-20220510213637957](pictures/1483bd50890ef67bc276ad0b239b61c0.png)

## 原理解析

### profile 配置文件切换

我们一般情况测试开发环境所用的配置文件和上线部署后用的配置文件一般不同，比如测试环境中我们可以用localhost，但是上线部署的生产环境中就需要切换到部署环境，而我们直接修改配置文件有些麻烦，所以Spring给我们提供了profile配置文件切换功能。

我们先编写两种配置文件，配置文件的名字必须是application-xxx.yml，xxx是配置文件的名称（测试环境的名称）：

比如：

测试环境所用的配置文件：applcation-test.yml

```yml
person:
  name: test
```

生产环境所用的配置文件：application-prod.yml

```yml
person:
  name: prod
```

然后我们在测试用手动controller中获取配置文件的值并输出

```java
package com.demo.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author 李天航
 */
@RestController
public class TestController {

    @Value("${person.name:default}")
    private String name;

    @GetMapping("/")
    public Object test(){
        return name;
    }
}
```

这样根据name的值就知道当前使用的是哪个配置文件

name标注了@Value("${person.name:default}")，从配置文件中获取值，如果配置文件没有相关的配置则值默认是default（上一节也提到过）

然后设置默认配置文件application.properties：

```
person.name=okk
spring.profiles.active=test
```

application.properties是一定会被加载的配置文件，其中spring.profiles.active自动用于设置当前使用哪个配置文件

spring.profiles.active=test表示使用application-test.yml配置文件，得到结果test

spring.profiles.active=prod表示使用application-prod.yml配置文件，得到结果prod

如果application.properties和选择的yml配置文件中有同名的配置，则优先使用选择的yml中的配置，如果yml中没有配置（获取选择的配置文件不存在）则使用application.properties配置文件，如果application.properties中也没有相关的配置则使用设置的默认值（例如这里是default）

打包后如果想要切换配置文件，可以在后面用--加上启动参数，启动参数的优先级最高，可以设置多个参数，参数名称和配置项的名称一致

```
java -jar demo2-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod --server.port=8888
```

--spring.profiles.active=prod 使用prod配置文件

--server.port=8888 切换端口至8888

获取配置文件的信息除了可以用@Value注解，还可以使用@ConfigurationProperties注解，这个注解之前在阅读Spring源码的时候我们见过很多次，每一个自动配置类都需要一个配置类，而配置类就是使用@ConfigurationProperties注解获取到配置文件的信息

例如配置文件中是这么写的：

```yml
person:
  name: test
  age: 88
```

我们想要获取配置信息：

```java
@Component
@ConfigurationProperties("person")
@Data
public class Person {
    String name;
    String age;
}
```

用@ConfigurationProperties("person")绑定要获取的配置项，然后根据属性名称将值装配进去，需要加上@Component注解

（这个注解会让idea报错，但是运行没有问题）

```java
@RestController
public class TestController {
    @Resource
    Person person;

    @GetMapping("/")
    public Object test(){
        return person;
    }
}
```

经过测试成功得到返回值person的值

![image-20220510232548151](pictures/da0266f5b88527d8c11b1e6a04cd825e.png)

假如一个环境中包含多个配置文件，我们可以设置配置文件组：

```properties
spring.profiles.active=production

spring.profiles.group.production[0]=proddb
spring.profiles.group.production[1]=prodmq
```

假如有个生产环境叫production，这个生产环境包含两个配置文件：proddb，prodmq，可以通过下面这两行配置实现

```properties
spring.profiles.group.production[0]=proddb
spring.profiles.group.production[1]=prodmq
```

然后选择生产环境的时候选择组即可：

```properties
spring.profiles.active=production
```

选择的组中的配置文件都会生效

### Profile条件装配

假如我们有一个类叫Person：

```java
@Data
public class Person {
    protected String name;
    protected String age;
}
```

它有两个子类：

```java
@Data
@Component
public class Boss extends Person{
    String type="boss";
}
```

```java
@Data
@Component
public class Worker extends Person{
    String type="worker";
}
```

测试类中是

```java
@RestController
public class TestController {
    @Resource
    Person person;

    @GetMapping("/")
    public Object test(){
        return person;
    }
}
```

我们想要在test环境下返回Worker对象，在prod环境下返回Boss对象，此时Spring容器中有两个Person对象，所以Spring不知道装配哪个对象所以会报错。所以这时候可以使用条件装配，在不同的环境下选择让一些类在特性的测试环境下生效。

```java
@Profile("prod")
@Component
@ConfigurationProperties("person")
@Data
public class Boss extends Person{
    String type="boss";
}
```

```java
@Profile("test")
@Component
@ConfigurationProperties("person")
@Data
public class Worker extends Person{
    String type="worker";
}
```

@Profile("prod")表示这个类只在运行环境为prod时才放入Spring容器中（并不影响编译）

例如当前运行环境是test，即spring.profiles.active=test，则会返回Worker对象

![image-20220510234810086](pictures/5a0e92a91297e1bc9d8c55826bf790e4.png)

@Profile可以标注在带有@Bean注解的方法上来选择性在Spring容器中注册bean

@Profile如果不设置value字段的值，则value字段的值默认是default，也就是默认环境下会使用的配置，不加@Profile则是在任何环境都会加载的bean。如果不激活任何环境也就是不设置spring.profiles.active的值（或者设置为default），这个值默认是default，默认会加载默认环境下的bean

### 配置文件加载的优先级

#### 配置信息的来源

properties文件，yml文件，环境变量，命令行参数（除了环境变量外我们都使用过，下面演示环境变量）

获取环境变量，使用方法就和控制台中一样，${环境变量名}：

```java
@RestController
public class TestController {

    @Value("${person.name:default}")
    private String name;

    @Resource
    Person person;

    @Value("${JAVA_HOME}")
    private String JAVA_HOME;

    @GetMapping("/")
    public Object test(){
        System.out.println(JAVA_HOME);
        return person;
    }
}
```

Springboot在启动的时候也会获取当前机器的环境变量和各种属性值：

```java
@SpringBootApplication
public class Demo2Application {

    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(Demo2Application.class, args);
        ConfigurableEnvironment environment = run.getEnvironment();
        //获取环境变量
        System.out.println(environment.getSystemEnvironment());
        //获取各种JVM参数和操作系统等信息
        System.out.println(environment.getPropertySources());
    }
}
```

其中命令行参数设置配置项的时候有一点要注意：

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

在启动类中SpringApplication.run(DemoApplication.class, args)一定要把args传进去，我们设置的命令行参数才能生效QWQ

#### 配置文件的优先级

1. Default properties (specified by setting SpringApplication.setDefaultProperties).
2. @PropertySource annotations on your @Configuration classes. Please note that such property sources are not added to the Environment until the application context is being refreshed. This is too late to configure certain properties such as logging.* and spring.main.* which are read before refresh begins.
3. Config data (such as application.properties files)
4. A RandomValuePropertySource that has properties only in random.*.
5. OS environment variables.
6. Java System properties (System.getProperties()).
7. JNDI attributes from java:comp/env.
8. ServletContext init parameters.
9. ServletConfig init parameters.
10. Properties from SPRING_APPLICATION_JSON (inline JSON embedded in an environment variable or system property).
11. Command line arguments.
12. properties attribute on your tests. Available on @SpringBootTest and the test annotations for testing a particular slice of your application.
13. @TestPropertySource annotations on your tests.
14. Devtools global settings properties in the $HOME/.config/spring-boot directory when devtools is active.

后面的会覆盖前面的同名配置项

#### 配置文件的位置

1. classpath 根路径（resource目录是classpath的根路径）。
2. classpath 根路径下config目录。
3. jar包当前目录。
4. jar包当前目录的config目录。
5. /config子目录的直接子目录。

后面的优先级更高

我们可以使用外部配置文件来修改配置，这样就不用重新打包编译文件也能修改配置

#### 配置文件加载顺序

1. 当前jar包内部的application.properties和application.yml。
2. 当前jar包内部的application-{profile}.properties 和 application-{profile}.yml。
3. 引用的外部jar包的application.properties和application.yml。
4. 引用的外部jar包的application-{profile}.properties和application-{profile}.yml。

后面的优先级更高

（测试的时候不要使用idea直接运行，使用命令行来启动）

### 自定义starter和自动配置类

如果我们使用Spring-Initializer时，没有选择任何场景，则会自动帮我们导入

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
```

这个依赖抱哈Spring的基本功能（Spring容器和自动配置的的依赖）

我们创建一个名为lth-spring-boot-starter的MAVEN项目，也就是我们自定义的starter，这这个starter中引入我们想要引入的依赖，然后其他项目想引入这些依赖时，直接引入这个starter即可

这个starter没有业务逻辑，起到统合依赖的作用：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.lth</groupId>
    <artifactId>lth-spring-boot-starter</artifactId>
    <version>1.0-SNAPSHOT</version>
    <dependencies>
        <dependency>
            <groupId>com.lth</groupId>
            <artifactId>lth-spring-boot-starter-autoconfiguration</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```

这个starter引入了lth-spring-boot-starter-autoconfiguration，其他项目引入这个starter时也会自动引入autoconfiguration

在lth-spring-boot-starter-autoconfiguration模块中编写一些具体的业务逻辑，比如我们想要根据配置文件设置打招呼的前缀和后缀

pom文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.6.7</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.lth</groupId>
    <artifactId>lth-spring-boot-starter-autoconfiguration</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>lth-spring-boot-starter-autoconfiguration</name>
    <description>lth-spring-boot-starter-autoconfiguration</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>

</project>
```

引入依赖时是根据，这两个属性引入到项目中的

```xml
    <groupId>com.lth</groupId>
    <artifactId>lth-spring-boot-starter-autoconfiguration</artifactId>
	<version>0.0.1-SNAPSHOT</version>
```

我们设置一个配置类来绑定依赖：

```java
package com.lth.bean;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("lth.hello")
@Data
public class HelloProperties {
    private String prefix;
    private String suffix;
}
```

然后编写一个业务类实现具体的业务逻辑：从Spring容器中获取helloProperties，然后利用这个配置项在名称前后加上前缀和后缀

```java
package com.lth.bean;

import javax.annotation.Resource;

public class HelloService {
    @Resource
    HelloProperties helloProperties;
    public String helloWorld(String name){
        return helloProperties.getPrefix()+" name "+helloProperties.getSuffix();
    }
}
```

但是此时helloProperties并不在Spring容器中，HelloService也不在Spring容器中，我们可以通过编写自动配置类将这两个bean注入到Spring容器中：

```java
@Configuration
//注入配置类
@EnableConfigurationProperties(HelloProperties.class)
public class HelloAutoConfiguration {
    //注入业务类
    @Bean
    public HelloService helloService(){
        return new HelloService();
    }
}
```

@EnableConfigurationPropertie注解用于向Spring容器中添加配置类的bean（也就是向容器中添加一个带有@ConfigurationProperties注解的类的对象），等价于通过@Bean注解向Spring容器添加带有@ConfigurationProperties注解的bean，通过@EnableConfigurationPropertie，@Bean，@Component注解注入的bean都会经过Spring容器的自动装配，相关的注解都会生效。

然后我们使用maven的lifecycle中clean，install将当前项目编译，然后安装到我们的项目中

先安装自动配置类lth-spring-boot-starter-autoconfiguration，再安装我们的lth-spring-boot-starter，因为starter编译需要用到autoconfiguration的jar包，实际上我们需要将starter所引用的jar都编译好，再编译starter进行总体上的打包

测试：

```java
@SpringBootTest
class DemoApplicationTests {

    @Resource
    HelloService helloService;

    @Test
    void contextLoads() {
        System.out.println(helloService.helloWorld("LTH"));
    }
}
```

properties配置文件：

```properties
lth.hello.prefix=hello
lth.hello.suffix=come on
```

输出hello name come on，代表成功

## 补充：IOC容器的创建流程

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
        //上锁
		synchronized (this.startupShutdownMonitor) {
            //通知监听器开始创建IOC容器
			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

			//创建容器前的预处理
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end();

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
				contextRefresh.end();
			}
		}
	}
```

### 1. 预处理前的初始化prepareRefresh()

```java
	protected void prepareRefresh() {
		//记录时间
		this.startupDate = System.currentTimeMillis();
        //设置状态，表示激活IOC容器
		this.closed.set(false);
		this.active.set(true);
		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment.
        //初始化属性设置(默认为空，我们可以重写这个方法)
		initPropertySources();

		//验证一些必须的属性是否合法
		getEnvironment().validateRequiredProperties();

		//将早期事件监听器注册为监听器，并清空早期事件
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```

### 2.创建bean工厂beanFactories

### ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()

```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		//创建bean工厂
		refreshBeanFactory();
        //获取刚才创建的bean工厂并返回
		return getBeanFactory();
	}
```

创建的beanFactory的类型是DefaultListableBeanFactory，也就是默认bean工厂

### 3.准备bean工厂prepareBeanFactory(beanFactory)

在这个方法中，向bean工厂设置一些属性

```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
        //设置类加载器
		beanFactory.setBeanClassLoader(getClassLoader());
        //设置表达式解析器
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

#### 1.设置加载bean所需的工具类

```java
//类加载器
beanFactory.setBeanClassLoader(getClassLoader());
//表达式解析器
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
//属性编辑器
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
```

#### 2.设置一些回调方法

```java
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

放入ApplicationContextAwareProcessor（添加部分beanPostProcessor）

忽略以这些接口创建的bean：EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware，ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware

#### 3.设置可以通过自动装配获取的bean（@Autowire，@Resource）

```
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

可以通过自动装配拿到BeanFactory（bean工厂），ResourceLoader（资源加载器），ApplicationEventPublisher（事件推送器），ApplicationContext（IOC容器）

#### 4.注册ApplicationListenerDetector

```
// Register early post-processor for detecting inner beans as ApplicationListeners.
beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
```

#### 5.添加AspectJ动态代理的支持

#### 6.注册和环境（系统属性，环境变量）相关的组件

### 4.进行bean工厂创建完成后的后置处理

postProcessBeanFactory(beanFactory)

这个方法默认为空，我们重写这个方法，在beanFactory加载完成后进行一些操作

====================================通过以上方法完成了beanFactory的创建和预处理工作=========================

### 5.执行所有的BeanFactoryPostProcessors

invokeBeanFactoryPostProcessors

在beanFactory标准初始化完成后执行这个这个方法

两个接口：

```java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

#### 1.执行所有BeanFactoryPostProcessors

##### 1.获取所有的BeanFactoryPostProcessor

##### 2.优先执行实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessor

##### 3.然后执行实现了Order接口的BeanDefinitionRegistryPostProcessor

##### 4.执行剩下的BeanDefinitionRegistryPostProcessor

##### 5.获取所有的BeanFactoryPostProcessor

##### 6.依次执行实现了PriorityOrdered，Order，没实现接口的BeanFactoryPostProcessor

### 6.注册bean的后置处理器

registerBeanPostProcessors(beanFactory);

也是依次注册实现了PriorityOrdered接口，实现了Order接口，没有实现任何接口的BeanFactoryPostProcessor

然后注册MergedBeanDefinitionPostProcessor和ApplicationListenerDetector

### 7.初始化消息（消息绑定，消息解析）

initMessageSource

如果容器中有MessageSource，则赋值给MessageSource，如果没有则自己创建一个默认的对象

MessageSource：取出某个key的值，安装区域获取值

然后将MessageSource注册进Spring容器中，然后我们就能通过自动装配得到MessageSource

### 8.初始化事件派发器

initApplicationEventMulticaster()

1.获取BeanFactory

2.从容器中获取applicationEventMulticaster，如果没有就创建一个SimpleApplicationEventMulticaster并注册进Spring容器

### 9.刷新容器onRefresh()

onRefresh()默认为空，留给我们来实现

### 10.注册事件派发器

获取所有的事件监听器，去重后将所有的监听器注册进事件派发器

派发之前步骤产生的事件earlyApplicationEvents

### 11.初始化所有所有单实例bean

```
finishBeanFactoryInitialization
```

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no BeanFactoryPostProcessor
		// (such as a PropertySourcesPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

这个方法的核心语句是beanFactory.preInstantiateSingletons()，预加载单实例bean

```java
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

        //拿到所有bean的定义信息
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

#### 1.preInstantiateSingletons

##### 1.拿到扫描路径下所有带有@Controller，@Service，@Repository，@Configuration，@Component等向Spring容器中注册组件的注解的类的信息

![image-20220512223534964](pictures/189200b12b1d1c8ec72936c1539a4d80.png)

如上图所示，包含Spring容器中默认加载的组件和我们自己编写的

##### 2.遍历所有的bean的全限定名，创建和初始化对应的对象

- 拿到一个类的全限定名beanName
- 获取这个类的定义信息RootBeanDefinition

RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName)

![image-20220512232616126](pictures/f0b2393a61b48e7f903b136fab5d1a19.png)

- 如果这个bean不是抽象的，也不是单实例的，也不是懒加载的
    - 然后判断是不是FactoryBean
    - 如果是FactoryBean，则使用FactoryBean的getObect方法创建bean
    - 如果不是FactoryBean，则使用getBean方法创建对象，getBean调用下面的doGetBean方法



```java
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
      throws BeansException {
   //拿到bean的名称
   String beanName = transformedBeanName(name);
   Object bean;

   //从缓存中获取单实例bean，如果能获取到说明已经被创建过了
   Object sharedInstance = getSingleton(beanName);
    //如果缓存中拿不到(不是调用了beanFactory创建bean了吗为什么拿不到，这里先伏笔一下)
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      //拿到父工厂(如果有的话)
      BeanFactory parentBeanFactory = getParentBeanFactory();
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else if (requiredType != null) {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
         else {
            return (T) parentBeanFactory.getBean(nameToLookup);
         }
      }

      if (!typeCheckOnly) {
         //标记当前bean已经被创建了，防止多个线程创建bean
         markBeanAsCreated(beanName);
      }

      try {
         //获取bean的定义信息
         RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
          //获取当前bean依赖的其他bean
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            //如果当前有依赖的bean，则遍历所有依赖的bean,创建所有依赖的bean
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               registerDependentBean(dep, beanName);
               try {
                  //尝试获取或者创建所依赖的bean(这里发生了递归)
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         // Create bean instance.
         //如果这是一个单实例bean，则采用单实例bean的创建方法
         if (mbd.isSingleton()) {
            //调用getSingleton方法(上面也调用这个方法)创建或者从一二级缓存中获取bean，这里的lamda表达式省略的是beanFactory的getObject方法
            sharedInstance = getSingleton(beanName, () -> {
               try {
                   //调用createBean方法创建bean
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
               beforePrototypeCreation(beanName);
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         else {
            String scopeName = mbd.getScope();
            if (!StringUtils.hasLength(scopeName)) {
               throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
            }
            Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
               Object scopedInstance = scope.get(beanName, () -> {
                  beforePrototypeCreation(beanName);
                  try {
                     return createBean(beanName, mbd, args);
                  }
                  finally {
                     afterPrototypeCreation(beanName);
                  }
               });
               bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // Check if required type matches the type of the actual bean instance.
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         if (logger.isTraceEnabled()) {
            logger.trace("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```

**核心方法doGetBean**

1.从缓存中获取单实例bean，如果能获取到说明已经被创建过了

Object sharedInstance = getSingleton(beanName)

（单例设计模式）

```java
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		//尝试从一级缓存中拿到bean
		Object singletonObject = this.singletonObjects.get(beanName);
        //如果没有拿到
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            //再尝试从二级缓存中找
			singletonObject = this.earlySingletonObjects.get(beanName);
            //如果二级缓存中也没有找到，并且允许提前创建bean
			if (singletonObject == null && allowEarlyReference) {
                //锁住一级缓存(单例模式)
				synchronized (this.singletonObjects) {
					//再尝试从一级缓存中找
					singletonObject = this.singletonObjects.get(beanName);
                    //如果一级缓存中没有找到
					if (singletonObject == null) {
                        //从二级缓存中找
						singletonObject = this.earlySingletonObjects.get(beanName);
                        //如果二级缓存中没有找到
						if (singletonObject == null) {
                            //从三级中找到对应的beanFactory，准备执行创建的bean的流程
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                            //如果找到了beanFactory
							if (singletonFactory != null) {
                                //使用beanFactory创建bean
								singletonObject = singletonFactory.getObject();
                                //将这个bean放入二级缓存
								this.earlySingletonObjects.put(beanName, singletonObject);
                                //从三级缓存中移除beanFactory
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}
```

singletonObjects：一级缓存：单例池

```
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

其实就是一个线程安全的map的

earlySingletonObjects：二级缓存，用于保存半成品的bean

```
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
```

同样是一个线程安全的map

singletonFactories：三级缓存，用于保存bean工厂

```
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

同样是一个线程安全的map，但是保存的是 ObjectFactory<?>

**核心方法createBean**

```java
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
            //给这个bean一个返回代理对象的机会
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
            //如果没有返回代理对象，则创建bean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

**创建bean：doCreateBean**

```java
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
            //创建bean实例
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
        //加上锁，防止多次后置处理，确保只处理一次
		synchronized (mbd.postProcessingLock) {
            //如果没有被后置处理
			if (!mbd.postProcessed) {
				try {
                    //执行一些后置处理器
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
                //标志位已经被后置处理
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
        //第二级缓存能处理循环依赖，及时有了生命周期的处理方法
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            //为bean赋值
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
            //获取早期保留的bean的引用
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
            //注册bean的销毁
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}
		//返回创建好的bean
		return exposedObject;
	}
```

创建对象实例 createBeanInstance

```java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
        //获取当前的bean是什么类型
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
        //如果是实例bean(@Component)
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		//如果是用@bean注解创建
		if (mbd.getFactoryMethodName() != null) {
            //利用对象的构造器创建bean实例
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		return instantiateBean(beanName, mbd);
	}
```

属性赋值populateBean



![image-20220513101910052](pictures/059bb96aa2f5fd724bcaea38a591f7c6.png)

### 12.完成beanFactory的创建工作

```java
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
        //初始化
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

#### 1.初始化LifecycleProcessor（需要我们来实现）

#### 2.执行getLifecycleProcessor().onRefresh();

#### 3.发布容器创建完成事件

publishEvent(new ContextRefreshedEvent(this))