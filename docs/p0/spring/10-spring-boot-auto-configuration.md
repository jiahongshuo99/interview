# Spring Boot 自动配置原理

## 面试定位

Spring Boot 自动配置是 Spring Boot 面试的核心。面试官通常不满足于“约定大于配置”，而会追问：

- `@SpringBootApplication` 做了什么。
- 自动配置类如何被加载。
- `spring.factories` 和 `AutoConfiguration.imports` 的关系。
- 条件注解如何决定配置是否生效。
- starter 的作用是什么。
- 如何自定义一个 starter。
- 自动配置和用户自定义 Bean 的优先级关系。

回答时要抓住主线：Spring Boot 通过启动注解开启自动配置，批量导入自动配置类，再利用条件注解根据 classpath、配置项、已有 Bean 等条件决定是否注册 Bean。

## `@SpringBootApplication`

常见启动类：

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`@SpringBootApplication` 是组合注解，核心包含：

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

### `@SpringBootConfiguration`

本质上是特殊的 `@Configuration`，表示这是 Spring Boot 应用配置类。

### `@ComponentScan`

从启动类所在包开始扫描组件。

因此启动类通常放在项目根包下，否则可能扫描不到其他包中的组件。

### `@EnableAutoConfiguration`

开启自动配置，是自动配置机制入口。

## 自动配置加载主线

简化链路：

```text
@SpringBootApplication
  -> @EnableAutoConfiguration
     -> @Import(AutoConfigurationImportSelector.class)
        -> AutoConfigurationImportSelector.selectImports()
           -> 读取自动配置候选类
           -> 去重、排除、过滤
           -> 返回需要导入的自动配置类
  -> Spring 容器处理这些 @Configuration 类
  -> 条件注解决定是否注册 Bean
```

关键类：

```text
AutoConfigurationImportSelector
```

它负责找出所有候选自动配置类。

## 自动配置候选类来自哪里

### Spring Boot 2 时代

自动配置类通常配置在：

```text
META-INF/spring.factories
```

键为：

```text
org.springframework.boot.autoconfigure.EnableAutoConfiguration
```

示例：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.FooAutoConfiguration,\
com.example.BarAutoConfiguration
```

### Spring Boot 3 时代

自动配置类主要配置在：

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

示例：

```text
com.example.FooAutoConfiguration
com.example.BarAutoConfiguration
```

面试回答可以说明：早期通过 `spring.factories` 发现自动配置类，Spring Boot 2.7 开始引入新的 imports 文件，Spring Boot 3 主要使用 `AutoConfiguration.imports`。

## 条件注解

自动配置不是无条件生效。它依赖大量 `@Conditional` 派生注解。

常见条件注解：

- `@ConditionalOnClass`：classpath 中存在指定类。
- `@ConditionalOnMissingClass`：classpath 中不存在指定类。
- `@ConditionalOnBean`：容器中存在指定 Bean。
- `@ConditionalOnMissingBean`：容器中不存在指定 Bean。
- `@ConditionalOnProperty`：配置项满足条件。
- `@ConditionalOnResource`：资源存在。
- `@ConditionalOnWebApplication`：Web 应用环境。
- `@ConditionalOnNotWebApplication`：非 Web 应用环境。
- `@ConditionalOnExpression`：表达式满足。

示例：

```java
@AutoConfiguration
@ConditionalOnClass(RedisClient.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public RedisClient redisClient(RedisProperties properties) {
        return new RedisClient(properties.getHost(), properties.getPort());
    }
}
```

含义：

- classpath 中有 `RedisClient` 才启用。
- 绑定配置属性。
- 用户没有自定义 `RedisClient` 时才创建默认 Bean。

## `@ConditionalOnMissingBean`

这是自动配置“让位于用户配置”的关键。

如果用户自己定义：

```java
@Bean
RedisClient redisClient() {
    return new RedisClient("custom-host", 6379);
}
```

自动配置中的：

```java
@Bean
@ConditionalOnMissingBean
RedisClient redisClient(...) {
}
```

就不会再创建默认 Bean。

这体现了 Spring Boot 的原则：

- 自动配置提供默认值。
- 用户显式配置优先。

## starter 的作用

starter 本质上是依赖聚合和自动配置入口。

一个 starter 通常包含：

- 业务或客户端核心依赖。
- 自动配置模块。
- 配置属性类。
- 默认 Bean 声明。
- 条件注解。
- metadata。

使用方只需要引入依赖：

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>example-spring-boot-starter</artifactId>
</dependency>
```

