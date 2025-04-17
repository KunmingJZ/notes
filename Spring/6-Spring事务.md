# Spring事务

关于数据库事务、锁可以先行查看此文：[MySQL数据库读写锁示例详解、事务隔离级别示例详解](https://github.com/Byron4j/CookBook/blob/master/MySQL/1-MySQL数据库读写锁示例详解、事务隔离级别示例详解.md)。

Spring事务属于Data Access模块中的内容，该模块包含事务管理支持以及其它数据访问的集成。

## 事务管理

全面的事务支持是使用Spring框架的最重要原因之一。Spring为事务管理提供了一个始终如一的抽象，优点如下：
- 提供不同事务的API但是一致的编程模型，如Java事务API（JTA）、JDBC、Hibernate和Java持久化API（JPA）。
- 支持声明式事务
- 比JTA更简单的编程式事务API
- 与Spring数据访问抽象的优秀集成

### Spring框架事务模型的优点

习惯上，Java EE 开发者有两种事务管理方式：全局事务管理、本地事务管理，两者都有很大的局限性。

#### 全局性事务管理

全局事务允许你操作多个事务资源，典型的是关系型数据库和消息队列。应用服务器通过JTA管理全局性事务，而JTA API是非常笨重的。另外，一个JTA的```UserTransaction```通常需要从JNDI中加载资源，意味着使用JTA必须配置JNDI。全局性事务限制了代码的重用性，因为JTA通常只在应用服务器环境中可用。


#### 本地事务管理

本地事务是特定于资源的，例如与JDBC关联的事务。本地事务更容易使用，但是也有一个重大的缺陷：不能跨多个事务资源工作。例如，使用JDBC连接的事务管理代码不能在一个JTA的全局性事务中使用。因为应用服务器不参与事务管理，它不能帮助确保跨多个资源的正确性。


#### Spring框架一致性编程模型

Spring解决了全局性事务和本地事务的缺陷，它可以让应用开发者在任何环境下使用一致的编程模型API。你在一个地方编写你的代码，它可以在不同环境的不同事务管理策略中工作。Spring框架提供了```声明式事务```和```编程式事务```。大都数用户偏爱声明式事务，因为编码更简单。

通过编程式事务，开发者通过Spring框架事务抽象来进行开发，可以运行在任何底层事务基础设施上。
使用首选的声明式事务模型，开发者仅需要编写一点点与事务管理关联的代码，因此，不需要依赖Spring框架事务的API或其他事务API。

## Spring事务相关的类


- ```org.springframework.transaction.PlatformTransactionManager``` 事务管理器接口。
- ```org.springframework.transaction.TransactionDefinition``` 事务定义。
- ```org.springframework.transaction.TransactionStatus``` 事务状态。
- ```org.springframework.transaction.support.TransactionSynchronization```
- ```org.springframework.transaction.support.AbstractPlatformTransactionManager``` 实现了PlatformTransactionManager。其它框架集成Spring一般会继承该类。


![](pictures/Spring-tx.png)

Spring框架事务抽象的关键点是事务策略的概念。一个事务策略通过```org.springframework.transaction.PlatformTransactionManager```接口来定义，像以下所展示的：



```java
/**事务管理器*/
public interface PlatformTransactionManager {
    /**根据事务定义获取事务状态*/
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
    /**提交事务*/
    void commit(TransactionStatus status) throws TransactionException;
    /**回滚事务*/
    void rollback(TransactionStatus status) throws TransactionException;
}
```

这主要是一个服务提供者接口(**SPI**)，尽管你可以使用编程方式使用它。因为```PlatformTransactionManager```是一个接口，它可以根据需要很容易地被mock或作为存根使用。它没有绑定到查找策略，比如JNDI等。
PlatformTransactionManager 实现的定义与Spring框架IOC容器中其他任何bean是一样的，仅这一点就使得Spring事务是一个有价值的抽象，甚至你在使用JTA的时候。

同样，为了保持和Spring理念一致，PlatformTransactionManager 接口的方法可以抛出 ```TransactionException ```异常。
**getTransaction(..)** 方法返回一个 ```TransactionStatus```对象，依赖于一个 ```TransactionDefinition```参数，返回的TransactionStatus可能代表一个新的事务或者一个已经存在的事务（如果当前调用堆栈中存在事务）。后一种情况的含义是，与Java EE事务上下文一样，事务状态与执行线程相关联。



## 七种事务传播行为、四种事务隔离级别

```TransactionDefinition ``` 接口指定了：

- **传播性Propagation**：通常，在事务范围内执行的所有代码都在该事务中运行。但是，事务方法在已存在事务上下文执行时，你可以指定其行为。例如，代码可以继续在已经存在的事务中运行（通过是这样的），或者已存在的事务会挂起然后创建一个新的事务。Spring提供了和EJM CMT类似的所有事务传播性操作。
    - **PROPAGATION_REQUIRED**：支持当前事务，如果当前没有事务则新建一个事务。这是默认的事务传播行为。
    - **PROPAGATION_SUPPORTS**：支持当前事务，如果不存在事务则以非事务形式执行。
    - **PROPAGATION_MANDATORY**：支持当前事务，如果没有事务则抛出异常，transaction synchronization还是可用的。
    - **PROPAGATION_REQUIRES_NEW**：新建一个事务，如果当前存在事务则还会挂起已经存在的事务。
    - **PROPAGATION_NOT_SUPPORTED**：不支持当前事务，总是以非事务方式执行。
    - **PROPAGATION_NEVER**：不支持事务，存在事务则抛出异常，transaction synchronization不可用。
    - **PROPAGATION_NESTED**：如果当前存在事务则在嵌套事务中执行，有点类似PROPAGATION_REQUIRED。

- **隔离性Isolation**：指定了事务的隔离性。
    - **ISOLATION_READ_UNCOMMITTED**：读未提交。可能出现脏读、不可重复读、幻读。这个隔离级别，一个事务可以读取另一个事务未提交的内容。
    - **ISOLATION_READ_COMMITTED**：读已提交。阻止了脏读，但是不可重复读、幻读可能会发生。此级别仅禁止事务读取包含未提交更改的行。
    - **ISOLATION_REPEATABLE_READ**：可重复度。阻止了脏读、不可重复度，但是幻读可能会发生。这个级别禁止事务读取包含未提交更改的行，还禁止一个事务读取行、第二个事务更改行、第一个事务重新读取行，第二次获得不同的值(“不可重复读取”)。
    - **ISOLATION_SERIALIZABLE**：串行化。解决了脏读、不可重复度和幻读的问题。效率低，一般生产不用。

- **超时Timeout**：此事务在超时并由事务基础设施自动回滚之前运行多长时间。

- **是否只读Read-only**：当你的代码仅仅读取数据不会更改数据时可以设置只读属性。

这些设置反映了标准的事务概念。理解这些概念，是使用Spring框架或其它事务管理解决方案的基本前提。

**TransactionStatus** 接口为事务代码提供了一种简单的方法来控制事务执行和查询事务状态。

```java
public interface TransactionStatus extends SavepointManager {

    boolean isNewTransaction();

    boolean hasSavepoint();

    void setRollbackOnly();

    boolean isRollbackOnly();

    void flush();

    boolean isCompleted();

}
```

无论您在Spring中选择声明式事务管理还是编程式事务管理，定义正确的PlatformTransactionManager实现都是绝对必要的。通常是通过依赖注入来定义此实现。

```PlatformTransactionManager``` 实现通常需要了解他们的环境：JDBC，JTA，Hibernate等等。以下示例展示了定义了一个本地的```PlatformTransactionManager```实现（此例中，使用了简单的JDBC）。
你可以像以下一样创建一个类似的beam，定义一个

- 1.```JDBC DataSource```配置如下：

```xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driverClassName}" />
    <property name="url" value="${jdbc.url}" />
    <property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
</bean>
```

与之关联的 ```PlatformTransactionManager ``` bean定义则可以引用 DataSource的定义，例如：

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

注解方式：
```java
@Bean(name = "myTxManager")
public PlatformTransactionManager txManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```


如果你是在Java EE容器中使用JTA，你可以使用一个容器DataSource，可以通过JNDI获取数据源，再结合Spring框架的```JtaTransactionManager```。
- 2.```JTA和JDNI查找配置如下```：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/jee
        https://www.springframework.org/schema/jee/spring-jee.xsd">

    <jee:jndi-lookup id="dataSource" jndi-name="jdbc/jpetstore"/>

    <bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager" />

    <!-- other <bean/> definitions here -->

</beans>
```

```JtaTransactionManager``` 不需要知道DataSource（或其他指定的数据源）因为它使用了容器的全局事务管理基础设施。


你也可以使用Hibernate本地事务，像以下示例展示的一样。在此案例中，你需要定义一个Hibernate的```LocalSessionFactoryBean``` bean，则你的应用可以使用来获取Hibernate的会话```session```实例，而DataSource bean则和本地JDBC示例类似。

>❕ ❕
>
>如果```DataSource```(被任何非JTA事务管理器使用的)是在一个Java EE容器中管理且通过JNDI查找到的，则它应该是非事务的，因为Spring框架(而不是Java EE容器)负责管理事务。

在这个案例中的 ```txManager``` bean是一个```HibernateTransactionManager```类型。和```DataSourceTransactionManager```类似，也需要依赖一个DataSource的引用，```HibernateTransactionManager```需要一个```SessionFactory```的引用。示例如下：

```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mappingResources">
        <list>
            <value>org/springframework/samples/petclinic/hibernate/petclinic.hbm.xml</value>
        </list>
    </property>
    <property name="hibernateProperties">
        <value>
            hibernate.dialect=${hibernate.dialect}
        </value>
    </property>
</bean>

<bean id="txManager" class="org.springframework.orm.hibernate5.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory"/>
</bean>
```

如果你是使用Hibernate和Java EE容器管理JTA事务，你应该和之前一样使用```JtaTransactionManager```：
```xml
<bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager"/>
```

>❕ ❕
>
>如果你使用的是JTA，你的事务管理器则应该看起来很像，不管使用什么数据访问技术，不管是JDBC、Hibernate JPA还是任何其他受支持的技术。这是因为JTA事务是全局性事务，它可以征募任何事务资源。

在这些案例中，应用的代码是不需要变更的。您可以仅通过更改配置来更改事务的管理方式，即使这种变化意味着从本地事务转移到全局事务，或者反之亦然。


## 事务资源同步

怎样创建不同的事务管理器和它们是怎样关联那些需要同步到事务中的资源的（例如，```DataSourceTransactionManager```之余一个JDBC DataSource，```HibernateTransactionManager```之于一个Hibernate的```SessionFactory，等等）现在应该是比较清晰的了。

这个部分描述应用代码（直接或间接使用持久化API如JDBC、Hibernate，或者JPA）怎样确保这些资源是如何创建、复用和清除的。

也讨论事务同步（transaction synchronization）是如何通过关联的```PlatformTransactionManager```触发的。


### 高级同步方法

首选的方法是使用Spring最高级的基于模板的持久性集成api或者使用基于transaction-aware factory的原生的ORM API ben 或者代理 去管理本地资源工厂。
这种transaction-aware（事务感知）解决方案是在内部处理资源、重用、清除，资源的可选事务同步，异常映射。
因此，用户数据访问代码可以不用关心这些处理而仅仅将关注持久化逻辑的编写。
一般而言，你可以使用原生ORM API或者使用JdbcTemplate处理JDBC数据访问。

### 低级同步方法

像```DataSourceUtils```类（for JDBC）一样，```EntityManagerFactoryUtils```类(for JPA),```SessionFactoryUtils```类(for Hibernate  )，等等就是比较低级的API了。
当你想在应用代码中直接处理原生持久API的资源类型的时候，你可以使用这些类确保实例是由Spring框架管理的、事务同步是可选的、异常映射到合适的持久化API中。

例如，在 JDBC 的案例中，在DataSource中用于替代传统的 JDBC的```getConnection()```的方法，可以使用 ```org.springframework.jdbc.datasource.DataSourceUtils```类：

```java
Connection conn = DataSourceUtils.getConnection(dataSource);
```

如果一个已经存在的事务已经有一个连接connection同步给它了，则会返回该connection。否则，该方法会触发创建一个新的connection，它(可选地)同步到任何现有事务，并可用于该事务的后续重用。
像之前提到过的一样，任何 SQLException 都被包装在Spring框架中的```CannotGetJdbcConnectionException```（这是一个Spring框架的未检查unchecked的DataAccessException类型的层次结构之一）。
这种方法提供的信息比从SQLException获得的信息要多，并且确保了跨数据库甚至跨不同持久性技术的可移植性。
这种方式是没有在Spring事务管理机制下工作的，因此，无论你是否使用Spring事务管理机制都可以使用它。

### ```TransactionAwareDataSourceProxy``` 事务感知数据源代理类

在最底层存在 ```TransactionAwareDataSourceProxy```类。这是一个数据源DataSource的代理类，包装了一个数据源并且将其添加到Spring事务的感知中。在这方面，类似于Java EE服务器提供的传统的JNDI数据源。

你应该几乎从不会使用这个类，除非当前的代码必须通过一个标准的JDBC数据源接口调用实现。在这个场景中，这些代码是有用的，但是它参与了Spring管理的事务。你可以使用高级的抽象编写新的代码。

## 声明式事务管理

>❕ ❕
>
>大部分的Spring框架使用者会选择声明式事务管理。这个选择对应用代码影响更小，因此，它更符合非侵入式轻量级容器理念。

<u>**Spring框架的声明式事务管理是通过Spring面向切面编程（AOP）实现的。**</u>

然而，因为事务相关的代码是随Spring框架发行版本一块发布的，可以以样板方式使用，通常不需要理解AOP的概念就可以使用这些代码了。

Spring框架的声明式事务管理机制类似于EJB CMT，在这种情况下，你可以将事务行为(或缺少事务行为)指定到单个方法级别。如果有必要的话，你可以在一个事务上下文中调用```setRollbackOnly()```方法。这两种类型的事务管理的差异在于：

- 不像EJB CMT是绑定了JTA的。Spring框架的声明式事务管理可以在任何环境中工作，它可以通过调整配置文件就可以轻易地和JTA事务、使用JDBC的本地事务、JPA或者Hibernate一块工作。
- 你可以在任何类中使用Spring框架声明式事务，而不是像EJB一样只能指定某些类。
- Spring框架提供了声明式回滚规则，这是和EJB等同的特性。编程式、声明式的回滚规则都提供了。
- Spring框架可以让你通过AOP自定义事务行为。例如，你可以在事务回滚的时候插入自定义行为。还可以添加任意的advice(通知)，以及事务advice。而如果是EJB CMT的话，你不可能影响容器的事务管理机制，除非使用```setRollbackOnly()```。
- Spring框架不像高端应用服务器那样支持在远程调用之间传播事务上下文。如果你需要这个特性，推荐你使用EJB。但是，在你使用该特性之前需要慎重，因为，正常情况下，是不想在远程调用之间传播事务的。

<u>**回滚规则的概念是非常重要的。**</u> 它们可以让你指定哪些异常应该引发自动回滚。你可以在配置中而不是Java代码中指定这些声明。所以，尽管你可以在```TransactionStatus```对象中调用```setRollbackOnly()```方法去回滚当前的事务，大都数情况下你可以指定一个规则，即可以自定义异常必须导致事务回滚。这种选择的重要优点是业务对象不依赖事务基础设施。例如，它们通常不需要导入Spring事务API或者其它Spring API。

尽管EJB容器默认行为是在事务发生系统异常（通常是运行时异常）时自动回滚，EJB CMT并不会在出现应用异常时自动回滚。但是Spring声明式事务的默认行为是允许自定义异常变更回滚策略的。

### 理解Spring声明式事务实现

仅仅告诉你使用 ```@Transactional```注解标注你的类是不够的，添加```EnabledTransactionManagement```到你的配置中，并希望你理解它是如何工作的。为了提供一个深刻的理解，这个部分解释在发生与事务相关的问题时，Speing声明式事务机制的内部工作原理。

掌握Spring框架声明式事务的最重要的概念是通过AOP代理实现的，事务通知由元数据（XML或者基于注解的）驱动。

<u>**AOP与事务元数据的结合产生了一个AOP代理，它使用一个事务拦截器```TransactionInterceptor```和一个适当的```PlatformTransactionManager```实现来驱动围绕方法调用的事务。**</u>

以下是通过事务代理调用方法的概念视图：

![](pictures/Spring声明式事务代理调用方法的概念视图.png)


### 声明式事务实现示例

考虑以下接口以及它的实现，这个示例使用了```Foo```和```Bar```类，这样你就可以专注于事务的实现而不用关注具体的域模型了。就这个示例而言，```DefaultFooService```类的每个方法抛出```UnsupportedOperationException```异常是OK的。该行为允许创建事务，然后回滚以响应UnsupportedOperationException实例。

```java
// 我们想进行事务性操作的目标接口

package x.y.service;

public interface FooService {

    Foo getFoo(String fooName);

    Foo getFoo(String fooName, String barName);

    void insertFoo(Foo foo);

    void updateFoo(Foo foo);

}
```

实现类：

```java
package x.y.service;

public class DefaultFooService implements FooService {

    public Foo getFoo(String fooName) {
        throw new UnsupportedOperationException();
    }

    public Foo getFoo(String fooName, String barName) {
        throw new UnsupportedOperationException();
    }

    public void insertFoo(Foo foo) {
        throw new UnsupportedOperationException();
    }

    public void updateFoo(Foo foo) {
        throw new UnsupportedOperationException();
    }

}
```

假设FooService接口的前两个方法getFoo(String)和getFoo(String, String)必须在具有只读语义的事务上下文中执行，而其他方法insertFoo(Foo)和updateFoo(Foo)必须在具有读写语义的事务上下文中执行。以下是符合要求的配置信息：

```xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- 将进行事务性操作的bean -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>
    
    <!-- 数据源配置 -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"/>
        <property name="url" value="jdbc:oracle:thin:@rj-t42:1521:elvis"/>
        <property name="username" value="scott"/>
        <property name="password" value="tiger"/>
    </bean>
    
    <!-- 配置事务管理器 PlatformTransactionManager -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- 配置事务通知 (将会发生什么，查看下面的 <aop:advisor/> 配置) -->
    <tx:advice id="txAdvice" transaction-manager="txManager">
        <!-- 事务性语义配置 -->
        <tx:attributes>
            <!-- 所有get开头的方法都是只读性事务 -->
            <tx:method name="get*" read-only="true"/>
            <!-- 其他的方法使用默认的事务性行为 -->
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- 确保上面配置的事务通知可以在FooService接口的任意操作中执行-->
    <aop:config>
        <!-- 配置切面(切点集合) -->
        <aop:pointcut id="fooServiceOperation" expression="execution(* x.y.service.FooService.*(..))"/>
        
        <!-- 配置AOP通知 -->
        <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceOperation"/>
    </aop:config>
</beans>

```



检查上面的配置，假设你想使一个service对象（fooService bean）可以进行事务性操作。事务性语义可以使用```<tx:advice/>```定义来封装。```<tx:advice/>```定义会读取所有的方法，如果是 get 开头的方法则执行只读性事务，其它的方法则执行默认的事务语义。```<tx:advice/>```的```transaction-manager```属性标签会设置为将要驱动事务的```PlatformTransactionManager```实现bean的name。

>🎻
>如果你配置的```PlatformTransactionManager```的name或者id是```transactionManager```的话，事务通知(```<tx:advice/>```)的```transaction-manager```属性则可以忽略，如果不是的话则必须配置该属性标签。

```<aop:config>```定义确保```txAdvice```bean事务通知可以在适当的切点执行。
首先，首先，定义一个切点，它匹配在FooService接口(```fooServiceOperation```)中定义的任何操作的执行；
然后，使用一个advisor关联切点和事务通知。
结果就是，只要切点fooServiceOperation匹配的方法执行了，txAdvice中定义的事务通知就会运行。

```<aop:pointcut/>```元素定义是AspectJ的切点表达式。可以查看[AOP](3-SpringAOP.md)获取Spring-AOP的信息。

一个常见的需求是使整个service具有事务性。最好的方式就是改变切点表达式：

```xml
<aop:config>
    <aop:pointcut id="fooServiceMethods" expression="execution(* x.y.service.*.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceMethods"/>
</aop:config>
```

现在我们分析过了配置信息了，你也许会问自己：这些配置将会做什么？

```java
public final class Boot {

    public static void main(final String[] args) throws Exception {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("context.xml", Boot.class);
        FooService fooService = (FooService) ctx.getBean("fooService");
        fooService.insertFoo (new Foo());
    }
}
```

运行后查看输出信息可以发现事务性操作的整个流程：
```xml
<!--  Spring 容器启动... -->
[AspectJInvocationContextExposingAdvisorAutoProxyCreator] - Creating implicit proxy for bean 'fooService' with 0 common interceptors and 1 specific interceptors

<!-- DefaultFooService 被动态代理 -->
[JdkDynamicAopProxy] - Creating JDK dynamic proxy for [x.y.service.DefaultFooService]

<!-- ... the insertFoo(..) 在代理中开始调用-->
[TransactionInterceptor] - Getting transaction for x.y.service.FooService.insertFoo

<!-- 事务通知在这里起作用... -->
[DataSourceTransactionManager] - Creating new transaction with name [x.y.service.FooService.insertFoo]
[DataSourceTransactionManager] - Acquired Connection [org.apache.commons.dbcp.PoolableConnection@a53de4] for JDBC transaction

<!--  DefaultFooService 的 insertFoo(..) 抛出一个异常... -->
[RuleBasedTransactionAttribute] - Applying rules to determine whether transaction should rollback on java.lang.UnsupportedOperationException
<!-- 事务拦截器，抛出异常，则回滚insertFoo方法上的事务 -->
[TransactionInterceptor] - Invoking rollback for transaction on x.y.service.FooService.insertFoo due to throwable [java.lang.UnsupportedOperationException]

<!-- 事务回滚，事务完毕后释放连接 -->
[DataSourceTransactionManager] - Rolling back JDBC transaction on Connection [org.apache.commons.dbcp.PoolableConnection@a53de4]
[DataSourceTransactionManager] - Releasing JDBC Connection after transaction
<!-- 归还连接给数据源 -->
[DataSourceUtils] - Returning JDBC Connection to DataSource

Exception in thread "main" java.lang.UnsupportedOperationException at x.y.service.DefaultFooService.insertFoo(DefaultFooService.java:14)
<!-- 为了清晰起见，AOP基础设施堆栈跟踪元素被删除 -->
at $Proxy0.insertFoo(Unknown Source)
at Boot.main(Boot.java:11)
```

### 回滚声明式事务

向Spring框架的事务基础结构表明要回滚事务的推荐方法是从当前正在事务上下文中执行的代码中抛出异常.
Spring框架事务基础结构代码会捕获任何没有处理的异常因为它会从堆栈中冒泡出来从而决定是否标记该事务需要回滚。

在默认配置中，Spring框架事务基础机构代码标记事务回滚只会在运行时异常、非检查异常时回滚。```RuntimeException```（Error实例默认会导致事务回滚）。检查的异常在默认情况下不会引起事务回滚操作。

你可以在配置中准确指明哪种异常类型会导致事务回滚，可以包括检查异常（checked exception），例如：

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
    <tx:method name="get*" read-only="true" rollback-for="NoProductInStockException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

<u>**如果你想在异常抛出时让一个事务回滚，你也可以指定回滚规则。**</u> 如下示例告诉Spring框架事务基础结构，即使面对未处理的```InstrumentNotFoundException```异常，也要提交事务：

```xml
<tx:advice id="txAdvice">
    <tx:attributes>
    <tx:method name="updateStock" no-rollback-for="InstrumentNotFoundException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

当Spring事务框架基础结构捕获一个异常时，它会咨询配置的事务回滚规则从而决定是否回滚事务，最强的匹配的规则获胜。所以，以下示例中，所有除了InstrumentNotFoundException的异常均会导致事务回滚：

```xml
<tx:advice id="txAdvice">
    <tx:attributes>
    <tx:method name="*" rollback-for="Throwable" no-rollback-for="InstrumentNotFoundException"/>
    </tx:attributes>
</tx:advice>
```

你也可以通过编程式指定一个需要的回滚机制，尽管简单但是耦合了Spring框架事务基础结构在你的代码中：

```java
public void resolvePosition() {
    try {
        // some business logic...
    } catch (NoProductInStockException ex) {
        // trigger rollback programmatically
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

强烈推荐你使用声明式方式处理回滚。编程式回滚可以使用，但是它的使用与面向一个纯净的POJO架构背道而驰。


### 为不同的bean配置不同的事务语义

考虑您拥有许多service层对象的场景，并且你想对他们使用完全不同的事务配置。你可以定义不同的```<aop:advisor/>```元素通过```advice-ref```关联不同的```pointcut```。

作为个比较点，首先假设你所有的service层都位于x.y.service包下面。使这个包下面的类所有以Service结尾的类的所有方法都有默认的事务配置，可以如下配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:config>
        <!-- 定义切点，所有Service结尾的类的所有方法 -->
        <aop:pointcut id="serviceOperation"
                expression="execution(* x.y.service..*Service.*(..))"/>

        <!-- 定义AOP通知，引用事务通知和切点 -->
        <aop:advisor pointcut-ref="serviceOperation" advice-ref="txAdvice"/>

    </aop:config>

    <!-- 这两个bean会被事务性控制 -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>
    <bean id="barService" class="x.y.service.extras.SimpleBarService"/>

    <!-- 这些bean不会被事务性处理 -->
    <bean id="anotherService" class="org.xyz.SomeService"/> <!-- (包名不匹配) -->
    <bean id="barManager" class="x.y.service.SimpleBarManager"/> <!-- (类名不是Service结尾) -->

    <!-- 事务通知 -->
    <tx:advice id="txAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- PlatformTransactionManager 配置省略... -->

</beans>
```

以下则是两个不同的bean使用不同的事务配置信息，定义了两组事务通知、两组AOP通知、两个切点：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:config>

        <aop:pointcut id="defaultServiceOperation"
                expression="execution(* x.y.service.*Service.*(..))"/>

        <aop:pointcut id="noTxServiceOperation"
                expression="execution(* x.y.service.ddl.DefaultDdlManager.*(..))"/>

        <aop:advisor pointcut-ref="defaultServiceOperation" advice-ref="defaultTxAdvice"/>

        <aop:advisor pointcut-ref="noTxServiceOperation" advice-ref="noTxAdvice"/>

    </aop:config>

    <!-- this bean will be transactional (see the 'defaultServiceOperation' pointcut) -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- this bean will also be transactional, but with totally different transactional settings -->
    <bean id="anotherFooService" class="x.y.service.ddl.DefaultDdlManager"/>

    <tx:advice id="defaultTxAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <tx:advice id="noTxAdvice">
        <tx:attributes>
            <tx:method name="*" propagation="NEVER"/>
        </tx:attributes>
    </tx:advice>

    <!-- other transaction infrastructure beans such as a PlatformTransactionManager omitted... -->

</beans>
```

### ```<tx:advice/>``` 事务通知配置选项详解

这个部分总结了使用```<tx:advice/>```标签的多种类型的事务配置操作。默认的```<tx:advice/>```配置如下：

- 事务传播类型默认是 PROPAGATION_REQUIRED。
- 隔离级别是 DEFAULT。
- 事务是可读可写的。
- 事务超时为底层事务系统的默认超时，如果不支持超时，则为none。
- 任意```RuntimeException```触发回滚，checked 异常则不会导致回滚。

你可以修改这些默认配置，下表总结了```<tx:advice/>```和```<tx:attributes/> ```标签嵌套的```<tx:method/>```标签的各个属性:

|属性|必须？|默认值|描述|
|---|---|---|---|
|name|Yes||方法名。占位符(*)表示全符合（例如，*表示所有方法、get*表示所有get开头的方法，on*Event表示on开头且Event结尾的方法等等）|
|propagation|No|REQUIRED|和前面的事务传播行为PROPAGATION_REQUIRED一样|
|isolation|No|DEFAULT|事务隔离级别。仅适用于REQUIRED或REQUIRES_NEW的传播设置。|
|timeout|No|-1|事务超时设置。仅适用于REQUIRED或REQUIRES_NEW的传播设置。|
|read-only|No|false|设置为只读事务。仅适用于REQUIRED或REQUIRES_NEW的传播设置。|
|rollback-for|No||指定需要回滚事务的异常|
|no-rollback-for|No||指定不回滚事务的异常|

### 使用 ```@Transaction``` 注解

除了基于XML声明事务配置外，还可以使用基于注解的方式配置事务。
声明式事务语义直接在Java源代码中声明。
不存在过度耦合的危险，因为用于事务的代码几乎总是这样部署的。

>标准的```javax.transaction.Transactional```注解还支持作为Spring自身注解```org.springframework.transaction.annotation.Transactional```的一个替代。

最好的关于```@Transactional```注解的最容易使用的方式：

```java
// 在你想事务性操作的类或者方法标注 @Transactional 注解 
@Transactional
public class DefaultFooService implements FooService {

    Foo getFoo(String fooName);

    Foo getFoo(String fooName, String barName);

    void insertFoo(Foo foo);

    void updateFoo(Foo foo);
}
```

上面的示例使用了类级别的注解，则注解默认在所有的方法中使用。你还可以在每个方法中单独标注使用。注意，类级别的注解并不会对其祖先类作用，在这种情况下，需要在祖先类本地重新声明方法，以便参与子类级别的注释。

当一个POJO类类似上面作为一个bean在Spring上下文中定义的一样，你可以在一个```@Configuration```的配置类中通过一个```@EnableTransactionManagerment```注解使bean实例具有事务性。更多细节可以查看[```org.springframework.transaction.annotation.EnableTransactionManagement```](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/transaction/annotation/EnableTransactionManagement.html)。
该注解启用Spring的注释驱动的事务管理功能，类似于Spring的<tx:*> XML名称空间中的支持，@Configuration 类中可以如下编写代码：
```java
@Configuration
@EnableTransactionManagement
public class AppConfig {

   @Bean
   public FooRepository fooRepository() {
       // 配置并返回具有@Transactional方法的类
       return new JdbcFooRepository(dataSource());
   }

   @Bean
   public DataSource dataSource() {
       // 配置并返回必要的 JDBC DataSource
   }

   @Bean
   public PlatformTransactionManager txManager() {
       return new DataSourceTransactionManager(dataSource());
   }
}
``` 

类似XML中的配置：
```xml
   <beans>

     <tx:annotation-driven/>

     <bean id="fooRepository" class="com.foo.JdbcFooRepository">
         <constructor-arg ref="dataSource"/>
     </bean>

     <bean id="dataSource" class="com.vendor.VendorDataSource"/>

     <bean id="transactionManager" class="org.sfwk...DataSourceTransactionManager">
         <constructor-arg ref="dataSource"/>
     </bean>

 </beans>
```

>**方法可见性和```@Transactional```**
>
>当你使用代理时，你应该将```@Transactional```注解应用于public方法中。如果注解到protected、private或者包级别的方法中，不会报异常，但是事务配置不会生效。可以查看```org.springframework.transaction.annotation.SpringTransactionAnnotationParser#parseTransactionAnnotation方法```。
>
>该方法调用```org.springframework.core.annotation.AnnotationUtils#getAnnotatedMethodsInBaseType```方法。如果你需要使用非public方法中，请考虑使用 AspectJ。


你可以把@Transactional标注到接口定义上、接口中的方法、类定义上、或者类的public方法上。但是仅存在@Transactional注释不足以激活事务行为。@Transactional注解只是元数据会在运行时被事务基础设施感知然后使用元数据配置合适的bean产生事务行为。在前面的示例中，```<tx:annotation-driven/>```元素会开启事务行为。

>🔕🔕🔕
>
>Spring组推荐你将```@Transactional```注解使用在具体的类上，而不是接口。当然，您可以将@Transactional注释放在接口(或接口方法)上，但是只有在使用基于接口的代理时，才会像您所期望的那样工作。事实上Java注解并不会从接口中继承意味着，如果你使用基于类的代理（```proxy-target-class="true"）或者基于aspect的织入，事务设置不会被代理和aspect织入机制识别，则对象就不会被包装进事务代理中。

>🔕🔕🔕
>
>在代理模式中(默认)，只有通过代理传入的外部方法调用才会被拦截。这意味着自身调用（实际上，目标对象的一个方法调用该目标对象的另一个方法）在运行时是不会产生真实事务的，即使被调用的方法被```@Transactional```标注了。而且，代理必须完全初始化以提供预期行为，因此，您不应该在初始化代码(即```@PostConstruct```)中依赖该特性。

考虑使用 AspectJ 模式如果你希望自身调用可以进行事务性操作的话。在这个情况下，没有代理。而目标类是被织入（字节码被修改）后的任何方法的运行时将@Transactional加入其中。

<tx:annotation-driven/> 注解驱动事务设置清单：

|XML属性|注解属性|默认值|描述|
|---|---|---|---|
|transaction-manager|N/A（查看[TransactionManagementConfigurer](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/transaction/annotation/TransactionManagementConfigurer.html)）|transactionManager|事务管理器的name。当不是transactionManager时则需要配置|
|mode|mode|proxy|默认模式(proxy)处理注解bean，使用Spring AOP框架代理。可选项模式（aspectj）通过织入（修改字节码）改变事务行为|
|proxy-target-class|proxyTargetClass|false|仅仅在proxy模式下有用。控制使用@Transactional注释为类创建什么类型的事务代理。如果设置为```true```，基于类的代理会被创建。为false或者忽略设置，则基于JDK接口动态代理的类被创建。可以查看[Spring AOP](3-SpringAOP.md)获取更多关于代理机制的信息。|
|order|order|Ordered.LOWEST_PRECEDENCE|定义@Transactional标注的事务通知的顺序。更多关于通知顺序的信息可以参考[Spring AOP](3-SpringAOP.md)|


>🔕🔕🔕
>
>默认处理```@Transactional```注解的通知模式是```proxy```，只允许通过代理拦截调用。同一类内的本地调用不能以这种方式被拦截。对于更高级的拦截模式，可以考虑结合编译时或加载时织入切换到```aspectj```模式。


>🔕🔕🔕
>
>```proxy-target-class```属性控制被```@Transactional```标注时创建何种类型的事务代理类。如果设置为```true```，基于proxy的类会被创建。如果设置为false或者忽略这个属性，标准的基于JDK接口的代理会被创建。

>🔕🔕🔕
>
>```@EnableTransactionManagement``` 和 ```<tx:annotation-driven/>```会查找在同一个上下文中被 ```@Transactional``` 标注的bean。意味着，如果你在一个```WebApplicationContext```为```DispatcherServlet```配置了annotation-driven的话，它会检查```@Transactional```标记的controller的bean而不是你的service bean，可以查看[SpringMVC](https://github.com/Byron4j/CookBook/blob/master/Spring/SpringMVC处理流程.png)。


**计算事务设置时，最派生的位置优先。**  如下示例中，```DefaultFooService```类，在类定义中被注解标记为一个只读事务。但是在updateFoo(Foo)方法中标记了非只读事务，且传播行为设置为新建事务，则update方法的事务设置以```@Transactional(readOnly = false, propagation = Propagation.REQUIRES_NEW)```为准：
```java
@Transactional(readOnly = true)
public class DefaultFooService implements FooService {

    public Foo getFoo(String fooName) {
        // do something
    }

    // 该方法以这个事务设置优先级别高
    @Transactional(readOnly = false, propagation = Propagation.REQUIRES_NEW)
    public void updateFoo(Foo foo) {
        // do something
    }
}
```

#### ```@Transactional```属性设置

@Transactional 注解是拥有事务性语义的接口、类、方法的元数据（例如，开启一个read-only事务，则当一个方法被调用时，会挂起已经存在的事务）。默认情况下@Transactional的属性设置如下：

- 传播行为设置为 PROPAGATION_REQUIRED。
- 隔离级别设置为 ISOLATION_DEFAULT。
- 事务是可读可写的。
- 事务超时时间默认依赖底层事务系统，不支持超时则为none。
- 运行时异常会回滚事务，任何checked异常则不会。

```java
@Transactional(rollbackFor = { Exception.class }, propagation = Propagation.REQUIRED)
public CoreOffsetReturn2AgentData handlerRepayFlowAndOffsetBill(
        String dataSourceKey, RepayServiceModel model,
        RepayOffsetCommonRequestDTO offsetCommonRequestDTO,
        RepayOrderNo repayOrderNo, Date offsetDate, List<String> loanNos,
        List<CoreOffsetCoupon2AgentParam> couponList, String mctNo)
        throws Exception {
            // TODO
}
```

设置属性清单：

|属性|类型|描述|
|---|---|---|
|value|String|指定要使用的事务管理器的可选限定符。|
|propagation|enum: Propagation|可选的传播行为设置|
|isolation|enum: Isolation|可选的事务隔离级别。仅仅在传播行为为REQUIRED和REQUIRES_NEW时才有效|
|timeout|int 秒单位|可选的事务超时设置。仅仅在传播行为为REQUIRED和REQUIRES_NEW时才有效|
|readOnly|boolean|读写事务与只读事务的设置。仅仅在传播行为为REQUIRED和REQUIRES_NEW时才有效|
|rollbackFor|Class数组，类型必须为Throwable的派生类|可选的事务回滚指定的异常|
|rollbackForClassName|String数组，指定类名|可选|
|noRollbackFor|Class数组|可选项，用于指定不会引发事务回滚的异常|
|noRollbackForClassName|String数组|可选|

目前，您无法显式控制事务的名称，其中“name”表示出现在事务监视器(如果适用的话)和日志输出中的事务名称(例如，WebLogic的事务监视器)。
对于声明式事务，事务名称总是为 ```类的全限定名+.+方法名称```。例如，如果```BusinessService```类的```handlePayment(...)```方法被事务注解标记，则该事务名称为：```com.example.BusinessService.handlePayment```。

#### 使用```@Transactional```的多个事务管理器

大都数Spring应用只需要一个事务管理器，但是还是可能会在单个应用中使用多个独立的事务管理器的。你可以使用```value```属性指定需要使用的```PlatformTransactionManager```事务管理器。这可以是一个bean的name或者是事务管理器bean的限定名。例如，使用限定符，你可以将java代码结合上下文中的事务管理bean一块使用：

```java
public class TransactionalService {

    @Transactional("order")
    public void setSomething(String name) { ... }

    @Transactional("account")
    public void doSomething() { ... }
}
```

以下是事务bean的配置声明：
```xml
<tx:annotation-driven/>

<bean id="transactionManager1" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    ...
    <qualifier value="order"/>
</bean>

<bean id="transactionManager2" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    ...
    <qualifier value="account"/>
</bean>
```
在这个情况中，```TransactionService```中的两个方法在不同的事务管理器中运行。

#### 自定义快捷注解

如果你需要在不同方法中重复使用 ```@Transactional```注解的相同属性，[Spring元注解支持](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-meta-annotations)可以让你自定义快捷注解。

自定义快捷注解如下：
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional("order")
public @interface OrderTx {
}
```

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional("account")
public @interface AccountTx {
}
```

使用自定义快捷注解：
```java
public class TransactionalService {

    @OrderTx
    public void setSomething(String name) { ... }

    @AccountTx
    public void doSomething() { ... }
}
```

在前面的示例中，我们使用语法来定义事务管理器限定符，但是我们也可以包含传播行为、回滚规则、超时和其他特性。

### 事务传播行为

详细描述了Spring中关于事务传播的一些语义.

在spring管理的事务中，请注意物理事务和逻辑事务之间的差异，以及传播设置如何应用于这种差异。

#### 理解 ```PROPAGATION_REQUIRED```

![](pictures/Spring事务传播性---PROPAGATION_REQUIRED.png)

```PROPAGATION_REQUIRED``` 强制执行物理事务，如果当前范围还不存在事务，则在本地执行当前范围的事务，或者参与为更大范围定义的现有“外部”事务。这在同一个线程中通常是一个比较好的处理方式（例如，一个service门面委托给几个repository的方法，其中底层资源必须参与service层的事务）。

>默认情况下，一个事务参与了外部事务的特征的话，会静默地忽略本地事务隔离级别、超时设置、read-only标志。考虑在你的事务管理中开启```validateExistingTransactions```标志为true，如果你想拒绝接收外部事务隔离级别设置的话。这种非宽松模式还拒绝只读不匹配(即，试图参与只读外部范围的内部读写事务)。

当传播行为设置为 PROPAGATION_REQUIRED 时，就会为应用该设置的每个方法创建逻辑事务范围。每个这样的逻辑事务范围都可以单独确定回滚状态，外部事务范围在逻辑上独立于内部事务范围。标准的 PROPAGATION_REQUIRED 传播行为，所有这些事务范围都会映射到物理事务中。所以如果一个内部事务标记了仅仅回滚的标志会影响到外部事务提交的机会。

但是，当一个内部事务设置为仅仅回滚的标记时，外部事务并没有决定回滚本身，所以被内部事务触发回滚操作不是外部事务所期望的。一个相应的```UnexpectedRollbackException ```异常会被抛出。这是所期望的行为，因此事务调用者永远不会被误导，以为提交是在实际没有执行的情况下执行的。So，如果一个内部事务(外部调用方并不知道)静默地标记为一个事务为仅仅回滚，外部调用者仍然会调用commit。外部调用者需要接受一个```UnexpectedRollbackException```以清楚地表明执行了回滚。

#### 理解 ```PROPAGATION_REQUIRES_NEW```

![](pictures/Spring事务传播性---PROPAGATION_REQUIRES_NEW.png)

```PROPAGATION_REQUIRES_NEW``` 和 ```PROPAGATION_REQUIRED``` 刚好相反，总是为每个可影响的事务范围使用一个独立的物理事务，从来不会参与外部已经存在的事务。
在这个布置中，底层资源事务是不同的，因此，可以独立提交或者回滚，外部事务不会受内部事务回滚状态的影响，并且内部事务锁会在它执行完后立马释放。
这样的独立的内部事务也可以声明自己的隔离级别、超时时间、read-only属性，并不会继承外部事务的特征。

#### 理解 ```PROPAGATION_NESTED```

```PROPAGATION_NESTED``` 在多个保存点savepoints中使用一个物理事务。所以一个内部事务的回滚会触发其事务范围内的回滚，外部事务可以继续处理物理事务尽管已经回滚了一些操作。这个设置通常映射到JDBC保存点，所以仅仅在JDBC资源事务才会工作。

### 事务通知操作

假设你想同时执行事务操作和profiling通知，如何在```<tx:annotation-driven/>```上下文中实现？

当你调用 ```updateFoo(Foo)``` 方法的时候，你可能会看到以下行为：
- 启动已配置的profiling aspect。
- 执行事务通知(transactional advice)。
- 被adviced的对象的执行方法。
- 事务提交。
- profiling aspect报告在整个事务方法调用时间。

以下是一个简单的展示profiling aspect案例(StopWatch是一个关于打印时间的封装，会记录执行方法的耗时类似于System.currentTimeMillis()的一个改进，不建议使用在生产环境中。)：

```java
package x.y;

import org.aspectj.lang.ProceedingJoinPoint;
import org.springframework.util.StopWatch;
import org.springframework.core.Ordered;

public class SimpleProfiler implements Ordered {

    private int order;

    // allows us to control the ordering of advice
    public int getOrder() {
        return this.order;
    }

    public void setOrder(int order) {
        this.order = order;
    }

    // this method is the around advice
    public Object profile(ProceedingJoinPoint call) throws Throwable {
        Object returnValue;
        StopWatch clock = new StopWatch(getClass().getName());
        try {
            clock.start(call.toShortString());
            returnValue = call.proceed();
        } finally {
            clock.stop();
            System.out.println(clock.prettyPrint());
        }
        return returnValue;
    }
}
```

通知的顺序是通过```Ordered```接口控制的。
以下配置创建了一个```fooService```bean，并且通过切面和事务通知指定了期望的顺序：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- this is the aspect -->
    <bean id="profiler" class="x.y.SimpleProfiler">
        <!-- 更低的order为1会使得其在事务前面执行，因为后面的事务通知的order是200 -->
        <property name="order" value="1"/>
    </bean>

    <tx:annotation-driven transaction-manager="txManager" order="200"/>

    <aop:config>
            <!-- 这个通知会环绕事务通知执行 -->
            <aop:aspect id="profilingAspect" ref="profiler">
                <aop:pointcut id="serviceMethodWithReturnValue"
                        expression="execution(!void x.y..*Service.*(..))"/>
                <aop:around method="profile" pointcut-ref="serviceMethodWithReturnValue"/>
            </aop:aspect>
    </aop:config>

    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"/>
        <property name="url" value="jdbc:oracle:thin:@rj-t42:1521:elvis"/>
        <property name="username" value="scott"/>
        <property name="password" value="tiger"/>
    </bean>

    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

</beans>
```

你可以以类似的方式配置任意数量的切面。
以下示例创建了同样的设置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- the profiling advice -->
    <bean id="profiler" class="x.y.SimpleProfiler">
        <!-- execute before the transactional advice (hence the lower order number) -->
        <property name="order" value="1"/>
    </bean>

    <aop:config>
        <aop:pointcut id="entryPointMethod" expression="execution(* x.y..*Service.*(..))"/>
        <!-- will execute after the profiling advice (c.f. the order attribute) -->

        <aop:advisor advice-ref="txAdvice" pointcut-ref="entryPointMethod" order="2"/>
        <!-- order value is higher than the profiling aspect -->

        <aop:aspect id="profilingAspect" ref="profiler">
            <aop:pointcut id="serviceMethodWithReturnValue"
                    expression="execution(!void x.y..*Service.*(..))"/>
            <aop:around method="profile" pointcut-ref="serviceMethodWithReturnValue"/>
        </aop:aspect>

    </aop:config>

    <tx:advice id="txAdvice" transaction-manager="txManager">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- other <bean/> definitions such as a DataSource and a PlatformTransactionManager here -->

</beans>
```

上面的示例中```fooService``` bean被带有order属性的切面和事务通知作用。order值越小优先级越高。
参考：org.springframework.core.annotation.Order以及org.springframework.core.Ordered

### 通过AspectJ使用```@Transactional```注解

您还可以通过AspectJ切面在Spring容器之外使用Spring框架的```@Transactional```支持。
为了达到这个目标，首先使用```@Transactional```注解标注你的类，然后使用spring-aspects.jar中定义的```org.springframework.transaction.aspectj.AnnotationTransactionAspect```织入到你的应用中去。
你也可以通过事务管理器配置切面。
你可以使用Spring框架的IOC容器来处理依赖注入的切面。
配置事务管理的切面的最简单方式是使用```<tx:annotation-driven/>```元素并且指定```mode```属性为```aspectj```（上面已经提到过了）。因为我们在这里聚焦于在Spring容器外面使用，展示如何编程式处理。

以下示例展示了如何创建一个事务管理器、配置```AnnotationTransactionAspect ```使用它：
```java
// 构造一个核实的事务管理器
DataSourceTransactionManager txManager = new DataSourceTransactionManager(getDataSource());AnnotationTransactionAspect

// 配置 AnnotationTransactionAspect 使用；必须在事务方法之前执行。
AnnotationTransactionAspect.aspectOf().setTransactionManager(txManager);
```

>当你使用这个切面的时候，你必须注解到实现类上面（或者实现类的方法中），而不是接口上。Aspectj遵循Java规则--在接口上的注解是不会继承的。

类中方法的```@Transactional```注解指定了类中的public方法的默认的事务语义。
类中方法的```@Transactional```注解覆盖了类中（如果指定了的话）的事务语义。
为了在你的应用中织入```AnnotationTransactionAspect```，你必须通过Aspectj构建你的应用（[Aspectj开发指南](https://www.eclipse.org/aspectj/doc/released/devguide/index.html)）或者在加载时织入。加载时织入可以查看[在Spring框架中通过AspectJ织入](https://docs.spring.io/spring/docs/5.1.6.RELEASE/spring-framework-reference/core.html#aop-aj-ltw)



## 编程式事务管理

Spring框架提供了两种编程式事务管理方式，通过使用：

- ```TransactionTemplate```
- ```PlatformTransactionManager```实现

Spring项目组推荐编程式事务管理使用```TransactionTemplate```。第二种方式和使用 JTA UserTransaction API 类似，尽管异常处理没那么麻烦。

### 使用事务模板类```TransactionTemplate```和```TransactionCallback```事务回调类

```TransactionTemplate```采用了和Spring中其他模板类如```JdbcTemplate```类似的方式。使用回调的方式（将应用代码从必须执行样板文件获取和释放事务资源中解放出来）产生意图驱动的代码，在你的代码中你只需聚焦于做你想做的。

>如下示例中，使用```TransactionTemplate```从Spring事务基础结构的API中完全解耦出来。编程式事务管理是否适合你的开发需要由你自己决定。

应用代码必须在事务上下文中执行并显示使用```TransactionTemplate```。
作为一个程序开发者，你可以编写一个```TransactionCallback```实现(通常表示为匿名内部类)包含你想在事务上下文中执行的代码。
然后，你需要将自定义的```TransactionCallback```实例传入到```TransactionTemplate```暴露的```execute(...)```方法中，以下是两个示例：

```java
public class SimpleService implements Service {

    // single TransactionTemplate shared amongst all methods in this instance
    private final TransactionTemplate transactionTemplate;

    // use constructor-injection to supply the PlatformTransactionManager
    public SimpleService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }

    public Object someServiceMethod() {
        return transactionTemplate.execute(new TransactionCallback() {
            // the code in this method executes in a transactional context
            public Object doInTransaction(TransactionStatus status) {
                updateOperation1();
                return resultOfUpdateOperation2();
            }
        });
    }
}
```


如果没有返回值的话，可以使用```TransactionCallbackWithoutResult```更简洁：

```java
transactionTemplate.execute(new TransactionCallbackWithoutResult() {
    @Override
    protected void doInTransactionWithoutResult(TransactionStatus status) {
        try {
            repayConfirm(dataSourceKey, resultDto);
        } catch (Exception e) {
            logger.error("确认还款处理数据失败:" , e);
            throw new BusinessException(e);
        }
    }
});
```

- org.springframework.transaction.support.TransactionTemplate 操作事务模板类
- org.springframework.transaction.support.TransactionCallback 事务回调，在该回调中可以编写自己的代码
- org.springframework.transaction.support.TransactionCallbackWithoutResult Spring中自带的一个事务回调实现抽象类

事务回调中的代码可以调用```TransactionStatus```的 ```setRollbackOnly()```方法回滚事务：
```java
transactionTemplate.execute(new TransactionCallbackWithoutResult() {

    protected void doInTransactionWithoutResult(TransactionStatus status) {
        try {
            updateOperation1();
            updateOperation2();
        } catch (SomeBusinessException ex) {
            status.setRollbackOnly();//回滚事务
        }
    }
});
```

**指定事务设置**

你可以在通过编程式或者配置中指定```TransactionTemplate```的事务设置（例如传播行为、隔离级别、超时设置或者其他的设置）。
默认情况下，```TransactionTemplate```具有默认的事务设置。

编程式设置：
```java
public class SimpleService implements Service {

    private final TransactionTemplate transactionTemplate;

    public SimpleService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);

        // the transaction settings can be set here explicitly if so desired
        this.transactionTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_UNCOMMITTED);
        this.transactionTemplate.setTimeout(30); // 30 seconds
        // and so forth...
    }
}
```

XML配置：
```xml
<bean id="sharedTransactionTemplate"
        class="org.springframework.transaction.support.TransactionTemplate">
    <property name="isolationLevelName" value="ISOLATION_READ_UNCOMMITTED"/>
    <property name="timeout" value="30"/>
