---
short_title: Core
---

# Spring Core

Spring 是一个轻量级的 Java 企业级开发框架，它的核心是 IoC（控制反转）和 AOP（面向切面编程）。通过 IoC 容器统一管理对象的创建和依赖关系，通过 AOP 将事务、日志等横切逻辑从业务代码中解耦。

## IoC & DI

### 控制反转

什么是控制反转（IoC）？
: 控制反转（Inversion of Control）是一种设计思想，它将对象的创建和依赖管理的控制权从程序本身转移给外部容器。传统代码中对象自己通过 new 创建依赖，而在 IoC 中，对象的创建、组装和生命周期都由 Spring 容器负责，从而降低组件之间的耦合度。

IoC 容器的作用？
: IoC 容器的核心作用是管理对象的创建、依赖关系和生命周期。它负责实例化 Bean、解析依赖、注入属性、调用初始化方法，并在容器关闭时销毁对象。同时它还支持作用域管理、AOP 代理生成以及条件装配等高级特性。

为什么需要 IoC？
: 使用 IoC 可以降低系统的耦合度，提高代码的可测试性和可维护性。传统模式下类与类之间强依赖，难以替换或测试；而通过 IoC，组件只依赖接口，具体实现由容器提供，从而实现解耦和更灵活的扩展能力。

BeanFactory 和 ApplicationContext 区别？
: BeanFactory 是 Spring 最基础的 IoC 容器，提供最核心的 Bean 创建和依赖注入能力，默认是懒加载；而 ApplicationContext 是 BeanFactory 的高级子接口，提供更完整的企业级功能，比如国际化、事件机制、AOP 支持、自动注册 BeanPostProcessor，并且默认在容器启动时，通过在 refresh() 方法中调用 finishBeanFactoryInitialization() 进行完成单例 Bean 的初始化。

| 对比点        | BeanFactory              | ApplicationContext     |
| ------------- | ------------------------ | ---------------------- |
| 层级          | 最顶层接口               | BeanFactory 子接口     |
| Bean 创建时机 | 懒加载（getBean 时创建） | 默认启动时预实例化单例 |
| 国际化支持    | ❌ 不支持                | ✅ 支持                |
| 事件机制      | ❌ 不支持                | ✅ 支持                |
| AOP 自动支持  | ❌ 需要手动注册          | ✅ 自动注册            |
| 使用场景      | 轻量级容器               | 企业级开发（常用）     |

### Bean的注册

1. 显式注册 Bean

显式注册 Bean 是通过 \@Configuration + \@Bean 的方式，将方法返回的对象注册到 Spring 容器中。当容器启动并执行 refresh() 时，Spring 会通过 ConfigurationClassPostProcessor 解析带有 \@Configuration 的类，找到其中的 \@Bean 方法，并将每个方法转换成对应的 BeanDefinition 注册到 BeanFactory。在 refresh() 的后期阶段（finishBeanFactoryInitialization()），Spring 根据这些 BeanDefinition 实例化单例 Bean，并将其放入单例缓存中，从而完成显式注册的整个流程。

```java
@Configuration
public class AppConfig {

    @Bean
    public UserRepository userRepository() {
        return new UserRepository();
    }

    @Bean
    public UserService userService(UserRepository userRepository) {
        return new UserService(userRepository);
    }
}
```

\@Configuration 为什么要用代理？
: \@Configuration 使用代理（默认通过 CGLIB 生成子类）是为了保证 配置类中多个 \@Bean 方法之间互相调用时，返回的是容器中管理的同一个单例 Bean，而不是每次调用都 new 一个新对象。代理会拦截对 \@Bean 方法的调用，先从 BeanFactory 中获取已存在的 Bean，如果不存在才执行原始方法创建并注册，从而保证单例语义和依赖一致性，这也被称为 inter-bean method call 语义。

2. 组件扫描注册

Spring 通过 \@ComponentScan 指定要扫描的包路径，在容器启动的 refresh() 过程中由 ConfigurationClassPostProcessor 触发扫描逻辑，使用类路径扫描器（ClassPathBeanDefinitionScanner）查找带有 \@Component 及其派生注解（如 \@Controller、\@Service、\@Repository）的类，然后为这些类生成 BeanDefinition 并注册到 BeanFactory 中，后续再统一完成实例化和依赖注入。

下面的例子中，Spring 扫描 com.example.app，发现 \@Service 和 \@Repository，生成对应的 BeanDefinition，在容器初始化阶段创建单例 Bean。

