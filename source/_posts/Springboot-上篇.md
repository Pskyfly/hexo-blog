---
title: Springboot(上篇)
date: 2022-11-30 18:53:29
tags:
---
## Springboot核心功能（2.2.4）

### 配置文件

#### Yaml语法

properties的优先级高于yml

- key: value；kv之间有空格
- 大小写敏感
- 使用缩进表示层级关系
- 缩进不允许使用tab，只允许空格
- 缩进的空格数不重要，只要相同层级的元素左对齐即可
- '#'表示注释
- 字符串无需加引号，如果要加，单引号’’、双引号""表示字符串内容会被 转义、不转义

kv表示：k: v 

注意要有空格

数组可以用y: [xx,xxx]来表示

也可以用

```
y: 
  - xx
  - xxx
```

`-`代表集合中的一个元素

可以用一个类和配置文件来绑定

```java
package com.demo.test;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import java.util.*;

@Data
@Configuration
@ConfigurationProperties(prefix = "person")
public class Person {
    private String userName;
    private Boolean boss;
    private Date birth;
    private Integer age;
    private Pet pet;
    private String[] interests;
    private List<String> animal;
    private Map<String, Object> score;
    private Set<Double> salarys;
    private Map<String, List<Pet>> allPets;
}

@Data
class Pet {
    private String name;
    private Double weight;
}

```

在yml配置对应的属性：

```yml
person:
  userName: zhangsan
  boss: false
  birth: 2019/12/12 20:12:33
  age: 18
  pet: 
    name: tomcat
    weight: 23.4
  interests: [篮球,游泳]
  animal: 
    - jerry
    - mario
  score:
    english: 
      first: 30
      second: 40
      third: 50
    math: [131,140,148]
    chinese: {first: 128,second: 136}
  salarys: [3999,4999.98,5999.99]
  allPets:
    sick:
      - {name: tom}
      - {name: jerry,weight: 47}
    health: [{name: mario,weight: 47}]
```

#### 编写配置文件时，添加提示

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>

<!-- 下面插件作用是工程打包时，不将spring-boot-configuration-processor打进包内，让其只在编码的时候有用 -->
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-configuration-processor</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

大写字母等价于小写字母前加上- 也就是：N 和-n的意义相同

### Web开发

Springboot框架是框架的框架

#### 静态资源

Springboot默认的静态资源目录是在resources目录下的：

/static

/public 

/resources  

/META-INF/resources

这些目录静态资源都可以直接访问

例如：http://localhost:8080/123.png

如果是他们在他们的子目录下，则需要加上子目录的包名

##### 请求顺序

在请求进来时，先判断Controller能不能处理，如果不能处理再交给静态资源处理器来处理，否则返回404

##### 配置静态资源的访问前缀

访问静态资源默认是没有前缀的，但是实际上我们需要加上前缀来对资源进行一些个性化的拦截（登录拦截动态资源，而为静态资源放行）

设置静态资源前缀：

```yml
spring:
  mvc:
    static-path-pattern: /res/**
```

表示和这个正则表达式匹配的可以由静态资源处理器来处理

##### 设置静态资源的目录

```yml
spring:
  resources:
    static-locations: [classpath:/static/,classpath:/static/img/]
```
![在这里插入图片描述](../../../../学习笔记/picture/13440a06372548d791163c0170b582af.png)


底层是一个String数组，所以我们采用数组（列表）的写法

webjars：用于编写web应用的jar包（例如JQuery）

在pom引入后，可以在webjars/目录下访问

##### 欢迎页

如果静态目录下有index.html页面，访问`http://localhost:8080/`也就是项目路径时，会默认显示index.html页面

但是如果配置了

```
spring:
  mvc:
    static-path-pattern: /res/**
```

会让欢迎页功能失效，也会让图标功能失效

##### 图标功能

在静态目录下添加favicon.ico作为所有页面的图标，然后用ctrl+F5强制刷新并清空缓存可以看到效果

##### 静态资源访问底层原理

```java
		@Override
		public void addResourceHandlers(ResourceHandlerRegistry registry) {
			if (!this.resourceProperties.isAddMappings()) {
				logger.debug("Default resource handling disabled");
				return;
			}
			Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
			CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
			if (!registry.hasMappingForPattern("/webjars/**")) {
				customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
						.addResourceLocations("classpath:/META-INF/resources/webjars/")
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
			String staticPathPattern = this.mvcProperties.getStaticPathPattern();
			if (!registry.hasMappingForPattern(staticPathPattern)) {
				customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
						.addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
		}
```

`resourceProperties.isAddMappings() `对应配置：

```yml
spring:
  resources:
    add-mappings: false
```

从源码可知，如果配置成了false，后面的逻辑都不会执行，也就禁用了静态资源的访问功能（默认是true）

`Duration cachePeriod = this.resourceProperties.getCache().getPeriod();`

这条语句用于获取配置：

```yml
spring:
  resources:
    cache:
      period: 11000
```

也就是设置静态资源的缓存时间，在这段时间内不用再重新加载静态资源，可以直接从浏览器缓存中获取，单位是秒

通过缓存拿到的资源状态码会显示304

webjars访问规则：

```java
			if (!registry.hasMappingForPattern("/webjars/**")) {
				customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
						.addResourceLocations("classpath:/META-INF/resources/webjars/")
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
```

如果在Controller中没有设置`/webjars/**`的路由，就在访问带有webjars的前缀时，访问classpath:/META-INF/resources/webjars/这个目录下的资源，同时设置缓存时间和缓存控制

##### 静态资源访问规则：

```java
			String staticPathPattern = this.mvcProperties.getStaticPathPattern();
			if (!registry.hasMappingForPattern(staticPathPattern)) {
				customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
						.addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
```

如果在Controller中没有设置staticPathPattern的url访问规则，则在访问staticPathPattern规则下的资源时，访问this.resourceProperties.getStaticLocations()路径下对应静态资源，同时设置缓存时间。

staticPathPattern：
![(img-3hCK5Zyu-1653538758227)(D:/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/picture/image-20220429154625565.png)\]](../../../../学习笔记/picture/2828370373d4434e8af200e8b5402cb5.png)

这就解释了为什么静态资源访问的url是/ 而没有前缀

staticLocations：
![在这里插入图片描述](../../../../学习笔记/picture/c3e55103057046e6ad8e10110499c585.png)




这就也就静态资源默认路径的由来，如果我们进行了配置，staticLocations就会被更新为配置文件中的值。

关于欢迎页：

```java
WelcomePageHandlerMapping(TemplateAvailabilityProviders templateAvailabilityProviders,
			ApplicationContext applicationContext, Optional<Resource> welcomePage, String staticPathPattern) {
		if (welcomePage.isPresent() && "/**".equals(staticPathPattern)) {
			logger.info("Adding welcome page: " + welcomePage.get());
			setRootViewName("forward:index.html");
		}
		else if (welcomeTemplateExists(templateAvailabilityProviders, applicationContext)) {
			logger.info("Adding welcome page template: index");
			setRootViewName("index");
		}
	}
```

`"/**".equals(staticPathPattern)`我们可以看到，只有在静态路径没有被配置时，欢迎页才会生效

### Restful风格开发

对于原生的HTML中的form元素没有PUT和DELETE方法，可以使用post方法模拟这两个请求（如果用一些能直接发这两种请求的工具则不需要以下流程，因为在HTTP层就已经是PUT和DELETE了，所以这一项是选择性开启）

WebMvcAutoConfiguration类中有这样的一段配置：

```java
	@Bean
	@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
	@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = false)
	public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
		return new OrderedHiddenHttpMethodFilter();
	}
```

注意到@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = false)，需要我们在配置文件中，将spring.mvc.hiddenmethod.filter 设置为enabled 才可以：

```yml
spring:
  mvc:
    hiddenmethod:
      filter:
        enabled: true
```

前端需要添加隐藏参数_method，才能使用PUT方法和DELETE方法：

```html
<form action="/user" method="get">
    <input value="REST-GET提交" type="submit" />
</form>

<form action="/user" method="post">
    <input value="REST-POST提交" type="submit" />
</form>

<form action="/user" method="post">
    <input name="_method" type="hidden" value="DELETE"/>
    <input value="REST-DELETE 提交" type="submit"/>
</form>

<form action="/user" method="post">
    <input name="_method" type="hidden" value="PUT" />
    <input value="REST-PUT提交"type="submit" />
<form>

```

Rest原理（表单提交要使用REST的时候）

```java
	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		HttpServletRequest requestToUse = request;

		if ("POST".equals(request.getMethod()) && request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) == null) {
			String paramValue = request.getParameter(this.methodParam);
			if (StringUtils.hasLength(paramValue)) {
				String method = paramValue.toUpperCase(Locale.ENGLISH);
				if (ALLOWED_METHODS.contains(method)) {
					requestToUse = new HttpMethodRequestWrapper(request, method);
				}
			}
		}

		filterChain.doFilter(requestToUse, response);
	}
```

在执行拦截器前，会先获取到我们_method字段的参数，然后根据这个字段重新设置我们的请求方法，然后生成一个HttpServletRequest的包装类（这个类也实现了HttpServletRequest接口），然后将原来的request和新设置的方法传进去，从而完成方法的替换，然后再去执行接下来的逻辑。

使用@GetMapping("/") @PostMapping("/") 等更方便

### 请求映射原理

DispatcherServlet实现了HttpServlet接口，所以本质上就是一个Servlet，而Servlet的功能就是接收从服务器发送来的请求，并予以返回值的框架。

DispatcherServlet里面实现了doGet，doPost等方法，这些方法都会调用processRequest方法，在这个方法中调用doService方法，在doService方法再调用doDispatch方法，而处理请求的核心代码就在这个方法中。

```java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
                //判断是不是文件上传请求
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

**mappedHandler = getHandler(processedRequest)**

根据请求获取对应url的处理器

```java
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

遍历容器中所有的HandlerMapping，找到第一个能处理这个请求的handler并返回

Spring帮我们注册的handler有欢迎页的handler，我们在Controller定义的handler，以及我们自定义的handler

先根据URL找到URL匹配的处理器handler，先找到方法也匹配的handler，如果有多个匹配则报错

### Springboot参数注解

#### @PathVariable  路径参数

将路径的一部分作为参数/user/{id}
数字或者字符串变量，加上这个注解后可以获取到路径参数中对应的名称。如果是一个Map型变量加上了这个注解则会将所有参数以kv的形式传入到这个Map中

#### @RequestHeader  请求头参数

可以拿到请求头中对应的参数，如果参数类型是Map,MultiValueMap,HttpHeaders则会拿到所有的请求头参数

#### @RequestParam  请求参数

用来获取路由参数
例如/user?age=13
如果等号左边有相同的值，则会以列表的形式读取进来
如果参数列表是Map类型，则会将所有方法参数都读进来，类型是String,String或者String,Object

#### @CookieValue  Cookie参数

可以获取指定Cookie的值
参数类型可以是String，也可以是Cookie类型的变量，用getName和getValue来获取KV的值

#### @RequestBody  请求体

获取请求体中的所有参数，如果参数类型是String会把参数url原样拿过来，如果是其他对象类型，会将参数按照属性名装配进去后返回

#### @RequestAttribute 请求域参数

设置获取请求域的参数，请求域的参数可以通过request的setAttribute来设置，也可以用过getAttribute来获取，在进行路由转发的时候可以使用这种方式传递参数，转发方式:
return "forward:/success" 在forward后面设置转发的路由，这样可以让多个路由映射到同一个功能上

#### @MatrixVariable  矩阵变量

```
/cars/sell;low=34;brand=byd,audi,yd
```

URL中还可以通过矩阵变量传递参数，每个参数用分号`;`分割，List类型的参数可以直接用逗号`,`分割，相同参数会被封装成一个list

在参数中加上这个注解@MatrixVariable来获取值

每个矩阵变量依附于它前面的路由变量，每个路径变量都可以有一个一系列矩阵变量，可以通过设置@MatrixVariable中的pathVar属性来获取指定变量参数后面的矩阵变量

```java
@RestController
public class ParameterTestController {

    ///cars/sell;low=34;brand=byd,audi,yd
    @GetMapping("/cars/{path}")
    public Map carsSell(@MatrixVariable("low") Integer low,
                        @MatrixVariable("brand") List<String> brand,
                        @PathVariable("path") String path){
        Map<String,Object> map = new HashMap<>();

        map.put("low",low);
        map.put("brand",brand);
        map.put("path",path);
        return map;
    }

    // /boss/1;age=20/2;age=10

    @GetMapping("/boss/{bossId}/{empId}")
    public Map boss(@MatrixVariable(value = "age",pathVar = "bossId") Integer bossAge,
                    @MatrixVariable(value = "age",pathVar = "empId") Integer empAge){
        Map<String,Object> map = new HashMap<>();
        map.put("bossAge",bossAge);
        map.put("empAge",empAge);
        return map;
    }
}
```

