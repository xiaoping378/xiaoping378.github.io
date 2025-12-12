---
tags: ["Dev","Java"]
title: "Goper直接撸java代码，真是操碎了心，tips总结①"
linkTitle: "Java tips的总结"
weight: 32
description: >
 Java Tips总结
---

{{% pageinfo %}}
Java发展的这些年，为了复杂大型项目的工程实践能傻瓜落地，真是封了一层又一层啊，C/C++型选手转过来，会感觉各种看不透，总结了一些tips，尽量追根问底，分享给大家。
{{% /pageinfo %}}

## @SpringBootApplication 注解

java spring项目，第一个看到的基本就是这个注解，只能加到main方法所在的主类上，标识这是一个Spring Boot应用程序的main入口类。

> @(注解)的本质是给代码打标签，告诉需要处理注解的工具或框架: 这里的代码需要特殊处理。

“@SpringBootApplication”注解，是个组合注解，是表示要在程序启动运行阶段被特殊处理的(其实现完全依赖于Java的反射机制)，是spring框架的核心机制，依靠Spring容器的运行时处理能力，体现了「约定优于配置」的设计思想。

> 没点儿Java经验，「约定优于配置」的设计思想，会让学习成本增加，但是在大型项目中，会让开发效率大大提高。
> spring 容器的运行时处理能力，是实现「约定优于配置」的关键。

### Spring 容器

spring 容器是一个IoC容器（简单理解为一个对象工厂），用于管理应用程序中的Bean对象。

> Bean的生命周期包括：实例化、依赖注入、初始化、使用、销毁。
> 依赖注入是指在运行时，通过容器自动将Bean的依赖对象注入到Bean中。
> 初始化是指在Bean实例化后，调用Bean的初始化方法。
> 使用是指在Bean初始化后，通过容器获取Bean实例，调用其方法。
> 销毁是指在应用程序关闭时，调用Bean的销毁方法。

简单来说：
- 你写好代码，贴上标签（注解）
- 容器帮你管理这些代码（Bean）
- 你需要什么，容器就给你什么（依赖注入）

### 为什么要使用 Spring 容器？

> 理由有一堆，还都是高大上的那种....

拿用户注册功能对比，举例说明，对比「不用Spring容器」和「用Spring容器」的代码差异，直观展示Spring容器的价值：

**场景：用户注册功能（需要3层结构）**
1. Controller ：接收注册请求
2. Service ：处理注册业务逻辑（密码加密、保存用户）
3. Repository ：操作数据库（保存用户信息）

**方式1：不用Spring容器 → 传统Java开发**

```java
// 1. Repository层：操作数据库
public class UserRepository {
    public void save(User user) {
        System.out.println("保存用户到数据库：" + user.getName());
    }
}

// 2. Service层：业务逻辑
public class UserService {
    // 必须手动创建依赖对象 → 「自己new」，初始化省略..
    private UserRepository userRepository = new UserRepository();
    
    public void registerUser(User user) {
        // 密码加密（业务逻辑）
        user.setPassword(encryptPassword(user.getPassword()));
        // 调用Repository保存
        userRepository.save(user);
    }
    
    private String encryptPassword(String password) {
        return "加密后的密码：" + password;
    }
}

// 3. Controller层：接收请求
public class UserController {
    // 必须手动创建依赖对象 → 「自己new」
    private UserService userService = new UserService();
    
    // 可以改进为接收参数
    public void handleRegisterRequest(String name, String password) {
        User user = new User();
        user.setName(name);
        user.setPassword(password);
        userService.registerUser(user);
    }
}

// 4. 启动类：手动创建所有对象
public class App {
    public static void main(String[] args) {
        // 必须手动创建所有层的对象 → 「自己管理依赖」
        UserController controller = new UserController();
        // 调用注册接口
        controller.handleRegisterRequest("张三", "123456");
    }
}
```

有点儿偏面向过程的开发思路，需要自己管理各层依赖对象的创建初始化 ：
- 必须手动 new 所有对象，代码冗余
- 依赖关系硬编码（UserService直接依赖UserRepository的具体实现）
- 修改依赖时有可能需要修改所有相关代码（比如把UserRepository换成Redis实现）
- 无法集中管理对象生命周期

