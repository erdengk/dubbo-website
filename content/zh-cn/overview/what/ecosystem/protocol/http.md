---
aliases:
    - /zh/overview/what/ecosystem/protocol/http/
description: ""
linkTitle: http
title: http
type: docs
weight: 3
---



基于标准的 Java REST API——JAX-RS 2.0（Java API for RESTful Web Services 的简写）实现的 REST 调用支持

## 特性说明
此协议提供通过 web 访问服务的简单方式，将服务与其他基于 web 的应用程序集成。
支持 JSON、XML 和 Text 格式的请求和响应，发布和使用服务的便捷方式,也提供了服务版本控制、服务过滤、服务元数据和服务参数, 实现 Dubbo 框架的灵活性和可伸缩性。

## 使用场景

## 使用方式

### 快速入门

在 dubbo 中开发一个 REST 风格的服务会比较简单，下面以一个注册用户的简单服务为例说明。

这个服务要实现的功能是提供如下 URL（注：这个URL不是完全符合 REST 的风格，但是更简单实用）
```
http://localhost:8080/users/register
```
而任何客户端都可以将包含用户信息的 JSON 字符串 POST 到以上 URL 来完成用户注册。

首先，开发服务的接口

```java
public class UserService {    
   void registerUser(User user);
}
```

然后，开发服务的实现

```java
@Path("users")
public class UserServiceImpl implements UserService {
       
    @POST
    @Path("register")
    @Consumes({MediaType.APPLICATION_JSON})
    public void registerUser(User user) {
        // save the user...
    }
}
```
上面的实现非常简单，但是由于该 REST 服务是要发布到指定 URL 上，供任意语言的客户端甚至浏览器来访问，所以这里额外添加了几个 JAX-RS 的标准 annotation 来做相关的配置。

@Path("users")：指定访问 UserService 的 URL 相对路径是 /users，即 http://localhost:8080/users

@Path("register")：指定访问 registerUser() 方法的 URL 相对路径是 /register，再结合上一个 @Path为UserService 指定的路径，则调用 UserService.register() 的完整路径为 http://localhost:8080/users/register

@POST：指定访问 registerUser()用HTTP POST方法

@Consumes({MediaType.APPLICATION_JSON})：指定 registerUser() 接收 JSON 格式的数据。REST 框架会自动将 JSON 数据反序列化为 User 对象

最后，在 spring 配置文件中添加此服务，即完成所有服务开发工作

 ```xml
<!-- 用rest协议在8080端口暴露服务 -->
<dubbo:protocol name="rest" port="8080"/>
 
<!-- 声明需要暴露的服务接口 -->
<dubbo:service interface="xxx.UserService" ref="userService"/>
 
<!-- 和本地bean一样实现服务 -->
<bean id="userService" class="xxx.UserServiceImpl" />
``` 

### REST 服务提供端

下面我们扩充“快速入门”中的UserService，进一步展示在dubbo中REST服务提供端的开发要点。

### HTTP POST/GET 的实现

REST 服务中虽然建议使用 HTTP 协议中四种标准方法 POST、DELETE、PUT、GET 来分别实现常见的“增删改查”，但实际中，我们一般情况直接用POST来实现“增改”，GET 来实现“删查”即可（DELETE 和 PUT 甚至会被一些防火墙阻挡）。

前面已经简单演示了 POST 的实现，在此，我们为 UserService 添加一个获取注册用户资料的功能，来演示 GET 的实现。

这个功能就是要实现客户端通过访问如下不同 URL 来获取不同 ID 的用户资料

```
http://localhost:8080/users/1001
http://localhost:8080/users/1002
http://localhost:8080/users/1003
```

当然，也可以通过其他形式的URL来访问不同 ID 的用户资料，例如

```
http://localhost:8080/users/load?id=1001
```