Springboot禁用了矩阵变量的功能，需要我们手动开启

原因：

```java
		@Override
		@SuppressWarnings("deprecation")
		public void configurePathMatch(PathMatchConfigurer configurer) {
			configurer.setUseSuffixPatternMatch(this.mvcProperties.getPathmatch().isUseSuffixPattern());
			configurer.setUseRegisteredSuffixPatternMatch(
					this.mvcProperties.getPathmatch().isUseRegisteredSuffixPattern());
			this.dispatcherServletPath.ifAvailable((dispatcherPath) -> {
				String servletUrlMapping = dispatcherPath.getServletUrlMapping();
				if (servletUrlMapping.equals("/") && singleDispatcherServlet()) {
					UrlPathHelper urlPathHelper = new UrlPathHelper();
					urlPathHelper.setAlwaysUseFullPath(true);
					configurer.setUrlPathHelper(urlPathHelper);
				}
			});
		}
```

路由匹配在上述方法中进行，而路由解析需要用到UrlPathHelper，而在UrlPathHelper中：

```java
/**
	 * Set if ";" (semicolon) content should be stripped from the request URI.
	 * <p>Default is "true".
	 */
	public void setRemoveSemicolonContent(boolean removeSemicolonContent) {
		checkReadOnly();
		this.removeSemicolonContent = removeSemicolonContent;
	}
```

上面提到如果removeSemicolonContent这个变量是true，则会移除我们分号后面的内容，因而我们获取不到矩阵参数

所以我们在组件中设置一个这个变量为false的组件即可：

可以单独写一个类实现接口，JDK8有接口默认方法，所以们不用实现所有的类：

```java
@Configuration(proxyBeanMethods = false)
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {

        UrlPathHelper urlPathHelper = new UrlPathHelper();
        // 不移除；后面的内容。矩阵变量功能就可以生效
        urlPathHelper.setRemoveSemicolonContent(false);
        configurer.setUrlPathHelper(urlPathHelper);
    }
}
```

也可以在配置类中用@Bean注入：

```java
@Configuration(proxyBeanMethods = false)
public class WebConfig{
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {
                        @Override
            public void configurePathMatch(PathMatchConfigurer configurer) {
                UrlPathHelper urlPathHelper = new UrlPathHelper();
                // 不移除；后面的内容。矩阵变量功能就可以生效
                urlPathHelper.setRemoveSemicolonContent(false);
                configurer.setUrlPathHelper(urlPathHelper);
            }
        }
    }
}
```

（但是重写这个方法的话，其他方法怎么办呢……可能Spring还做了一些其他的事情……，不过我们知道这个怎么配置，大致的原因是什么即可）

#### Springboot参数注解原理

```java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
                //判断是不是文件上传请求
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

##### 获取handler  

mappedHandler = getHandler(processedRequest);

获取能处理这个请求的handler，而所谓的handler就是在Controller中通过URL找到的对应的方法，拿到方法的各种信息。

##### 获取适配器

HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

获取能处理这个handler的适配器，适配器用于解析上述各种注解的参数，相当于一个大的反射工具

HandlerAdapter 里面有这些方法：

```java
public interface HandlerAdapter {
	//是否能处理这个handler
	boolean supports(Object handler);
    /*
    对应的实现类，直接比较是不是我们想要的类型的对象
    @Override
	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}
    
    */
    
    
	//如果能处理则调用这个方法处理请求
	@Nullable
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	long getLastModified(HttpServletRequest request, Object handler);

}
```

获取对应的handlerAdapter的方法：

```java
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

遍历容器中注册的所有HandlerAdapter，找到能支持这个handler的HandlerAdapter

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-uZvIvF2E-1653538758228)(D:/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/picture/image-20220430192846168.png)]

RequestMappingHandlerAdapter ：用于处理Controller的方法中带哟@RequestMapping注解的方法（也就是我们所写的普通方法）

HandlerFunctionAdapter ：用于处理函数式编程的方法对应的Controller

##### 浏览器缓存

```java
    String method = request.getMethod();
    boolean isGet = "GET".equals(method);
    if (isGet || "HEAD".equals(method)) {
        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
        if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
            return;
        }
    }
```

判断是不是GET方法或者方法，这个String method是我们之前设置的方法名（回顾之前用内置参数模拟PUT,DELETE方法，这里的HEAD方法也是这样），如果是HEAD方法则直接返回（并不是真正的请求），如果是GET方法的则判断静态资源最后的修改时间，如果没有修改则提示客户端可以从浏览器缓存中获取静态资源

##### 执行方法

```
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

返回值是视图解析器(->代表调用)

```
handle -> handleInternal->invokeHandlerMethod
```

invokeHandlerMethod方法中

根据不同的类型的注解解析参数，并设置参数的值：

```
if (this.argumentResolvers != null) {
    invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
}
```

解析返回值：

```
if (this.returnValueHandlers != null) {
    invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
}
```

准备工作完成，真正执行方法：

```
invocableMethod.invokeAndHandle(webRequest, mavContainer);
```

在这个方法中调用**invokeForRequest**方法：

```java
	@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Arguments: " + Arrays.toString(args));
		}
		return doInvoke(args);
	}
```

第一条语句：

```
Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
```

用于获取这个方法所有所需的参数

```java
	protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		//拿到所有参数的信息，但是此时参数还没有值
		MethodParameter[] parameters = getMethodParameters();
		if (ObjectUtils.isEmpty(parameters)) {
			return EMPTY_ARGS;
		}
		//创建等大的数组作为参数列表，准备设置值
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
            //判断在所有的视图解析器中是否有能够处理这个参数的解析器
			if (!this.resolvers.supportsParameter(parameter)) {
				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
			}
			try {
            //解析参数的值
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				// Leave stack trace for later, exception may actually be resolved and handled...
				if (logger.isDebugEnabled()) {
					String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
						logger.debug(formatArgumentError(parameter, exMsg));
					}
				}
				throw ex;
			}
		}
		return args;
	}
```

**this.resolvers.supportsParameter(parameter)** //判断在所有的视图解析器中是否有能够处理这个参数的解析器

判断方法是看是否能找到合适的视图解析器：

```
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return getArgumentResolver(parameter) != null;
	}
```

查找过程getArgumentResolver：

先判断缓存map里面有没有，如果有就直接拿到，如果没有则遍历所有的视图解析器，判断是否支持解析这个参数，如果支持则放入缓存中并返回这个视图解析器。

判断方法：1. 是否有对应的参数注解 2.参数类型是否满足要去 3.其他

```java
@Nullable
	private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		if (result == null) {
			for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
				if (resolver.supportsParameter(parameter)) {
					result = resolver;
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```

**args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory)** 解析参数的值

进入后来到可以来到：

```java
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
		Map<String, String> uriTemplateVars = (Map<String, String>) request.getAttribute(
				HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
		return (uriTemplateVars != null ? uriTemplateVars.get(name) : null);
	}
```

这个方法用于获取参数的值，不同注解的的解析器有不同的实现，上面这个是@PathVariable参数注解的解析器。

之前我们看到Springboot用urlPathHelper解析了URL中的各种参数，解析后Springboot会将其放到HttpServletRequest的请求域中，然后再这里直接根据参数名从请求域中获取参数的值

获取后回到原来的方法中，设置参数的值

##### 进行一些善后处理

mappedHandler.applyPostHandle(processedRequest, response, mv);

##### 处理最后的结果

也就设置最后要去哪个页面，需要处理哪些参数

processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

### Servlet API

Springboot给Servlet API类型的参数赋值时，解析用的方法和上面加了注解的参数一致，只是用的参数解析器不同。这里用的参数解析器只用判断参数的类型即可，如果是指定的类型比如HttpServletRequest 类型，他就会封装出一个对应的请求对象并进行引用赋值。

### 复杂参数

Map，Model类型的参数对应HttpServletRequest的请求域，操作这两个参数（map.put）就相当于操作request的请求域(request.setAttribute)

```java
@GetMapping("/params")
public String testParam(Map<String,Object> map,
                        Model model,
                        HttpServletRequest request,
                        HttpServletResponse response){
    //下面三位都是可以给request域中放数据
    map.put("hello","world666");
    model.addAttribute("world","hello666");
    request.setAttribute("message","HelloWorld");

    Cookie cookie = new Cookie("c1","v1");
    response.addCookie(cookie);
    return "forward:/success";
}

@ResponseBody
@GetMapping("/success")
public Map success(@RequestAttribute(value = "msg",required = false) String msg,
                   @RequestAttribute(value = "code",required = false)Integer code,
                   HttpServletRequest request){
    Object msg1 = request.getAttribute("msg");

    Map<String,Object> map = new HashMap<>();
    Object hello = request.getAttribute("hello");//得出testParam方法赋予的值 world666
    Object world = request.getAttribute("world");//得出testParam方法赋予的值 hello666
    Object message = request.getAttribute("message");//得出testParam方法赋予的值 HelloWorld

    map.put("reqMethod_msg",msg1);
    map.put("annotation_msg",msg);
    map.put("hello",hello);
    map.put("world",world);
    map.put("message",message);

    return map;
}
```

response可以方Cookie

#### Map，Model

底层都会调用ModelAndViewContainer的getModel方法获取到一个MAP型的变量，因而在经过参数解析器解析后，这两个指向的对象实际上是同一个

ModelAndViewContainer 故名意思就是模型和视图的容器，Model用于存放数据，View用于存放视图（页面的地址），这两个都在这个容器中

这两个参数操作的是request中请求域的参数，而这两个类型的参数是怎么操作请求域的呢？

解析参数的时候如果参数类型是Map或者Model，则会创建一个BindingAwareModelMap变量来装载请求域中的参数，这个类既是Map也是Model，所以可以完成赋值。

![](../../../../学习笔记/picture/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE4NjMwMjQ=,size_16,color_FFFFFF,t_70#pic_center.png)

请求结束后，我们再来到具体的逻辑：

```java
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		//执行方法，得到返回值是"forward:/success"
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
        //设置请求状态
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				disableContentCachingIfNecessary(webRequest);
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
            //处理返回值,这里有我们要的转发逻辑，里面会传入我们方法的返回值returnValue
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}
```

我们深入handleReturnValue方法可以来到这个方法里面：

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		//判断返回值是不是字符串
		if (returnValue instanceof CharSequence) {
			String viewName = returnValue.toString();
            //如果是字符串则设置容器中view的名称(转发路径)
			mavContainer.setViewName(viewName);
			if (isRedirectViewName(viewName)) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		else if (returnValue != null) {
			// should not happen
			throw new UnsupportedOperationException("Unexpected return type: " +
					returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
		}
	}
```

在方法执行完成后我们会得到一个ModelAndView对象：

```
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

mv里面包含我们想要的数据（model）和转发的地址（view）

然后传入到

```
mappedHandler.applyPostHandle(processedRequest, response, mv);
```

进行最后结果的处理

深入这个方法后来到：

```
render(mv, request, response);
```

这个方法用于渲染页面

核心逻辑是：

封装成视图对象：

```java
view = resolveViewName(/*视图名*/viewName,/*视图数据*/ mv.getModelInternal(), locale, request);
```

然后渲染视图：

```java
view.render(mv.getModelInternal(), request, response);
/////////////////////////////////////////////////////////////
//这个方法的逻辑是：
@Override
	public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {

		if (logger.isDebugEnabled()) {
			logger.debug("View " + formatViewName() +
					", model " + (model != null ? model : Collections.emptyMap()) +
					(this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
		}
		//这一步，将我们model中的数据放到一个新的map里面mergedModel
		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
		prepareResponse(request, response);
		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
	}
```

renderMergedOutputModel方法中会执行语句：exposeModelAsRequestAttributes(model, request);

```java
	protected void exposeModelAsRequestAttributes(Map<String, Object> model,
			HttpServletRequest request) throws Exception {

		model.forEach((name, value) -> {
			if (value != null) {
				request.setAttribute(name, value);
			}
			else {
				request.removeAttribute(name);
			}
		});
	}
```

显然这个方法的作用就是将model中的数据放到新的request的请求域中，这就解释了我们转发请求后，为啥新的方法中能拿到上一个请域的参数

### Springboot自定义参数

```java
@RestController
public class ParameterTestController {
    @PostMapping("/saveuser")
    public Person saveuser(Person person){
        return person;
    }
}
```

参数列表是我们自定义的对象时，Spring会自动帮我们将参数按照参数名装配进去（如果包含了其他引用类型要用pet.name,pte.age的形式传过来才能解析）

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-n0KSzKyJ-1653538758229)(D:/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/picture/image-20220502111900935.png)]

参数解析过程和前面讲的一样，只是用的参数解析器不同，这里用的参数解析器是`ServletModelAttributeMethodProcessor`

能用这个的处理器的条件是加了@ModelAttribute的注解或者它不是简单数据类型（即是引用类型）

```java
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
				(this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
	}
```

然后给Person参数的赋值过程如下：

```java
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		Assert.state(mavContainer != null, "ModelAttributeMethodProcessor requires ModelAndViewContainer");
		Assert.state(binderFactory != null, "ModelAttributeMethodProcessor requires WebDataBinderFactory");

		String name = ModelFactory.getNameForParameter(parameter);
		ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
		if (ann != null) {
			mavContainer.setBinding(name, ann.binding());
		}

		Object attribute = null;
		BindingResult bindingResult = null;
		//如果请求域中已经有了就直接返回
		if (mavContainer.containsAttribute(name)) {
			attribute = mavContainer.getModel().get(name);
		}
		else {
			// Create attribute instance
			try {
                //根据对象属性创建一个空对象（也就是上文中属性值为null的对象）
				attribute = createAttribute(name, parameter, binderFactory, webRequest);
			}
			catch (BindException ex) {
				if (isBindExceptionRequired(parameter)) {
					// No BindingResult parameter -> fail with BindException
					throw ex;
				}
				// Otherwise, expose null/empty value and associated BindingResult
				if (parameter.getParameterType() == Optional.class) {
					attribute = Optional.empty();
				}
				bindingResult = ex.getBindingResult();
			}
		}

		if (bindingResult == null) {
			// Bean property binding and validation;
			// skipped in case of binding failure on construction.
            //为我们刚才创建的空对象绑定属性
			WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
			if (binder.getTarget() != null) {
				if (!mavContainer.isBindingDisabled(name)) {
                    //实际绑定属性
					bindRequestParameters(binder, webRequest);
				}
				validateIfApplicable(binder, parameter);
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new BindException(binder.getBindingResult());
				}
			}
			// Value type adaptation, also covering java.util.Optional
			if (!parameter.getParameterType().isInstance(attribute)) {
				attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
			}
			bindingResult = binder.getBindingResult();
		}

		// Add resolved attribute and BindingResult at the end of the model
		Map<String, Object> bindingResultModel = bindingResult.getModel();
		mavContainer.removeAttributes(bindingResultModel);
		mavContainer.addAllAttributes(bindingResultModel);

		return attribute;
	}
```

WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);

WebDataBinder 是属性绑定器，这个类里面有各种数据类型之间的转换器，可以利用反射和转换器根据webRequest里面拿到的数据为对象属性赋值

![image-20220501131802770](../../../../学习笔记/picture/2ff7dcf9c81914153c44f0a432d9bc63.png)

实际绑定的过程是在`bindRequestParameters(binder, webRequest);`这个方法里面，这个方法过后对象属性就有值了

`MutablePropertyValues mpvs = new MutablePropertyValues(request.getParameterMap());`通过这个方法拿到request中的所有kv属性值，然后接下来遍历对象中的所有参数，然后根据属性名从这个mpvs里面找就能拿到对应的属性值，但是拿到后还需要将原来的类型（一般是String，也可能是文件流之类的）转换为我们需要的类型，所以在绑定的时候还会遍历所有的属性转换器（Converter），找到可以进行转换的属性转换器，然后将其放入缓存，用转换器来进行属性值的转换，然后就可以为对象中的属性值赋值。

#### 自定义类型转换器

上面使用的都是Springboot提供的转换器，使用Spring为我们提供的转换规则，我们也可以自定义一个转换规则。

```java
@Configuration
public class MyConfig {
    //1、WebMvcConfigurer定制化SpringMVC的功能
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {

            @Override
            public void addFormatters(FormatterRegistry registry) {
                registry.addConverter(new Converter<String, Pet>() {

                    @Override
                    public Pet convert(String source) {
                        // 啊猫,3
                        if(!StringUtils.isEmpty(source)){
                            Pet pet = new Pet();
                            String[] split = source.split(",");
                            pet.setName(split[0]);
                            pet.setAge(Integer.parseInt(split[1]));
                            return pet;
                        }
                        return null;
                    }
                });
            }
        };
    }
}
```

**WebMvcConfigurer**是Spring给我们提供的扩展功能的接口，我们可以重写其中的很多方法来定制化我们想要的功能

在我们添加自定义的转换器后，Springboot在处理参数的时候就可以根据转换前后的参数类型找到能够使用的Converter进行转换，这样就不会报String无法转换成Pet的异常。

并且Converter类带有@FunctionalInterface注解，申明了是一个函数式接口，我们可以直接传入Lamda表达式来进行设置。

### 响应数据与内容协商

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

在这个依赖中会自动帮我们引入JSON的依赖，可以帮我们将返回值处理成JSON格式的数据

在Controller中，如果方法上带有@ResponBody注解，则会将返回值以JSON格式返回给前端

#### 原理解析

我们再来到处理请求的流程里面：

```java
	@Override
	protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ModelAndView mav;
		checkRequest(request);

		// Execute invokeHandlerMethod in synchronized block if required.
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No synchronization on session demanded at all...
            //因为没有session锁，所以我们会来到这个方法中
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
```

mav = invokeHandlerMethod(request, response, handlerMethod)  这个方法的逻辑如下：（其实解析参数的时候我们也进去过）

```java
    if (this.argumentResolvers != null) {
        //传入所有参数解析器
        invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
    }
    if (this.returnValueHandlers != null) {
        //传入所有的返回值处理器
        invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
    }