**方式2：用Spring容器 → 依赖注入开发**

```java
// 1. Repository层：用@Repository标记，告诉容器「UserRepository类是要管理的Bean对象」
@Repository
public class UserRepository {
    public void save(User user) {
        System.out.println("保存用户到数据库：" + user.getName());
    }
}

// 2. Service层：用@Service标记，@Autowired让容器自动注入依赖
@Service
public class UserService {
    // 容器自动注入UserRepository → 「不用自己new」,配置已初始化
    @Autowired
    private UserRepository userRepository;
    
    public void registerUser(User user) {
        // 专注于业务逻辑（密码加密）
        user.setPassword(encryptPassword(user.getPassword()));
        userRepository.save(user);
    }
    
    private String encryptPassword(String password) {
        return "加密后的密码：" + password;
    }
}

// 3. Controller层：用@RestController标记，@Autowired自动注入Service
@RestController
public class UserController {
    // 容器自动注入UserService → 「自动解决依赖」
    @Autowired
    private UserService userService;
    
    @PostMapping("/register")
    public String registerUser(@RequestBody User user) {
        // 专注于处理请求，不用关心Service怎么来的
        userService.registerUser(user);
        return "注册成功";
    }
}

// 4. 启动类：用@SpringBootApplication标记，容器自动创建所有对象
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        // 一行代码启动 → 容器自动处理所有对象创建和依赖
        SpringApplication.run(App.class, args);
    }
}
```

**具体价值体现 1. 价值1：不用自己new对象**
1. 价值1：不用自己new对象
- 传统方式 ：每个类都要手动 new 依赖对象，并自己管理参数初始化
- Spring方式 ：只需要用 @Service / @Repository 标记，容器自动创建，并配合其他注解可自动配置参数。
- 例子 ： UserService 不用写 new UserRepository() ，容器会自动创建并注入 

2. 价值2：自动解决依赖
- 传统方式 ：依赖关系硬编码，修改时要改所有相关代码
- Spring方式 ： @Autowired 自动找到并注入（嵌入）合适的对象
- 例子 ：如果把 UserRepository 换成Redis实现，只需要给Redis实现类加 @Repository ，不需要修改 UserService 代码

3. 价值3：专注业务逻辑
- 传统方式 ：花大量时间管理对象创建和依赖
- Spring方式 ：只需要关心业务逻辑（如密码加密、用户验证）
- 例子 ： UserService 只需要写 registerUser 的业务代码，不用关心 UserRepository 怎么来的

### spring 启动过程

回到开头提到的spring应用中常遇到的第一个注解，会出现在主类上，“@SpringBootApplication”注解，标识这是一个Spring Boot应用程序的main入口类。

“@SpringBootApplication”注解，包含了以下3个注解：

1. @SpringBootConfiguration: 标识这是应用的主配置类（入口），是@Configuration的派生注解，提供了Spring Boot特定的配置语义，确保配置类能被正确识别和处理，优先级说明：默认配置 → 自动配置 → 外部配置（application.yml） → 手动配置（@Configuration）。

2. @EnableAutoConfiguration: 启用Spring Boot的自动配置机制。
- 自动扫描 classpath 下所有 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（Spring Boot 2.7+）
- 根据pom中的配置的依赖包、条件配置类等信息，自动配置相应的Bean（比如数据源、Web MVC配置等）。
- 另外会看application.yml配置文件中的属性，覆盖默认配置。举例：修改Tomcat默认端口为9090。

```yml
# application.yml
server:
  port: 9090  # 覆盖默认的 8080 端口

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/yudao
    username: root
    password: 123456
```

- Profile多环境支持
    - 可以根据不同的环境（如开发、测试、生产），加载不同的配置文件（如application-dev.yml、application-test.yml、application-prod.yml）。
    - 可以通过spring.profiles.active属性指定当前激活的环境，默认是default环境。

3. @ComponentScan: 开启组件扫描，Spring Boot会自动扫描主类所在的包及其子包，重点找到使用了@Controller、@Service、@Repository等注解的类，并将这些Bean对象注册到Spring容器中。
    - 支持通过basePackages参数设置，指定要扫描的包名，控制扫描范围，灵活组织应用的功能。

经过以上3个隐含注解的协同处理，程序启动完成，会进入运行状态，等待接收外部请求。

