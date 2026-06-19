# 配置加载优先级、Profile

## 面试定位

配置加载优先级和 Profile 是 Spring Boot 工程面试常见题。面试官通常会问：

- `application.yml`、`application-dev.yml`、环境变量、命令行参数谁优先。
- Profile 如何激活。
- 多个 Profile 同时激活时如何合并。
- `bootstrap.yml` 和 `application.yml` 的区别。
- `@Value` 和 `@ConfigurationProperties` 的区别。
- 为什么线上配置没有按预期生效。

回答时要把重点放在 Spring Environment 和 PropertySource 顺序上：配置最终都会进入 Environment，属性解析时按 PropertySource 优先级查找。

## 核心概念

### Environment

`Environment` 是 Spring 对运行环境的抽象。

它包含：

- Profiles。
- PropertySources。
- 属性解析能力。

常见使用：

```java
String port = environment.getProperty("server.port");
```

### PropertySource

`PropertySource` 表示一个配置来源。

常见来源：

- 命令行参数。
- JVM 系统属性。
- 操作系统环境变量。
- 配置文件。
- 外部配置中心。
- 默认属性。

属性解析时，如果多个来源有同名 key，优先级高的覆盖优先级低的。

### Profile

Profile 用于按环境或场景激活不同配置和 Bean。

典型环境：

- `dev`。
- `test`。
- `staging`。
- `prod`。

配置文件：

```text
application.yml
application-dev.yml
application-prod.yml
```

Bean 条件：

```java
@Profile("dev")
@Bean
MockSmsClient mockSmsClient() {
    return new MockSmsClient();
}
```

## Spring Boot 配置优先级概览

常见优先级从高到低可以概括为：

1. 命令行参数。
2. Java 系统属性。
3. 操作系统环境变量。
4. 外部配置文件。
5. classpath 内部配置文件。
6. 默认配置。

更细一点，常见配置文件位置优先级一般是：

```text
file:./config/
file:./
classpath:/config/
classpath:/
```

同名属性通常高优先级覆盖低优先级。

注意：Spring Boot 2.4 之后引入 Config Data 机制，配置加载细节更系统；面试中可以回答主规则，并说明具体顺序可通过 Environment 中 PropertySource 顺序确认。

## 常见配置来源

### 命令行参数

```bash
java -jar app.jar --server.port=9090
```

通常优先级很高。

### JVM 系统属性

```bash
java -Dserver.port=9090 -jar app.jar
```

通过：

```java
System.getProperty("server.port")
```

也可读取。

### 操作系统环境变量

```bash
export SERVER_PORT=9090
```

Spring Boot 支持宽松绑定，`SERVER_PORT` 可绑定到 `server.port`。

常见转换：

```text
SERVER_PORT -> server.port
SPRING_DATASOURCE_URL -> spring.datasource.url
MY_APP_USER_NAME -> my.app.user-name
```

### 配置文件

常见：

```text
application.properties
application.yml
application.yaml
```

Profile 配置：

```text
application-dev.yml
application-prod.yml
```

### 默认属性

可以通过代码设置：

```java
SpringApplication app = new SpringApplication(DemoApplication.class);
app.setDefaultProperties(Map.of("server.port", "8080"));
app.run(args);
```

默认属性优先级较低，适合兜底。

## 配置文件位置优先级

假设同时存在：

```text
./config/application.yml
./application.yml
classpath:/config/application.yml
classpath:/application.yml
```

通常外部配置优先于 jar 内配置，`config` 目录优先于根目录。

这意味着部署时可以不改 jar，通过外部配置覆盖内部默认值。

示例：

```text
classpath:/application.yml
  server.port=8080

file:./config/application.yml
  server.port=9090
```

最终 `server.port` 为 `9090`。

## Profile 激活方式

### 配置文件激活

```yaml
spring:
  profiles:
    active: dev
```

### 命令行激活

```bash
java -jar app.jar --spring.profiles.active=prod
```

### JVM 参数激活

```bash
java -Dspring.profiles.active=prod -jar app.jar
```

### 环境变量激活

```bash
export SPRING_PROFILES_ACTIVE=prod
```

### 测试中激活

```java
@SpringBootTest
@ActiveProfiles("test")
class OrderServiceTest {
}
```

## 多 Profile

可以同时激活多个 Profile：

```properties
spring.profiles.active=dev,mysql
```

或：

```bash
java -jar app.jar --spring.profiles.active=prod,gray
```