```

然后来到invokeAndHandle方法来处理请求：

```java
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		//执行方法并拿到返回值（里面的逻辑就是获取参数值和执行controller的方法，在上一节分析过）
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
        //设置请求返回值状态
		setResponseStatus(webRequest);
		//如果返回值为空，则不用处理返回值，直接返回
		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				disableContentCachingIfNecessary(webRequest);
				mavContainer.setRequestHandled(true);
				return;
			}
		}//判断请求处理是否失败，如果失败也不处理返回值直接返回
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
        //重点：处理返回值的方法，参数为返回值，返回值类型，容器，请求
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}
```

处理返回值的方法：

```
this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
```

方法体：

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		//根据返回值和返回值类型获得返回值处理器
		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
        //用返回值处理器处理返回值
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}
```

selectHandler(returnValue, returnType) ：

```java
	@Nullable
	private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}
```

遍历所有的返回值处理器，判断哪个能够用来处理返回值，判断依据大多都是判断返回值类型是不是这个处理器想要的类型，或者有没有对应的注解

SpringMVC能支持的返回值类型有：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-eB7Ul0Pv-1653538758231)(D:/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/picture/image-20220502131627510.png)]

我们在方法上加了@ResponseBody注解，所以使用最后一种处理器

找到返回值处理器后，用处理器处理返回值：

```java
handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
```

方法体：

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		mavContainer.setRequestHandled(true);
        //请求
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
        //响应
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		// Try even with null return value. ResponseBodyAdvice could get involved.
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}
```

其中核心的是：

```
writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
```

方法体：

```java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		Object body;
		Class<?> valueType;
		Type targetType;

		if (value instanceof CharSequence) {
			body = value.toString();
			valueType = String.class;
			targetType = String.class;
		}
		else {
            //获取返回值
			body = value;
            //原类型
			valueType = getReturnValueType(body, returnType);
			targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
		}
		//判断返回值是否是资源文件
		if (isResourceType(value, returnType)) {
			outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
			if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
					outputMessage.getServletResponse().getStatus() == 200) {
				Resource resource = (Resource) value;
				try {
					List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
					outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
					body = HttpRange.toResourceRegions(httpRanges, resource);
					valueType = body.getClass();
					targetType = RESOURCE_REGION_LIST_TYPE;
				}
				catch (IllegalArgumentException ex) {
					outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
					outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
				}
			}
		}

		MediaType selectedMediaType = null;
    	//判断响应中是否已经有了返回类型，如果有就赋值，因为之前可能已经处理了一部分而确定了返回值
		MediaType contentType = outputMessage.getHeaders().getContentType();
		boolean isContentTypePreset = contentType != null && contentType.isConcrete();
    	//如果找到了返回值类型
		if (isContentTypePreset) {
			if (logger.isDebugEnabled()) {
				logger.debug("Found 'Content-Type:" + contentType + "' in response");
			}
			selectedMediaType = contentType;
		}
		else {
            //如果没找到返回类型
            //获得被包装的请求
			HttpServletRequest request = inputMessage.getServletRequest();
            //获得浏览器能接收什么样的媒体类型(text/html之类的)
			List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
            //获得服务器能生产什么样的媒体类型(json之类的)
			List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

			if (body != null && producibleTypes.isEmpty()) {
				throw new HttpMessageNotWritableException(
						"No converter found for return value of type: " + valueType);
			}
			List<MediaType> mediaTypesToUse = new ArrayList<>();
            //暴力的两层for循环，找出浏览器能接受并且服务器能生产的数据
			for (MediaType requestedType : acceptableTypes) {
				for (MediaType producibleType : producibleTypes) {
					if (requestedType.isCompatibleWith(producibleType)) {
						mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
					}
				}
			}
			if (mediaTypesToUse.isEmpty()) {
				if (body != null) {
					throw new HttpMediaTypeNotAcceptableException(producibleTypes);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
				}
				return;
			}
            //按照优先级排序(q的值)
			MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
			//确定返回的媒体类型（优先级最高的）
			for (MediaType mediaType : mediaTypesToUse) {
				if (mediaType.isConcrete()) {
					selectedMediaType = mediaType;
					break;
				}
				else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
					selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
					break;
				}
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Using '" + selectedMediaType + "', given " +
						acceptableTypes + " and supported " + producibleTypes);
			}
		}

		if (selectedMediaType != null) {
			selectedMediaType = selectedMediaType.removeQualityValue();
            //遍历所有的类型转换器，找到能实现的转换器
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
						(GenericHttpMessageConverter<?>) converter : null);
                //判断是否支持我们协商的返回值
				if (genericConverter != null ?
						((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
						converter.canWrite(valueType, selectedMediaType)) {
					body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
							inputMessage, outputMessage);
					if (body != null) {
						Object theBody = body;
						LogFormatUtils.traceDebug(logger, traceOn ->
								"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
						addContentDispositionHeader(inputMessage, outputMessage);
						if (genericConverter != null) {
                            //往outMessage中写入转换后的JSON数据
							genericConverter.write(body, targetType, selectedMediaType, outputMessage);
						}
						else {
							((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
						}
					}
					else {
						if (logger.isDebugEnabled()) {
							logger.debug("Nothing to write: null body");
						}
					}
					return;
				}
			}
		}

		if (body != null) {
			Set<MediaType> producibleMediaTypes =
					(Set<MediaType>) inputMessage.getServletRequest()
							.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

			if (isContentTypePreset || !CollectionUtils.isEmpty(producibleMediaTypes)) {
				throw new HttpMessageNotWritableException(
						"No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
			}
			throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
		}
	}
```

用MessageConverters将返回值转化为JSON格式

1.内容协商：浏览器会告诉服务器它能接收什么样的数据

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-3BaTQJ6L-1653538758232)(D:/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/picture/image-20220502145831045.png)]

q代表权值，也就是优先级，表示优先接收text/html之类的数据，如果没有再接收image/webp，如果还没有就接收所有类型的数据

```java
            //获得被包装的请求
			HttpServletRequest request = inputMessage.getServletRequest();
            //获得浏览器能接收什么样的数据(text/html之类的，这个方法会获取request中ACCEPT字段的值，并封装成List)
			List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
            //获得服务器能生产什么样的数据(json之类的)
			List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

			if (body != null && producibleTypes.isEmpty()) {
				throw new HttpMessageNotWritableException(
						"No converter found for return value of type: " + valueType);
			}
			List<MediaType> mediaTypesToUse = new ArrayList<>();
            //暴力的两层for循环，找出浏览器能接受并且服务器能生产的数据
			for (MediaType requestedType : acceptableTypes) {
				for (MediaType producibleType : producibleTypes) {
					if (requestedType.isCompatibleWith(producibleType)) {
						mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
					}
				}
			}
			if (mediaTypesToUse.isEmpty()) {
				if (body != null) {
					throw new HttpMediaTypeNotAcceptableException(producibleTypes);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
				}
				return;
			}
            //按照优先级排序(q的值)
			MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
			//确定返回值类型（优先级最高的）
			for (MediaType mediaType : mediaTypesToUse) {
				if (mediaType.isConcrete()) {
					selectedMediaType = mediaType;
					break;
				}
				else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
					selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
					break;
				}
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Using '" + selectedMediaType + "', given " +
						acceptableTypes + " and supported " + producibleTypes);
			}