</bean>"
```

然后你可以将```sharedTransactionTemplate```这个bean注入到任何需要的service中去。

最后，```TransactionTemplate```类是线程安全的，在则种情况下不会维护任何会话状态。
但是，```TransactionTemplate```实例，会维护配置状态。所以，很多类会共享一个单例的```TransactionTemplate```实例。
如果一个类需要使用不同的配置，则需要才能创建不同的```TransactionTemplate```实例。

### 使用```PlatformTransactionManager```

也可以使用```org.springframework.transaction.PlatformTransactionManager```直接管理你的应用。为此，请通过bean引用将使用的```PlatformTransactionManager```的实现传递给bean。然后，通过使用```TransactionDefinition```和```TransactionStatus```对象，你可以初始化事务、回滚、提交。
```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
// 显示指定事务名称
def.setName("SomeTxName");
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

TransactionStatus status = txManager.getTransaction(def);
try {
    // 执行业务逻辑代码
}
catch (MyException ex) {
    txManager.rollback(status);
    throw ex;
}
txManager.commit(status);
```

## 在编程式事务管理和声明式事务管理二者的选择

编程式事务管理通常是一个好点子但仅仅在于存在少部分的事务操作的场景中。例如，如果你有一个web应用仅仅在update操作中需要事务，你不想使用Spring或者其他技术设置事务代理。在这种情况下，使用```TransactionTemplate```可能是一个很好的方式。
另一方面，如果你的应用存在大量的事务性操作，声明式事务管理更好，它容易配置、并且是在业务逻辑外面管理事务。
使用Spring框架而不是EJB CMT，声明性事务管理的配置成本将大大降低。


## Spring事务监听器、事务绑定的事件

从Spring4.2开始，一个事件的监听器可以绑定到事务的各个阶段。典型的示例就是在事务成功处理完成后处理事件。
当事务结果对监听器很重要的时候，这个特点可以使得事务处理更灵活。

你可以使用```@EventListener```注解注册一个常规的事件监听器。如果你想将其绑定到事务中去，则使用```@TransactionEventListener```。默认情况下，是绑定到commit阶段的。

以下示例展示了该概念，假设组件发布了一个订单创建的事件，并且我们希望定义一个监听器，该监听器应该只在发布事件的事务提交成功后才处理该事件。
```java
@Component
public class MyComponent{
    @TransactionalEventListener
    public void handleOrderCreatedEvent(CreationEvent<Order> creationEvent){
        ...
    }
}
```

### 事务的阶段

```@TransactionalEventListener```注解暴露了一个事务阶段的属性。可以让你自定义绑定到事务的某个阶段。```TransactionPhase```枚举类指定了有效的事务阶段值：

- **BEFORE_COMMIT** : 事务提交之前
- **AFTER_COMMIT** ： 事务提交成功之后。默认值。
- **AFTER_ROLLBACK** ： 事务回滚。
- **AFTER_COMPLETION** : 事务完成（包含回滚、或者提交完成）。

如果没有事务运行，监听器则不会被调用，因为不能遵守所需的事务语义。但是你可以通过设置```fallbackExecution```属性为```true```来覆盖其行为。



## 特定于应用服务器的集成

Spring的事务抽象通常与应用服务器无关。另外，Spring的```JtaTransactionManager```类（可以选择通过JNDI查找JTA UserTransaction和TransactionManager对象）自动发现后一个对象的位置，后者因应用服务器的不同而不同。
在访问JTA ```TransactionManager```时允许增强的事务语义——特别的，支持事务挂起。可以查看[JtaTransactionManager](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html)获取更多的信息。

Spring的```JtaTransactionManager```是众所周知的运行Java EE应用服务的标准选择。更高级的功能：例如事务挂起，在很多服务器中（GlassFish、JBoss等）不需要做额外的配置就能运行很好。但是，为了支持完整的事务挂起和更多高级功能的集成，Spring为WebLogic服务器和WebSphere服务器指定了特殊的适配器。

对于标准的场景，包括WebLogic服务器和WebSphere服务器，考虑使用```<tx:jta-transaction-manager/>```配置元素。在配置的时候，这个元素会自动地查找出下面的服务并选择适用于平台最好的事务管理器。这意味着你不必显示地配置服务指定地适配器，相反```JtaTransactionManager```会自动地选择。

### IBM WebSphere

在WebSphere 6.1.0.9及以上版本中，推荐使用的Spring JTA事务管理器是```WebSphereUowTransactionManager```。这个适配器使用了IBM的```UOWManager```的API，存在于WebSphere 6.1.0.9及以上版本的服务器中。
使用这个适配器，支持Spring驱动的事务挂起(挂起和恢复由PROPAGATION_REQUIRES_NEW启动)。

### Oracle WebLogic 服务器

在WebLogic Server 9.0或更高版本上，通常使用```WebLogicJtaTransactionManager```而不是```JtaTransactionManager```类。这是```JtaTransactionManager```的一个子类，在weblogic管理的事务环境中支持Spring事务定义的全部功能，超出标准JTA语义。
特性包括：事务名称、事务隔离级别设置、合适的事务重启机制等。

## 常见问题的解决方案

### 对于指定的```DataSource```使用了错误的事务管理器

根据你的事务技术选择和需求选择正确的```PlatformTransactionManager```实现。为了使用适当，Spring仅仅提供了一个简单且可移植的抽象。如果你要使用全局性事务，你必须使用```org.springframework.transaction.jta.JtaTransactionManager```类。否则，事务基础结构会将尝试对容器数据源实例等资源执行本地事务。这样的本地事务没有意义，好的应用程序服务器会将它们视为错误。

## 更多资源

更多关于Spring框架事务支持，可以查看：

- [Spring中的分布式事务，使用XA和不使用XA](https://www.javaworld.com/javaworld/jw-01-2009/jw-01-spring-transactions.html)
- [Java事务设计策略](https://www.infoq.com/minibooks/JTDS)




## Spring框架事务配置总结

- 配置数据源DataSource
- 配置事务管理器即```PlatformTransactionManager```相应的合适实现
- 有必要则配置事务通知（或者注解 @Transactional）
- 有必要则配置自定义AOP通知

## Spring框架事务读源码类

- org.springframework.jdbc.datasource.DataSourceTransactionManager#doGetTransaction
- org.springframework.jdbc.datasource.DataSourceTransactionManager.DataSourceTransactionObject
- org.springframework.jdbc.datasource.JdbcTransactionObjectSupport#createSavepoint
- org.springframework.jdbc.datasource.JdbcTransactionObjectSupport#rollbackToSavepoint
- org.springframework.jdbc.datasource.ConnectionHolder 事务提交、回滚的委托类，占有一个java.sql.Connection

- org.springframework.transaction.annotation.EnableTransactionManagement 事务管理配置注解

- org.springframework.transaction.support.DefaultTransactionDefinition 默认事务行为类

- org.springframework.transaction.support.TransactionTemplate 操作事务模板类
- org.springframework.transaction.support.TransactionCallback 事务回调，在该回调中可以编写自己的代码
- org.springframework.transaction.support.TransactionCallbackWithoutResult Spring中自带的一个事务回调实现抽象类

- org.springframework.transaction.event.TransactionalEventListener  事务监听器
- org.springframework.transaction.event.TransactionPhase 事务阶段
- org.springframework.transaction.support.TransactionSynchronization 用于事务同步回调的接口

- org.springframework.transaction.interceptor.TransactionInterceptor 事务拦截器
- org.springframework.transaction.interceptor.TransactionProxyFactoryBean 事务代理AOP工厂bean
- org.springframework.transaction.interceptor.TransactionAttributeSource  TransactionInterceptor用于元数据检索的策略接口。其实现知道如何从配置、源级别的元数据属性(如Java 5注释)或其他任何地方获取事务属性。```org.springframework.core.annotation.AnnotatedElementUtils#searchWithFindSemantics(java.lang.reflect.AnnotatedElement, java.lang.Class<? extends java.lang.annotation.Annotation>, java.lang.String, java.lang.Class<? extends java.lang.annotation.Annotation>, org.springframework.core.annotation.AnnotatedElementUtils.Processor<T>, java.util.Set<java.lang.reflect.AnnotatedElement>, int)```


## 使用 ```TransactionProxyFactoryBean```类

```xml
<bean id="baseTransactionProxy" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean"
     abstract="true">
   <property name="transactionManager" ref="transactionManager"/>
   <property name="transactionAttributes">
     <props>
       <prop key="insert*">PROPAGATION_REQUIRED</prop>
       <prop key="update*">PROPAGATION_REQUIRED</prop>
       <prop key="*">PROPAGATION_REQUIRED,readOnly</prop>
     </props>
   </property>
 </bean>

 <bean id="myProxy" parent="baseTransactionProxy">
   <property name="target" ref="myTarget"/>
 </bean>

 <bean id="yourProxy" parent="baseTransactionProxy">
   <property name="target" ref="yourTarget"/>
 </bean>
```