然后写配置：

```yaml
example:
  enabled: true
  endpoint: http://example.com
```

Spring Boot 会根据 classpath 和配置自动装配。

## 自动配置类结构

典型自动配置：

```java
@AutoConfiguration
@ConditionalOnClass(ExampleClient.class)
@EnableConfigurationProperties(ExampleProperties.class)
public class ExampleAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public ExampleClient exampleClient(ExampleProperties properties) {
        return new ExampleClient(properties.getEndpoint());
    }
}
```

配置属性：

```java
@ConfigurationProperties(prefix = "example")
public class ExampleProperties {
    private boolean enabled = true;
    private String endpoint;
}
```

Spring Boot 2 中也常见：

```java
@Configuration
```

Spring Boot 3 推荐：

```java
@AutoConfiguration
```

## 自动配置顺序

自动配置之间可能有顺序要求。

常见注解：

- `@AutoConfigureBefore`。
- `@AutoConfigureAfter`。
- `@AutoConfigureOrder`。

示例：

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
public class MyBatisAutoConfiguration {
}
```

顺序只影响自动配置类处理顺序，不应该作为复杂业务依赖控制手段。

## 排除自动配置

### 注解排除

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class DemoApplication {
}
```

### 配置排除

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

适合：

- classpath 中有相关依赖，但当前应用不需要该自动配置。
- 避免错误创建 Bean。
- 手动接管配置。

## 查看自动配置报告

开启 debug：

```properties
debug=true
```

启动日志会输出条件评估报告，帮助判断自动配置为什么生效或不生效。

Actuator 也可查看 conditions 端点：

```text
/actuator/conditions
```

常用于排查：

- 某个 Bean 为什么没创建。
- 哪个条件不满足。
- 用户自定义 Bean 是否覆盖了自动配置。

## SpringApplication 启动流程中的位置

简化启动流程：

```text
SpringApplication.run()
  -> 创建 ApplicationContext
  -> 准备 Environment
  -> 加载 BeanDefinition
     -> 处理 @SpringBootApplication
     -> 处理 @EnableAutoConfiguration
     -> 导入自动配置类
  -> refresh()
     -> BeanFactoryPostProcessor
     -> BeanPostProcessor
     -> 实例化单例 Bean
  -> runners
  -> ready event
```

自动配置类本质上是在 BeanDefinition 加载阶段被导入，再在容器刷新过程中按普通配置类处理。

## Web 自动配置例子

引入：

```xml
spring-boot-starter-web
```

会带来：

- Spring MVC。
- Jackson。
- Tomcat。
- Validation 等常用依赖。

自动配置可能包括：

- `ServletWebServerFactoryAutoConfiguration`：内嵌 Web 容器。
- `DispatcherServletAutoConfiguration`：`DispatcherServlet`。
- `WebMvcAutoConfiguration`：MVC 基础组件。
- `HttpMessageConvertersAutoConfiguration`：消息转换器。
- `JacksonAutoConfiguration`：ObjectMapper。

条件决定是否生效：

- 是否 Web 应用。
- classpath 是否有 Servlet。
- 是否已经有用户自定义 Bean。

## 数据源自动配置例子

引入 JDBC 相关依赖并配置：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
    username: root
    password: root
