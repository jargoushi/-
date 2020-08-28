# mybatis

## MyBatis是什么？

mybatis是一个半自动持久化框架， 主要是与数据库进行交互。封装了JDBC，以及对象与数据库数据的映射。开发人员只需要关心sql语句即可。

### 优点

sql语句与java代码分离。 一般出问题都是sql语句的问题，方便查找问题

动态sql语句， 支持逻辑拼接sql语句

java对象与数据库字段的映射

因为手写sql，所以可以对sql优化提升性能

### 缺点

需要开发人员来写sql语句

## mybatis与hibernate的区别

### 共同点

都是持久化层框架，与数据库交互

### 不同点

mybatis是半自动框架，需要开发人员写sql，方便优化sql

hibernate是全自动框架，不需要开发人员写sql，不方便优化sql

## jdbc的不足，以及mybatis如何解决

- jdbc需要频繁创建关闭数据库连接， mybatis整合连接池
- jdbc使用时java代码和sql耦合， mybatis将sql语句写在xml中
- jdbc使用时对象解析复杂， mybatis会自动将查询结果映射为java对象，列名不同时三种解决方案如下
  - 开启自动驼峰映射
  - resultMap
  - sql语句写别名
- jdbc拼接sql语句麻烦， mybatis支持动态sql标签

## mybatis的编码流程

- 创建SqlSessionFactory
- 通过SqlSessionFactory创建SqlSession
- 通过sqlsession执行数据库操作
- 调用session.commit()提交事务
- 调用session.close()关闭会话

## mybatis的工作原理

1. 解析mybatis的全局配置文件， 配置了数据库四大参数，连接池等参数
2. 解析每个mapper文件， 解析其中的select，insert，update，delete标签映射为java中的MappedStatement对象（包含了需要执行的sql语句， 以及#{}占位符**（PS:在执行sql语句时需要替换）**，请求响应参数类型等）。以namespace+sql语句的id属性值作为map中的key，mappedStatement对象为value存入到map容器中。
3. 我们只会写mapper接口，并不会写mapper的实现类。当调用mapper接口中的方法时mybatis会动态的帮我们生成一个代理对象。
4. 所以我们最终调用的其实是代理类中的逻辑（从map中获取mappedstatement对象，得到sql语句封装jdbc执行）。
5. 代理对象将获取到的结果映射为java中的对象

### #{}和${}的区别

#{}是占位符，预编译处理；${}是拼接符，字符串替换，没有预编译处理。

Mybatis在处理#{}时，会将SQL中的#{}替换为?号，调用PreparedStatement的set方法来赋值。

## 模糊查询like语句该怎么写