JAX-RS 本身可以支持所有这些形式。但是上面那种在 URL 路径中包含查询参数的形式（http://localhost:8080/users/1001） 更符合 REST 的一般习惯，所以更推荐大家来使用。下面我们就为 UserService 添加一个 getUser() 方法来实现这种形式的 URL 访问

```java
@GET
@Path("{id : \\d+}")
@Produces({MediaType.APPLICATION_JSON})
public User getUser(@PathParam("id") Long id) {
    // ...
}
```

@GET：指定用 HTTP GET 方法访问

@Path("{id : \\d+}")：根据上面的功能需求，访问 getUser() 的 URL 应当是 “http://localhost:8080/users/ + 任意数字"，并且这个数字要被做为参数传入 getUser() 方法。 这里的 annotation 配置中，@Path中间的 {id: xxx} 指定 URL 相对路径中包含了名为id参数，而它的值也将被自动传递给下面用 @PathParam("id") 修饰的方法参数 id。{id:后面紧跟的\\d+ 是一个正则表达式，指定了 id 参数必须是数字。

@Produces({MediaType.APPLICATION_JSON})：指定getUser()输出JSON格式的数据。框架会自动将User对象序列化为JSON数据。

### Annotation

在 Dubbo 中开发 REST 服务主要都是通过 JAX-RS的annotation 来完成配置的，在上面的示例中，我们都是将 annotation 放在服务的实现类中。但其实，我们完全也可以将 annotation 放到服务的接口上，这两种方式是完全等价的，例如：

```java
@Path("users")
public interface UserService {
    
    @GET
    @Path("{id : \\d+}")
    @Produces({MediaType.APPLICATION_JSON})
    User getUser(@PathParam("id") Long id);
}
```

在一般应用中，我们建议将 annotation 放到服务实现类，这样 annotation 和 java 实现代码位置更接近，更便于开发和维护。另外更重要的是，我们一般倾向于避免对接口的污染，保持接口的纯净性和广泛适用性。

但是，如后文所述，如果我们要用 dubbo 直接开发的消费端来访问此服务，则 annotation 必须放到接口上。

如果接口和实现类都同时添加了 annotation，则实现类的 annotation 配置会生效，接口上的 annotation 被直接忽略。

### 多数据格式支持

在 dubbo 中开发的 REST 服务可以同时支持传输多种格式的数据，以给客户端提供最大的灵活性。其中我们目前对最常用的 JSON 和 XML 格式特别添加了额外的功能。

比如，我们要让上例中的getUser()方法支持分别返回 JSON 和 XML 格式的数据，只需要在 annotation 中同时包含两种格式即可

```java
@Produces({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})
User getUser(@PathParam("id") Long id);
```

或者也可以直接用字符串（还支持通配符）表示 MediaType

```java
@Produces({"application/json", "text/xml"})
User getUser(@PathParam("id") Long id);
```

如果所有方法都支持同样类型的输入输出数据格式，则我们无需在每个方法上做配置，只需要在服务类上添加 annotation 即可

```java
@Path("users")
@Consumes({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})
@Produces({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})
public class UserServiceImpl implements UserService {
    // ...
}
```

在一个 REST 服务同时对多种数据格式支持的情况下，根据 JAX-RS 标准，一般是通过HTTP中的MIME header（content-type和accept）来指定当前想用的是哪种格式的数据。

但是在 dubbo 中，我们还自动支持目前业界普遍使用的方式，即用一个 URL 后缀（.json和.xml）来指定想用的数据格式。例如，在添加上述 annotation后，直接访问 http://localhost:8888/users/1001.json 则表示用 json 格式，直接访问 http://localhost:8888/users/1002.xml 则表示用 xml 格式，比用 HTTP Header 更简单直观。Twitter、微博等的 REST API 都是采用这种方式。
如果你既不加 HTTP header，也不加后缀，则 dubbo 的 REST 会优先启用在以上 annotation 定义中排位最靠前的那种数据格式。