```java
@Configuration
@ComponentScan("com.example.app")
public class AppConfig {
}
```

```java
@Service
public class UserService {
}
```

```java
@Repository
public class UserRepository {
}
```

3. 导入注册

\@Import 用于将指定的类直接注册到 Spring 容器中，它是比组件扫描更底层、更灵活的注册方式。Spring 在解析 \@Configuration 类时，会处理 \@Import 注解，根据导入类型的不同，可能直接注册普通类、通过 ImportSelector 动态选择要导入的类，或者通过 ImportBeanDefinitionRegistrar 手动向容器注册 BeanDefinition。Spring Boot 的自动装配机制本质上就是基于 \@Import + ImportSelector 实现的。

- 导入普通类：即使 UserService 没有 \@Component，也会被注册为 Bean，Bean 名默认是全类名

```java
@Configuration
@Import(UserService.class)
public class AppConfig {
}
```

```java
public class UserService {
}
```

- 使用 ImportSelector（动态选择）：运行时动态决定要注册哪些类，Spring Boot 自动装配就是用这种方式

```java
public class MyImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        return new String[] {
            "com.example.UserService",
            "com.example.UserRepository"
        };
    }
}
```

```java
@Configuration
@Import(MyImportSelector.class)
public class AppConfig {
}
```

- 使用 ImportBeanDefinitionRegistrar（手动注册）：可以完全自定义 BeanDefinition，可以设置作用域、懒加载等属性，是最底层、最灵活的方式

```java
public class MyRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {

        RootBeanDefinition beanDefinition =
                new RootBeanDefinition(UserService.class);

        registry.registerBeanDefinition("userService", beanDefinition);
    }
}
```

```java
@Configuration
@Import(MyRegistrar.class)
public class AppConfig {
}
```

4. 条件注册

条件注册是通过 \@Conditional 机制，在 Bean 注册前进行条件判断，只有条件成立才会将 Bean 注册到容器中。其底层核心是 Condition 接口，所有条件判断最终都会实现该接口，并通过 matches(ConditionContext context, AnnotatedTypeMetadata metadata) 方法决定是否注册。Spring 在解析配置类时会读取 \@Conditional，调用 matches()：返回 true 则注册 Bean，返回 false 则不注册。

Spring Boot 在此基础上提供了派生注解，例如 \@ConditionalOnMissingBean 等，本质仍然是对 \@Conditional 的封装。\@ConditionalOnMissingBean 的含义是：当容器中不存在指定类型（或名称）的 Bean 时才会注册当前 Bean。它的作用是避免重复注册，允许用户自定义 Bean 覆盖默认配置，是 Spring Boot 自动装配“可覆盖”机制的关键。

- 自动配置类：如果容器中 没有 User 类型的 Bean，才会注册这个默认的 User。

```java
@Configuration
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public User user() {
        return new User("默认用户");
    }
}
```

- 用户自己定义：容器中已经存在 User，\@ConditionalOnMissingBean 条件不成立，自动配置中的 Bean 不会注册

```java
@Configuration
public class UserConfig {

    @Bean
    public User user() {
        return new User("自定义用户");
    }
}
```

5. FactoryBean注册

FactoryBean 是一种特殊的 Bean 注册方式，它本身是一个工厂类，用来创建另外一个对象交给 Spring 容器管理。当一个类实现 FactoryBean<T> 接口时，Spring 注册到容器中的并不是这个工厂对象本身，而是通过其 getObject() 方法返回的对象。也就是说，默认通过 getBean("xxx") 获取到的是 getObject() 生成的实例，而不是 FactoryBean 本身。这种机制常用于创建复杂对象（如代理对象）。

getBean("&xxx") 和 getBean("xxx") 区别？
: getBean("xxx")获取的是 FactoryBean 通过 getObject() 方法生产的对象。
: getBean("&xxx")获取的是 FactoryBean 本身（加 & 表示获取工厂对象）。

### 作用域 & 懒加载

\@Scope
: \@Scope 用来指定 Bean 在 Spring 容器中的作用范围。默认是 singleton（单例），即整个容器中只有一个实例。除了单例外，还可以定义为 prototype（多例），以及 Web 环境下的 request、session 等。不同作用域决定了 Bean 的创建时机、实例数量以及生命周期范围。