```

2.浏览器会根据自己能生产的类型的数据进行内容协商，确定最后返回值的类型

3.消息转换

HttpMessageConverter消息转换器是一个接口，里面定义了消息转换的相关方法，用这些方法来进行返回值类型的转换

```java
public interface HttpMessageConverter<T> {
    //是否能将mediaType媒体类型的数据转换为clazz类型的数据
	boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
    //是否能将clazz类型的数据转换为mediaType媒体类型的数据
	boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);
rn the list of supported media types, potentially an immutable copy
	 */
    //能支持转换的媒体类型
	List<MediaType> getSupportedMediaTypes();
	//从转换器中读取T类型数据
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;
    //向outputMessage中写入T类型的数据
	void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;

}
```

SpringMVC中内置的所有类型转换器

![image-20220502154807257](../../../../学习笔记/picture/e88297aa7f78e61a190f4e24215d3dc4.png)

遍历所有的类型转化器，判断哪个类型转换器能处理这个请求（将对象类型转换为JSON数据）

其中MappingJackson2HttpMessageConverter类向的能处理我们的对象类型（实际上它能处理所有类型的返回值）

然后用MappingJackson2HttpMessageConverter的write方法向outputMessage中写入转换后的JSON数据

#### 原理总结

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-rWsYvfrn-1653538758233)(D:/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/picture/image-20220502160312589.png)]

根据@ResponBody注解判断使用RequestResponseBodyMethodProccessor这个返回值处理器，这个返回值处理器又会根据返回值选择不同的Converter来转换数据的格式，例如返回资源文件：

```java
   /*
   import org.springframework.core.io.FileSystemResource;
   import org.springframework.core.io.Resource;
   */
   @GetMapping("/file")
    @ResponseBody
    public Resource testParam(){
        return new FileSystemResource("src/main/resources/application.yml");
    }
```

最后得到的就不是JSON格式的数据了：

![image-20220502161738933](../../../../学习笔记/picture/2b112b9e9c73d551f6add66310d03af4.png)

#### 内容协商

如果在pom文件中引入这个依赖（这个jar包可以把对象转换为XML格式的数据）

```xml
 <dependency>
     <groupId>com.fasterxml.jackson.dataformat</groupId>
     <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

那么返回给浏览器的数据就是XML的数据，这是因为在浏览器的响应头中设置的优先级

![image-20220502145831045](../../../../学习笔记/picture/26526d63fbf4143fa0ff1bbcd10be844.png)

xhtml+xml的优先级高(q=0.9)，比q=0.8的`*/*`要高，所以Spring会优先将其转换为XML格式的数据，而如果我们在PostMan中将Accept字段的值设置为`*/*`，就会得到JSON格式的数据。我们需要不同格式的数据只需要改变Header中Accept的字段的值即可。这些得益于Spring的内容协商功能。

原理：

```java
List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
```

获取浏览器支持的类型，这个方法中会获取request中的ACCEPT字段并解析成List<MediaType>类型

```java
List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
```

获得服务器可以返回的媒体类型

方法体：

```java
	protected List<MediaType> getProducibleMediaTypes(
			HttpServletRequest request, Class<?> valueClass, @Nullable Type targetType) {
		//从请求域中获取媒体类型
		Set<MediaType> mediaTypes =
				(Set<MediaType>) request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
        //如果非空则直接返回
		if (!CollectionUtils.isEmpty(mediaTypes)) {
			return new ArrayList<>(mediaTypes);
		}
		else if (!this.allSupportedMediaTypes.isEmpty()) {
			List<MediaType> result = new ArrayList<>();
            //遍历所有的类型转换器Converter
			for (HttpMessageConverter<?> converter : this.messageConverters) {
                //如果这个类型转换器是一个合法的转换器
				if (converter instanceof GenericHttpMessageConverter && targetType != null) {
                    //转换器是否支持valueClass类型的数据（GenericHttpMessageConverter这个类有三个参数，媒体类型为空）
                    //targetType,valueClass都是从返回参数中得到的，targetType只为GenericHttpMessageConverter类服务
					if (((GenericHttpMessageConverter<?>) converter).canWrite(targetType, valueClass, null)) 					 {
                        //将转换器能转换出的媒体类型添加到集合中
						result.addAll(converter.getSupportedMediaTypes());
					}
				}
				else if (converter.canWrite(valueClass, null)) {
					result.addAll(converter.getSupportedMediaTypes());
				}
			}
			return result;
		}
		else {
			return Collections.singletonList(MediaType.ALL);
		}
	}
```

内容协商原理（writeWithMessageConverters方法执行流程，源码在上一章有）：

1. 判断请求域中是否已经有返回值类型（可能在拦截的时候做了处理）

2. 获得浏览器支持的媒体类型（基于内容协商管理器contentNegotiationManager，使用请求头策略HeaderContentNegotiationStrategy获取）

3. 获得服务器能产生的媒体类型：

   1.遍历所有的转换器，找到所有支持返回类型的转换器(A -> 转换器 -> B，已知A，找到所有的转换器)

   2.将这些转换器能转换出的媒体类型统计出来

4. 遍历浏览器支持的媒体类型和服务器能产生的媒体类型，找到所有能匹配的媒体类型

5. 对找到的媒体类型按照优先级排序（设置的q的值），取最大的作为返回的媒体类型

6. 再次遍历所有的转化器，找到能转换的转换器（A（返回值类型） -> 转换器 -> B（媒体类型），已知A,B找到转换器）

7. 用转换器实现A（返回值类型） -> 转换器 -> B（媒体类型）的转换

#### 自定义内容协商策略

浏览器的ACCEPT字段我们提交form表单后没办法随意修改，所以我们可以将协商内容放在参数部分

```yml
spring:
  mvc:
    contentnegotiation:
      favor-parameter: true  #开启请求参数内容协商模式
```

开启参数内容协商后，我们就可以用format参数决定返回值的类型（json，xml）

（如果内容协商失败，会返回406）

获取浏览器的请求类型：

```java
	@Override
	public List<MediaType> resolveMediaTypes(NativeWebRequest request) throws HttpMediaTypeNotAcceptableException {
		for (ContentNegotiationStrategy strategy : this.strategies) {
			List<MediaType> mediaTypes = strategy.resolveMediaTypes(request);
			if (mediaTypes.equals(MEDIA_TYPE_ALL_LIST)) {
				continue;
			}
			return mediaTypes;
		}
		return MEDIA_TYPE_ALL_LIST;
	}
```

在将favor-parameter设置为true后，这里再寻找浏览器能接受的媒体类型时会多一种策略：根据参数确定媒体类型，而这个策略排在根据请求头确定媒体类型的策略之前，所以会按照参数策略确定媒体类型。

![image-20220502222239380](../../../../学习笔记/picture/9d9db3d038855be9198ee2d091f714a0.png)

确定过程是先拿到请求参数对应的format字段的值（比如json），然后根据这个值得到对应的媒体类型，可以忽略大小写

```java
	@Nullable
	protected MediaType lookupMediaType(String extension) {
		return this.mediaTypes.get(extension.toLowerCase(Locale.ENGLISH));
	}
```

```java
	public List<MediaType> resolveMediaTypeKey(NativeWebRequest webRequest, @Nullable String key)
			throws HttpMediaTypeNotAcceptableException {

		if (StringUtils.hasText(key)) {
			MediaType mediaType = lookupMediaType(key);
			if (mediaType != null) {
				handleMatch(key, mediaType);
				return Collections.singletonList(mediaType);
			}
			mediaType = handleNoMatch(webRequest, key);
			if (mediaType != null) {
				addMapping(key, mediaType);
				return Collections.singletonList(mediaType);
			}
		}
		return MEDIA_TYPE_ALL_LIST;
	}
```

如果拿到的媒体类型是`"*/*"`，则使用下一个策略，如果所有策略都返回`*/*`，则返回`*/*`

#### 内容协商适用场景

假如我们有这个场景：

1.浏览器发请求，返回xml格式的数据

2.AJAX发请求返回JSON格式的数据

3.App发请求返回一个名为"x-atguigu"格式的数据

在容器启动的时候，Spring会帮我们注册默认的Converter进入Spring容器：

```java
	protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
		messageConverters.add(new ByteArrayHttpMessageConverter());
		messageConverters.add(new StringHttpMessageConverter());
		messageConverters.add(new ResourceHttpMessageConverter());
		messageConverters.add(new ResourceRegionHttpMessageConverter());
		try {
			messageConverters.add(new SourceHttpMessageConverter<>());
		}
		catch (Throwable ex) {
			// Ignore when no TransformerFactory implementation is available...
		}
		messageConverters.add(new AllEncompassingFormHttpMessageConverter());

		if (romePresent) {
			messageConverters.add(new AtomFeedHttpMessageConverter());
			messageConverters.add(new RssChannelHttpMessageConverter());
		}

		if (jackson2XmlPresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
		}
		else if (jaxb2Present) {
			messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
		}

		if (jackson2Present) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2HttpMessageConverter(builder.build()));
		}
		else if (gsonPresent) {
			messageConverters.add(new GsonHttpMessageConverter());
		}
		else if (jsonbPresent) {
			messageConverters.add(new JsonbHttpMessageConverter());
		}

		if (jackson2SmilePresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.smile();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2SmileHttpMessageConverter(builder.build()));
		}
		if (jackson2CborPresent) {
			Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.cbor();
			if (this.applicationContext != null) {
				builder.applicationContext(this.applicationContext);
			}
			messageConverters.add(new MappingJackson2CborHttpMessageConverter(builder.build()));
		}
	}
```

其中我们注意到有诸如jackson2XmlPresent是否为true的判断，而这个值的true还是false取决于：

```java
	static {
		ClassLoader classLoader = WebMvcConfigurationSupport.class.getClassLoader();
		romePresent = ClassUtils.isPresent("com.rometools.rome.feed.WireFeed", classLoader);
		jaxb2Present = ClassUtils.isPresent("javax.xml.bind.Binder", classLoader);
		jackson2Present = ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", classLoader) &&
				ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator", classLoader);
		jackson2XmlPresent = ClassUtils.isPresent("com.fasterxml.jackson.dataformat.xml.XmlMapper", classLoader);
		jackson2SmilePresent = ClassUtils.isPresent("com.fasterxml.jackson.dataformat.smile.SmileFactory", classLoader);
		jackson2CborPresent = ClassUtils.isPresent("com.fasterxml.jackson.dataformat.cbor.CBORFactory", classLoader);
		gsonPresent = ClassUtils.isPresent("com.google.gson.Gson", classLoader);
		jsonbPresent = ClassUtils.isPresent("javax.json.bind.Jsonb", classLoader);
	}
```

所以引入这个依赖后，才可以实现对象和XML格式之间的转换。

适用类工具ClassUtils判断某个类是否存在

我们想自定义消息转换器，方法和前面一样，向Spring容器中注册WebMvcConfigurer组件，在里面通过实现里面的方法来定制化我们想要的功能。

这里面有两个方法可以让我们定制化消息转换器：

```java
	@Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {

    }

    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        WebMvcConfigurer.super.extendMessageConverters(converters);
    }
```

上面那个会覆盖默认的类型转换器，下面那个会在默认类型转换器的基础上添加新的消息转换器

```
Class<T> 类的 isAssignableFrom方法 用于判断某个类是不是一个类或者它的子类
```

```java
    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return MediaType.parseMediaTypes("application/atguigu");
    }
```

通过字符串得到一个application/atguigu类型的消息转换器（集合类型）

实现一个自定义的消息转换器：

```java
public class AtGuiguConverter implements HttpMessageConverter<Pet> {
    
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        return false;
    }
    //支持转换什么类型的数据（Pet）
    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        return clazz.isAssignableFrom(Pet.class);
    }
    //支持转换成什么类型的数据（application/atguigu类型）
    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return MediaType.parseMediaTypes("application/atguigu");
    }
    @Override
    public Pet read(Class<? extends Pet> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return null;
    }
    //如何转换，定制化转换规则
    @Override
    public void write(Pet pet, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        //转换后得到数据
        String data=pet.getName()+":"+pet.getAge();
        //拿到封装在outputMessage的输出流
        OutputStream body = outputMessage.getBody();
        //往输出流中写入数据
        body.write(data.getBytes(StandardCharsets.UTF_8));
    }
}
```

![image-20220503004520534](../../../../学习笔记/picture/ec08f6e770684795ca48d0efb0f03ed1.png)