配置会按激活顺序加载和合并。相同 key 后加载的 Profile 配置通常覆盖先加载的 Profile 配置。

示例：

```yaml
# application-prod.yml
feature:
  enabled: false
```

```yaml
# application-gray.yml
feature:
  enabled: true
```

激活：

```properties
spring.profiles.active=prod,gray
```

最终：

```text
feature.enabled=true
```

实际排查时不要只靠记忆，最好查看 Environment 中 PropertySource 顺序或启动日志。

## `spring.config.activate.on-profile`

Spring Boot 2.4 后推荐在多文档 YAML 中使用：

```yaml
server:
  port: 8080

---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081

---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 8082
```

激活 `dev` 时，`server.port=8081`。

旧写法：

```yaml
spring:
  profiles: dev
```

在新版本中不推荐继续使用。

## `spring.profiles.include` 与 Profile Group

### include

```yaml
spring:
  profiles:
    include: mysql,redis
```

激活当前配置时额外包含其他 Profile。

### group

Profile Group 用于把多个 Profile 组合成一个逻辑环境：

```yaml
spring:
  profiles:
    group:
      prod: mysql,redis,monitor
```

激活：

```bash
--spring.profiles.active=prod
```

会同时激活：

```text
prod,mysql,redis,monitor
```

适合复杂环境组合。

## `@Profile`

`@Profile` 可控制 Bean 是否注册。

```java
@Configuration
class SmsConfig {
    @Bean
    @Profile("dev")
    SmsClient mockSmsClient() {
        return new MockSmsClient();
    }

    @Bean
    @Profile("prod")
    SmsClient realSmsClient() {
        return new RealSmsClient();
    }
}
```

也支持表达式：

```java
@Profile("prod & !gray")
```

注意：

- `@Profile` 影响 Bean 是否注册。
- 配置文件 Profile 影响属性是否加载。
- 二者都由 Environment 中的 active profiles 驱动。

## `@Value` 与 `@ConfigurationProperties`

### `@Value`

适合少量简单配置：

```java
@Value("${order.timeout:3000}")
private int timeout;
```

优点：

- 使用简单。
- 支持默认值。
- 支持 SpEL。

缺点：

- 分散。
- 不利于批量校验。
- 不利于 IDE metadata。
- 类型复杂时可维护性差。

### `@ConfigurationProperties`

适合结构化配置：

```java
@ConfigurationProperties(prefix = "order")
public class OrderProperties {
    private int timeout = 3000;
    private Retry retry = new Retry();

    public static class Retry {
        private int maxAttempts = 3;
    }
}
```

启用方式：

```java
@EnableConfigurationProperties(OrderProperties.class)
```

或作为 Bean 注册。

优点：

- 结构化。
- 类型安全。
- 支持校验。
- 支持配置元数据。
- 适合 starter 和复杂配置。

## 宽松绑定

Spring Boot 支持 relaxed binding。

这些写法可绑定到同一属性：

```text
my.service.timeout
my.service-timeout
my.serviceTimeout
MY_SERVICE_TIMEOUT
```

环境变量一般不能使用点号，所以常写成大写下划线：

```text
MY_SERVICE_TIMEOUT=3000
```

会绑定到：

```text
my.service.timeout
```

## 配置刷新与动态配置

Spring Boot 本地配置默认在启动时加载。修改 `application.yml` 不会自动刷新已经运行的应用。

动态配置通常依赖：

- Spring Cloud Config。
- Nacos。
- Apollo。
- Consul。
- 组织内部配置中心。

使用动态配置时要关注：

- 是否支持运行期刷新。
- Bean 是否能感知新值。
- `@Value` 是否刷新。
- `@ConfigurationProperties` 是否刷新。
- 是否需要 `@RefreshScope`。
- 配置变更是否线程安全。

面试中可说明：配置优先级是 Spring Boot 本地配置机制，动态配置中心还会插入额外 PropertySource，具体优先级取决于集成实现。

## `bootstrap.yml` 与 `application.yml`

在 Spring Cloud 体系中，`bootstrap.yml` 曾常用于应用上下文启动前的引导配置，例如配置中心地址。

典型用途：

- 应用名。
- 配置中心地址。
- 加密解密配置。
- 远程配置拉取前置参数。

`application.yml` 则是普通应用配置。

注意：

- Spring Boot 本身主要使用 `application.yml`。
- Spring Cloud 新版本中 bootstrap 机制需要按依赖和配置启用。
- 是否存在 `bootstrap.yml` 以及优先级，取决于 Spring Cloud 版本和项目配置。