> 注意：这里要支持 XML 格式数据，在 annotation 中既可以用 MediaType.TEXT_XML，也可以用 MediaType.APPLICATION_XML，但是 TEXT_XML 是更常用的，并且如果要利用上述的 URL 后缀方式来指定数据格式，只能配置为 TEXT_XML 才能生效。
### 中文字符支持

为了在 dubbo REST 中正常输出中文字符，和通常的 Java web 应用一样，我们需要将 HTTP 响应的 contentType 设置为 UTF-8编码。

基于 JAX-RS 的标准用法，我们只需要做如下 annotation 配置即可：

```java
@Produces({"application/json; charset=UTF-8", "text/xml; charset=UTF-8"})
User getUser(@PathParam("id") Long id);
```

为了方便用户，我们在 dubbo REST 中直接添加了一个支持类，来定义以上的常量，可以直接使用，减少出错的可能性。

```java
@Produces({ContentType.APPLICATION_JSON_UTF_8, ContentType.TEXT_XML_UTF_8})
User getUser(@PathParam("id") Long id);
```

### XML 数据格式

由于 JAX-RS 的实现一般都用标准的 JAXB（Java API for XML Binding）来序列化和反序列化 XML 格式数据，所以我们需要为每一个要用 XML 传输的对象添加一个类级别的 JAXB annotation，否则序列化将报错。例如为 getUser() 中返回的 User 添加如下

```java
@XmlRootElement
public class User implements Serializable {
    // ...
}
```	

此外，如果service方法中的返回值是Java的 primitive类型（如int，long，float，double等），最好为它们添加一层wrapper对象，因为JAXB不能直接序列化primitive类型。

例如，我们想让前述的registerUser()方法返回服务器端为用户生成的ID号：

```java
long registerUser(User user);
```

由于 primitive 类型不被 JAXB 序列化支持，所以添加一个 wrapper 对象：

```java
@XmlRootElement
public class RegistrationResult implements Serializable {
    
    private Long id;
    
    public RegistrationResult() {
    }
    
    public RegistrationResult(Long id) {
        this.id = id;
    }
    
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
}
```

并修改 service 方法：

```java
RegistrationResult registerUser(User user);
```

这样不但能够解决 XML 序列化的问题，而且使得返回的数据都符合 XML 和 JSON 的规范。例如，在 JSON中，返回的将是如下形式

```javascript
{"id": 1001}
```

如果不加 wrapper，JSON 返回值将直接是

```
1001 	
```

而在 XML 中，加 wrapper 后返回值将是：

```xml
<registrationResult>
    <id>1002</id>
</registrationResult>
```

这种 wrapper 对象其实利用所谓 Data Transfer Object（DTO）模式，采用 DTO 还能对传输数据做更多有用的定制。

### 定制序列化

如上所述，REST 的底层实现会在 service 的对象和 JSON/XML 数据格式之间自动做序列化/反序列化。但有些场景下，如果觉得这种自动转换不满足要求，可以对其做定制。

Dubbo 中的 REST 实现是用 JAXB 做 XML 序列化，用 Jackson 做 JSON 序列化，所以在对象上添加 JAXB 或 Jackson 的 annotation 即可以定制映射。

例如，定制对象属性映射到 XML 元素的名字：

```java
@XmlRootElement
@XmlAccessorType(XmlAccessType.FIELD)
public class User implements Serializable {
    
    @XmlElement(name="username") 
    private String name;  
}
```

定制对象属性映射到 JSON 字段的名字：

```java
public class User implements Serializable {
    
    @JsonProperty("username")
    private String name;
}
```

更多资料请参考 JAXB 和 Jackson 的官方文档，或自行 google。

### REST Server 的实现

目前在 dubbo 中，我们支持5种嵌入式 rest server 的实现，并同时支持采用外部应用服务器来做 rest server 的实现。rest server 可以通过如下配置实现：

```xml
<dubbo:protocol name="rest" server="jetty"/>
```