在WebMvcConfigurer中添加转换器：

```java
@Configuration
public class MyConfig {
    //1、WebMvcConfigurer定制化SpringMVC的功能
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {
            @Override
            public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
                converters.add(new AtGuiguConverter());
            }
}

```

实现功能：

![image-20220503004520534](../../../../学习笔记/picture/ec08f6e770684795ca48d0efb0f03ed1.png)

我们新添加的转换器和默认转换器的适用流程都是一样的

![image-20220503142857435](../../../../学习笔记/picture/01f007429027944c2b9573c5b33110a0.png)

#### 添加参数和媒体映射关系

如果我们想在url中设置format字段，当format=gg（可以是url参数，也可以是请求体中的参数）时，内容协商后的媒体类型是atguigu，那么就需要我们在内容协商管理器中添加我们自定义的映射规则。（和前面一样要在WebMvcConfigurer里面实现里面的方法configureContentNegotiation）

```java
@Configuration
public class MyConfig {
    //1、WebMvcConfigurer定制化SpringMVC的功能
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {

            @Override
            public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
                converters.add(new AtGuiguConverter());
            }

            @Override
            public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
                Map<String, MediaType> map=new HashMap<>();
                map.put("json",MediaType.APPLICATION_JSON);
                map.put("xml",MediaType.APPLICATION_XML);
                map.put("gg",MediaType.parseMediaType("application/atguigu"));
                ParameterContentNegotiationStrategy paramStrage = new ParameterContentNegotiationStrategy(map);
                configurer.strategies(Arrays.asList(paramStrage));
            }
        };
    }
}
```

我们回顾一下之前所讲的内容：

在进行内容协商的时候要获取浏览器能接受的媒体类型，服务器要根据浏览器能接受的媒体类型返回对应格式的数据，而获取媒体类型时Spring会使用内容协商管理器遍历所有注册到Spring容器中的内容协商策略（获取浏览器支持的媒体类型的途径），在默认情况下，内容协商策略只有根据请求头获取媒体类型（HeaderContentNegotiationStrategy），而在spring.mvc.contentnegotiation.favor-parameter设置为true后，Spring容器中会多出一种策略：按照请求参数获取媒体类型（ParameterContentNegotiationStrategy），创建这个策略对象需要传入一个`Map<String, MediaType>`类型的参数，代表format值和媒体类型的对应关系，默认情况下只有json和xml，所以我们需要把这两个加上的同时将gg和application/atguigu媒体类型建立关系，然后创建新的策略对象添加进策略协商管理器中。可以看到结果生效：

![image-20220503152216636](../../../../学习笔记/picture/699d77c0ad1025cb2eb22d124ca0f817.png)

但是我们也发现根据请求设置媒体类型的策略失效了：

![image-20220503152319677](../../../../学习笔记/picture/9975c1b5f3f8dc3166a1d399e1f9c493.png)

这是因为我们在配置类中用configurer.strategies(Arrays.asList(paramStrage));重新设置了内容协商管理器的所有策略（覆盖了默认情况，而不是添加），我们没有添加HeaderContentNegotiationStrategy策略，所以请求头会失效。同时在没有获取到浏览器的媒体类型时，会默认将媒体类型视为`*/*`，即接受所有的类型：

```java
	@Override
	public List<MediaType> resolveMediaTypes(NativeWebRequest request) throws HttpMediaTypeNotAcceptableException {
		for (ContentNegotiationStrategy strategy : this.strategies) {
			List<MediaType> mediaTypes = strategy.resolveMediaTypes(request);
			if (mediaTypes.equals(MEDIA_TYPE_ALL_LIST)) {
				continue;
			}
			return mediaTypes;
		}
		return MEDIA_TYPE_ALL_LIST;
	}
```

而服务器能产生json,xml,atguigu等类型的数据，都能与`*/*`匹配，其中json优先级最高，排序后是第一个，所以会默认使用json格式的数据返回。

```java
                ParameterContentNegotiationStrategy paramStrage = new ParameterContentNegotiationStrategy(map);
                HeaderContentNegotiationStrategy headerStrage=new HeaderContentNegotiationStrategy();
                configurer.strategies(Arrays.asList(paramStrage,headerStrage));
```

添加请求头策略后又重新生效

在请求头策略和参数策略同时存在时，优先使用参数策略。

如果不想获取format字段的数据作为协商依据，可以通过paramStrage.setParameterName("ff")方法更换为其他字段。

```java
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    Map<String, MediaType> map=new HashMap<>();
    map.put("json",MediaType.APPLICATION_JSON);
    map.put("xml",MediaType.APPLICATION_XML);
    map.put("gg",MediaType.parseMediaType("application/atguigu"));
    ParameterContentNegotiationStrategy paramStrage = new ParameterContentNegotiationStrategy(map);
    paramStrage.setParameterName("ff");
    HeaderContentNegotiationStrategy headerStrage=new HeaderContentNegotiationStrategy();
    configurer.strategies(Arrays.asList(paramStrage,headerStrage));
}
```

### 视图解析

https://blog.csdn.net/u011863024/article/details/113667946

#### Thymeleaf模板引擎

Thymeleaf模板引擎适用于开发后台管理界面（给管理人员使用而非具体的用户），没有与后端分离，性能也较差，但是开发起来会容易很多。

使用Thymeleaf模板的html页面，放在前面也能运行，使用的是没有数据的普通页面，放在Spring的资源目录下就会经过视图的渲染而获得数据

引入依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

自动配好的策略

1. 所有thymeleaf的配置值都在 ThymeleafProperties
2. 配置好了 **SpringTemplateEngine**
3. 配好了 **ThymeleafViewResolver**
4. 我们只需要直接开发页面

在寻找html页面时会在classpath:/templates/目录下面找，并且会自动帮我们加上.html的后缀名，这两个和我们的字符串拼接再一起共同构成html的请求路径

```
public static final String DEFAULT_PREFIX = "classpath:/templates/";//模板放置处
public static final String DEFAULT_SUFFIX = ".html";//文件的后缀名
```

JSP语法：

##### 基本语法

| 表达式名字 | 语法 | 用途                               |
| ---------- | ---- | ---------------------------------- |
| 变量取值   | ${…} | 获取请求域、session域、对象等值    |
| 选择变量   | *{…} | 获取上下文对象值                   |
| 消息       | #{…} | 获取国际化等值                     |
| 链接       | @{…} | 生成链接                           |
| 片段表达式 | ~{…} | jsp:include 作用，引入公共页面片段 |

##### 简单使用

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<h1 th:text="${msg}">nice</h1>
<h2>
    <a href="www.baidu.com" th:href="${link}">去百度</a>  <br/>
    <a href="www.google.com" th:href="@{/link}">去百度</a>
</h2>
</body>
</html>
```

直接打开这个html页面显示的"去百度"这个原始内容，经过Spring加载后会显示变量

```
xmlns:th="http://www.thymeleaf.org"
```

这个用于引入命名空间

修改标签的值：`th:text="${msg}"`

设置页面跳转的值：

`th:href="${link}"` 将链接内容替换为model中link变量的值（替换的是变量的值）

`th:href="@{/link}"`  将链接内容替换为/link（替换的字面量的值）

用${}获取我们放在model中的数据

```java
@Controller
public class ViewTestController {
    @GetMapping("/hello")
    public String hello(Model model){
        //model中的数据会被放在请求域中 request.setAttribute("a",aa)
        model.addAttribute("msg","一定要大力发展工业文化");
        model.addAttribute("link","http://www.baidu.com");
        return "success";
    }
}
```

设置标签内部属性的值：

```hmtl
<img src="../../images/gtvglogo.png"  
     th:attr="src=@{/images/gtvglogo.png},title=#{logo},alt=#{logo}" />
```

可以在双引号中使用单引号进行字符串拼接操作

Tip：@GetMapping(value={}) 这些注解的value字段可以是数组，表示这些注解对应到同一个controller

thymleaf原则:model有值就用model里的值，model里没有值就用html中的值。

官网:thymeleaf.org/doc

th:action="@{/login}" 加在form表单上，表示设置form表单请求的url

controller返回 "redirect:/main.html"表示进行请求重定向

th:text=${msg} 修改内容，用这个可以动态修改文本自己标签页

除了能获得model中的数据，也默认能有session中的数据（参数名需要叫session）

thymeleaf行内写法:[[${session.user.name}]]

跳转到template目录下的basic目录下，返回"basic/index"即可

html中需要用src属性，thymeleaf用th:src="@{/}"

html需要用href属性的，thymeleaf用th:href="@{/}"

##### 模板引入

html页面可能会有很多功能的部分，例如导航条，侧边栏等。如果要修改这些部分的话需要修改所以的html页面，十分繁琐，所以我们可以使用thymeleaf的模板语法来将html可能会用到的功能组件保存起来，再需要使用的时候从组件库中引入组件（组件可以是任何公共的部分，例如公共的css，js，html元素），这样在修改组件的时候直接修改组件库的内容即可。

引入组件的方式可以使用thymleaf提供的fragment字段来设置一个唯一的标识，也可以使用html'的属性选择器（比如设置了id，引入的时候使用#id来引入）

使用fragment字段：

```html
<head th:fragment="commonheader">
    <!--common-->
    <link href="css/style.css" th:href="@{/css/style.css}" rel="stylesheet">
</head>
```

引入的时候：

```html
<div th:include="common :: commonheader"> </div>
```

common是存放组件的**html文件**的名称，commonheader是我们设置的th:fragmen字段的值

使用id：

```html
<div id="commonscript">
    <!-- Placed js at the end of the document so the pages load faster -->
    <script th:src="@{/js/jquery-1.10.2.min.js}"></script>
    <script th:src="@{/js/jquery-ui-1.9.2.custom.min.js}"></script>
    <script th:src="@{/js/jquery-migrate-1.2.1.min.js}"></script>
    <script th:src="@{/js/bootstrap.min.js}"></script>
    <script th:src="@{/js/modernizr.min.js}"></script>
    <script th:src="@{/js/jquery.nicescroll.js}"></script>
    <!--common scripts for all pages-->
    <script th:src="@{/js/scripts.js}"></script>
</div>
```

引入的时候：

```html
<div th:replace="common :: #commonscript"></div>
```

其实就是多加了一个#

组件库里的链接(href)和内容(src) ，都要替换成th的格式

编写组件库common.html：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"><!--注意要添加xmlns:th才能添加thymeleaf的标签-->
<head th:fragment="commonheader">
    <!--common-->
    <link href="css/style.css" th:href="@{/css/style.css}" rel="stylesheet">
    <link href="css/style-responsive.css" th:href="@{/css/style-responsive.css}" rel="stylesheet">
    ...
</head>
<body>
<!-- left side start-->
<div id="leftmenu" class="left-side sticky-left-side">
	...

    <div class="left-side-inner">
		...

        <!--sidebar nav start-->
        <ul class="nav nav-pills nav-stacked custom-nav">
            <li><a th:href="@{/main.html}"><i class="fa fa-home"></i> <span>Dashboard</span></a></li>
            ...
            <li class="menu-list nav-active"><a href="#"><i class="fa fa-th-list"></i> <span>Data Tables</span></a>
                <ul class="sub-menu-list">
                    <li><a th:href="@{/basic_table}"> Basic Table</a></li>
                    <li><a th:href="@{/dynamic_table}"> Advanced Table</a></li>
                    <li><a th:href="@{/responsive_table}"> Responsive Table</a></li>
                    <li><a th:href="@{/editable_table}"> Edit Table</a></li>
                </ul>
            </li>
            ...
        </ul>
        <!--sidebar nav end-->
    </div>
</div>
<!-- left side end-->


<!-- header section start-->
<div th:fragment="headermenu" class="header-section">

    <!--toggle button start-->
    <a class="toggle-btn"><i class="fa fa-bars"></i></a>
    <!--toggle button end-->
	...

</div>
<!-- header section end-->

<div id="commonscript">
    <!-- Placed js at the end of the document so the pages load faster -->
    <script th:src="@{/js/jquery-1.10.2.min.js}"></script>
    <script th:src="@{/js/jquery-ui-1.9.2.custom.min.js}"></script>
    <script th:src="@{/js/jquery-migrate-1.2.1.min.js}"></script>
    <script th:src="@{/js/bootstrap.min.js}"></script>
    <script th:src="@{/js/modernizr.min.js}"></script>
    <script th:src="@{/js/jquery.nicescroll.js}"></script>
    <!--common scripts for all pages-->
    <script th:src="@{/js/scripts.js}"></script>
</div>
</body>
</html>
```

引入组件：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0">
  <meta name="description" content="">
  <meta name="author" content="ThemeBucket">
  <link rel="shortcut icon" href="#" type="image/png">

  <title>Basic Table</title>
    <div th:include="common :: commonheader"> </div><!--将common.html的代码段 插进来-->