面试回答不要绝对化说所有 Boot 项目都有 bootstrap。

## 常见排查方法

### 1. 打印 Environment

```java
@Component
class EnvPrinter implements ApplicationRunner {
    private final Environment environment;

    EnvPrinter(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void run(ApplicationArguments args) {
        System.out.println(environment.getProperty("server.port"));
    }
}
```

### 2. 查看 PropertySource 顺序

```java
ConfigurableEnvironment env = (ConfigurableEnvironment) environment;
for (PropertySource<?> propertySource : env.getPropertySources()) {
    System.out.println(propertySource.getName());
}
```

### 3. 使用 Actuator

常见端点：

```text
/actuator/env
/actuator/configprops
/actuator/conditions
```

注意线上开启这些端点要做权限控制，避免泄露敏感配置。

### 4. 检查启动参数

很多配置被命令行或环境变量覆盖，排查时要看：

- 启动脚本。
- 容器环境变量。
- Kubernetes ConfigMap / Secret。
- JVM 参数。
- 配置中心。
- CI/CD 注入变量。

## 工程实践

- 默认配置放 `application.yml`，环境差异放 Profile 文件或配置中心。
- 生产环境优先使用外部配置或配置中心，不改 jar 内配置。
- 复杂配置用 `@ConfigurationProperties`，少量简单值才用 `@Value`。
- 敏感信息不要提交到仓库，使用 Secret、环境变量或配置中心加密能力。
- 不要在代码里硬编码环境判断，优先用 Profile 和条件配置。
- 多 Profile 组合要控制复杂度，避免覆盖关系不可读。
- 配置排查时直接看 Environment 和 Actuator，不靠猜。

## 常见追问

### 1. 配置加载优先级怎么理解

最终所有配置都会进入 Environment 的 PropertySource 列表。解析属性时按优先级查找，高优先级来源覆盖低优先级来源。常见高优先级包括命令行参数、JVM 系统属性、环境变量和外部配置文件。

### 2. 外部配置和 jar 内配置谁优先

通常外部配置优先于 classpath 内部配置。因此部署时可以通过 `./config/application.yml` 或命令行参数覆盖 jar 内默认配置。

### 3. 如何激活 Profile

可以通过 `spring.profiles.active`，来源可以是配置文件、命令行参数、JVM 参数、环境变量，也可以在测试中用 `@ActiveProfiles`。

### 4. 多个 Profile 同时激活时谁覆盖谁

相同 key 通常后加载的 Profile 覆盖先加载的 Profile。实际项目中应通过 Environment PropertySource 顺序或启动日志确认。

### 5. `@Profile` 和 `application-dev.yml` 是一回事吗

不是。`@Profile` 控制 Bean 是否注册；`application-dev.yml` 控制某组配置属性是否加载。它们都由 active profiles 驱动。

### 6. `@Value` 和 `@ConfigurationProperties` 怎么选

少量简单配置可用 `@Value`；结构化、多字段、需要校验和 metadata 的配置应使用 `@ConfigurationProperties`。

### 7. 为什么环境变量 `SERVER_PORT` 能覆盖 `server.port`

Spring Boot 支持宽松绑定，会把大写下划线形式映射到点分隔属性名。

### 8. 修改配置文件后应用会自动生效吗

本地 `application.yml` 默认不会自动刷新运行中应用。动态刷新需要配置中心或额外刷新机制支持。

## 易错点

- 只背固定优先级，不知道本质是 PropertySource 顺序。
- 以为 jar 内 `application.yml` 一定优先于外部配置。
- 以为 `@Profile` 控制配置文件加载，实际它控制 Bean 注册。
- 以为 `@Value` 适合所有配置。
- 忽略环境变量的宽松绑定和覆盖。
- 多 Profile 叠加后不知道哪个值最终生效。
- 把 Spring Cloud 的 `bootstrap.yml` 当成所有 Spring Boot 项目的必备文件。
- 在线上暴露 `/actuator/env` 导致敏感配置泄露。

## 自检清单

- 能否解释 Environment 和 PropertySource。
- 能否说出常见配置来源和大致优先级。
- 能否说明外部配置为什么能覆盖 jar 内配置。
- 能否列出 Profile 激活方式。
- 能否解释多 Profile 合并和覆盖。
- 能否区分 `@Profile` 与 Profile 配置文件。
- 能否比较 `@Value` 和 `@ConfigurationProperties`。
- 能否说明动态配置和本地配置的区别。
- 能否给出配置不生效的排查步骤。
