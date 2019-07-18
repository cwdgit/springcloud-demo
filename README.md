## 1.部署目的：

为了方便制定项目配置信息规范，让大家了解springcloud项目的相关配置。部署这个demo供大家参考。

## 2.demo说明：

源项目地址

https://github.com/cattles/spring-cloud-event-sourcing-example

该项目是一个电商类型平台，包含下列模块。

- Discovery Service
- Edge Service
- User Service
- Catalog Service
- Account Service
- Order Service
- Inventory Service
- Online Store Web
- Shopping Cart Service

![Online Store Architecture Diagram](http://i.imgur.com/PBCVt90.png)

选取下列服务来部署demo

- Discovery Service
- Edge Service
- User Service
- Online Store Web
- Config Service

## 3.服务部署

说明：maven中集成了docker，编译会自动build镜像，修改父pom文件中关于docker镜像的配置，可以指定镜像name等配置。

pom.xml文件中docker镜像配置

```xml
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
        <docker.image.prefix>10.11.0.8:5000</docker.image.prefix>
        <docker.plugin.version>0.3.258</docker.plugin.version>
        <dockerfile.plugin.version>1.3.6</dockerfile.plugin.version>
    </properties>
```

修改pom文件或者执行 mvn package -D docker.image.prefix=${REGISTRY}  -D maven.test.skip=true 都可以指定镜像名称。

### Config service(配置中心)

bootstrap.yml

config service 配置管理分为本地目录和远端git管理，本地目录指定file路径，在平台中挂载configmap来实现配置文件灵活配置。

```yaml
server:
  port: 8888
spring:
  profiles:
    active: native
  application:
    name: config-server
  cloud:
    config:
      server:
        native:
          search-locations: file:/app/config

---
server:
  port: 8888
spring:
  profiles: docker
  application:
    name: config-service
  cloud:
    config:
      server:
        git:
          uri: http://${GIT_HOST}/root/config-server
          username: root
          password: alauda1234

server.session.cookie.name: config-service
```

运行配置文件时指定profiles，来设置使用哪里的配置环境。

```yaml
SPRING_PROFILES_ACTIVE=native
```
![Online Store Architecture Diagram](https://github.com/cwdgit/springcloud-demo/blob/master/img/config-service1.png)
![Online Store Architecture Diagram](https://github.com/cwdgit/springcloud-demo/blob/master/img/config-service2.png)
### Discovery Service(注册中心)

bootstrap.yml

```yaml
server:
  port: 8761
spring:
  application:
    name: discovery-service

---
spring:
  profiles: docker
  cloud:
    config:
      uri: http://config-service:8888

server.session.cookie.name: discovery-service
```

启动时指定了自身相关信息，指定注册中心地址

启动时在pass平台添加环境变量,来指定使用的配置。

```yaml
SPRING_PROFILES_ACTIVE=docker
```
![Online Store Architecture Diagram](https://github.com/cwdgit/springcloud-demo/blob/master/img/discovery.png)
### Edge Service(服务网关)

bootstrap.yml

```yaml
spring:
  application:
    name: edge-service
server:
  port: 9999
---
spring:
  profiles: docker
  cloud:
    config:
      uri: http://config-service:8888
---
spring:
  profiles: default
  cloud:
    config:
      uri: http://localhost:8888
spring.profiles.active: development
---
spring:
  profiles: cloud
  cloud:
    config:
      uri: ${vcap.services.config-service.credentials.uri:http://localhost:8888}
```

application.yml

```yaml
---
spring:
  profiles: docker
  application:
    name: edge-service
  sleuth:
    sampler:
      percentage: 1.0
    traceId128: true
  zipkin:
    base-url: http://zipkin:9411/
zuul:
  ignored-services: '*'
  ignoredPatterns: /**/api/**
  routes:
    account-service: /account/**
    payment-service: /payment/**
    inventory-service: /inventory/**
    order-service: /order/**
    user-service: /user/**
    catalog-service: /catalog/**
    shopping-cart-service: /shoppingcart/**
security:
  oauth2:
    resource:
      userInfoUri: http://user-service:8181/uaa/user   
#这里跳转使用的是servicename，前端跳转是无法解析的，在实际部署是可以填写为user-service服务对外提供访问的地址。或者在本地配置host解析，解析user-serive到user-service服务
      filter-order: 3
        ignored: /catalog/**
eureka:
  instance:
    prefer-ip-address: true
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://discovery-service:8761/eureka/

server.session.cookie.name: edge-service

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 60000
ribbon:
  ConnectTimeout: 3000
  ReadTimeout: 60000
```



指定使用profile=docker

```yaml
SPRING_PROFILES_ACTIVE=docker
```

### User Service(用户模块)

bootstrap.yml

```yaml
spring:
  application:
    name: user-service
---
spring:
  profiles: docker
  cloud:
    config:
      uri: http://config-service:8888
---
spring:
  profiles: default
  cloud:
    config:
      uri: http://localhost:8888
spring.profiles.active: development
---
spring:
  profiles: cloud
  cloud:
    config:
      uri: ${vcap.services.config-service.credentials.uri:http://localhost:8888}
```

application.yml

```yaml
server:
  port: 8181
  contextPath: /uaa
security:
  user:
    password: password
  enable-csrf: false
logging.level.org.springframework.security: DEBUG
spring:
  profiles: development
  datasource:
    url: jdbc:mysql://192.168.99.100:3306/dev
    username: root
    password: dbpass
    initialize: true
security.ignored: /resources/**
---

server:
  port: 8181
  contextPath: /uaa
security:
  user:
    password: password
  enable-csrf: false
  # https://github.com/spring-projects/spring-security-oauth/issues/993
  oauth2:
    resource:
      filter-order: 3
spring:
  profiles: docker
  cloud:
    bus:
      trace:
        enabled: true     # 开启cloud bus的跟踪
  sleuth:
    sampler:
      percentage: 1.0
    traceId128: true
  zipkin:
    base-url: http://zipkin:9411/
security.ignored: /resources/**
logging:
  level:
    org:
      springframework:
        bean: DEBUG
        context: DEBUG
        security: DEBUG
        web: DEBUG
        data: DEBUG
        http: DEBUG

server.session.cookie.name: user-service
```

指定profile

```
SPRING_PROFILES_ACTIVE=docker
```

### Online Store Web(前端门户)

application.yml

```yaml
spring:
  profiles:
    active: development
---
spring:
  profiles: development
  application:
    name: online-store-web
zuul:
  ignored-services: '*'
  routes:
    edge-service:
      path: /api/**
      url: http://edge-service:9999
    auth-service:
      path: /user/**
      url: http://user-service:8181/uaa/user
security:
  enable-csrf: false
  oauth2:
    resource:
      userInfoUri: http://user-service:8181/uaa/user
    client:
      accessTokenUri: http://user-service:8181/uaa/oauth/token
      userAuthorizationUri: http://user-service:8181/uaa/oauth/authorize
      clientId: acme
      clientSecret: acmesecret
  ignored: /assets/**
  eureka:
  instance:
    non-secure-port: 8787
    prefer-ip-address: true
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://discovery-service:8761/eureka/
---
spring:
  profiles: docker
  application:
    name: online-store-web
  sleuth:
    sampler:
      percentage: 1.0
    traceId128: true
  zipkin:
    base-url: http://zipkin:9411/
zuul:
  ignored-services: '*'
  routes:
    edge-service:
      path: /api/**
      url: http://edge-service:9999
    auth-service:
      path: /user/**
      url: http://user-service:8181/uaa/user
security:
  enable-csrf: false
   oauth2:
    resource:
      userInfoUri: http://user-service:8181/uaa/user
      filter-order: 3
    client:
      accessTokenUri: http://user-service:8181/uaa/oauth/token
      userAuthorizationUri: http://user-service:8181/uaa/oauth/authorize
      ##这里跳转使用的是servicename，前端跳转是无法解析的，在实际部署是可以填写为user-service服务对外提供访问的地址。或者在本地配置host解析，解析user-serive到user-service服务
      clientId: acme
      clientSecret: acmesecret
eureka:
  instance:
    non-secure-port: 8787
    prefer-ip-address: true
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://discovery-service:8761/eureka/

logging:
  level:
    org:
      springframework:
        bean: DEBUG
        context: DEBUG
        security: DEBUG
        web: DEBUG
        data: DEBUG
        http: DEBUG

server.session.cookie.name: online-store-web
```

![Online Store Architecture Diagram](https://github.com/cwdgit/springcloud-demo/blob/master/img/user登录前.png)
![Online Store Architecture Diagram](https://github.com/cwdgit/springcloud-demo/blob/master/img/登录中.png)
![Online Store Architecture Diagram](https://github.com/cwdgit/springcloud-demo/blob/master/img/登录成功.png)