</head>

<body class="sticky-header">

<section>
<div th:replace="common :: #leftmenu"></div>
    
    <!-- main content start-->
    <div class="main-content" >

        <div th:replace="common :: headermenu"></div>
        ...
    </div>
    <!-- main content end-->
</section>

<!-- Placed js at the end of the document so the pages load faster -->
<div th:replace="common :: #commonscript"></div>
</body>
</html>
```

##### 引入语法

https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html#difference-between-thinsert-and-threplace-and-thinclude

（其实都用div即可）

假如在footer.html中有

```html
<footer th:fragment="copy">
	hello,lth
</footer>
```

1.insert 引入用的标签在外，被引入的标签在内

```html
<div th:insert="footer :: copy"></div>
```

替换后的效果是：

```html
<div>
    <footer>
        hello,lth
    </footer>
</div>
```

2.replace 只保留被引入的标签

```html
<div th:replace="footer :: copy"></div>
```

替换后：

```html
  <footer>
    hello,lth
  </footer>
```

3.include 只保留引入用的标签

```html
<div th:include="footer :: copy"></div>
```

替换后：

```html
  <div>
    hello,lth
  </div>
```

##### 集合遍历

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <tr th:each="pet,status:${pets}">
        <td>[[${status.index}]]</td>
        <td>[[${pet.name}]]</td>
        <td>[[${pet.age}]]</td>
        <br>
    </tr>
</body>
</html>
```

除了我们集合中对应的对象，每个集合还默认会有一个status对象（默认在第二个参数里），里面有相关的索引信息

#### 视图解析原理

视图解析流程与前面所说一致，拿到返回值后，会根据返回值的类型以及注解判断使用哪种视图解析器，而对于返回值是String类型，且没有加上@ResponBody注解，则会使用ViewNameMethodReturnValueHandler这个返回值解析器来解析返回值。

处理过程：

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		//如果返回值是字符串
		if (returnValue instanceof CharSequence) {
			String viewName = returnValue.toString();
            //将视图的地址放入到视图容器中
			mavContainer.setViewName(viewName);
            //判断是否能重定向
			if (isRedirectViewName(viewName)) {
                //将重定向标志设置为true
				mavContainer.setRedirectModelScenario(true);
			}
		}
		else if (returnValue != null) {
			// should not happen
			throw new UnsupportedOperationException("Unexpected return type: " +
					returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
		}
	}
```

在方法执行过程中，方法中数据（model）和视图地址（view）都会放在一个ModelAndViewContainer视图容器中

isRedirectViewName(viewName)方法：

```java
	protected boolean isRedirectViewName(String viewName) {
		return (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) || viewName.startsWith("redirect:"));
	}
```

如果返回的字符符串和redirectPatterns设置的正则表达式匹配，或者以"redirect:"开头，则将这个字符串视为重定向。所以我们在加上redirect:作为前缀后可以进行请求重定向。

如果我们方法去请求参数中有我们的自定义对象，那么这个自定义对象也会被放到mavContainer中

在invokeHandlerMethod方法执行完后，会执行下面的getModelAndView方法

```java
	@Nullable
	private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
			ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
		//拿到mavContainer容器
		modelFactory.updateModel(webRequest, mavContainer);
		if (mavContainer.isRequestHandled()) {
			return null;
		}
        //获取容器中的model，这里的model和我们在方法参数中通过设置Map型参数或者Model型参数拿到的对象是同一个，类型都是ModelMap类型，对应request的请求域
		ModelMap model = mavContainer.getModel();
        //使用model(数据),视图名(view)创建一个ModelAndView对象
		ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
		if (!mavContainer.isViewReference()) {
			mav.setView((View) mavContainer.getView());
		}
        //如果model带有@RedirectAttribute注解，则会将这个model放入到下一次请求的参数中
		if (model instanceof RedirectAttributes) {
			Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
			HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
			if (request != null) {
				RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
			}
		}
		return mav;
	}
```

1.所有请求的执行结果都是一个ModelAndView对象：

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

2.如果视图名称为null，则会根据uri给它一个默认的视图名

```
applyDefaultViewName(processedRequest, mv);
```

例如：

```java
    @GetMapping("/success")
    public String hello(Model model, HttpSession httpSession){
        return null;
    }
```

会返回template目录下的success.html页面

3.处理派发结果

```
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

```java
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;
		//如果有异常，处理异常
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// 如果视图不为空，渲染视图
		if (mv != null && !mv.wasCleared()) {
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("No view rendering, null ModelAndView returned.");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		if (mappedHandler != null) {
			// Exception (if any) is already handled..
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
```

渲染视图：

```
render(mv, request, response);
```

```java
	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale =
				(this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
		response.setLocale(locale);
		//根据视图名，拿到视图对象
		View view;
		String viewName = mv.getViewName();
		if (viewName != null) {
			// We need to resolve the view name.
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
            //如果无法解析就抛出异常
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		}
		else {
			// No need to lookup: the ModelAndView object contains the actual View object.
			view = mv.getView();
			if (view == null) {
				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
						"View object in servlet with name '" + getServletName() + "'");
			}
		}

		// Delegate to the View object for rendering.
		if (logger.isTraceEnabled()) {
			logger.trace("Rendering view [" + view + "] ");
		}
		try {
			if (mv.getStatus() != null) {
				response.setStatus(mv.getStatus().value());
			}
            //得到视图后，调用view的render方法来觉得最后的视图如何渲染
			view.render(mv.getModelInternal(), request, response);
		}
		catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Error rendering view [" + view + "]", ex);
			}
			throw ex;
		}
	}
```

1.根据视图名拿到视图对象View，View中会定义页面的渲染逻辑（也就是得到返回给前端的文本）

```java
	@Nullable
	protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
			Locale locale, HttpServletRequest request) throws Exception {

		if (this.viewResolvers != null) {
			for (ViewResolver viewResolver : this.viewResolvers) {
				View view = viewResolver.resolveViewName(viewName, locale);
				if (view != null) {
					return view;
				}
			}
		}
		return null;
	}
```

遍历所有的视图解析器，尝试解析视图名，如果能成功解析就直接返回，否则返回null

包含的视图解析有：

![image-20220505003040015](../../../../学习笔记/picture/56b3e65906cfc0e40d46ea341d3a3a7c.png)

第0个是内容协商视图解析器，里面内容协商管理器中包含下面所有的视图解析器，因而还是会遍历下面所有的视图解析器，尝试解析viewName得到view。所以在这个循环中不会进入到下面中，但是解析过程还是用下面的解析器完成

第2个视图解析器是Thymeleaf视图解析器，会创建RedirectView对象

视图渲染逻辑：

```java
	@Override
	public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {

		if (logger.isDebugEnabled()) {
			logger.debug("View " + formatViewName() +
					", model " + (model != null ? model : Collections.emptyMap()) +
					(this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
		}
		//这一步，将我们model中的数据放到一个新的map里面mergedModel
		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
		prepareResponse(request, response);
        //将需要的参数都统合起来，觉得最后的视图渲染逻辑
		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
	}
```

其中：renderMergedOutputModel(mergedModel, getRequestToExpose(request), response)方法：

```java
	@Override
	protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
			HttpServletResponse response) throws IOException {
		//获取模板URL，拼接URL并将model中的参数作为URL的路径参数放在后面
		String targetUrl = createTargetUrl(model, request);
		targetUrl = updateTargetUrl(targetUrl, model, request, response);

		// 保存参数
		RequestContextUtils.saveOutputFlashMap(targetUrl, request, response);

		//使用原生的response.sendRedirect(encodedURL)方法进行重定向
		sendRedirect(request, response, targetUrl, this.http10Compatible);
	}
```

返回值如果是以**"forward:"**开始，则返回new InternalResourceView(forwardUrl)视图对象

功能是**转发**：request.getRequestDispatcher(URL).forward(request,response)

转发是以当前请求为代理，生产一次的新的请求，将新的请求的返回值作为当前请求的返回值返回，调用的是request的方法，转发新的请求是服务器发起的，所以浏览器只会发送一次请求（相当于处理请求的时候调用了其他请求对应的方法），并且地址栏不会发送变化

返回值如果以**"redirect:"**开始，则返回new RedirectView()视图对象

功能是**重定向**：response.sendRedirect(URL)

重定向是返回下一次应当查询的URL，让浏览器向这个URL发请求，调用的是response的方法，浏览器会发送多次请求直到得到结果，地址栏的请求地址会变成最后一次重定向的地址

补充：转发和重定向的区别

![在这里插入图片描述](../../../../学习笔记/picture/e7462351ee1c77cf5f552cd388028a2d.png)

返回值如果是普通字符串，则返回new ThymeleafView()视图对象，这个view会使用HTML解析器等工具填充数据，返回HTML文本

我们可以实现一个View接口和一个自定义的视图解析器，这样就可以返回我们自定义的文本内容

### 拦截器

添加拦截器需要我们实现HandlerInterceptor接口，里面有三个方法

```java
public interface HandlerInterceptor {
	//用于前置拦截，在方法执行前执行
	default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		return true;
	}
    //后置拦截，在方法执行完，还没有渲染页面的时候，如果我们需要添加一些数据进model里面的可以使用这个方法
	default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable ModelAndView modelAndView) throws Exception {
	}
    //在视图渲染完成后执行，用于进行一些清理工作
	default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable Exception ex) throws Exception {
	}

}
```

定制化SpringMVC的功能都需要我们实现一个WebMvcConfigurer

#### preHandle 前置拦截

实现一个拦截器：

如果session没有对应的值，说明没有登录，返回false表示进行拦截，返回true表示放行

```java
public class LoginIntercepter implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if(request.getSession().getAttribute("loginUser")!=null){
            return false;
        }
        return true;
    }
}
```

在实现的WebMvcConfigurer接口中，实现方法：

```java
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                registry.addInterceptor(new LoginIntercepter())
                        .addPathPatterns("/**")
                        .excludePathPatterns("/login","/","/css/**","/js/**");
            }
```

addInterceptor：添加一个拦截器

addPathPatterns：添加拦截的路由，动态路由和静态资源都会被拦截，所以要为静态资源的路径也放行

excludePathPatterns：添加放行的路由

重定向会丢失原来request中的数据（因为发了一个新的request），所以使用转发功能即可保留请求域中的数据

#### 拦截器原理

```java
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
        }
```

在执行mv = ha.handle(processedRequest, response, mappedHandler.getHandler());方法前，会先执行上述方法，可以看到只要这个方法返回false，请求过程就结束了。

applyPreHandle：

```java
	boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
            //顺序执行所有的拦截器
			for (int i = 0; i < interceptors.length; i++) {
				HandlerInterceptor interceptor = interceptors[i];
				if (!interceptor.preHandle(request, response, this.handler)) {
                    //如果被拦截了则逆序执行返回true的拦截器的AfterCompletion方法
					triggerAfterCompletion(request, response, null);
					return false;
				}
				this.interceptorIndex = i;
			}
		}
		return true;
	}
```

如代码所示，请求会顺序执行我们添加的拦截器列表，执行里面的preHandle方法。如果拦截器返回true则执行下一个拦截器，如果有拦截器返回false，也就是请求被拦截了，在返回doDispatch之前会执行triggerAfterCompletion方法：

```java
	void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = this.interceptorIndex; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				try {
					interceptor.afterCompletion(request, response, this.handler, ex);
				}
				catch (Throwable ex2) {
					logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
				}
			}
		}
	}
```

这个方法中会逆序执行先前已经返回true的拦截器中的afterCompletion方法（最后那个返回false的拦截器不会执行afterCompletion方法）

方法执行完成后会执行applyPostHandle方法：

```java
mappedHandler.applyPostHandle(processedRequest, response, mv);
```

```java
	void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = interceptors.length - 1; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				interceptor.postHandle(request, response, this.handler, mv);
			}
		}
	}
```

在这个方法中会逆序执行所有的拦截器的postHandle方法（能执行这里说明所有拦截器的preHandle方法都返回了true）

如果正常结束，会在processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);中逆序触发triggerAfterCompletion方法。

如果出现异常，则直接触发triggerAfterCompletion方法

triggerAfterCompletion只会执行已经执行了preHandle并且返回true的拦截器的方法

![image-20220505172331957](../../../../学习笔记/picture/3ec2d5bca54c51b80b6d2c5a9f1fbf32.png)

### 文件上传

文件上传页面