| 作用域    | 含义   | 实例个数           | 创建时机           | 适用场景         |
| --------- | ------ | ------------------ | ------------------ | ---------------- |
| singleton | 单例   | 容器中唯一         | 容器启动时（默认） | 无状态、通用组件 |
| prototype | 多例   | 每次获取一个新对象 | 每次 getBean 时    | 有状态对象       |
| request   | 请求级 | 每个 HTTP 请求一个 | 每次请求创建       | Web 请求数据     |
| session   | 会话级 | 每个 Session 一个  | Session 存在期间   | 用户会话数据     |

\@Lazy
: \@Lazy 用于控制 Bean 是否在容器启动时立即创建。默认情况下，singleton Bean 会在容器启动时实例化；加上 \@Lazy 后，会在第一次被获取或注入时才创建。它只影响创建时机，不改变作用域本身。

单例 Bean 是线程安全的吗？
: Spring 只保证 singleton Bean 在容器中只有一个实例，但并不保证线程安全。如果单例 Bean 是无状态的（不保存可变成员变量），通常是线程安全的；如果包含可变成员变量，在多线程环境下可能存在线程安全问题，需要自行加锁或避免共享状态。

三级缓存如何解决循环依赖？
: 三级缓存是 Spring 在创建单例 Bean 过程中为了解决循环依赖问题而设计的内部机制，属于 Bean 生命周期中“实例化 + 属性填充阶段”的知识点。本质上，它发生在单例 Bean 创建流程里，当 Bean A 依赖 Bean B，Bean B 又依赖 Bean A 时，Spring 需要在 A 尚未完全初始化完成时，提前暴露一个“早期引用”给 B 使用，从而打破循环依赖。
: 它具体存在于 Spring 的单例注册流程中（DefaultSingletonBeanRegistry），通过三级缓存机制分别保存不同阶段的单例对象：一级缓存是单例池，用于存放完全创建好的 Bean；二级缓存用于存放早期暴露的对象，也就是尚未完成初始化的半成品 Bean；三级缓存存放的是对象工厂，用于在需要时生成早期引用，并且支持 AOP 代理的创建。这个机制的核心目的，是在支持循环依赖的同时，还能兼容 AOP 代理对象的创建。

### 依赖注入

\@Autowired
: \@Autowired 是 Spring 提供的自动注入注解，默认 按类型（byType）注入。当容器中存在唯一匹配类型的 Bean 时即可注入成功；如果存在多个同类型 Bean，就会发生冲突。required=true 是默认值，表示必须注入成功，否则启动报错；设置 required=false 时，如果找不到匹配 Bean，则注入 null 不报错。注入顺序上：先按类型匹配 → 如果有多个候选，再按名称匹配 → 仍无法确定则报错。

```java
@Service
public class UserService {
}

@RestController
public class UserController {

    @Autowired
    private UserService userService;  // 按类型注入
}
```

\@Qualifier 和 \@Primary
: 当容器中有 两个相同类型的实现类 时，单纯使用 \@Autowired 会报错，因为按类型无法唯一确定。解决方式有两种：\@Primary：标记某个 Bean 为首选；\@Qualifier：明确指定要注入的 Bean 名称。

```java
public interface PaymentService {
    void pay();
}

@Service
@Primary
public class AlipayService implements PaymentService {
    public void pay() {}
}

@Service
public class WechatPayService implements PaymentService {
    public void pay() {}
}
```

- 默认注入 AlipayService（因为有 \@Primary）

```java
@Autowired
private PaymentService paymentService;
```

- 指定注入 WechatPayService

```java
@Autowired
@Qualifier("wechatPayService")
private PaymentService paymentService;
```

\@Resource
: \@Resource 是 JSR-250 标准注解（不是 Spring 原生），默认 按名称（byName）注入，如果找不到同名 Bean，才会按类型匹配。它没有 required 属性。

```java
@Service
public class UserService {
}

@RestController
public class UserController {

    @Resource
    // @Resource(name = "userService")
    private UserService userService;  // 默认按名称 userService 注入
}
```

字段注入、Setter注入和构造器注入比较
: 见下表

| 对比项           | 构造器注入 ✅（推荐） | Setter 注入       | 字段注入 ❌（不推荐）           |
| ---------------- | --------------------- | ----------------- | ------------------------------- |
| 注入方式         | 通过构造方法传入      | 通过 set 方法注入 | 直接在成员变量上加 `@Autowired` |
| 依赖是否强制     | 强制依赖（必须传入）  | 可选依赖          | 容器控制，不明显                |
| 是否可用 `final` | ✅ 可以               | ❌ 不可以         | ❌ 不可以                       |
| 依赖是否清晰     | ✅ 明确               | 较明确            | ❌ 隐式依赖                     |
| 是否利于单元测试 | ✅ 非常方便           | 一般              | ❌ 不方便（依赖容器）           |
| 是否符合设计原则 | ✅ 符合（依赖不可变） | 一般              | ❌ 破坏封装                     |
| Spring 官方推荐  | ✅ 推荐               | 可用              | ❌ 不推荐                       |