以上配置选用了嵌入式的 jetty 来做 rest server，同时，如果不配置 server 属性，rest 协议默认也是选用 jetty。jetty 是非常成熟的 java servlet 容器，并和 dubbo 已经有较好的集成（目前5种嵌入式 server 中只有 jetty 和后面所述的 tomcat、tjws，与 dubbo 监控系统等完成了无缝的集成），所以，如果你的 dubbo 系统是单独启动的进程，你可以直接默认采用 jetty 即可。


```xml
<dubbo:protocol name="rest" server="tomcat"/>
```

以上配置选用了嵌入式的 tomcat 来做 rest server。在嵌入式 tomcat 上，REST 的性能比 jetty 上要好得多（参见后面的基准测试），建议在需要高性能的场景下采用 tomcat。

```xml
<dubbo:protocol name="rest" server="netty"/>
```

以上配置选用嵌入式的 netty 来做 rest server。（TODO more contents to add）

```xml
<dubbo:protocol name="rest" server="tjws"/> (tjws is now deprecated)
<dubbo:protocol name="rest" server="sunhttp"/>
```

以上配置选用嵌入式的 tjws 或 Sun HTTP server 来做 rest server。这两个 server 实现非常轻量级，非常方便在集成测试中快速启动使用，当然也可以在负荷不高的生产环境中使用。	注：tjws目前已经被deprecated掉了，因为它不能很好的和servlet 3.1 API工作。

如果你的 dubbo 系统不是单独启动的进程，而是部署到了 Java 应用服务器中，则建议你采用以下配置

```xml
<dubbo:protocol name="rest" server="servlet"/>
```

通过将 server 设置为 servlet，dubbo 将采用外部应用服务器的 servlet 容器来做 rest server。同时，还要在 dubbo 系统的 web.xml 中添加如下配置

```xml
<web-app>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/classes/META-INF/spring/dubbo-demo-provider.xml</param-value>
    </context-param>
    
    <listener>
        <listener-class>org.apache.dubbo.remoting.http.servlet.BootstrapListener</listener-class>
    </listener>
    
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.apache.dubbo.remoting.http.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
</web-app>
```

即必须将 dubbo 的 BootstrapListener 和 DispatherServlet 添加到 web.xml，以完成 dubbo 的 REST 功能与外部 servlet 容器的集成。

> 注意：如果你是用 spring 的 ContextLoaderListener 来加载 spring，则必须保证 BootstrapListener 配置在 ContextLoaderListener 之前，否则 dubbo 初始化会出错。
其实，这种场景下你依然可以坚持用嵌入式 server，但外部应用服务器的 servlet 容器往往比嵌入式 server 更加强大（特别是如果你是部署到更健壮更可伸缩的 WebLogic，WebSphere 等），另外有时也便于在应用服务器做统一管理、监控等等。

### 获取 Context 信息

在远程调用中，值得获取的上下文信息可能有很多种，这里特别以获取客户端 IP 为例。

在 dubbo 的 REST 中，我们有两种方式获取客户端 IP。

第一种方式，用 JAX-RS 标准的 @Context annotation

```java
public User getUser(@PathParam("id") Long id, @Context HttpServletRequest request) {
    System.out.println("Client address is " + request.getRemoteAddr());
} 
```

用 Context 修饰 getUser() 的一个方法参数后，就可以将当前的 HttpServletRequest 注入进来，然后直接调用 servlet api 获取 IP。

> 注意：这种方式只能在将server设置为 tjws、tomcat、jetty 或者 servlet 的时候才能工作，因为只有这几种 server 的实现才提供了 servlet 容器。另外，标准的JAX-RS还支持用@Context修饰service类的一个实例字段来获取HttpServletRequest，但在dubbo中我们没有对此作出支持。
第二种方式，用 dubbo 中常用的 RpcContext

```java
public User getUser(@PathParam("id") Long id) {
    System.out.println("Client address is " + RpcContext.getContext().getRemoteAddressString());
} 
```

