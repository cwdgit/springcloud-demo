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