```html
<!-- role 申明这是个表单 th:action表示表单提交的路由 method表示请求方法是post enctype表示多文件上传-->
<form role="form" th:action="@{/upload}" method="post" enctype="multipart/form-data">
    <div class="form-group">
        <label for="exampleInputEmail1">邮箱</label>
        <input type="email" name="email" class="form-control" id="exampleInputEmail1" placeholder="Enter email">
    </div>
    
    <div class="form-group">
        <label for="exampleInputPassword1">名字</label>
        <input type="text" name="username" class="form-control" id="exampleInputPassword1" placeholder="Password">
    </div>
    
    <div class="form-group">
        <label for="exampleInputFile">头像</label>
        <input type="file" name="headerImg" id="exampleInputFile">
    </div>
    
    <div class="form-group">
        <label for="exampleInputFile">生活照</label>
        <input type="file" name="photos" multiple>
    </div>
    
    <div class="checkbox">
        <label>
            <input type="checkbox"> Check me out
        </label>
    </div>
    <button type="submit" class="btn btn-primary">提交</button>
</form>
```

文件上传处理的Controller：

```java
    @PostMapping("/upload")
    public Object upload(@RequestParam("email") String email,
                         @RequestPart("headerImg") MultipartFile headerImage,
                         @RequestPart("photos") MultipartFile[] photos){
        return new Object[]{email,headerImage.getName(),photos.length};
    }
```

单个文件使用MultipartFile headerImage

多个文件上传使用数组MultipartFile[] photos

使用@RequestPart("headerImg") 来接收文件

在配置中设置文件大小：（因为Spring有某人的文件上传大小限制）

```
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=100MB
```

文件下载：

```java
@RestController
public class FileController {

    final static String LOCATION =new File("").getAbsolutePath()+"/src/main/resources/static/";

    @PostMapping("/upload")
    public Object upload(@RequestParam("email") String email,
                         @RequestPart("headerImg") MultipartFile headerImage,
                         @RequestPart("photos") MultipartFile[] photos){
        if(!headerImage.isEmpty()) {
            try {
                System.out.println(headerImage.getName());
                System.out.println(headerImage.getOriginalFilename());

                headerImage.transferTo(new File(LOCATION+headerImage.getOriginalFilename()));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return new Object[]{email,headerImage.getName(),photos.length};
    }
}
```

使用headerImage.transferTo(new File(LOCATION+headerImage.getOriginalFilename()));保存文件

使用的是底层使用的是FileCopyUtils.copy(getInputStream(), Files.newOutputStream(dest));实现文件拷贝

#### 文件上传原理

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Servlet.class, StandardServletMultipartResolver.class, MultipartConfigElement.class })
@ConditionalOnProperty(prefix = "spring.servlet.multipart", name = "enabled", matchIfMissing = true)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(MultipartProperties.class)
public class MultipartAutoConfiguration {

	private final MultipartProperties multipartProperties;

	public MultipartAutoConfiguration(MultipartProperties multipartProperties) {
		this.multipartProperties = multipartProperties;
	}

	@Bean
	@ConditionalOnMissingBean({ MultipartConfigElement.class, CommonsMultipartResolver.class })
	public MultipartConfigElement multipartConfigElement() {
		return this.multipartProperties.createMultipartConfig();
	}
//向容器中添加文件上传解析器
	@Bean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
	@ConditionalOnMissingBean(MultipartResolver.class)
	public StandardServletMultipartResolver multipartResolver() {
		StandardServletMultipartResolver multipartResolver = new StandardServletMultipartResolver();
		multipartResolver.setResolveLazily(this.multipartProperties.isResolveLazily());
		return multipartResolver;
	}

}
```

如果我们向自定义文件解析过程，往Spring容器中添加我们自定义的文件解析器即可

在doDispatch方法中，在解析参数之前会先判断当前请求是否是文件上传请求：

```
processedRequest = checkMultipart(request);
```

```java
	protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
        //使用文件上传解析器判断是不是文件上传请求
		if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
			if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
				if (request.getDispatcherType().equals(DispatcherType.REQUEST)) {
					logger.trace("Request already resolved to MultipartHttpServletRequest, e.g. by MultipartFilter");
				}
			}
			else if (hasMultipartException(request)) {
				logger.debug("Multipart resolution previously failed for current request - " +
						"skipping re-resolution for undisturbed error rendering");
			}
			else {
				try {
                    //如果是文件上传请求则对原请求进行包装
					return this.multipartResolver.resolveMultipart(request);
				}
				catch (MultipartException ex) {
					if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
						logger.debug("Multipart resolution failed for error dispatch", ex);
						// Keep processing error dispatch with regular request handle below
					}
					else {
						throw ex;
					}
				}
			}
		}
		// 不是文件上传请求则直接返回原请求
		return request;
	}
```

this.multipartResolver.isMultipart(request) 使用这个方法判断是不是文件上传请求

如果是文件上传请求则将原请求进行包装

return this.multipartResolver.resolveMultipart(request);

然后返回doDispatch方法，判断返回的请求和原来的请求是否一样：

```
multipartRequestParsed = (processedRequest != request);
```

如果不一样，说明对原请求进行了包装，因而是文件上传请求

如果一样，说明没有包装，则不是文件上传请求

解析参数的过程和前面一样，根据@RequestPart注解判断使用RequestPartMethodArgumentResolver这个文件上传解析器来解析文件参数。

```java
	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest request, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
		Assert.state(servletRequest != null, "No HttpServletRequest");
		//获取注解信息，判断这个参数是不是必须的
		RequestPart requestPart = parameter.getParameterAnnotation(RequestPart.class);
		boolean isRequired = ((requestPart == null || requestPart.required()) && !parameter.isOptional());
		//获得参数名
		String name = getPartName(parameter, requestPart);
		parameter = parameter.nestedIfOptional();
		Object arg = null;
		//解析文件上传参数
		Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
		if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
			arg = mpArg;
		}
		else {
			try {
				HttpInputMessage inputMessage = new RequestPartServletServerHttpRequest(servletRequest, name);
				arg = readWithMessageConverters(inputMessage, parameter, parameter.getNestedGenericParameterType());
				if (binderFactory != null) {
					WebDataBinder binder = binderFactory.createBinder(request, arg, name);
					if (arg != null) {
						validateIfApplicable(binder, parameter);
						if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
							throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
						}
					}
					if (mavContainer != null) {
						mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
					}
				}
			}
			catch (MissingServletRequestPartException | MultipartException ex) {
				if (isRequired) {
					throw ex;
				}
			}
		}

		if (arg == null && isRequired) {
			if (!MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
				throw new MultipartException("Current request is not a multipart request");
			}
			else {
				throw new MissingServletRequestPartException(name);
			}
		}
		return adaptArgumentIfNecessary(arg, parameter);
	}
```

在文件上传请求发送过来后，所有的文件的文件流都被被直接封装在一个MultiValueMap中，而文件上传解析器的作用则是从这个MultiValueMap中根据字段名拿到对应的MultiPartFile（数组）对象。

MultiPartFile类有很多好用的方法：

```java
public interface MultipartFile extends InputStreamSource {
	//获取上传文件的参数名
	String getName();
	//获取上传的文件原来的名字
	@Nullable
	String getOriginalFilename();
	//获取文件类型
	@Nullable
	String getContentType();
	//判断文件是否合法
	boolean isEmpty();
	//获取文件大小
	long getSize();
	//获得字节数组形式的文件
	byte[] getBytes() throws IOException;
	//获取文件输入流
	@Override
	InputStream getInputStream() throws IOException;
	//获取资源类型
	default Resource getResource() {
		return new MultipartFileResource(this);
	}
    //保存文件
	void transferTo(File dest) throws IOException, IllegalStateException;
	//保存文件实际就是调用FileCopyUtils进行流拷贝
	default void transferTo(Path dest) throws IOException, IllegalStateException {
		FileCopyUtils.copy(getInputStream(), Files.newOutputStream(dest));
	}

}
```

### 错误处理

Springboot在执行过程中如果出现了异常，会默认转发到/error路由上

如果是机器客户端（如PostMan）则会返回JSON格式id错误信息以及状态码

如果是浏览器客户端则会返回一个错误页

在template目录下创建一个error目录，这个目录下的4xx.html和5xx.html（泛指以4开头和以5开头的状态码对于的页面）,页面会被自动解析，在状态码为对应值时会自动跳转到这个错误页，可以用具体的404.html,500.html来精确定位

也可以根据错误信息使用thymleaf语法设置错误页面的信息

#### 错误处理原理

我们来到配置类：ErrorMvcAutoConfiguration

和异常处理相关的配置都设置在这里

添加了一个错误处理组件：

```java
	@Bean
	@ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
	public DefaultErrorAttributes errorAttributes() {
		return new DefaultErrorAttributes();
	}
```

这个组件实现了接口： ErrorAttributes, HandlerExceptionResolver, Ordered

##### BasicErrorController

添加了一个Controller：

```java
	@Bean
	@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
	public BasicErrorController basicErrorController(ErrorAttributes errorAttributes,
			ObjectProvider<ErrorViewResolver> errorViewResolvers) {
		return new BasicErrorController(errorAttributes, this.serverProperties.getError(),
				errorViewResolvers.orderedStream().collect(Collectors.toList()));
	}
```

这个Controller中：

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
```

如果我们配置了server.error.path，就用这个路由，如果没有配置再看error.path有没有配置，如果也没有就按照/error路由来进行映射

也就是如果没有配置，这个Controller默认处理/error为前缀的请求

如果内容协商的结果是返回HTML页面：

```java
	@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
	public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
		HttpStatus status = getStatus(request);
		Map<String, Object> model = Collections
				.unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
		response.setStatus(status.value());
		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
        //如果没有找到404.html文件，也没有找到4xx.html文件，则会返回默认的异常界面
		return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
	}
```

会返回一个new ModelAndView("error", model)

如果协商结果不是HTML则返回一个Entity：

相当于返回了JSON

```java
	@RequestMapping
	public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
		HttpStatus status = getStatus(request);
		if (status == HttpStatus.NO_CONTENT) {
			return new ResponseEntity<>(status);
		}
		Map<String, Object> body = getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.ALL));
		return new ResponseEntity<>(body, status);
	}
```

容器中如果没有名为error的组件，会向容器中加入一个View类型的组件error

```java
		@Bean(name = "error")
		@ConditionalOnMissingBean(name = "error")
		public View defaultErrorView() {
			return this.defaultErrorView;
		}
```

所以如果返回的是HTML页面，返回new ModelAndView("error", model)时，会从Spring容器中拿到error组件作为视图返回

同时会放入视图解析器：

```java
		@Bean
		@ConditionalOnMissingBean
		public BeanNameViewResolver beanNameViewResolver() {
			BeanNameViewResolver resolver = new BeanNameViewResolver();
			resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
			return resolver;
		}
```

使用视图解析器就可以根据error这个id找到对于的view对象

然后就可以使用前面处理请求的逻辑来处理/error请求，也就是拿到包含由数据和视图的ModelAndView对象后，在处理返回值的流程中，调用view的render方法来渲染视图：

```java
@Override
		public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
				throws Exception {
			if (response.isCommitted()) {
				String message = getMessage(model);
				logger.error(message);
				return;
			}
			response.setContentType(TEXT_HTML_UTF8.toString());
			StringBuilder builder = new StringBuilder();
			Object timestamp = model.get("timestamp");
			Object message = model.get("message");
			Object trace = model.get("trace");
			if (response.getContentType() == null) {
				response.setContentType(getContentType());
			}
			builder.append("<html><body><h1>Whitelabel Error Page</h1>").append(
					"<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>")
					.append("<div id='created'>").append(timestamp).append("</div>")
					.append("<div>There was an unexpected error (type=").append(htmlEscape(model.get("error")))
					.append(", status=").append(htmlEscape(model.get("status"))).append(").</div>");
			if (message != null) {
				builder.append("<div>").append(htmlEscape(message)).append("</div>");
			}
			if (trace != null) {
				builder.append("<div style='white-space:pre-wrap;'>").append(htmlEscape(trace)).append("</div>");
			}
			builder.append("</body></html>");
			response.getWriter().append(builder.toString());
		}
```

所以其实就是根据数据拼接成一个HTML格式的字符串返回，也就是我们看到的错误页的来源

##### DefaultErrorViewResolver 异常视图解析器

这个视图用于根据异常名称解析错误页的，解析过程如下：

```java
	@Override
	public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
        //解析视图
		ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
		if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
			modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
		}
		return modelAndView;
	}
```

上面会调用resove方法：

```java
	private ModelAndView resolve(String viewName, Map<String, Object> model) {
		String errorViewName = "error/" + viewName;
		TemplateAvailabilityProvider provider = this.templateAvailabilityProviders.getProvider(errorViewName,
				this.applicationContext);
		if (provider != null) {
			return new ModelAndView(errorViewName, model);
		}
		return resolveResource(errorViewName, model);
	}

```