> 注意：这种方式只能在设置 server="jetty" 或者 server="tomcat" 或者 server="servlet" 或者 server="tjws" 的时候才能工作。另外，目前 dubbo 的 RpcContext 是一种比较有侵入性的用法，未来我们很可能会做出重构。
如果你想保持你的项目对 JAX-RS 的兼容性，未来脱离 dubbo 也可以运行，请选择第一种方式。如果你想要更优雅的服务接口定义，请选用第二种方式。

此外，在最新的 dubbo rest 中，还支持通过 RpcContext 来获取 HttpServletRequest和 HttpServletResponse，以提供更大的灵活性来方便用户实现某些复杂功能，比如在 dubbo 标准的 filter 中访问 HTTP Header。用法示例如下

```java
if (RpcContext.getContext().getRequest() != null && RpcContext.getContext().getRequest() instanceof HttpServletRequest) {
    System.out.println("Client address is " + ((HttpServletRequest) RpcContext.getContext().getRequest()).getRemoteAddr());
}
if (RpcContext.getContext().getResponse() != null && RpcContext.getContext().getResponse() instanceof HttpServletResponse) {
    System.out.println("Response object from RpcContext: " + RpcContext.getContext().getResponse());
}
```

> 注意：为了保持协议的中立性，RpcContext.getRequest()和RpcContext.getResponse()返回的仅仅是一个Object类，而且可能为null。所以，你必须自己做null和类型的检查。
> 注意：只有在设置server="jetty"或者server="tomcat"或者server="servlet"的时候，你才能通过以上方法正确的得到HttpServletRequest和HttpServletResponse，因为只有这几种server实现了servlet容器。
为了简化编程，在此你也可以用泛型的方式来直接获取特定类型的 request/response：

```java
if (RpcContext.getContext().getRequest(HttpServletRequest.class) != null) {
    System.out.println("Client address is " + RpcContext.getContext().getRequest(HttpServletRequest.class).getRemoteAddr());
}
if (RpcContext.getContext().getResponse(HttpServletResponse.class) != null) {
    System.out.println("Response object from RpcContext: " + RpcContext.getContext().getResponse(HttpServletResponse.class));
}
```

如果 request/response 不符合指定的类型，这里也会返回 null。

### 端口号和 Context Path

dubbo 中的 rest 协议默认将采用80端口，如果想修改端口，直接配置：

```xml
<dubbo:protocol name="rest" port="8888"/>
```

另外，如前所述，我们可以用 @Path 来配置单个 rest 服务的 URL 相对路径。但其实，我们还可以设置一个所有 rest 服务都适用的基础相对路径，即 java web 应用中常说的 context path。

只需要添加如下 contextpath 属性即可：

```xml
<dubbo:protocol name="rest" port="8888" contextpath="services"/>
```

以前面代码为例：

```java
@Path("users")
public class UserServiceImpl implements UserService {
       
    @POST
    @Path("register")
    @Consumes({MediaType.APPLICATION_JSON})
    public void registerUser(User user) {
        // save the user...
    }	
}
```

现在 registerUser() 的完整访问路径

```
http://localhost:8888/services/users/register
```

注意：如果你是选用外部应用服务器做 rest server，即配置

```xml
<dubbo:protocol name="rest" port="8888" contextpath="services" server="servlet"/>
```

则必须保证这里设置的 port、contextpath，与外部应用服务器的端口、DispatcherServlet 的上下文路径（即 webapp path 加上 servlet url pattern）保持一致。例如，对于部署为 tomcat ROOT 路径的应用，这里的 contextpath 必须与 web.xml 中 DispacherServlet 的`<url-pattern/>` 完全一致：

```xml
<servlet-mapping>
     <servlet-name>dispatcher</servlet-name>
     <url-pattern>/services/*</url-pattern>
</servlet-mapping>
```

### 线程数和 IO 线程数

可以为 rest 服务配置线程池大小

```xml
<dubbo:protocol name="rest" threads="500"/>
```