示例：
```java
/**
 * Spring Boot应用程序的主启动类
 * 
 * @SpringBootApplication 是一个组合注解，包含以下三个核心功能：
 * 1. @SpringBootConfiguration: 标识这是一个Spring Boot应用的主配置类
 * 2. @EnableAutoConfiguration: 启用Spring Boot的自动配置机制
 * 3. @ComponentScan: 自动扫描当前包及其子包下的组件（如@Controller、@Service、@Repository等）
 * 
 * 这个注解告诉Spring Boot框架：这是一个Spring Boot应用程序的入口点
 */
@SpringBootApplication
public class ServerApplication {
    
    /**
     * 应用程序的主入口方法
     * 
     * 当JVM启动时，会首先执行这个main方法
     * 该方法负责启动整个Spring Boot应用程序
     * 
     * @param args 命令行参数，可以在启动时传入配置参数
     *             例如：--server.port=8081 或 --spring.profiles.active=dev
     */
    public static void main(String[] args) {
        /**
         * SpringApplication.run() 方法执行以下关键操作：
         * 1. 创建Spring应用上下文（ApplicationContext）
         * 2. 启用自动配置（根据classpath中的依赖自动配置Bean）
         * 3. 启动内嵌的Web服务器（如Tomcat、Jetty等）
         * 4. 扫描并注册所有使用Spring注解的组件
         * 5. 启动完成后，应用程序进入运行状态，等待HTTP请求
         * 
         * 参数说明：
         * - ServerApplication.class: 主配置类，Spring Boot会从这个类所在的包开始扫描组件
         * - args: 命令行参数，用于覆盖默认配置
         * 
         * 返回值：ConfigurableApplicationContext对象，代表整个Spring应用上下文
         * 可以用于获取Bean、关闭应用等操作
         */
        SpringApplication.run(ServerApplication.class, args);
    }
}
```

这个过程体现了spring boot生态的核心价值 - 配置自动化，开发人员可以通过pom引入下依赖包（内部包也一样），然后直接改改yml配置，就可以轻松写出生产级的代码了。

> SpringBootApplication注解工作的机制中，还有不少细节，属于用到了可以现查资料的知识，此处先不展开。
> spring框架还有很多AOP、事务管理等其他优势，此处先不展开。

## lombok
解决Java样板代码的利器，通过**@注解**的方式，在编译期自动生成对应的代码，减少了样板代码的编写，提高了开发效率。

解决了哪些样板代码的编写？ 比如：getter/setter、toString、equals、hashCode等。这些方法在大多数Java类中都需要，但是编写逻辑高度重复啰嗦，通过lombok注解，就可以自动生成这些代码，减少了开发人员的机械工作量。举例：

```java
@Data // @Data组合注解，自动为该类生成getter/setter/toString/equals/hashCode等方法，节省了约50行左右的代码量。
// 相当于以下注解组合
// @Getter、@Setter、@ToString、@EqualsAndHashCode等
public class User {
    private Long id;
    private String name;
    private Integer age;
}
```

**为什么每个Java类都需要这些方法 ？**

因为这些方法是Java类的基础（所有类都直接或间接的继承了Object类），没有它们，大多数Java类就不能正常工作。

```java
// 所有类都继承 Object
public class User {  // 隐式继承 Object
    // 自动拥有 Object 的方法：toString(), equals(), hashCode() 等
}
```

比如：
- 没有getter/setter方法，就不能通过对象的属性来访问和修改属性值，Java封装思想的具体实现。
- 没有toString方法，就不能通过System.out.println()方法来打印对象的信息。
- 没有 equals 方法，虽然仍然可以使用equals方法（继承自Object），只是比较的是引用相等性，不能用于比较对象内容是否相等。
- 没有 hashCode 方法，就不能将对象存储在HashSet、HashMap等集合中。

```java
Map<User, String> map = new HashMap<>();
User user = new User();
map.put(user, "value");  // 需要拥有正确的 hashCode() 和 equals()

Set<User> set = new HashSet<>();
set.add(user);  // 同样需要正确的 hashCode() 和 equals()
```

确保了 Java 中所有对象都具有统一的基本行为，也是哈希集合（如 HashMap、HashSet）等功能的基础。