```

可能触发：

- `DataSourceAutoConfiguration`。
- `DataSourceTransactionManagerAutoConfiguration`。
- `JdbcTemplateAutoConfiguration`。

如果没有配置 URL，也没有嵌入式数据库，可能启动失败。

排除：

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

## 自定义 starter 思路

常见模块拆分：

```text
example-client
example-spring-boot-autoconfigure
example-spring-boot-starter
```

### 1. 定义属性类

```java
@ConfigurationProperties(prefix = "example")
public class ExampleProperties {
    private String endpoint;
    private int timeout = 3000;
}
```

### 2. 定义自动配置

```java
@AutoConfiguration
@ConditionalOnClass(ExampleClient.class)
@EnableConfigurationProperties(ExampleProperties.class)
public class ExampleAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public ExampleClient exampleClient(ExampleProperties properties) {
        return new ExampleClient(properties.getEndpoint(), properties.getTimeout());
    }
}
```

### 3. 声明自动配置导入文件

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

内容：

```text
com.example.ExampleAutoConfiguration
```

### 4. starter 聚合依赖

starter 本身通常不写太多代码，只聚合：

- autoconfigure 模块。
- client 模块。
- 必要第三方依赖。

## 工程实践

- 自动配置类尽量使用 `@ConditionalOnMissingBean`，给用户覆盖空间。
- 条件注解要精确，避免引入依赖后误装配。
- 配置属性要有合理默认值，并提供 metadata。
- starter 不要强行扫描业务包。
- 自动配置不要做重业务逻辑，主要负责装配基础设施 Bean。
- 排查自动配置问题优先看 conditions report。
- 用户自定义 Bean 应明确命名、类型和优先级，避免和自动配置冲突。

## 常见追问

### 1. `@SpringBootApplication` 包含哪些核心注解

包含 `@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan`。

### 2. 自动配置类是如何被加载的

`@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入候选自动配置类。候选类来自 `spring.factories` 或 `AutoConfiguration.imports`，再经过去重、排除和条件过滤。

### 3. 自动配置为什么不会覆盖用户自己的 Bean

自动配置通常使用 `@ConditionalOnMissingBean`。当用户已经定义同类型 Bean 时，默认 Bean 不再创建。

### 4. starter 和 auto-configuration 区别是什么

auto-configuration 是自动配置代码；starter 是依赖聚合入口，通常依赖 autoconfigure 模块和相关第三方库，方便使用方一键引入。

### 5. 如何排查某个自动配置为什么没生效

开启 `debug=true` 查看 conditions report，或使用 Actuator `/actuator/conditions`，检查 classpath、配置项、已有 Bean、Web 环境等条件是否满足。

### 6. 如何禁用某个自动配置

可以在 `@SpringBootApplication(exclude = XxxAutoConfiguration.class)` 中排除，也可以配置 `spring.autoconfigure.exclude`。

### 7. `@ConditionalOnClass` 有什么作用

根据 classpath 中是否存在指定类决定配置是否生效。starter 常用它判断某个第三方库是否被引入。

## 易错点

- 以为 starter 本身就是自动配置，实际 starter 更多是依赖聚合。
- 以为自动配置无条件生效，忽略条件注解。
- 以为用户 Bean 一定被自动配置覆盖，实际通常用户 Bean 优先。
- 启动类放错包，导致组件扫描范围不完整。
- 随便使用 `@EnableWebMvc` 导致 Boot MVC 自动配置退让。
- 自定义 starter 中使用 `@ComponentScan` 扫用户包，造成不可控装配。
- 不看 conditions report，靠猜排查自动配置问题。

## 自检清单

- 能否说明 `@SpringBootApplication` 三个核心组成。
- 能否说清 `@EnableAutoConfiguration` 和 `AutoConfigurationImportSelector` 的关系。
- 能否说明自动配置候选类文件位置。
- 能否解释常见条件注解。
- 能否说明 `@ConditionalOnMissingBean` 的意义。
- 能否讲清 starter、autoconfigure、client 的模块关系。
- 能否给出自定义 starter 的基本步骤。
- 能否说明如何排除和排查自动配置。