> 注意：目前线程池的设置只有当server="netty"或者server="jetty"或者server="tomcat"的时候才能生效。另外，如果server="servlet"，由于这时候启用的是外部应用服务器做rest server，不受dubbo控制，所以这里的线程池设置也无效。
如果是选用 netty server，还可以配置 Netty 的 IO worker 线程数

```xml
<dubbo:protocol name="rest" iothreads="5" threads="100"/>
```

### 配置长连接

Dubbo 中的 rest 服务默认都是采用 http 长连接来访问，如果想切换为短连接，直接配置

```xml
<dubbo:protocol name="rest" keepalive="false"/>
```

> 注意：这个配置目前只对 server="netty"和server="tomcat" 才能生效。
### 最大 HTTP 连接数

可以配置服务器提供端所能同时接收的最大 HTTP 连接数，防止 REST server 被过多连接撑爆，以作为一种最基本的自我保护机制

```xml
<dubbo:protocol name="rest" accepts="500" server="tomcat/>
```
> 注意：这个配置目前只对server="tomcat"才能生效。
### 每个消费端的超时时间和 HTTP 连接数
如果 rest 服务的消费端也是 dubbo 系统，可以像其他 dubbo RPC 机制一样，配置消费端调用此 rest 服务的最大超时时间以及每个消费端所能启动的最大 HTTP 连接数。
```xml
<dubbo:service interface="xxx" ref="xxx" protocol="rest" timeout="2000" connections="10"/>
```
当然，由于这个配置针对消费端生效的，所以也可以在消费端配置
```xml
<dubbo:reference id="xxx" interface="xxx" timeout="2000" connections="10"/>
```
但是，通常我们建议配置在服务提供端提供此类配置。按照 dubbo 官方文档的说法：“Provider 上尽量多配置 Consumer 端的属性，让 Provider 实现者一开始就思考 Provider 服务特点、服务质量的问题。”
> 注意：如果 dubbo 的 REST 服务是发布给非 dubbo 的客户端使用，则这里 `<dubbo:service/>` 上的配置完全无效，因为这种客户端不受 dubbo 控制。
### Annotation 取代部分 Spring XML 配置
以上所有的讨论都是基于 dubbo 在 spring 中的 xml 配置。但是，dubbo/spring 本身也支持用 annotation 来作配置，所以我们也可以按dubbo官方文档中的步骤，把相关 annotation 加到 REST 服务的实现中，取代一些 xml 配置，例如
```java
@Service(protocol = "rest")
@Path("users")
public class UserServiceImpl implements UserService {
    @Autowired
    private UserRepository userRepository;
       
    @POST
    @Path("register")
    @Consumes({MediaType.APPLICATION_JSON})
    public void registerUser(User user) {
        // save the user
        userRepository.save(user);
    }	
}
```
annotation 的配置更简单更精确，通常也更便于维护（当然现代IDE都可以在xml中支持比如类名重构，所以就这里的特定用例而言，xml 的维护性也很好）。而 xml 对代码的侵入性更小一些，尤其有利于动态修改配置，特别是比如你要针对单个服务配置连接超时时间、每客户端最大连接数、集群策略、权重等等。另外，特别对复杂应用或者模块来说，xml 提供了一个中心点来涵盖的所有组件和配置，更一目了然，一般更便于项目长时期的维护。
当然，选择哪种配置方式没有绝对的优劣，和个人的偏好也不无关系。
### 添加自定义的 Filter、Interceptor
Dubbo 的 REST 也支持 JAX-RS 标准的 Filter 和 Interceptor，以方便对 REST 的请求与响应过程做定制化的拦截处理。
其中，Filter 主要用于访问和设置 HTTP 请求和响应的参数、URI 等等。例如，设置 HTTP 响应的 cache header：
```java
public class CacheControlFilter implements ContainerResponseFilter {
    public void filter(ContainerRequestContext req, ContainerResponseContext res) {
        if (req.getMethod().equals("GET")) {
            res.getHeaders().add("Cache-Control", "someValue");
        }
    }
}
```
Interceptor 主要用于访问和修改输入与输出字节流，例如，手动添加 GZIP 压缩
```java
public class GZIPWriterInterceptor implements WriterInterceptor {
 
    @Override
    public void aroundWriteTo(WriterInterceptorContext context)
                    throws IOException, WebApplicationException {
        OutputStream outputStream = context.getOutputStream();
        context.setOutputStream(new GZIPOutputStream(outputStream));
        context.proceed();
    }
}
```
在标准 JAX-RS 应用中，我们一般是为 Filter 和 Interceptor 添加 @Provider annotation，然后 JAX-RS runtime 会自动发现并启用它们。而在 dubbo 中，我们是通过添加XML配置的方式来注册 Filter 和 Interceptor：
```xml
<dubbo:protocol name="rest" port="8888" extension="xxx.TraceInterceptor, xxx.TraceFilter"/>
```
在此，我们可以将 Filter、Interceptor 和 DynamicFeature 这三种类型的对象都添加到 `extension` 属性上，多个之间用逗号分隔。（DynamicFeature 是另一个接口，可以方便我们更动态的启用 Filter 和 Interceptor，感兴趣请自行 google。）
当然，dubbo 自身也支持 Filter 的概念，但我们这里讨论的 Filter 和 Interceptor 更加接近协议实现的底层，相比 dubbo 的 filter，可以做更底层的定制化。
> 注：这里的 XML 属性叫 extension，而不是叫 interceptor 或者 filter，是因为除了 Interceptor 和 Filter，未来我们还会添加更多的扩展类型。
如果 REST 的消费端也是 dubbo 系统（参见下文的讨论），则也可以用类似方式为消费端配置 Interceptor 和 Filter。但注意，JAX-RS 中消费端的 Filter 和提供端的 Filter 是两种不同的接口。例如前面例子中服务端是 ContainerResponseFilter 接口，而消费端对应的是 ClientResponseFilter:
```java
public class LoggingFilter implements ClientResponseFilter {
 
    public void filter(ClientRequestContext reqCtx, ClientResponseContext resCtx) throws IOException {
        System.out.println("status: " + resCtx.getStatus());
	    System.out.println("date: " + resCtx.getDate());
	    System.out.println("last-modified: " + resCtx.getLastModified());
	    System.out.println("location: " + resCtx.getLocation());
	    System.out.println("headers:");
	    for (Entry<String, List<String>> header : resCtx.getHeaders().entrySet()) {
     	    System.out.print("\t" + header.getKey() + " :");
	        for (String value : header.getValue()) {
	            System.out.print(value + ", ");
	        }
	        System.out.print("\n");
	    }
	    System.out.println("media-type: " + resCtx.getMediaType().getType());
    } 
}
```
### 添加自定义的 Exception 处理
Dubbo 的 REST 也支持 JAX-RS 标准的 ExceptionMapper，可以用来定制特定 exception 发生后应该返回的 HTTP 响应。
```java
public class CustomExceptionMapper implements ExceptionMapper<NotFoundException> {
    public Response toResponse(NotFoundException e) {     
        return Response.status(Response.Status.NOT_FOUND).entity("Oops! the requested resource is not found!").type("text/plain").build();
    }
}
```
和 Interceptor、Filter 类似，将其添加到 XML 配置文件中即可启用
```xml
<dubbo:protocol name="rest" port="8888" extension="xxx.CustomExceptionMapper"/>
```
### HTTP 日志输出
Dubbo rest 支持输出所有 HTTP 请求/响应中的 header 字段和 body 消息体。
在 XML 配置中添加如下自带的 REST filter：
```xml
<dubbo:protocol name="rest" port="8888" extension="org.apache.dubbo.rpc.protocol.rest.support.LoggingFilter"/>
```
然后配置在 logging 配置中至少为 org.apache.dubbo.rpc.protocol.rest.support 打开 INFO 级别日志输出，例如，在 log4j.xml 中配置
```xml
<logger name="org.apache.dubbo.rpc.protocol.rest.support">
    <level value="INFO"/>
    <appender-ref ref="CONSOLE"/>
</logger>
```
当然，你也可以直接在 ROOT logger 打开 INFO 级别日志输出
```xml
<root>
	<level value="INFO" />
	<appender-ref ref="CONSOLE"/>
</root>
```
然后在日志中会有类似如下的内容输出
```
The HTTP headers are: 
accept: application/json;charset=UTF-8
accept-encoding: gzip, deflate
connection: Keep-Alive
content-length: 22
content-type: application/json
host: 192.168.1.100:8888
user-agent: Apache-HttpClient/4.2.1 (java 1.5)
```
```
The contents of request body is: 
{"id":1,"name":"dang"}
```
打开 HTTP 日志输出后，除了正常日志输出的性能开销外，也会在比如 HTTP 请求解析时产生额外的开销，因为需要建立额外的内存缓冲区来为日志的输出做数据准备。
### 输入参数的校验
dubbo 的 rest 支持采用 Java 标准的 bean validation annotation（JSR 303) 来做输入校验 http://beanvalidation.org/
为了和其他 dubbo 远程调用协议保持一致，在 rest 中作校验的 annotation 必须放在服务的接口上，例如
```java
public interface UserService {
   
    User getUser(@Min(value=1L, message="User ID must be greater than 1") Long id);
}
```
当然，在很多其他的 bean validation 的应用场景都是将 annotation 放到实现类而不是接口上。把 annotation 放在接口上至少有一个好处是，dubbo 的客户端可以共享这个接口的信息，dubbo 甚至不需要做远程调用，在本地就可以完成输入校验。
然后按照 dubbo 的标准方式在 XML 配置中打开验证：
```xml
<dubbo:service interface=xxx.UserService" ref="userService" protocol="rest" validation="true"/>
```