- ’%${question}%’ 可能引起SQL注入，不推荐
- "%"#{question}"%" 
- **CONCAT(’%’,#{question},’%’) 使用CONCAT()函数，（推荐）**

## 在mapper中如何传递多个参数

- **顺序传参法**，在xml写sql语句时#{0}，#{1}获取参数。 不推荐
- **@Param注解传参法**，推荐
- **Map传参法**，推荐
- **Java Bean传参法**，推荐

## Mybatis如何执行批量操作

- **使用foreach标签**

```xml
foreach标签的属性主要有item(每次遍历的元素)，index（每次遍历的索引），collection（需要遍历的集合），open（以什么开头），separator（以什么分割），close（以什么结束）。

int addEmpsBatch(@Param("emps") List<Employee> emps);

<insert id="addEmpsBatch">
    INSERT INTO emp(ename,gender,email,did)
    VALUES
    <foreach collection="emps" item="emp" separator=",">
        (#{emp.eName},#{emp.gender},#{emp.email},#{emp.dept.id})
    </foreach>
</insert>

<insert id="addEmpsBatch">
    <foreach collection="emps" item="emp" separator=";">                                 
        INSERT INTO emp(ename,gender,email,did)
        VALUES(#{emp.eName},#{emp.gender},#{emp.email},#{emp.dept.id})
    </foreach>
</insert>
```

- java代码中循环单笔插入

  ```java
  for (Student stu: studentList) {
  	studentMapper.insert(stu);
  }
  ```

## 如何获取生成的主键

useGeneratedKeys="true" keyProperty="id" 

## 什么是MyBatis的接口绑定？有哪些实现方式？

任意定义接口，然后把接口里面的方法和SQL语句绑定，直接调用接口方法就可以。

- 注解， 接口方法上加上@Select， @Update, @Delete, @Insert。 注解参数中写sql语句
- 在xml文件中写sql语句。 推荐

## 这个Mapper接口的工作原理是什么？Mapper接口里的方法，参数不同时，方法能重载吗

Mapper接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Mapper接口生成代理proxy对象，代理对象proxy会拦截接口方法，转而执行MappedStatement所代表的sql，然后将sql执行结果返回。



不可以， 因为xml中namespace与接口全限定名相同。 xml中sql语句的id与方法名相同。 namespace+id作为map中的key，重复会被覆盖掉。

## Mybatis不同的Xml映射文件，id是否可以重复？

不同的xml文件id是可以重复的， 因为namespace+id作为map中的key。 而不同的xml文件namespace不同，id自然无所谓

## Mybatis是如何将sql执行结果封装为目标对象并返回的？都有哪些映射形式？

- 使用resultMap
- 使用别名
- 使用resultType（只能映射数据库列名与java字段名相同的属性）

## Xml映射文件中，除了常见的select|insert|updae|delete标签之外，还有哪些标签？

动态sql让我们在Xml映射文件内，以标签的形式编写动态sql，完成逻辑判断和动态拼接sql的功能

- resultMap， 响应参数映射

- parameterMap， 请求参数映射

- sql， sql片段

  ```xml
  <sql id="test">id, name, age, address</sql>
  ```

- include， 引用sql片段

```xml
<select id="queryUser">select <include refid="test"></include></select>
```

- trim,set,choose,when,otherwise,bind不常用
- where   
- if
- foreach
  - item(每次遍历的元素)
  - index（每次遍历的索引）
  - collection（需要遍历的集合）
  - open（以什么开头）
  - separator（以什么分割）
  - close（以什么结束）。



## 你们项目中是如何进行分页的？分页插件的原理是什么？

项目中使用的是PageHelper插件来实现的分页



分页插件的基本原理是使用Mybatis提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的sql，然后重写sql，根据dialect方言，添加对应的物理分页语句和物理分页参数。

举例：select * from student，拦截sql后重写为：select t.* from (select * from student) t limit 0, 10

# springmvc

## 什么是springmvc

springmvc是一个web层框架， 主要是与前端页面进行交互。 主要功能

- 接收参数（参数绑定）

  - 形式参数绑定， 可以是多个字段，可以是java对象
  - @RequestParam注解， 当请求字段名与形参名不同时使用
  - @PathVariable注解， 获取url中的请求参数
  - 通过servlet原生的HttpServletRequest对象

- 响应参数

  - model
  - modelMap
  - modelAndView
  - 通过servlet原生的HttpServletResponse对象

- 跳转页面

  - 重定向（两次请求，无法携带参数，url会发生改变）

    - ```java
      return "redirect:main";
      ```

    - ```
      response.sendRedirect("/www.baidu.com");
      ```

      

  - 转发（一次请求，可以携带参数，url不会发生变化）

    - ```java
      request.getRequestDispatcher("/main").forward(request, response);
      ```

    - ```java
      return "forward:main";
      ```

## springmvc工作原理

1. springmvc需要运行一个web容器（tomcat， jetty, jboss, weblogin）， 而web容器会监听一个端口号。比如启动8080端口，那么当前端发起http 8080端口的请求之后会被tomcat接收，tomcat会将请求转发给springmvc的核心容器DispatcherServlet
2. DispatcherServlet会查找标注了@Controller注解的controller。我们一般会在controller加上@RequestMapping的注解，标注说那些controller处理哪些请求。 此时根据请求的url去定位到哪个controller来进行处理
3. 根据@RequestMapping去查找， 使用这个controller这个类中的哪个方法来进行处理，因为我们也会在方法上加上@RequestMapping这个注解
4. DispatcherServlet会直接调用这个方法来进行处理
5. 方法执行完毕会有一个返回值。 以前的时候一般来说有jsp还在后端项目中，需要返回一个页面，需要经过渲染。  目前的话基本都是前后端分离。 我们只需要返回一个json数据即可。 加上@ResponseBody注解

## Spring MVC常用的注解有哪些？

- @RequestMapping：用于处理请求 url 映射的注解，可用于类或方法上。
- @RequestBody：注解实现接收http请求的json数据，将json转换为java对象。
- @ResponseBody：注解实现将conreoller方法返回对象转化为json对象响应给客户。
- @Controller：控制器的注解，表示是表现层,不能用用别的注解代替
- @RestController: @ResponseBody+@Controller
- @PathVariable: 获取url参数
- @RequestParam: 获取参数

## Spring MVC与Struts2区别

### 共同点

都是web层框架，与前端页面交互。 功能差不多，只是实现方式不同

### 不同点

- 前端控制器不一样。Spring MVC的前端控制器是servlet（DispatcherServlet）， struts2的前端控制器是filter
- springmvc基于方法。 struts2基于类

## Spring MVC怎么和AJAX相互调用的？

加入Jackson.jar    加上@ResponseBody @RequestBody注解

## 如果在拦截请求中，我想拦截get方式提交的方法,怎么配置

- @RequestMapping注解里面加上method=RequestMethod.GET
- @GetMapping

# spring面试题

## 什么是spring

是**一个轻量级Java开发框架**，主要是为了解决应用开发过程中各层之前的依赖耦合关系，简化java开发，让我们更关注于业务逻辑的实现。最核心的特性

- 控制反转（原来需要我们自己来创建对象， 现在我们只需要从ioc容器中获取。控制权交给了spring容器）
- 依赖注入（属性赋值。 set方法或者构造器）
- AOP面向切面编程

## spring中用到了哪些设计模式

- 工厂设计模式。 IOC容器就是一个大的工厂，我们只需要从工厂中获取，无需关心其内部实现
- 单例设计模式。 IOC容器中的bean默认作用域就是单例的（在IOC容器中只会存在一个实例）
- 代理设计模式。 AOP用到了JDK的动态代理和CGLIB动态代理（对目标功能进行增强）
- 模板设计模式。 比如RestTemplate,JdbcTemplate

## 详细讲解一下核心容器（spring context应用上下文) 模块

上下文的意思：承上启下

BeanFactory是最基本的IOC容器。 ApplicationContext对BeanFactory进行了增强，支持了事件处理，国际化等。

而我们常用的IOC容器是ClassPathXmlApplicationContext， AnnotationConfigApplicationContext是ApplicationContext的子类。

## 什么是Spring IOC 容器

Spring IOC 负责创建对象，管理对象（通过依赖注入（DI），装配对象，配置对象，并且管理这些对象的整个生命周期。其实就是个Map。

### 什么是Spring bean？

spring将IOC容器中的对象称为bean。  会将xml中的bean标签解析为BeanDefinition对象存入到map中

<id, beandefinition>

### 如何给Spring 容器提供配置bean

- xml文件

```xml
<bean id="user" class="com.rwb.user.User"></bean>
```

- 基于java配置

  ```java
  @Configuration  //相当于beans
  public class Config {
  
      // bean
  	@Bean
  	public User getUser() {
  		return new User();
  	}
  }
  ```

- 基于注解

  ```java
  @Service
  public class UserServiceImpl implements UserService {
  
  }
  ```

  

## Spring基于xml注入bean的几种方式

- set方法

- 构造方法

  - 通过index设置

    ````xml
    <bean id="user" class="com.rwb.user.User">
    	<construcor index="0" property="张三"/>
    </bean>
    ````

  - 通过name设置，推荐

  ```xml
  <bean id="user" class="com.rwb.user.User">
  	<construcor name="name" property="张三"/>
  </bean>
  ```

  ## Spring支持的几种bean的作用域

- **singleton**   默认， bean的作用域为单例
- **prototype**  多例
- 以下三种为web应用才存在的作用域
  - **request **  每次http请求都会创建一个bean
  - **session **   在一个HTTP Session中，一个bean定义对应一个实例
  - **global-session **  在一个全局的HTTP Session中，一个bean定义对应一个实例

## Spring框架中的单例bean是线程安全的吗

不是， 因为默认都是单例的

## 解释Spring框架中bean的生命周期（bean从创建到销毁的过程）

1. Spring对bean进行实例化， 默认是单例的

2. Spring将值和bean的引用注入到bean对应的属性中；

3. 如果bean实现了一系列Aware接口，则会调用对应的set方法

   ```java
   @Component
   public class SpringUtil implements ApplicationContextAware {
   
   	private static ApplicationContext applicationContext;
   	
   	// 当实现了ApplicationContextAware接口， 会执行set方法将spring容器传递
   	@Override
   	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
   		this.applicationContext = applicationContext;
   	}
   }
   ```

4. 如果bean实现了BeanPostProcessor接口，则会调用postProcessBeforeInitialization()方法

5. 如果bean实现了InitializingBean接口，则会调用afterPropertiesSet()方法。

6. 如果bean实现了BeanPostProcessor接口，则会调用postProcessAfterInitialization()方法

7. 此时bean就可以被使用了

8. 如果bean实现了DisposableBean接口，spring容器关闭时会调用destroy()方法

## 什么是bean装配

bean 装配是指在Spring 容器中把bean组装到一起，前提是容器需要知道bean的依赖关系，如何通过依赖注入来把它们装配到一起

Spring 容器能够自动装配相互合作的bean。 

比如我们要在service中@Autowired一个mapper。 spring容器会自动找到并帮我们注入进来

## spring 自动装配 bean 有哪些方式

- no 默认的方式是不进行自动装配的，通过手工设置ref属性来进行装配bean
- byName：通过bean的名称进行自动装配，如果一个bean的 property 与另一bean 的name 相同，就进行自动装配
- byType：通过参数的数据类型进行自动装配。
- constructor：利用构造函数进行装配，并且构造函数的参数通过byType进行装配。
- autodetect：自动探测，如果有构造方法，通过 construct的方式自动装配，否则使用 byType的方式自动装配。

## 使用@Autowired注解自动装配的过程是怎样的？

1. 实例化bean
2. 发现bean中存在@Autowired需要注入bean时
3. 从容器中查找并注入

## @Component, @Controller, @Repository, @Service 有何区别？

这四个注解都是标识将该类实例化对象并放入到IOC容器中

后三个注解继承了第一个注解

目的是为了不同的层使用不同的注解， 比如web层用@Controller， 业务层用@Service， 持久层用@Repository

## Spring支持的事务管理类型

- **编程式事务管理**， 在代码中管理事务。 手动的commit， rollback
- **声明式事务管理**， 只需用注解和XML配置来管理事务。   推荐

## Spring事务的实现方式和实现原理

Spring事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，spring是无法提供事务功能的。真正的数据库层的事务提交和回滚是通过binlog或者redo log实现的。（InnoDB）

## Spring的事务传播行为

当多个事务同时存在的时候，spring如何处理这些事务的行为

- PROPAGATION_REQUIRED		如果当前没有事务就创建一个新事务，如果当前存在事务，就加入该事务，该设置是最常用的设置。
- PROPAGATION_SUPPORTS     支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行。
- PROPAGATION_MANDATORY   支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就抛出异常。



- PROPAGATION_REQUIRES_NEW   创建新事务，无论当前存不存在事务，都创建新事务。
- PROPAGATION_NOT_SUPPORTED   以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
- PROPAGATION_NEVER   以非事务方式执行，如果当前存在事务，则抛出异常。 
- PROPAGATION_NESTED   如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则按REQUIRED属性执行。

## spring 的事务隔离

- ISOLATION_DEFAULT    用底层数据库的设置隔离级别，数据库设置的是什么我就用什么；
- ISOLATION_READ_UNCOMMITTED  未提交读，最低隔离级别、事务未提交前，就可被其他事务读取（会出现幻读、脏读、不可重复读）；
- ISOLATION_READ_COMMITTED   提交读，一个事务提交后才能被其他事务读取到（会造成幻读、不可重复读）
- ISOLATION_REPEATABLE_READ   可重复读，保证多次读取同一个数据时，其值都和事务开始时候的内容是一致，禁止读取到别的事务未提交的数据（会造成幻读），MySQL 的默认级别；
- ISOLATION_SERIALIZABLE    串行化，代价最高最可靠的隔离级别，该隔离级别能防止脏读、不可重复读、幻读。

## 什么是AOP

OOP: 面向对象编程   AOP：面向切面编程

是对面向对象的一种补充。 将核心业务与边缘业务进行拆分。 在执行核心业务时织入边缘业务。 例如权限校验， 日志， 事务处理等

## JDK动态代理和CGLIB动态代理的区别

Spring AOP中的动态代理主要有两种方式，JDK动态代理和CGLIB动态代理

- JDK动态代理只提供接口的代理，不支持类的代理。
- 如果该类只有一个实现类，没有实现接口则spring会自动切换为CGLIB动态代理

## Spring AOP的几个名词

- 切入点      在什么样的地点
- 通知          在什么样的时间
- 切面          边缘业务

在什么样的时间什么样的地址织入什么样的业务逻辑

## Spring通知有哪些类型

- 前置通知（Before）：在目标方法被调用之前调用通知功能；

- 后置通知（After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么；

- 返回通知（After-returning ）：在目标方法成功执行之后调用通知；

- 异常通知（After-throwing）：在目标方法抛出异常后调用通知；

- 环绕通知（Around）：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为。

  