1. 构造器注入：依赖是强制的（必须传入），可以配合 final，更安全、更清晰，方便单元测试，Spring 4.3 以后，只有一个构造器时可以省略 \@Autowired

```java
@Service
public class UserService {

    private final OrderService orderService;

    public UserService(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

2. 通过 set 方法注入：适合可选依赖，依赖不是必须的，不如构造器注入严谨

```java
@Service
public class UserService {

    private OrderService orderService;

    @Autowired
    public void setOrderService(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

3. 字段注入：直接在成员变量上使用 \@Autowired，破坏封装，无法使用 final，不利于单元测试（必须依赖 Spring 容器），依赖关系不明显

```java
@Service
public class UserService {

    @Autowired
    private OrderService orderService;
}
```

属性注入
: 属性注入是指将外部配置（如 properties 文件）中的值注入到 Spring Bean 中，常用 \@Value 实现。\@Value 支持两种写法：${} 占位符用于读取配置文件中的属性值，\#{}（SpEL）用于表达式计算。\@PropertySource 用于指定额外的 properties 文件加载到环境中。Spring 启动时会将配置文件加载到 Environment，再通过属性解析器解析 \${} 占位符，最终完成字段注入。

1. ${} 占位符

```bash
# application.properties
user.name=zhangsan
user.age=18
```

```java
@Component
public class User {

    @Value("${user.name}")
    private String name;

    @Value("${user.age}")
    private int age;
}
```

2. SpEL 表达式

使用 \#{}，支持表达式计算、方法调用等。

```java
@Value("#{10 * 2}")
private int num;

@Value("#{systemProperties['os.name']}")
private String osName;
```

3. \@PropertySource

```java
@Configuration
@PropertySource("classpath:app.properties")
public class AppConfig {
}
```

Aware 接口机制
: xxxAware 是 Spring 提供的一组“感知接口”，用于让 Bean 感知容器内部对象。当一个 Bean 实现了某个 Aware 接口（如 BeanNameAware、ApplicationContextAware），Spring 在创建该 Bean 时，会在初始化阶段自动调用对应的回调方法，把相关对象注入进来。这是一种由容器主动回调、传递内部资源给 Bean 的机制。常见接口包括：BeanNameAware获取当前 Bean 在容器中的名字、ApplicationContextAware获取容器对象。
: 下面的例子中，当 Spring 创建 MyBean 时，会自动调用setBeanName()、setApplicationContext()，把对应对象传进来。

```java
@Component
public class MyBean implements BeanNameAware, ApplicationContextAware {

    private String beanName;
    private ApplicationContext applicationContext;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }
}
```

\@Profile
: \@Profile 用于标识 Bean 在特定环境下才会生效。Spring 启动时会根据当前激活的环境（active profile）决定是否注册对应的 Bean。常见环境有 dev、test、prod。当某个 Bean 标注了 \@Profile("dev")，只有在激活 dev 环境时才会被注册到容器中，从而实现多环境隔离。
: 下面的例子中，启动后只会加载 DevDataSource，不会加载 ProdDataSource。

```bash
spring.profiles.active=dev
```

```java
@Service
@Profile("dev")
public class DevDataSource {
}
```

```java
@Service
@Profile("prod")
public class ProdDataSource {
}
```

### Bean的生命周期

生命周期 8 步
: 见下表

| 步骤 | 阶段名称                 | 说明                         | 常见扩展点 / 对应机制                                  |
| ---- | ------------------------ | ---------------------------- | ------------------------------------------------------ |
| 1️⃣   | 实例化                   | 创建 Bean 对象（new 出对象） | 构造方法                                               |
| 2️⃣   | 属性填充                 | 依赖注入，给字段/属性赋值    | `@Autowired`、`@Value`                                 |
| 3️⃣   | Aware 回调               | 容器将内部对象传给 Bean      | `BeanNameAware`、`ApplicationContextAware`             |
| 4️⃣   | BeanPostProcessor before | 初始化前增强                 | `postProcessBeforeInitialization()`                    |
| 5️⃣   | 初始化方法               | 执行初始化逻辑               | `@PostConstruct`、`afterPropertiesSet()`、`initMethod` |
| 6️⃣   | BeanPostProcessor after  | 初始化后增强                 | `postProcessAfterInitialization()`（AOP 在此生成代理） |
| 7️⃣   | 使用                     | Bean 对外提供服务            | 业务方法调用                                           |
| 8️⃣   | 销毁                     | 容器关闭时回收资源           | `@PreDestroy`、`destroyMethod`                         |

初始化与销毁方法
: Spring 提供三类初始化扩展点：\@PostConstruct、InitializingBean#afterPropertiesSet()、以及 \@Bean(initMethod=...) 指定的方法。它们在初始化阶段的执行顺序是：先执行 \@PostConstruct，再执行 afterPropertiesSet()，最后执行 initMethod 指定的方法。也就是说，从注解方式到接口方式，再到显式指定的方法，依次执行。
: 销毁阶段同样有对应扩展点：\@PreDestroy、DisposableBean#destroy()（如果实现），以及 \@Bean(destroyMethod=...) 指定的方法。常见执行顺序是：先执行 \@PreDestroy，再执行 destroy()，最后执行 destroyMethod 指定的方法。

| 使用场景                         | 初始化方式              | 销毁方式                   | 说明                           |
| -------------------------------- | ----------------------- | -------------------------- | ------------------------------ |
| ✅ 我能修改源码（普通业务 Bean） | `@PostConstruct`        | `@PreDestroy`              | 最推荐方式，标准注解，侵入性小 |
| ✅ 我要和 Spring 生命周期强绑定  | `afterPropertiesSet()`  | `destroy()`                | 实现接口，Spring 侵入性较强    |
| ✅ 第三方类，不能改源码          | `@Bean(initMethod=...)` | `@Bean(destroyMethod=...)` | 在配置类中指定方法名           |

```java
@Configuration
public class AppConfig {
    @Bean(initMethod = "initMethod", destroyMethod = "destroyMethod")
    public LifeCycleBean lifeCycleBean() {
        return new LifeCycleBean();
    }
}
```

```java
public class LifeCycleBean implements InitializingBean {

    @PostConstruct
    public void postConstruct() {
        System.out.println("1 @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("2 afterPropertiesSet");
    }

    public void initMethod() {
        System.out.println("3 @Bean initMethod");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("1 @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("2 DisposableBean destroy（若实现）");
    }

    public void destroyMethod() {
        System.out.println("3 @Bean destroyMethod");
    }
}
```

BeanPostProcessor
: BeanPostProcessor 是 Spring 提供的生命周期扩展机制，它允许我们在 Bean 初始化前后对 Bean 进行统一处理。它的作用可以概括为三个方面：扩展、增强、基础设施。所谓扩展，是指在不修改业务代码的情况下，对容器中的 Bean 进行统一逻辑处理；所谓增强，是指可以修改、包装甚至替换 Bean（例如返回代理对象）；所谓基础设施，是指 Spring 中很多核心功能（如 AOP、依赖注入相关处理器等）都是基于它实现的，因此它是整个扩展体系的核心支撑点。
: AOP 的目标是对方法进行增强，而增强的方式是通过代理对象替代原始 Bean 对外提供服务。为了保证代理对象是基于一个“完整可用的目标对象”生成的，必须等到 Bean 完成依赖注入和初始化逻辑之后再进行包装。如果在 before 阶段创建代理，可能会影响初始化过程或导致代理对象参与初始化，增加复杂性。因此，Spring 通常在 postProcessAfterInitialization 阶段获取已经完成初始化的 Bean，然后将其包装成代理对象并返回，容器最终保存和对外暴露的就是这个代理对象，从而实现 AOP 增强。
: before 更偏向“加工”，after 更偏向“包装”，具体总结如下：

| 阶段                              | 执行时机       | Bean 状态                    | 适合做什么                        |
| --------------------------------- | -------------- | ---------------------------- | --------------------------------- |
| `postProcessBeforeInitialization` | 初始化方法之前 | 已实例化、已完成依赖注入     | 做校验、补充属性、预处理逻辑      |
| `postProcessAfterInitialization`  | 初始化方法之后 | 已完成初始化，是完整可用对象 | 包装 Bean、创建代理、替换原始对象 |

```java
@Component
public class MyBpp implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if ("lifeCycleBean".equals(beanName)) {
            System.out.println("4 BPP before: " + beanName);
        }
        return bean; // 注意：必须返回bean（或替换后的bean）
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if ("lifeCycleBean".equals(beanName)) {
            System.out.println("6 BPP after: " + beanName);
            // AOP 的“生成代理并返回”就发生在这里（本质：return proxy）
        }
        return bean;
    }
}
```

## AOP

动态代理
: 在 Java 中实现动态代理主要有两种方式：JDK 动态代理和 CGLIB 代理。JDK 动态代理要求目标类必须实现接口，它的原理是基于接口在运行时生成一个实现同样接口的代理类，然后在方法调用时通过 InvocationHandler 进行拦截和增强，因此本质是“基于接口生成代理对象”；而 CGLIB 则不要求目标类实现接口，它通过在运行时生成目标类的子类，并重写其中的方法来实现增强，本质是“通过继承目标类生成子类代理”。Spring Boot 默认使用 CGLIB，是为了避免 JDK 动态代理带来的类型转换问题，保证代理对象与目标类类型一致，同时统一代理行为。在现代 JVM 下性能差距很小，因此默认选择兼容性更强的 CGLIB。

名词解释
: 切面通过切入点选择连接点，在织入阶段生成代理对象，实现对目标对象的增强。

| 术语                | 含义                       | 例子（以日志 AOP 为例）                                     |
| ------------------- | -------------------------- | ----------------------------------------------------------- |
| Aspect（切面）      | 封装横切逻辑的类           | `LogAspect` 类，专门写日志增强逻辑                          |
| Advice（通知）      | 具体增强逻辑（方法）       | `@Before` 标注的 `logBefore()` 方法                         |
| JoinPoint（连接点） | 可以被增强的方法           | `UserService.save()` 方法                                   |
| Pointcut（切入点）  | 过滤哪些方法要增强         | `execution(* com.xxx.service.*.*(..))`                      |
| Target（目标对象）  | 原始业务对象               | `UserServiceImpl` 实例                                      |
| Proxy（代理对象）   | 增强后的对象               | 容器中实际获取到的 `UserServiceImpl$$EnhancerBySpringCGLIB` |
| Weaving（织入）     | 把增强应用到目标对象的过程 | Spring 在 Bean 初始化后生成代理对象并应用日志逻辑           |

通知类型
: 常见的通知类型见下表：

| 类型     | 注解             | 执行时机               |
| -------- | ---------------- | ---------------------- |
| 前置通知 | \@Before         | 方法执行前             |
| 后置通知 | \@After          | 方法执行后（无论异常） |
| 返回通知 | \@AfterReturning | 方法正常返回后         |
| 异常通知 | \@AfterThrowing  | 方法抛异常时           |
| 环绕通知 | \@Around         | 自己控制整个执行过程   |

```java
@Service
public class UserService {

    public String save(String name) {
        System.out.println("执行保存用户：" + name);
        return "success";
    }
}
```

```java
@Aspect
@Component
public class LogAspect {

    @Around("execution(* com.xxx.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {

        // 1️⃣ 方法执行前
        System.out.println("方法开始执行：" + pjp.getSignature().getName());

        long start = System.currentTimeMillis();

        // 2️⃣ 执行目标方法
        Object result = pjp.proceed();

        long end = System.currentTimeMillis();

        // 3️⃣ 方法执行后
        System.out.println("方法执行结束");
        System.out.println("耗时：" + (end - start) + "ms");

        return result;  // ⚠️ 一定要返回
    }
}
```

AspectJ 切入点表达式
: Spring AOP 使用的是 AspectJ 切入点表达式语法 来匹配需要增强的方法，其中最常见的写法是 execution(...)。它的基本结构是：execution(访问修饰符 返回值 包名.类名.方法名(参数))，用于精确或模糊匹配某一类方法。例如：execution(\* com.xxx.service.\*.\*(..)) 表示匹配 com.xxx.service 包下所有类的所有方法，其中第一个 \* 表示任意返回值，第一个 \* 表示任意类名，第二个 \* 表示任意方法名，(..) 表示任意参数列表。表达式中常用的通配符包括：\*（匹配任意字符或单个层级）和 ..（匹配任意参数或多层包路径）。通过这种表达式，Spring 可以灵活地筛选出需要进行 AOP 增强的目标方法。

\@Pointcut
: 在 Spring AOP 中，可以使用 \@Pointcut 将切入点表达式单独抽取出来，避免在多个通知中重复书写复杂的 execution(...) 表达式。例如，通过 \@Pointcut("execution(\* com.xxx.service.\*.\*(..))") 定义一个公共的切入点方法，然后在不同的通知中通过 \@Before("servicePointcut()") 等方式直接引用。这样做的好处是切入点表达式可以被多个通知复用，当匹配规则需要修改时只需改一处即可，显著提升了代码的可维护性和可读性。

```java
@Pointcut("execution(* com.xxx.service.*.*(..))")
public void servicePointcut(){}
```

```java
@Before("servicePointcut()")
```

通知执行流程总结
: 正常情况下是 Before → 方法 → AfterReturning → After；异常情况下是 Before → 方法 → AfterThrowing → After；\@After 类似 finally，而 \@AfterReturning 只在方法正常返回时执行。

1. 正常情况

| 执行顺序 | 通知类型          | 说明                         |
| -------- | ----------------- | ---------------------------- |
| 1        | `@Before`         | 方法执行前执行               |
| 2        | 目标方法          | 业务方法执行                 |
| 3        | `@AfterReturning` | 方法正常返回后执行           |
| 4        | `@After`          | 方法最终执行（类似 finally） |

2. 异常情况

| 执行顺序 | 通知类型         | 说明                     |
| -------- | ---------------- | ------------------------ |
| 1        | `@Before`        | 方法执行前执行           |
| 2        | 目标方法         | 抛出异常                 |
| 3        | `@AfterThrowing` | 捕获异常后执行           |
| 4        | `@After`         | 最终执行（类似 finally） |

\@After和\@AfterReturning比较
: 见下表

| 对比点           | `@After`                   | `@AfterReturning`   |
| ---------------- | -------------------------- | ------------------- |
| 执行时机         | 方法结束后（无论是否异常） | 方法正常返回后      |
| 类似语义         | 类似 finally               | 类似 try 中成功分支 |
| 是否能获取返回值 | ❌ 默认不能                | ✅ 可以获取返回值   |

JoinPoint
： 在 Spring AOP 中，JoinPoint 表示当前被增强的方法执行点，通过在通知方法中声明 JoinPoint 参数，可以获取当前方法的关键信息，例如方法名、方法参数、目标对象以及代理对象等。它常用于日志记录、权限校验、参数打印等场景。比如在一个前置通知中，可以通过 jp.getSignature().getName() 获取方法名，通过 jp.getArgs() 获取参数列表，从而在方法执行前打印调用信息。

```java
@Before("execution(* com.xxx.service.*.*(..))")
public void before(JoinPoint jp) {

    // 获取方法名
    String methodName = jp.getSignature().getName();

    // 获取参数
    Object[] args = jp.getArgs();

    System.out.println("调用方法：" + methodName);
    System.out.println("参数列表：" + Arrays.toString(args));
}
```

多切面执行顺序
: 在 Spring AOP 中，如果存在多个切面同时作用于同一个方法，默认情况下它们的执行顺序是不可控的。为了控制多个切面的执行顺序，可以在切面类上使用 \@Order(n) 注解，数字越小优先级越高。对于 \@Around 通知来说，执行顺序呈现出类似“栈结构”的嵌套效果：优先级高的切面会先进入（执行前半部分），最后退出（执行后半部分）。

```java
@Aspect
@Component
@Order(1)
public class LogAspect {

    @Around("execution(* com.xxx.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("日志切面 - 前");
        Object result = pjp.proceed();
        System.out.println("日志切面 - 后");
        return result;
    }
}
```

```java
@Aspect
@Component
@Order(2)
public class TxAspect {

    @Around("execution(* com.xxx.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("事务切面 - 前");
        Object result = pjp.proceed();
        System.out.println("事务切面 - 后");
        return result;
    }
}
```

当调用目标方法时，执行顺序为：

```text
日志切面 - 前
  事务切面 - 前
    目标方法
  事务切面 - 后
日志切面 - 后
```

## 事务

什么是 Spring 事务？
: Spring 事务是基于 AOP 实现的声明式事务管理机制，本质上是在方法执行前通过代理开启事务，在方法正常完成时提交事务，在出现异常时进行回滚，从而保证数据的一致性。Spring 提供两种事务管理方式：编程式事务和声明式事务。编程式事务由开发者手动控制事务的开启、提交与回滚，虽然控制粒度更精细，但代码侵入性强、耦合度高，因此在实际项目中较少使用；声明式事务则通过 \@Transactional 注解定义事务边界，由 Spring 借助 AOP 自动完成事务的创建与管理，代码更加简洁、业务逻辑与事务逻辑解耦，因此在实际开发和面试中通常指的都是声明式事务。

有哪些传播行为？
: 传播行为决定了当前方法调用时，如何处理已有事务。REQUIRED 是共用一个事务，REQUIRES_NEW 是强制独立事务。

|            | REQUIRED     | REQUIRES_NEW           |
| ---------- | ------------ | ---------------------- |
| 外部有事务 | 加入外部事务 | 挂起外部事务，新开事务 |
| 回滚影响   | 一起回滚     | 互不影响               |
| 使用场景   | 普通业务     | 日志、审计、独立操作   |

REQUIRED 调用 REQUIRES_NEW，内部方法异常会不会影响外部事务？
: 当 A 方法使用 REQUIRED、内部调用使用 REQUIRES_NEW 的 B 方法时，B 会开启一个全新的独立事务并挂起 A 的事务；如果 B 抛出异常，它自己的事务一定会回滚，但是否影响 A 取决于异常是否继续向外传播：如果异常被 A 捕获并处理，A 的事务可以正常提交；如果异常继续抛出到 A 的事务边界之外，A 也会因为异常而回滚。因此，REQUIRES_NEW 本身是独立事务，但异常传播仍然可能影响外层事务。

四种隔离级别
: MySQL默认REPEATABLE_READ

| 隔离级别         | 解决问题       | 含义说明                                                               |
| ---------------- | -------------- | ---------------------------------------------------------------------- |
| READ_UNCOMMITTED | 可能脏读       | 允许读取其他事务未提交的数据，事务之间几乎没有隔离，问题最多，性能最高 |
| READ_COMMITTED   | 避免脏读       | 只能读取其他事务已提交的数据，但同一事务中多次读取可能结果不同         |
| REPEATABLE_READ  | 避免不可重复读 | 同一事务中多次读取同一行数据结果一致（即使其他事务已提交修改）         |
| SERIALIZABLE     | 完全串行       | 所有事务按顺序串行执行，强制加锁，避免所有并发问题，但性能最低         |

rollbackFor回滚规则
: 在 Spring 声明式事务中，默认的回滚规则是：只有运行时异常（RuntimeException）和 Error 才会触发事务回滚，而受检异常（Checked Exception）默认不会导致事务回滚。例如，如果方法中抛出的是 IOException 这类受检异常，事务通常仍然会提交而不是回滚。如果希望在出现受检异常时也进行回滚，可以通过在 \@Transactional 注解中显式指定 rollbackFor 属性来定义需要回滚的异常类型，例如指定对 Exception 或某个具体异常类进行回滚控制，从而改变默认行为。

超时回滚
: timeout 属性用于设置事务的最大执行时间，例如设置 \@Transactional(timeout = 5) 表示该事务必须在 5 秒内执行完成，否则 Spring 会回滚事务并抛出超时异常，以防止长时间占用数据库资源。

事务管理器原理
: Spring 事务管理的底层核心接口是 PlatformTransactionManager，不同的数据访问技术对应不同的实现，例如 JDBC 使用 DataSourceTransactionManager，JPA 使用 JpaTransactionManager，Hibernate 使用 HibernateTransactionManager。在执行过程中，Spring 通过 AOP 拦截被 \@Transactional 标注的方法，然后调用事务管理器开启事务，获取数据库连接并将自动提交设置为 false，接着执行目标方法；如果方法正常结束则提交事务（commit），如果出现异常则回滚事务（rollback）。其本质就是利用 AOP 对方法进行包装，从而统一控制数据库连接的提交与回滚。

为什么private方法事务和同类方法调用不生效？
: Spring 声明式事务是基于 AOP 代理机制实现的，而代理对象只能拦截对外暴露的可被代理的方法。在默认情况下，Spring 通过代理对象对 public 方法进行增强，如果方法是 private 的，就无法被代理类重写或拦截，因此事务增强逻辑不会织入进去，最终导致事务不生效。本质原因在于：事务控制发生在代理对象上，而不是目标对象本身。
: 同类方法调用事务不生效的原因也与代理机制有关。当在同一个类内部通过 this.methodB() 调用另一个带有 \@Transactional 注解的方法时，调用是直接发生在目标对象内部，并没有经过 Spring 生成的代理对象，因此事务拦截器不会生效。换句话说，事务是通过代理对象拦截外部调用实现的，而内部调用绕过了代理，所以事务不会被触发。

## Spring MVC

## Spring Boot