在 dubbo 的其他很多远程调用协议中，如果输入验证出错，是直接将 `RpcException` 抛向客户端，而在 rest 中由于客户端经常是非 dubbo，甚至非 java 的系统，所以不便直接抛出 Java 异常。因此，目前我们将校验错误以 XML 的格式返回

```xml
<violationReport>
    <constraintViolations>
        <path>getUserArgument0</path>
        <message>User ID must be greater than 1</message>
        <value>0</value>
    </constraintViolations>
</violationReport>
```

稍后也会支持其他数据格式的返回值。至于如何对验证错误消息作国际化处理，直接参考 bean validation 的相关文档即可。

如果你认为默认的校验错误返回格式不符合你的要求，可以如上面章节所述，添加自定义的 ExceptionMapper 来自由的定制错误返回格式。需要注意的是，这个 ExceptionMapper 必须用泛型声明来捕获 dubbo 的 RpcException，才能成功覆盖 dubbo rest 默认的异常处理策略。为了简化操作，其实这里最简单的方式是直接继承 dubbo rest 的 RpcExceptionMapper，并覆盖其中处理校验异常的方法即可

```java
public class MyValidationExceptionMapper extends RpcExceptionMapper {
    protected Response handleConstraintViolationException(ConstraintViolationException cve) {
        ViolationReport report = new ViolationReport();
        for (ConstraintViolation cv : cve.getConstraintViolations()) {
            report.addConstraintViolation(new RestConstraintViolation(
                    cv.getPropertyPath().toString(),
                    cv.getMessage(),
                    cv.getInvalidValue() == null ? "null" : cv.getInvalidValue().toString()));
        }
        // 采用json输出代替xml输出
        return Response.status(Response.Status.INTERNAL_SERVER_ERROR).entity(report).type(ContentType.APPLICATION_JSON_UTF_8).build();
    }
}
```

然后将这个 ExceptionMapper 添加到 XML 配置中即可：

```xml
<dubbo:protocol name="rest" port="8888" extension="xxx.MyValidationExceptionMapper"/>
```