String errorViewName = "error/" + viewName 通过这条语句可以看到解析的视图地址是在/error目录下，并且视图名称是viewName

创建ModelAndView对象时，会默认从template目录寻找对于的html文件，而加上/error前缀后，默认的视图页就会从/templates/error目录下面找，而视图名称viewName 从哪里来呢，我们看调用这个方法的语句：

```
ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
```

将Http状态码作为viewName穿了进去，并且在寻找视图时会默认加上.html的后缀，所以在出现404的时候会找到404.html页面，依次类推。

而如果没有找到，则会来到下一条语句：

```java
	if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
		modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
	}
```

这条语句也是执行resolve方法，只是传入的viewName不一样，而SERIES_VIEWS.get(status.series())，追溯到最后就是

```java
		@Nullable
		public static Series resolve(int statusCode) {
			int seriesCode = statusCode / 100;
			for (Series series : values()) {
				if (series.value == seriesCode) {
					return series;
				}
			}
			return null;
		}
```

Series是个枚举类型，这个枚举类型有以下字段：

```
		INFORMATIONAL(1),
		SUCCESSFUL(2),
		REDIRECTION(3),
		CLIENT_ERROR(4),
		SERVER_ERROR(5);
```

这些字段都是Series类型，对于的value值是括号里的值。

所以这个方法的逻辑就是遍历这里所有的枚举类型，然后根据状态码/100判断是哪个series。也就是将状态码转换成2xx，3xx，4xx，5xx类型的格式，然后在template/error/目录下查找有无对于类型格式的html文件，例如404.html没有找到就会去找4xx.html文件

##### DefaultErrorAttributes

这个类中定义了返回值中需要包含的数据（需要包含在页面中，或者以JSON返回）：

如果就相关信息就添加相关信息，如果没有相关信息就从返回参数中移除

```java
	@Override
	public Map<String, Object> getErrorAttributes(WebRequest webRequest, ErrorAttributeOptions options) {
		Map<String, Object> errorAttributes = getErrorAttributes(webRequest, options.isIncluded(Include.STACK_TRACE));
		if (Boolean.TRUE.equals(this.includeException)) {
			options = options.including(Include.EXCEPTION);
		}
        //异常信息
		if (!options.isIncluded(Include.EXCEPTION)) {
			errorAttributes.remove("exception");
		}
        //调用路径
		if (!options.isIncluded(Include.STACK_TRACE)) {
			errorAttributes.remove("trace");
		}
        //相关信息
		if (!options.isIncluded(Include.MESSAGE) && errorAttributes.get("message") != null) {
			errorAttributes.put("message", "");
		}
        //错误
		if (!options.isIncluded(Include.BINDING_ERRORS)) {
			errorAttributes.remove("errors");
		}
		return errorAttributes;
	}
	@Override
	@Deprecated
	public Map<String, Object> getErrorAttributes(WebRequest webRequest, boolean includeStackTrace) {
		Map<String, Object> errorAttributes = new LinkedHashMap<>();
        //时间戳
		errorAttributes.put("timestamp", new Date());
		addStatus(errorAttributes, webRequest);
		addErrorDetails(errorAttributes, webRequest, includeStackTrace);
		addPath(errorAttributes, webRequest);
		return errorAttributes;
	}
	private void addStatus(Map<String, Object> errorAttributes, RequestAttributes requestAttributes) {
		Integer status = getAttribute(requestAttributes, RequestDispatcher.ERROR_STATUS_CODE);
		if (status == null) {
			errorAttributes.put("status", 999);
			errorAttributes.put("error", "None");
			return;
		}
        //状态码
		errorAttributes.put("status", status);
		try {
			errorAttributes.put("error", HttpStatus.valueOf(status).getReasonPhrase());
		}
		catch (Exception ex) {
			// Unable to obtain a reason
			errorAttributes.put("error", "Http Status " + status);
		}
	}
```

总结

BasicErrorController -》用于处理异常请求（/error），如果向定制化在发送错误时的响应则需要修改BasicErrorController 对象

DefaultErrorViewResolver -》用于查找错误页，如果不想根据Spring的规则返回错误页面可以修改这个视图解析器

DefaultErrorAttributes -》用于设置返回的参数，如果觉得返回的数据不够多，可以修改这个类，添加我们需要的参数（然后可以使用thymleaf定制我们想要的页面）

（不过一般情况下用Spring默认的错误处理机制即可）

#### 异常处理流程

我们再回顾以下doDispatch方法

```java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

既然是异常处理，所以我们关心try catch语句块即可，我们之前所讲的内容都是在第一层try 块中，所有的请求流程，包括解析url，拦截器，执行具体的方法等等只要出现异常就会跳转到catch语句块中。

所有的Exception和Error都会被记录在dispatchException中

如果是handle方法中出现了异常，会被catch，将当前请求状态设置为结束，然后向外抛出

执行请求以及处理完请求中的异常后会进入视图解析流程：

```
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

```java
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;

		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		if (mv != null && !mv.wasCleared()) {
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("No view rendering, null ModelAndView returned.");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		if (mappedHandler != null) {
			// Exception (if any) is already handled..
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
```

其中，如果在之前执行过程中出现了异常则会进入这个代码块，这个代码块中会获取错误页的ModelAndView数据

```java
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}
```

如果不是ModelAndViewException则会执行mv = processHandlerException(request, response, handler, exception)

```java
	@Nullable
	protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
			@Nullable Object handler, Exception ex) throws Exception {

		// Success and error responses may use different content types
		request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

		// Check registered HandlerExceptionResolvers...
		ModelAndView exMv = null;
		if (this.handlerExceptionResolvers != null) {
			for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
				exMv = resolver.resolveException(request, response, handler, ex);
				if (exMv != null) {
					break;
				}
			}
		}
		if (exMv != null) {
			if (exMv.isEmpty()) {
				request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
				return null;
			}
			// We might still need view name translation for a plain error model...
			if (!exMv.hasView()) {
				String defaultViewName = getDefaultViewName(request);
				if (defaultViewName != null) {
					exMv.setViewName(defaultViewName);
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using resolved error view: " + exMv, ex);
			}
			else if (logger.isDebugEnabled()) {
				logger.debug("Using resolved error view: " + exMv);
			}
			WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
			return exMv;
		}

		throw ex;
	}
```

用HandlerExceptionResolver来处理异常，遍历容器中所有的异常解析器，解析拿到ModelAndView后就退出循环。默认情况下没有解析器能处理这个异常，所以会被抛出。

然后就会doDispatch中，触发拦截器的后续的收尾方法后就结束了doDispatch方法，因而这个异常也就没有被处理，而如果异常没有被处理，会**转发一个error请求**（servlet规范规定的逻辑），然后会被自动配置类添加的**BasicErrorController**处理，而这个controller在处理异常的时候，会遍历所有的的ErrorViewResolver，尝试解析并拿到视图View，其中默认只有一个ErrorViewResolver（错误视图解析器）：DefaultErrorViewResolver ，在这个解析器中会根据Http状态码寻找HTML文件并返回。如果都没有找到就返回默认的空白异常界面。

#### 定制化错误处理

##### 在error目录下定值我们想要的404.html或者5xx.html（像这种写法的html文件）

html文件中可以使用thymleaf语法使用返回的数据，显示在界面上

##### 全局异常处理

全局范围内的所有异常都可以集中起来一起处理

```java
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandle {

    @ExceptionHandler(ArithmeticException.class)
    public String mathExceptionHandle(Exception e){
        log.error(e.getMessage());
        return "error/4xx";
    }
}
```

@ControllerAdvice申明这是一个处理异常的类，这个注解内部包含@Component注解，会把这个类注册进Spring容器中

@ExceptionHandler(ArithmeticException.class) 申明要捕获的异常，出现了异常后都会跳转到这里来处理

返回类型是String类型，就会也就返回View对象的地址，也可以直接返回ModelAndView对象，这样既返回视图也返回了数据。

如果加上了@ResponseBody则会返回JSON格式或者文本类型的数据

```java
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandle {

    @ExceptionHandler(ArithmeticException.class)
    @ResponseBody
    public String mathExceptionHandle(Exception e){
        log.error(e.getMessage());
        return "error/4xx";
    }
}
```

返回值规则和普通的Controller一样，只是这个类是专门用于处理异常的

原理如下：

之前我们提到过在执行mv = processHandlerException(request, response, handler, exception)方法时会遍历Spring容器中的异常解析器，Spring容器中的异常解析器有以下三种

![image-20220507004852744](../../../../学习笔记/picture/029c7937f23ed1c8dc2a5a81e74565d8-1669806992655-25.png) 

**ExceptionHandlerExceptionResolver**对应@ExceptionHandler(ArithmeticException.class)注解，在Spring启动时，会将括号中的class对象类型和方法建立映射关系并缓存起来。之前因为我们没有编写全局异常处理类，所以这里就没有解析器可以处理，而此时我们添加了对应的方法，并且出现了指定的异常，就可以用这个解析器执行我们设置的处理逻辑来处理这和异常

如果想抛出一个自定义异常，可以使用@ResponseStatus注解来自定义异常

```java
@NoArgsConstructor
@ResponseStatus(value = HttpStatus.FORBIDDEN,reason = "用户太多")
public class ToManyUserException extends RuntimeException {
    
}
```

在这个异常中可以重新设置自己的状态码和错误提示信息，并放到请求域中

使用这个注解后，在processHandlerException解析异常的时候，就可以使用**ResponseStatusExceptionResolver**这个解析器来处理这个异常，不过处理的时候并不会生产ModelAndView对象，而是调用response.sendError()方法向服务器发送一个Error，结束当前请求，然后按照Servlet的规则会转发一个/error请求，然后这个异常最后还是会根据状态码被错误页面处理，例如这里是403会返回4xx.html页面

而对于框架内部产生的异常（每一种状态码都对应一种异常），则是由第三种异常解析器**DefaultHandlerExceptionResolver**来解析异常，这个解析器能解析的异常如下：

```
Exception
HTTP Status Code
HttpRequestMethodNotSupportedException
405 (SC_METHOD_NOT_ALLOWED)
HttpMediaTypeNotSupportedException
415 (SC_UNSUPPORTED_MEDIA_TYPE)
HttpMediaTypeNotAcceptableException
406 (SC_NOT_ACCEPTABLE)
MissingPathVariableException
500 (SC_INTERNAL_SERVER_ERROR)
MissingServletRequestParameterException
400 (SC_BAD_REQUEST)
ServletRequestBindingException
400 (SC_BAD_REQUEST)
ConversionNotSupportedException
500 (SC_INTERNAL_SERVER_ERROR)
TypeMismatchException
400 (SC_BAD_REQUEST)
HttpMessageNotReadableException
400 (SC_BAD_REQUEST)
HttpMessageNotWritableException
500 (SC_INTERNAL_SERVER_ERROR)
MethodArgumentNotValidException
400 (SC_BAD_REQUEST)
MissingServletRequestPartException
400 (SC_BAD_REQUEST)
BindException
400 (SC_BAD_REQUEST)
NoHandlerFoundException
404 (SC_NOT_FOUND)
AsyncRequestTimeoutException
503 (SC_SERVICE_UNAVAILABLE)
```

而处理这些异常的方法相同：

都是直接向tomcat发送一个Error，表示结束当前请求，然后tomcat会再发送一个/error请求，然后被处理这个请求的controller捕获进行处理。

```java
	protected ModelAndView handleHttpMessageNotWritable(HttpMessageNotWritableException ex,
			HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws IOException {

		sendServerError(ex, request, response);
		return new ModelAndView();
	}
```

上述三个解析器都实现了HandlerExceptionResolver接口，我们也可以实现这个接口定义我们想要的异常解析器

```java
@Component
public class CustomerHandlerExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            response.sendError(505,"我的错误");
        } catch (IOException e) {
            e.printStackTrace();
        }
        return new ModelAndView();
    }
}
```

这样在解析错误的时候就会多出一种异常解析器，但是此时我们的异常解析器的优先级最低，Spring自带的解析器生效后就不会再去执行我们自定义的解析器。

如果想要我们设置的异常解析器生效，可以加上@Order注解来设置组件的加载顺序

比如这个注解可以设置最高优先级，其实就是一个INT数的最小值，value值越小，优先级越高，我们也可以直接填入一个数字来合理规划优先级顺序。

```
@Order(value = Ordered.HIGHEST_PRECEDENCE)
```

总结：

使用respond.sendError()方法或者出现了异常而Spring容器的异常解析器均无法处理，则Tomcat会转发一个/error请求，然后被basicController捕获，因而basicController可以处理所有的异常。