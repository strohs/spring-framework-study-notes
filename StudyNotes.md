Container, Dependency Injection, Inversion of Control
======================================================
## Dependency Injection Basics
* reduces coupling between classes
* enables easy switching of dependencies based on current needs. local development vs production
environment, easy mocking for unit tests, in memory DB vs production DB
* promotes programming to interfaces
* Beans are written as POJOs, no need to inherit Spring base classes or implement Spring interfaces
* lifecycle of beans is managed by spring container(eg. enforces singleton state of beans / one instance per session /
one instance per HTTP request / ...)
* Spring configuration is possible via XML or java config classes

## Application Context
* Spring beans are managed by the application context
* Application context is initialized by one or more configuration files which contain settings for the framework,
DI, persistence, transactions
* Application context can also be initialized in unit tests (NOTE: I would consider this an integration test)
* Ensures beans are always created in the right order based on their dependencies
* Ensures beans are always fully initialized before their first use
* Each bean has a unique identifier, by which it can later be referenced
  * should not contain specific implementation details, based on circumstances different implementations can be provided
* No need for a full fledged java EE application server


### Creating ApplicationContext

* XML Config: there are two main "convenience" classes for loading XML from classpath or file-system 
  * ```ClassPathXmlApplicationContext```
      * load XML config(s) from the classpath
      * interprets plain paths as class path resource names
      * Can use Ant-style patterns like "/myfiles/*-context.xml"
      * In case of multiple config locations, later bean definitions will override ones defined in earlier loaded files.
      * Constructor examples:
        * ClassPathXmlApplicationContext()
        * ClassPathXmlApplicationContext( ApplicationContext parent )
        * ClassPathXmlApplicationContext( String ... configLocations ) - load definitions from given xml files
        * ClassPathXmlApplicationContext( String configLocation )
        * ClassPathXmlApplicationContext( String[] configLocations, ApplicationContext parent )
        * ClassPathXmlApplicationContext( String[] configLocations, boolean refresh )
        * ClassPathXmlApplicationContext( String[] configLocations, boolean refresh, ApplicationContext parent )
        * ClassPathXmlApplicationContext( String[] paths, Class<?> clazz )
          * paths - array of relative (or absolute) paths within the class path
          * clazz - the class to load resources with (basis for the given paths)
          * The basic idea is that one supplies a String array containing just the filenames of the XML files themselves
            (without leading path information) and one also supplies a *Class*, the ClassPathXmlApplicationContext will derive
            the path information from the supplied class
          ```java
          ApplicationContext ctx = new ClassPathXmlApplicationContext(
              new String[] {"services.xml", "daos.xml"}, MessengerService.class);
          ```
        * ClassPathXmlApplicationContext( String[] paths, Class<?> clazz, ApplicationContext parent )
        * ClassPathXmlApplicationContext( String path, Class<?> clazz )

        ```java //creating the context
        ApplicationContext context = new ClassPathXmlApplicationContext ("classpath:spring/application-config.xml");
        ```
        * ```classpath*:conf/appContext.xml``` simply means that all appContext.xml files under conf folders in
        all your jars on the classpath will be picked up and joined into one big application context

  * ```FileSystemXmlApplicationContext```
    * load context definition from XML file(s) in the file system.
    * plain paths are interpreted as relative to the current VM working directory, **even if they start with a slash**
    * use an explicit **file:** prefix to enforce an absolute file path
    * In case of multiple config locations, later bean definitions will override ones defined in earlier loaded files
    * Constructor examples
      * FileSystemXmlApplicationContext()
      * FileSystemXmlApplicationContext( ApplicationContext parent )
      * FileSystemXmlApplicationContext( String... configLocations )
      * FileSystemXmlApplicationContext( String configLocation )
        * ```java ApplicationContext context = new FileSystemXmlApplicationContext("/knight.xml")```
      * FileSystemXmlApplicationContext( String[] configLocations, ApplicationContext parent )
      * FileSystemXmlApplicationContext( String[] configLocations, boolean refresh )
      * FileSystemXmlApplicationContext( String[] configLocations, boolean refresh, ApplicationContext parent )
        
* Java Config: 
  * org.springframework.context.annotation.AnnotationConfigApplicationContext
    * loads configuration from one or more Java configuration classes
  * Constructor examples:
      * AnnotationConfigApplicationContext()
      * AnnotationConfigApplicationContext( Class<?>... annotatedClasses )
        ```java
            ApplicationContext ctx = new AnnotationConfigApplicationContext(ApplicationConfig.class);
            // everything wires up across configuration classes...
            SimpleUserService simpleUserService = (SimpleUserService) ctx.getBean("simpleUserService");
        ```
      * AnnotationConfigApplicationContext( DefaultListableBeanFactory beanFactory )
      * AnnotationConfigApplicationContext( String... basePackages )
        * scans for bean definitions in the given packages and automatically refreshes the context
  * example java config class
    ```java
    //example Java configuration class
    @Configuration
    @PropertySource("classpath:db/datasource.properties")
    public class ApplicationConfig {
        @Value("${url}")
        private String url;
        
        @Value("${username}")
        private String username;
        
        @Value("${password}")
        private String password;
        
        @Bean
        public UserService simpleUserService() throws SQLException {
            return new SimpleUserService(userRepo());
        }
        @Bean
        public UserRepo userRepo() throws SQLException {
            return new JdbcUserRepo(dataSource());
        }
        @Bean
        public DataSource dataSource() throws SQLException {
            OracleDataSource ds = new OracleDataSource();
            ds.setURL(url);
            ds.setUser(username);
            ds.setPassword(password);
            return ds;
        }
    }
    ```
### Creating Application Context for a web application
* ```AnnotationConfigWebApplicationContext``` - loads Spring Web Application context from one or more Java based
configuration classes.
* ```XmlWebApplicationContext``` - loads context definitions from one or more XML files contained in a Web Application

Application Context can also be (automatically created) when creating a web application via:
* AbstractContextLoaderInitializer - ultimately creates a Root App.Context of type ```WebApplicationContext```
* AbstractAnnotationConfigDispatcherServletInitializer - used when creating a web app using only java configurations
  * creates a Root web application context (and one or more Web Application Contexts) 
  of type ```AnnotationConfigWebApplicationContext```

## Shutting down the Spring IoC container gracefully in non-web applications
To register a shutdown hook, you call the ```registerShutdownHook()``` method that is declared on the 
ConfigurableApplicationContext interface. Doing so ensures a graceful shutdown and calls the relevant destroy methods 
on your singleton beans so that all resources are released. Of course, you must still configure and implement these 
destroy callbacks correctly. **Spring’s web-based ApplicationContext implementations already have code in place to shut 
down the Spring IoC container gracefully when the relevant web application is shut down.**
```java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

        // add a shutdown hook for the above context...
        ctx.registerShutdownHook();

        // app runs here...

        // main method exits, hook is called prior to the app shutting down...
    }
}
```

## XML bean definition
* *id* attribute is used to uniquely identify a bean
  * uniqueness enforced by the container
  * must be a single value
  * alphanumeric by convention but may contain special characters
  * required if you are going to refer to the bean from other beans
* *name* attribute acts as an alias for the bean
  * cam have more than one name in the attribute separated by *,* or *;* or whitespace (e.g. <bean name="beanA1,beanB2" > )
* do not **have** to specify id and/or name attribute. Spring will auto-generate an id
  * eg. if bean class is ```<bean class="soundsystem.SgtPeppers" />``` then bean id would be ```soundsystem.SgtPeppers#0 ```

## XML Configuration Dependency Injection
Two types of DI for when using XML Config: **constructor** based and **setter** based

#### Constructor
* Constructor argument resolution occurs using the arguments **type**.
  * if no potential ambiguity exists in the constructor arguments of a bean definition, then the order in which the
  arguments are defined will be the order in which they are supplied to the actual constructor.
  * If there is ambiguity you are going to have to use: *name* or *index* attributes

* uses the **ref** attribute to refer to another bean id to inject as a constructor parameter.
```xml
<bean id="..." class="...">
        <constructor-arg ref="..."/>
</bean>
```
* There is also the **value** attribute that can be used when the value to inject is a scalar.
```xml
<beans ...>
  <bean id="complexBean" class="com.ps.beans.ctr.ComplexBeanImpl">
        <constructor-arg ref="simpleBean"/>
        <constructor-arg value="true"/>
  </bean>
</beans>
```
* examples of constructor injecting multiple scalars:
    * When using multiple simple types (scalars), **Spring cannot determine the type**, so you have to help it by 
    using: index,type, or name attributes
```java
package examples;

public class ExampleBean {

    // Number of years to calculate the Ultimate Answer
    private int years;

    // The Answer to Life, the Universe, and Everything
    private String ultimateAnswer;

    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```
* specifying parameter **type**
```xml
<bean id="exampleBean" class="examples.ExampleBean">
     <constructor-arg type="int" value="7500000"/>
     <constructor-arg type="java.lang.String" value="42"/>
 </bean>
```
* specifying **index** position
```xml
<bean id="exampleBean" class="examples.ExampleBean">
  <constructor-arg index="0" value="7500000"/>
  <constructor-arg index="1" value="42"/>
</bean>
```
* specifying parameter **name**
  * **in order to use name parameter, you must compile your code with debug flag enabled**
```xml
<bean id="exampleBean" class="examples.ExampleBean">
  <constructor-arg name="years" value="7500000"/>
  <constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```
  * if you can't set the debug flag, you can use  **@ConstructorProperties** JDK annotation to explicitly name
  the constructor parameters
  ```java
   @ConstructorProperties({"years", "ultimateAnswer"})
       public ExampleBean(int years, String ultimateAnswer) {
           this.years = years;
           this.ultimateAnswer = ultimateAnswer;
       }
  ```



#### Setters
*  Use setter methods defined in a java class to inject dependencies after invoking a no-argument constructor or
 no-argument static factory method to instantiate your bean.
```xml
<beans ...>
    <bean id="simpleBean0" class="com.ps.beans.SimpleBeanImpl"/>
    <bean id="complexBean" class="com.ps.beans.set.ComplexBeanImpl">
        <property name="simpleBean" ref="simpleBean0"/>
    </bean>
</beans>
```
The corresponding java class using setter injection will look like:
```java
public class ComplexBeanImpl implements ComplexBean {
   private SimpleBean simpleBean;
   // no-argument empty constructor, not mandatory
   public ComplexBeanImpl() {}
   public void setSimpleBean(SimpleBean simpleBean) {
        this.simpleBean = simpleBean;
   }
   public SimpleBean getSimpleBean() {
        return simpleBean;
   }
}
```
another property setter example:
```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <!-- setter injection using the nested ref element -->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>

    <!-- setter injection using the neater ref attribute -->
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```
* The ```<property />``` element defines the property to be set and the value to be set with and does so using
a pair of attributes: [name, ref] or [name,value].
* *name* attribute is **mandatory**.
* *ref* tells the container the value of this attribute is another bean.
* *value* indicates setting a scalar value.


#### Best practices
When deciding between getter and setter injection:
* Use constructor injection when properties are **required**
* Use setter injection when properties are **not required**
* Use the strategy matching the implementation of the bean type
* Be consistent

### Injecting dependencies that are not Beans
The spring container can automatically convert primitive types and their reference wrapper types. Spring can also
convert Date values if the syntax matches a date pattern with "/" separators (dd/MM/yyyy. yyyy/MM/dd, etc.) however
note that invalid numerical combinations can also be accepted and converted e.g.(25/2433/23)

##### Property Editors
Configure these to tell spring how to explicitly convert types, you have to define a custom implementation of 
java.beans.PropertyEditor and register it with the Spring Context.
```java
package com.ps.beans.others;
import org.springframework.beans.PropertyEditorRegistrar;
import org.springframework.beans.PropertyEditorRegistry;
import org.springframework.beans.propertyeditors.CustomDateEditor;

import java.text.SimpleDateFormat;
import java.util.Date;
public class DateConverter implements PropertyEditorRegistrar {

    public void registerCustomEditors(PropertyEditorRegistry registry) {
        registry.registerCustomEditor(Date.class,
                new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), false));
    }
}
```
And then in XML Config:
```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="propertyEditorRegistrars">
        <list>
            <bean class="com.ps.beans.others.DateConverter" />
        </list>
    </property>
</bean>
```

#### Injecting collections
The ```<list> <set> <map>``` tags can be used to inject collections.
List of String values:
```xml
<bean id="compactDisc" class="soundsystem.BlankDisc">
  <constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
  <constructor-arg value="The Beatles" />
  <constructor-arg>
    <list>
      <value>Sgt. Pepper's Lonely Hearts Club Band</value>
      <value>With a Little Help from My Friends</value>
      <value>Lucy in the Sky with Diamonds</value>
      <value>Getting Better</value>
      <value>Fixing a Hole</value>
      <!-- ...other tracks omitted for brevity... -->
    </list>
  </constructor-arg>
</bean>
```
List of bean refs
```xml
<bean id="beatlesDiscography" class="soundsystem.Discography">
  <constructor-arg value="The Beatles" />
  <constructor-arg>
    <list>
      <ref bean="sgtPeppers" />
      <ref bean="whiteAlbum" />
      <ref bean="hardDaysNight" />
      <ref bean="revolver" />
...
    </list>
  </constructor-arg>
</bean>
```
Example using ```<set>```, just like using a java.util.Set:
```xml
<bean id="compactDisc" class="soundsystem.BlankDisc">
  <constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
  <constructor-arg value="The Beatles" />
  <constructor-arg>
    <set>
      <value>Sgt. Pepper's Lonely Hearts Club Band</value>
      <value>With a Little Help from My Friends</value>
      <value>Lucy in the Sky with Diamonds</value>
      <value>Getting Better</value>
      <value>Fixing a Hole</value>
     <!-- ...other tracks omitted for brevity... -->
    </set>
  </constructor-arg>
</bean>
```
### Bean Factories
Can be used to declare beans that are created by a factory method or singleton objects.
* **factory-method** attribute is used to invoke the static method of the singleton class that creates the bean.
```xml
<beans .../>
  <bean id="simpleSingleton" class="com.ps.beans.others.SimpleSingleton" factory-method="getInstance" />
</beans>
```
* To use a factory object to create a bean, then **factory-bean** attribute is used with **factory-method** attribute
  * factory-bean - contains the name of the object that creates the bean
  * factory-method - is the actual method to invoke (non-static)
```xml
<beans .../>
  <bean id="simpleBeanFactory" class="com.ps.beans.others.SimpleFactoryBean"/>
  <bean id="simpleFB" factory-bean="simpleBeanFactory" factory-method="getSimpleBean" />
</beans>
```
### Xml Namespaces
* Default namespaces is usually ```beans``` but there are others:
* Many other namespaces available
  * ```context```
  * ```aop``` (Aspect Oriented Programming)
  * ```tx``` (transactions)
  * ```util```
  * ```jms```
  * ```jdbc```
  * ...
* Provides tags to simplify configuration and hide detailed bean configuration
```xml
<context:property-placeholder location="config.properties" /> instead of declaring PropertySourcesPlaceholderConfigurer bean manually
<aop:aspectj-autoproxy /> Enables AOP (5+ beans)
<tx:annotation-driven /> Enables Transactions (15+ beans)
```
* XML Schema files can have version specified (```spring-beans-4.2.xsd```) or not (```spring-beans.xsd```)
  * If version not specified, most recent is used
  * Usually, no version number is preferred as it means easier upgrade to newer framework version
  ```xml
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:p="http://www.springframework.org/schema/p"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd">
        <!-- bean configs here -->
  </beans>
  ```


#### c-Namespace
* Introduced in Spring 3.1
* Lets you declare constructor args inline
* the c-namespace is not defined in an XSD file, but you do have to include its namespace
* if the injected dependency is another bean, you must suffix the attribute with **-ref**
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:c="http://www.springframework.org/schema/c"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="bar" class="x.y.Bar"/>
    <bean id="baz" class="x.y.Baz"/>

    <!-- traditional declaration -->
    <bean id="foo" class="x.y.Foo">
        <constructor-arg ref="bar"/>
        <constructor-arg ref="baz"/>
        <constructor-arg value="foo@bar.com"/>
    </bean>

    <!-- c-namespace declaration -->
    <bean id="foo" class="x.y.Foo" c:bar-ref="bar" c:baz-ref="baz" c:email="foo@bar.com"/>

</beans>
```


#### p-Namespace
* alternative to the property element. Allows use if inline attributes in XML config.
*  the p-namespace is not defined in an XSD file and exists only in the core of Spring, you do have to declare its namespace
* The **-ref** postfix for the "p:" attributes has the role of telling the container that the value of the attribute 
is a reference to another bean.
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="john-classic" class="com.example.Person">
        <property name="name" value="John Doe"/>
        <property name="spouse" ref="jane"/>
    </bean>

    <!-- uses -ref attribute to set 'spouse' property to jane bean -->
    <bean name="john-modern"
        class="com.example.Person"
        p:name="John Doe"
        p:spouse-ref="jane"/>

    <bean name="jane" class="com.example.Person">
        <property name="name" value="Jane Doe"/>
    </bean>
</beans>
```

#### Using Autowire in XML config
* specified om the XML bean element as an attribute
   * ```<bean autowire="byType" ...>```
* Four Autowire modes in XML config:
  * **no** - (default) - no autowiring, bean references must be defined explicitly via *ref*
  * **byName** - autowire by property name
  * **byType** - allows a property to be autowired if **exactly one** bean of the property type exists in the container,else
  a fatal exception is thrown. If there are no beans, the property is not set.
  * **constructor** - analogous to byType but applies to constructor arguments. If there is not exactly one bean of the
  constructor argument type, a fatal error is raised.
* To exclude a bean from autowiring use *autowire-candidate* attribute
  * ```<bean autowire-candidate="false" ...>```
  * *autowire-candidate* attribute is designed to only affect type-based autowiring, name base autowiring can still happen

### PropertySourcesPlaceholderConfigurer
* Enables you to load in properties file(s) and use them in your XML config via SpEL ${} or in your JavaConfig 
(using **@PropertySource** and **@Value**)

* example XML config using default bean namespace (you have to declare a PropertyPlaceholderConfigurer
bean manually)
```xml
<beans xmlns="http://www.springframework.org/schema/beans" 
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans.xsd">

   <!-- reading the properties, a Spring infrastructure bean is exposed -->
   <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
     <property name="locations" value="classpath:db/datasource.properties"/>
   </bean>

   <!-- here the values are injected into a datasource -->
   <bean id="dataSource1" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="${driverClassName}"/>
    <property name="url" value="${url}"/>
    <property name="username" value="${username}"/>
    <property name="password" value="${password}"/>
   </bean>
</beans>
```
* example using the convenience ```<context:property-placeholder >``` namespace (you don't have to declare the
PropertyPlaceholderConfigurer Bean)
```xml
<!-- Spring infrastructure is not visible anymore -->
<context:property-placeholder location="classpath:db/datasource.properties" />

<!-- here the values are injected into a datasource -->
<bean id="dataSource2" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
 <property name="driverClassName" value="${driverClassName}"/>
 <property name="url" value="${url}"/>
 <property name="username" value="${username}"/>
 <property name="password" value="${password}"/>
</bean>
```
java config example using PropertySourcesPlaceholderConfigurer
```java
@Configuration
@PropertySource("classpath:db/datasource.properties")
public class DataSourceConfig {
    @Value("${driverClassName}")
    private String driverClassName;
    @Value("${url}")
    private String url;
    @Value("${username}")
    private String username;
    @Value("${password}")
    private String password;
    
    @Bean
    public DataSource dataSource() throws SQLException {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName(driverClassName);
        ds.setUrl(url);
        ds.setUsername(username);
        ds.setPassword(password);
        return ds;
    }
    
    // needed to resolve the properties injected with @Value
    @Bean
    public static PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```
## Bean Definition Inheritance
* Bean definitions can be inherited and enriched with extra details
  * Reduces xml file size
  * promotes common configuration re-use
  * Separates common configuration elements into a "template" bean definition that can be inherited by child beans 

### Child Beans
* Inherits configuration data from parent bean
* Can also add own configuration data    
* Uses the ```parent="beanIdHere"``` attribute to indicate the parent bean
* Can inherit and/or override the following:
  * bean scope
  * constructor arg values
  * property values
  * method overrides
  * init / destroy /factory methods
  * bean class 
    * Can override parent bean class, eg. class="someBeanType" 
      * **they do NOT have to have same type as the parent bean** BUT
      * child bean must be *compatible* with parent bean. It must accept parent bean's property values
* Settings always taken from child bean definition
  * *depends on*
  * *autowire mode*
  * *dependency check*
  * *singleton*
  * *lazy init*

### Abstract Beans  
* Use the ```abstract="true"``` attribute to declare a bean as abstract
* Useful to indicate a beans configuration data can be inherited
* If a parent bean definition **does not define a class**, explicitly marking the bean as abstract **is required**
  * "abstract only" beans cannot be instantiated by Spring IoC
    * It is an error to refer to them in a ref attribute or use getBean()
    * Can only be used as a template definition for child beans
* If an abstract bean **defines a class, and is Singleton scope**, must make sure to set ```abstract="true"``` otherwise
ApplicationContext will attempt to pre-instantiate the abstract bean 
    
### Bean Inheritance Example
```xml
<beans ...>
  <!-- abstract bean definition with two properties -->
  <bean id="abstractDataSource" class="oracle.jdbc.pool.OracleDataSource" abstract="true">
    <property name="user" value="admin"/>
    <property name="loginTimeout" value="300"/>
  </bean>
  
  <!-- two bean definitions that add a URL property as well as inherit abstractDataSource properties -->
  <bean id="dataSource-1" parent="abstractDataSource">
    <property name="URL" value="jdbc:oracle:thin:@192.168.1.164:1521:PET"/>
  </bean>
  <bean id="dataSource-2" parent="abstractDataSource">
    <property name="URL" value="jdbc:oracle:thin:@192.168.1.164:1521:PET"/>
  </bean>
  
  <~-- bean that inherits from abstractDataSource and overrides loginTimeout AND class  -->
  <bean id="dataSource3" parent="abstractDataSource" class="com.ps.CustomizedOracleDataSource">
    <property name="URL" value="jdbc:oracle:thin:@192.168.1.164:1521:PET"/>
    <property name="loginTimeout" value="100"/>
  </bean>
</beans>
```

Spring Expression Language (SpEL)
==================================================================================================================
Inspired by WebFlow EL, a superset of Unified EL, SpEL is a powerful expression language that supports querying 
and manipulating an object graph at runtime. SpEL provides:
* can access Spring managed beans by ID
* invoking methods and access to properties on objects
* access to indexed collections
* collection filtering / manipulation
* regular expression matching
* boolean and relational operators
* can be used in @Value annotation values
* SpEL expressions are enclosed in #{}
* ${} is for accessing properties, called a placeholder value
  * In order to use placeholder values, you must configure a PropertySourcesPlaceholderConfigurer bean
  * To Access property of a **bean**, use #{beanName.property}
* Can reference systemProperties and systemEnvironment
  * ```#{systemProperties['user.name']}```
  * ```#{systemEnvironment['JAVA_HOME']}```
* Used in other spring projects such as Security, Integration, Batch, WebFlow,...

## SpEL Operators
* Arithmetic
  * ```+,-,*,/,%,^```
* Comparison
  * ```<,lt,>,gt,==,eq,<=,le,>=,ge```
* Logical
  * ```and,or,not,|```
* Conditional
  * ```?: (ternary), ?: (Elvis)```
* Regular Expression
  * ```matches```


## SpEL examples
* ```#{T(System).currentTimeMillis()}``` - get the current system time in milliseconds
* ```#{sgtPeppers.artist}``` - get the artist property on the bean with ID *sgtPeppers*
* ```#{3.14159}``` - returns a floating point value of: ```3.14159```
* ```#{9.87E4}``` - evaluates to ```98,700```
* ```#{'Hello'}``` - evaluates to the String ```Hello```
* ```#{artistSelector.selectArtist()}``` - calls the selectArtist() method of the bean with ID *artistSelector*
* ```#{artistSelector.selectArtist()?.toUpperCase()}``` - uses the null check operator (?.) to provide null safety
  * the expression will evaluate to null if ```selectArtist()``` returns null
* ```T(java.lang.Math).PI``` - uses the type operator ```T()``` to access the java ```Math``` class
* ```#{2 * T(java.lang.Math).PI * circle.radius}``` - compute the circumference of a circle
* ```#{disc.title ?: 'Rattle and Hum'}``` - uses elvis operator to perform a safe check for null
* ```#{admin.email matches '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.com'}```
  * uses a regular expression to match an email address. Returns ```true``` if it matches else ```false```

### Evaluation Collections
* ```#{jukebox.songs[4].title}``` - return the fourth song title
* ```#{'This is a test'[3]}``` - returns ```s```
* ```#{jukebox.songs.?[artist eq 'Aerosmith']}```
  * filters the ```jukebox.songs``` collection where ```song.artist``` equals 'Aerosmith' into a new collection
  of ```song``` objects
* ```#{jukebox.songs.^[artist eq 'Aerosmith']}```
  * ```.^[]``` returns the first matching entry
* ```#{jukebox.songs.$[artist eq 'Aerosmith']}```
  * ```.$[]``` returns the last matching entry
* ```#{jukebox.songs.?[artist eq 'Aerosmith'].![title]}```
  * ```.![]``` projection operator
  * filters all ```songs.artist == 'Aerosmith'``` into a collection then projects that collection into a
  collection of ```String``` containing only the ```song.title```



Spring Application Lifecycle
====================================================================================================================
Three main phases:
* Initialization
  * Bean definitions are read, beans are created, dependencies are injected, resources are allocated 
  (aka bootstrap phase). After this phase is complete the application can be used
* Use
  * The application is up and running. Beans are retrieved and used to do work. This is main phase of the
  lifecycle and covers 99% of it
* Destruction
  * The context is being shut down, resources are released, beans are handed over to garbage collector

Lifecycle independent of configuration style used (Java/XML)

## Initialization Phase
### Bean Initialization Process
1. Bean *definitions* are loaded
2. Bean *definitions* are processed
    * processed by beans called **bean definition post processors**. They are automatically picked up by the App.Context,
  created, and applied before any other beans are created. These beans implement the 
  `org.springframework.bean.factory.config.BeanFactoryPostProcessor` interface.
    * **BeanFactoryPostProcessor** contains a single method that must be implemented `postProcessBeanFactory(BeanFactory)`
  Developers can then implement this interface in their own beans, add it to their configuration, and the App.Context will
  make sure to invoke them.
    * BeanFactoryPostProcessor operates on the **bean configuration metadata**; that is, the Spring IoC container allows a 
  BeanFactoryPostProcessor to read the configuration metadata and potentially change it before the container 
  instantiates any beans other than BeanFactoryPostProcessors
    * Special consideration must be taken for `@Bean` methods that return Spring BeanFactoryPostProcessor (BFPP) types. 
  Because BFPP objects must be instantiated very early in the container lifecycle, they can interfere with processing 
  of annotations such as @Autowired, @Value, and @PostConstruct within @Configuration classes. To avoid these 
  lifecycle issues, mark BFPP-returning `@Bean` methods as **static**
    * Example: PropertySourcesPlaceholderConfigurer is a BeanFactoryPostProcessor that resolves placeholders like ${propName} 
3. For each bean definition:
    1. instantiate bean (basically calls bean's constructor)
    2. inject dependencies (calls setters)
    3. Pre-Initialize - run BeanPostProcessor before initialization
        * ```public Object postProcessBeforeInitialization(Object bean,String beanName);```
    4. run initializers
        1. @PostConstruct - is run after dependencies are injected
            * method must be:  **void [any access right] and take no arguments**
        2. afterPropertiesSet() as defined by the ```InitializingBean``` callback interface
        3. init-method="..." in XML config OR @Bean(initMethod="...") in java config
            * method must be:  **void [any access right] and take no arguments**
    5. Post-Initialize - BeanPostProcessor beans are invoked after initialization
        * ```public Object postProcessAfterInitialization(Object bean, String beanName);```

#### Instantiating Beans
* Spring **creates beans eagerly by default**, unless declared as @Lazy (or lazy-init in XML)
  * **@Lazy**
    * bean is instantiated the first time it is used
    * can be used at injection points marked with @Autowired or @Inject
    * Can be placed on @Bean Definitions, at the class level, at field level
    * If a bean is lazy by default, you can explicitly make it eager by using ```@Lazy(false)```
    * useful when the dependency is a huge object and you don't want to keep it in memory until it is actually needed
* Beans are created in the right order (in order to satisfy dependencies)

#### BeanPostProcessors
* after a bean is instantiated it may be post-processed (similar to bean definition post processing)
* One kind of post processor are **initializers** - @PostConstruct (JSR-250) OR ```init-method=``` attr. (XML config)  OR 
```@Bean(initMethod="...")``` (java config)
  * The **@PostConstruct** (added in Spring 2.5) annotation is used on a method that needs to be executed after dependency injection 
  is done to perform initialization. The annotated method must be invoked before the bean is used, and, like any 
  other initialization method chosen, may be called only once during a bean lifecycle. If there are no dependencies 
  to be injected, the annotated method will be called after the bean is instantiated. Only one method should be 
  annotated with @PostConstruct.
* Other post processors can be invoked either before initialization or after initialization
  * The BeanPostProcessor interface defines callback methods that you can implement to provide your own 
  (or override the container’s default) instantiation logic, dependency-resolution logic, and so forth. If you want 
  to implement some custom logic after the Spring container finishes instantiating, configuring, and initializing a bean, 
  you can plug in one or more BeanPostProcessor implementations.
  * To create custom bean post processor, implement interface `BeanPostProcessor`, and add the bean to your config
    ```java
    public interface BeanPostProcessor {
        Object postProcessAfterInitialization(Object bean, String beanName); 
        Object postProcessBeforeInitialization(Object bean,String beanName);
    }
    ```
    * Note that return value is the post-processed bean. It may be the bean with altered state, however, it may be
    completely new object. It is very important, **post processor can return proxy of bean instead of original one**.
    Used a lot in spring - AOP, Transactions, ... A proxy can add dynamic behavior such as security, logging or
    transactions without its consumers knowing.

##### Some Common Spring BeanPostProcessors
* ```CommonAnnotationBeanPostProcessor```
  * Post processes JSR-250 annotated beans
  * Registered automatically is you use "context:annotation-config" or "context:component-scan"
* ```AutowiredAnnotationBeanPostProcessor```
  * Post processes @Autowired,@Value, and JSR-330 @Inject, annotated beans
  * Registered automatically is you use "context:annotation-config" or "context:component-scan"
* ...

#### Enabling JSR-250 annotations (like @PostConstruct or @PreDestroy)
* The ```<context:annotation-config />``` enables scanning of all the classes in the project for annotations,
so using it on large applications might make them slow. The solution is to use ```<context:component-scan />```,
because the base-package attribute will reduce the number of classes to be scanned.
* The ```<context:annotation-config />``` activates detection for @PostConstruct, @PreDestroy, @Resource,
@Autowired, and @Required and other JPA and EJB 3 annotations

* The ```<context:component-scan base-package="{package names}"/>``` activates detection for all annotations mentioned 
previously, plus Spring stereotype annotations: @Component and extensions (e.g., @Service, @Repository).

XML-Config component scan example:
```xml
<beans .../>
<context:component-scan base-package="com.ps.sample, com.ps.repos"/>
``` 



#### Additional bean config
* @PostConstruct is equivalent to *init-method=* attribute
* @PreDestroy is equivalent to *destroy-method=* attribute
* @Scope is equivalent to *scope=* attribute, singleton if not specified
* @Lazy is equivalent to *lazy-init=* attribute
* @Required
  * indicates that the affected bean property must be populated at configuration time
  * Application won't start if the required property can't be found
  * used on setters
  * Replaced by ```@Autowired``` which are required by default
* @Profile is equivalent to *profile=* attribute on <beans> tag, <beans> tags can be nested
```xml
<beans profile="barProfile">
    <beans profile="fooProfile">
        <bean id="fooBeanName" class="com.example.foo.Foo" scope="prototype" />
    </beans>
    <bean id="barBeanName" class="com.example.bar.Bar" lazy-init="true" init-method="setup" 
          destroy-method="teardown" />
</beans>
```

## Destruction Phase
* Application shuts down - use ```ctx.registerShutdownHook()``` on non-webapps to close Application Context gracefully
* resources are released
* happens when application context closes
* ```@PreDestroy``` (JSR-250) and *destroy-method=* methods are called
  * * method must be:  **void [any access right] and take no arguments**
  * in xml config use **destroy-method** attribute of ```<bean />```
  * The equivalent of destroy-method for Java Configuration ```@Bean(destroyMethod="...")```
  * the destroy method may be called only once during the bean lifecycle
* garbage collector still responsible for collecting objects

## Multiple lifecycle mechanisms configured for the same bean, with different initialization methods
* Bean initialization methods are called in the following order:
  1. Methods annotated with @PostConstruct
  2. afterPropertiesSet() as defined by the InitializingBean callback interface
  3. A custom configured init() method e.g. @Bean(initMethod="someInit")
* Bean Destruction methods called in the following order:
  1. Methods annotated with @PreDestroy
  2. destroy() as defined by the DisposableBean callback interface
  3. A custom configured destroy() method  e.g. @Bean(destroyMethod="someDestroy")

## Bean Scopes
* Singleton - one instance of bean created for entire application (default)
* Prototype - One instance of bean is created every-time the bean is injected into or retrieved from the 
Spring Application Context.
  * e.g. each time the prototype scoped bean is injected into another bean, or requested through *getBean()* 
  * In the case of prototypes, configured destruction lifecycle callbacks are not called (@PreDestroy).
* [A]Global-session - One global session shared among all portlets - Only in Web Portlet Environment
* Session - In a web application, one instance of bean is created for each session
* Request - In a web application, one instance of bean is created for each request
* Application - In a web application, one instance of bean is created per servlet context
* WebSocket - Scopes a single bean definition to the lifecycle of a WebSocket. Only valid in the context of a web-aware
 Spring ApplicationContext.
* *Custom* - Developers are provided the possibility to define their own scopes with their
own rules.
```java
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) //or @Scope("prototype")
public Notepad notepad() {
    return new Notepad();
}
```


### Spring Proxies
* Two types of proxies:
  * JDK Dynamic Proxies
    * proxied bean must implement at least one java interface
    * part of JDK
    * all interfaces implemented by the class are proxied
    * all collaborators into which the scoped bean is injected must reference the bean through one of its interfaces
    in order to be proxied
    * preferred proxying mechanism if you have a choice
    * based on proxy implementing interfaces
    * To **force** JDK Dynamic Proxy, in Java Config use ScopedProxyMode.INTERFACES: 
    ```@Scope( value=WebApplicationContext.SCOPE_SESSION, proxyMode=ScopedProxyMode.INTERFACES)```
    * In XML config use aop:scoped-proxy with ```proxy-target-class="false"```
    ```xml 
        <bean id="cart" class="com.myapp.ShoppingCart" scope="session">
          <aop:scoped-proxy proxy-target-class="false" />
        </bean>
    ``` 
  * CGLib proxies
    * not part of JDK, is included with Spring
    * used when class implements no interface, but a Concrete class
    * Cannot be applied to final classes or methods
    * based on proxy inheriting the base class
    * To **force** CGLIB, in Java Config use: 
    ```@Scope(value=WebApplicationContext.SCOPE_SESSION, proxyMode=ScopedProxyMode.TARGET_CLASS)```
    * In xml config use ```<aop:scoped-proxy />```  
      * ```<bean id="cart" class="com.myapp.ShoppingCart" scope="session">
             <aop:scoped-proxy /> <!-- TARGET_CLASS is the default -->
           </bean>
        ```
        
#### ScopedProxyMode
The problem: What to do if you need to inject a prototype,session,request scoped bean into
a singleton scoped bean? Or when injecting a shorter-lived scoped bean into a longer-lived scoped bean? Seen a lot in
webapps when a session/request scoped been is injected into another bean but the session/request does not exist yet.
```java
// The proxyMode attribute is necessary because at the moment of the instantiation of the web application context, 
// there is no active request. Spring will create a proxy to be injected as a dependency, and instantiate the target 
// bean when it is needed in a request.
@Bean
@Scope( value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS )
public HelloMessageGenerator requestMessage() {
    return new HelloMessageGenerator();
}
```
* in addition to ```ScopedProxyMode.TARGET_CLASS``` and ```ScopedProxyMode.INTERFACES``` there is also:
  * ```ScopedProxyMode.NO``` - DEFAULT - do not create a scoped proxy
  * ```ScopedProxyMode.DEFAULT``` - typically equals NO, unless different default was configured at component-scan level
* If using xml config: 
    * ```<aop:scoped-proxy />``` (**uses CGLIB by default**)  OR
    * ```<aop:scoped-proxy proxy-target-class="false"/>``` (uses JDK Dynamic Proxy)
    * NOTE: CGLIB proxies only intercept public method calls! Do not call non-public methods on such a proxy; they
    will not be delegated to the actual scoped target object
    * If using aop:scoped-proxy against a prototype scoped bean, **every method call on the shared proxy will lead to the
    creation of a new target instance which the call will then be forwarded to**
    ```xml
    <beans ...>
        <!-- an HTTP Session-scoped bean exposed as a proxy -->
        <bean id="userPreferences" class="com.foo.UserPreferences" scope="session">
            <!-- instructs the container to proxy the surrounding bean -->
            <aop:scoped-proxy/>
        </bean>
    
        <!-- a singleton-scoped bean injected with a proxy to the above bean -->
        <bean id="userService" class="com.foo.SimpleUserService">
            <!-- a reference to the proxied userPreferences bean -->
            <property name="userPreferences" ref="userPreferences"/>
        </bean>
    </beans>
    ```

#### Method Injection
* If you need a new instance of a prototype scoped bean (or some other shorter lifetime scoped bean) repeatedly
at runtime, use **Method Injection**
* Option 1- make your bean aware of the container, (NOT RECOMMENDED due to coupling)
    * implement *ApplicationContextAware*, and manually call *getBean()* each time to create your prototype bean
* Option 2 - **Method Injection** -
  * Lookup method injection is the ability of the container to override methods on container managed beans,
  to return the lookup result for another named bean in the container
  * uses CGLIB proxying to dynamically override the method that creates your prototype scoped bean
  * the method to be injected must have the following signature:
    * ```<public|protected> [abstract] <return-type> theMethodName(no-arguments);```
  * in xml config use <lookup-method ...>
  ```xml
  <!-- a stateful bean deployed as a prototype (non-singleton) -->
  <bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
      <!-- inject dependencies here as required -->
  </bean>

  <!-- commandProcessor uses statefulCommandHelper -->
  <!-- The bean identified as commandManager calls its own method createCommand()
       whenever it needs a new instance of the myCommand bean -->
  <bean id="commandManager" class="fiona.apple.CommandManager">
      <lookup-method name="createCommand" bean="myCommand"/>
  </bean>
  ```
  * in Java config use @Lookup annotation
  ```java
  public abstract class CommandManager {

      public Object process(Object commandState) {
          MyCommand command = createCommand();
          command.setState(commandState);
          return command.execute();
      }

      // the target bean gets resolved against the declared return type of the lookup method
      @Lookup
      protected abstract MyCommand createCommand();
  }
  ```
  * NOTE: you must make sure the bean being created (MyCommand in above example) is of scope *prototype*. If it
  is singleton scope, it will only get created once
  * NOTE: Component scanning filters out abstract beans. You must provide at least one stub/implementation of
  a method if your lookup method if it is in an Abstract class
  * NOTE: **lookup methods won't work on beans returned from @Bean methods in configuration classes**;
  you'll have to resort to ```@Inject Provider<TargetBean>```
* Option 3 - use ```aop:scoped-proxy``` or (eg. ```@Scope( proxyMode = ScopedProxyMode.TARGET_CLASS)``` )
     
           
Spring Configuration, Properties, Profiles
====================================================================================================================
* can be XML or Java based
* externalized from the bean class --> better separation of concerns

## Externalizing Configuration Properties
* Configuration values (DB connections, external endpoints) should not be hardcoded in the configuration files
* better to externalize to a properties file
* can then be easily changed without having to rebuild the application
* can have different values for different environments
* Spring provides an abstraction on top of many property sources:
  * JNDI
  * Java .properties files
  * JVM System properties
  * System environment variables
  * Servlet context parameters


### Spring Profiles
* beans can have an associated profile (e.g. "local","qa","mock")
* Use @Profile("profileName") either with the @Bean level or
* @Component class level (including @Configuration)
  * will apply to all beans in the config
* Beans with no profile are always active
* Beans with a specific profile are only loaded when the given profile(s) are active
* multiple profiles can be active
* Can also use "!" to NOT a profile:
  * @Profile({"p1", "!p2"}) - this profile is active only if p1 is active OR if p2 is NOT active

#### Activating Profiles
* Active profiles can be set in several different ways
  * @ActiveProfiles in spring integration tests
  * in web.xml deployment descriptor as context param ("spring.profiles.active")
    ```xml
    <context-param>
        <param-name>spring.profiles.active</param-name>
        <param-value>profile1,profile2</param-value> 
    </context-param>
    ```
  * in a Web Application using Java config:
    * set via the ```servletContext.setInitParamter(...)``` in the ```onStartup( )``` method
    ```java
        @Override
        public void onStartup( ServletContext servletContext ) throws ServletException {
            servletContext.setInitParameter( "spring.profiles.active","profile1, profile2" );
        }
    ```
  * By setting "spring.profiles.active" system property  ```-Dspring.profiles.active="profile1,profile2"```
    * Active profiles are comma-separated
  * programmatically via the Environment object:
    ```java
    //setting active profile programmatically
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.getEnvironment().setActiveProfiles("development");
    ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
    ctx.refresh();
    ```
* profiles can be used to load different property files under different profiles
    ```java
    @Configuration
    @Profile("local")
    @PropertySource("classpath:/com/example/myapp/config/application-local.properties")
    public class LocalConfig {
        //@PropertySource is loaded only when "local" profile is active

        //Config here
    }
    ```


##### Default Profile
* If an active profile is **NOT** set, spring looks to the **default** profile
  * the name of the default profile is *default*
* If neither an active profile or default profile is set, then only beans that are NOT in a profile are created
* Can set the name of the default profile (only takes effect if there is no active profile set)
  * Can be set with ```spring.profiles.default``` similar to active profile
  * Can also be set thru the Environment object ```ctx.getEnvironment().setDefaultProfiles("default1","default2")```




### Obtaining properties using Springs Environment object
* The **Environment** is an abstraction integrated in the container that models two key aspects of the application 
environment: profiles and properties.
  * The role of the Environment object with relation to profiles is in determining which profiles (if any) are 
  currently active, and which profiles (if any) should be active by default.
  * The role of the Environment object with relation to properties is to provide the user with a convenient service 
  interface for configuring property sources and resolving properties from them.
* **Environment** can be injected using @Autowired
* properties are obtained using environment.getProperty("propertyName")
```java
@Configuration
//declare a property source to load properties into the Spring Environment 
@PropertySource("classpath:/com/soundsystem/app.properties")
public class ExpressiveConfig {
 @Autowired
 Environment env;
 
 @Bean
 public BlankDisc disc() {
  return new BlankDisc(
  env.getProperty("disc.title"),    //get the properties from the environment
  env.getProperty("disc.artist"));
 } 
}
```

### Obtaining properties using @Value
* @Value("${propertyName}") can be used as alternative to Environment object
* can be used on fields and method parameters
* need to configure a property source that will load in the properties (see below)

#### Property Sources
* Spring loads system property sources automatically (JNDI, system variables, ...)
* Additional property sources can be specified in configuration using @PropertySource annotation on @Configuration class
* In order to use placeholder values e.g.```${}``` you must configure a ```PropertySourcesPlaceholderConfigurer```
 as a **static** bean
  * ```PropertySourcesPlaceholderConfigurer``` resolves ${...} placeholders within bean definition property
  values and ```@Value``` annotations against the current Spring Environment and its set of PropertySources
```java
@Configuration
@PropertySource("classpath:/com/example/myapp/config/application.properties")
public class ApplicationConfiguration {
    @Bean
    public static PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }    
    //Additional config here
}
```
```xml
<beans ...>
  <context:property-placeholder location="classpath:db/datasource.properties" />
</beans>
```
* can use property placeholders in @PropertySource names (e.g. different property based on environment, dev/staging/prod)
  * placeholders are resolved against known properties
  ```@PropertySource("classpath:/com/example/myapp/config/application-${ENV}.properties")```
* if a property key exists in more than one property file, the last @PropertySource annotation will win





Java Configuration
====================================================================================================================
* done in java classes with ```@Configuration``` on the class level
* beans can either be specifically declared in ```@Configuration``` using ```@Bean``` annotation or automatically 
discovered using component scan
* **Annotation injection is performed before XML injection, thus the latter configuration will override the former 
for properties wired through both approaches.**

## Full @Configuration vs 'lite' @Beans mode
* When @Bean methods are declared in classes **not** annotated with @Configuration, they are being processed in **lite mode**
  * e.g. bean methods declared in a @Component or even a POJO class will be considered 'lite'
* Unlike full @Configuration, lite @Bean methods **cannot** easily declare inter-bean dependencies.
* @Bean methods should **not** invoke another @Bean method when operating in 'lite' mode
* Should only use @Bean methods in @Configuration classes so that 'full' mode is always used.
  * Will prevent the same @Bean method from being invoked multiple times and helps to reduce subtle bugs

## Component Scan
* used with a @Configuration class, @ComponentScan will scan specified packaged (and sub-packages) for @Components
* if no packages specified, scans current package of the @ComponentScan class (and sub-packages)
* All classes with @Component annotation on class level in scanned packages are automatically considered Spring
managed beans
* Also applies to components annotated as: @Service, @Controller, @Repository, @Configuration
* can also be used for scanning jar dependencies
* recommended to declare specific package names in order to speed up scan time
  ```java
  @Configuration
  @ComponentScan( { "org.foo.myapp", "com.bar.anotherapp" })
  public class ApplicationConfiguration {
      //Configuration here
  }
  ```

## @Autowired
* beans declare their dependencies using @Autowired annotation - automatically injected by Spring
  * Field level - even private
  * Method level
  * Constructor level
* @Autowired dependencies are **required by default** and **will throw an exception if no matching bean is found**
  * can be changed to optional using ```@Autowired(required=false)```
* dependencies of type ```Optional<T>``` are automatically considered optional

### Autowire conflicts
* if no bean of given type for @Autowired dependency is found and it is not optional, exception is thrown
* If more than one bean found that satisfies the dependency, ```NoSuchBeanDefinitionException``` is thrown as well
* Can use @Qualifier to inject bean by name. The name is set in @Component annotation ```@Component("myName")```
  ```java
  @Component("myRegularDependency")
  public class MyRegularDependency implements MyDependency {...}

  @Component("myOtherDependency")
  public class MyOtherDependency implements MyDependency {...}

  @Component
  public class MyService {
      @Autowired
      public MyService( @Qualifier("myOtherDependency") MyDependency myDependency ) {...}
  }
  ```


### Autowired resolution sequence
1. Try to inject bean by type, if there is just one
2. Try to inject by @Qualifier, if present
    * After selecting candidate beans by type, the specified String qualifier value will be considered within those
      type-selected candidates only, e.g. matching an "account" qualifier against beans marked with the same
      qualifier label.
3. Try to inject by bean name matching name of property being set
    * bean 'foo' for 'foo' field in field injection
    * bean 'foo' for 'setFoo' setter in setter injection
    * bean 'foo' for constructor param named 'foo' in constructor injection
    * **When a bean name is not specified, one is auto-generated: De-capitalized non-qualified class name**



## @Value annotation
* is used to inject value from either system properties (using ${}) or SpEL (using #{})
* can be on fields, constructor parameters, or setter parameters
  * on constructors and setters @Value must be combined with @Autowired **at the method level**
  ```java
  public class MovieRecommender {
  
      private String defaultLocale;
  
      private CustomerPreferenceDao customerPreferenceDao;
  
      @Autowired
      public MovieRecommender(CustomerPreferenceDao customerPreferenceDao,
              @Value("#{systemProperties['user.country']}") String defaultLocale) {
          this.customerPreferenceDao = customerPreferenceDao;
          this.defaultLocale = defaultLocale;
      } 
      // ...
  }
  ```
  * for fields, @Value alone is all that is needed
* can specify default values
  * ```"${minAmount:100}"```  the : acts like a default separator. if minAmount not present then default to 100
  * ```#{environment['minAmount'] ?: 100}```  ?: is elvis operator. if minAmount is null then default to 100


## Setter vs Constuctor vs Field Injection
* all three types can be used and combined
* usage should be consistent
* constructors are preferred for mandatory, immutable dependencies
* setters are preferred for optional, changeable dependencies
* only way to resolve circular dependencies is using setter injection, constructors cannot be used
* Field injection can be used, but mostly discouraged



## Explicit Bean declaration using @Bean
* A method is explicitly marked (@Bean) as spring managed bean in @Configuration class
* All settings of the bean are present in the @Configuration class in the bean declaration
* Spring config is completely in the @Configuration. Bean is just a POJO with no Spring dependency
* Cleaner separation of concerns
* Only option for third party dependencies where you cannot change the source code
  ```java
  @Configuration
  public class ApplicationConfiguration {
      
      /***
      *  - Object returned by this method will be spring managed bean 
      *  - Return type is the type of the bean
      *  - Method name is the name of the bean
      */
      @Bean
      public MyBean myBean() {
          MyBean myBean = new MyBean();
          //Configure myBean here
          return myBean;
      }
  ```


## Component Scan vs Explicit Bean Declaration
* Same settings can be achieved either way
* Both approaches can be mixed
  * can use component scan for your code, @Configuration for third party and legacy code
  * use component scan only for Spring classes, explicit configuration for others
* Explicit bean declaration
  * Is more verbose
  * Achieves cleaner separation of concerns
  * Config is centralized in one or several places
  * Configuration can contain complex logic
* Component Scan
  * Configuration is scattered across many classes
  * **Cannot** be used for third party classes
  * Code and configuration violate single responsibility principle, bad separation of concerns
  * good for rapid prototyping and frequently changing code

## @PostConstruct, @PreDestroy
* Can be used on @Component methods
* methods can have any access modifier, but **must have no args and return void**
* applies to both beans discovered by component scan and declared by @Bean in @Configuration
* Defined by JSR-250 in javax.annotation package (not Spring)
* also used by EJB3
* alternative in @Configuration is @Bean(initMethod="init", destroyMethod="destroy") - can be used for third party beans

### @PostConstruct
* method is invoked by spring after dependency injection

### @PreDestroy
* method is invoked before bean is destroyed
* is **NOT called for prototype scoped beans**
* application context is closed, then destroy phase will begin
* **only happens if JVM exits normally**

## Stereotypes and Meta-annotations
* Stereotypes
  * Spring has several annotations which are themselves annotated by @Component
    * @Service, @Controller, @Repository, @Configuration  (more in other Spring projects)
  * are discoverable using component scan
  * Stereotype annotations are in package ```org.springframework.stereotype``` there are four:
    * @Component, @Service, @Repository, @Controller
  
* Meta-Annotations
  * used to annotate other annotations
  * e.g. you can create a custom annotation which combines @Service and @Transactional
```java
//creates a Custom Transaction annotation named CustomTx built on top of @Transactional
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional("customTransactionManager", timeout="90")
public @interface CustomTx {
boolean readOnly() default false;
}
```

## Composite Configuration
* Preferred way is to divide configuration into multiple files and then import them when used
* Usually based on application layers - web layer, service layer, DAO layer
* Enables better configuration organization
* Enables you to separate layers, which often change based on environment, etc. - eg switching services for mocks in
unit tests, having an in-memory database
* can import other config files in Java config using @Import annotation
```java
@Configuration
@Import({WebConfiguration.class, InfrastructureConfiguration})
public class ApplicationConfiguration {
    //Config here
}
```
* Beans defined in another @Configuration file can be injected using @Autowired annotation on field or setter level, 
**cannot use constructors here**
* Another way of referencing beans from another @Configuration is to declare them as method parameters of beans defined
in the current config file
```java
@Configuration
@Import({WebConfiguration.class, InfrastructureConfiguration})
public class ApplicationConfiguration {
    
    @Bean //Object returned by this method will be spring managed bean
    public MyBean myBean( BeanFromOtherConfig beanFromOtherConfig ) {
        //beanFromOtherConfig bean is automatically injected by Spring
        ...
        return myBean;
    }
}
```
* In case multiple beans of the same type are found in configuration, **spring injects the bean that was discovered last**





Testing Spring Applications
===================================================================================================================
## Test Types
### Unit Tests
* shouldn't use any spring functionality (like application context)
* Isolated tests for single unit of functionality, method level
* Dependency interactions are not tested - replaced by either mocks or stubs
* Controllers, Services, converters,...

### Integration Tests
* With spring
* Test interaction between components
* Dependencies are **not** mocked/stubbed
* Controllers + Services + DAO + DB

## Spring Test
* Distributed as separate artifact - spring-test.jar
* Allows testing application without the need to deploy to external container
* Allows to reuse most of the spring config in your tests
* Consists of several JUnit support classes
  * ```SpringJUnit4ClassRunner``` - application context is cached among test methods - only one instance of
  app context is created per test class
  * Spring caches application contexts by locations attribute so if the same locations appears for the second time,
  Spring uses the same context rather than creating a new one.
  * supports JUnit 4.12 or higher

### Steps to define a test class that runs in the Spring Context
1. Test class needs to be annotated with ```@RunWith(SpringJUnit4ClassRunner.class)```
2. Spring configuration classes to be loaded are specified in ```@ContextConfiguration``` annotation
      * If no value is provided (@ContextConfiguration), config file **${TestClassname}-context.xml** in the same
      package is imported.
      * XML config files are loaded by providing string value to annotation -
        ```@ContextConfiguration("classpath:com/example/test-config.xml")```
      * Java @Configuration files are loaded from *classes* attribute -
        ```@ContextConfiguration(classes={TestConfig.class,OtherConfig.class})```
      ```java
      @RunWith(SpringJUnit4ClassRunner.class)
      @ContextConfiguration(classes={TestConfig.class, OtherConfig.class})
      public final class FooTest  {
          //Autowire dependencies here as usual
          @Autowired
          private MyService myService;

          //...
      }
      ```
      * @Configuration inner classes (must be static) are automatically detected and loaded into tests
        * **these inner classes will be detected automatically and will cause any classes or xml files declared
        in the @ContextConfiguration annotation to be ignored**
      ```java
      @RunWith(SpringJUnit4ClassRunner.class)
      @ContextConfiguration
      public final class FooTest  {

          @Configuration
          public static class TestConfiguration {...}
      }
      ```
3. use @Autowired to inject the beans to be tested

### Testing by Specifying a Context Loader
* Allows you to declare an inner @Configuration class within your test class and then the beans contained therein
are loaded into your ApplicationContext
  * the inner class must be static and annotated with @Configuration
* Have to specify ```AnnotationConfigContextLoader.class``` as the value to @ContextConfiguration
  * ```@ContextConfiguration( loader=AnnotationConfigContextLoader.class )```

```java
package com.ps.integration;
import org.springframework.test.context.support.AnnotationConfigContextLoader;
//...
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(loader=AnnotationConfigContextLoader.class)
public class SpringPetServiceTest3 {
  public static final Long PET_ID = 1L;
  public static final User owner = buildUser("test@gmail.com", "test", UserType.OWNER);

  @Configuration
  static class TestCtxConfig {
      @Bean
      StubPetRepo petRepo(){
        return new StubPetRepo();
      }

      @Bean
      PetService simplePetService(){
        SimplePetService petService = new SimplePetService();
        petService.setRepo(petRepo());
        return petService;
      }
  }

    @Autowired
    PetService simplePetService;
    @Test
    public void findByOwnerPositive() {
    Set<Pet> result = simplePetService.findAllByOwner(owner);
    assertEquals(result.size(), 2);
    }
}
```


### Testing Transactions
* Default policy is to rollback transactions after every test method
* Can override using @Rollback or @Commit
* **@Rollback**
  * e.g. ```@Rollback(false)```
  * indicates whether the transaction for a transactional test method should be rolled back after the test method completes
  * default is true
  * can be declared at class and/or method level, method level overrides class level
* **@Commit**
  * can be used on class and/or method level, method level overrides class level
  * indicates that the transaction for a transactional test method should be committed after test method completes
  * can be used as a direct replacement for ```@Rollback(false)```
  
### Testing with Spring Profiles
* **@ActiveProfiles** annotation of test class activates the profiles listed
  * ```@ActiveProfiles({"foo","bar"})```

### Test Property Sources
* **@TestPropertySource** overrides any existing property of the same name
* Default location is **[classname].properties**
* ```@TestPropertySource( properties={"username=foo"} )```

### Testing with in-memory DB
* Real DB usually replaced with in-memory DB for integration tests
* no need to install a DB server
* in-memory DB can be initialized with scripts using **@Sql** annotation

### @Sql
* introduced in Spring 4.1
* can be either on class level or method level
* if on class level, SQL is executed before every test method in the class
* if on method level, SQL is executed before invoking that particular method
* can be declared with/without a value
* when value is present, specified SQL script is executed - ```@Sql({"/sql/foo.sql", "/sql/bar.sql"})```
* when no value present:
  * class-level declaration: if the annotated test class is com.example.MyTest, the corresponding
  default script is "classpath:com/example/MyTest.sql"
  * method-level declaration: if the annotated test method is named testMethod() and is defined in
  the class com.example.MyTest, the corresponding default script is "classpath:com/example/**MyTest.testMethod.sql**"
* can specify whether SQL is run before (default) or after test method
* ```@Sql(scripts="/sql/foo.sql", executionPhase=Sql.ExecutionPhase.AFTER_TEST_METHOD)```
* can provide further configuration using ```config``` param - error mode,comment prefix,separator...
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {TestDataConfig.class, AppConfig.class})
@ActiveProfiles("dev")
public class UserServiceTest {
  @Test
  //this Sql runs before the test method by default
  @Sql(value ="classpath:db/extra-data.sql",
       config = @SqlConfig(encoding = "utf-8", separator = ";", commentPrefix = "--"))
  //this Sql runs after Test method
  @Sql(
  scripts = "classpath:db/delete-test-data.sql",
  config = @SqlConfig(transactionMode = SqlConfig.TransactionMode.ISOLATED),
  executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD
  )
  public void testCount(){
    int count = userService.countUsers();
    assertEquals(8, count);
  }
```



Aspect Oriented Programming
==================================================================
* Solves modularization of cross-cutting concerns
* Functionality, which would be scattered all over the application, but is not core functionality of the class
  * Logging
  * Security
  * Transactions
  * Performance Monitoring
  * Caching
  * Error handling
  * ...
* Keeps good separation of concerns, does not violate single responsibility principle
* Solves two main problems:
  * Code Tangling - coupling of different concerns in one class- e.g. business logic coupled with security and logging
  * Code Scattering - same code scattered among multiple classes, code duplication - e.g. same security logic in each class
  
  
### Writing AOP Applications
1. write main application logic
2. write aspects to address cross cutting concerns
3. weave aspects into the application to be applied in the right places

## AOP Technologies
* AspectJ
  * original AOP technology
  * uses bytecode modification for aspect weaving
  * Complex, a lot of features
* Spring AOP
  * AspectJ integration
  * subset of AspectJ functionality
  * Uses dynamic proxies instead of bytecode manipulation for aspect weaving
  * Limitations:
    * For JDK proxies, **only public interface method calls on the proxy can be intercepted**
    * CGLIB can advise **non-private methods only**
      * However, common interactions through proxies should always be designed through **public** signatures
    * Only applicable to spring managed beans
    * AOP is not applied when one method of the class calls method on the same class (will bypass the proxy)

## AOP Concepts
* Join Point - a point in the execution of the app. where AOP code will be applied, - method call, exception thrown,...
* Pointcut - Expression that matches one or more Join Points
* Advice - Code to be executed at each matched Join Point
* Aspect - Module encapsulating Pointcut and Advice
* Weaving - Technique by which aspects are integrated into main code

## Enabling AOP
* In XML - ```<aop:aspectj-autoproxy />```
* In Java Config - **@EnableAspectJAutoProxy** on @Configuration class
* Create **@Aspect** classes, which encapsulate AOP behavior
  * Annotate with **@Aspect**
  * Must be spring managed bean - either explicitly declare in configuration or discover via component scan
    * note that the @Aspect annotation alone is not sufficient for auto-detection in the classpath: For that purpose,
    you need to add a separate @Component annotation
  * Class methods are annotated with annotation defining advice (e,g, before method call, After, if exception), the
  annotation value defines the pointcut
  * ```@Before("execution( void set*(*))" )```
```java
@Aspect
public class MyAspect {
  @Before("execution(void set*(*))") 
  public void beforeSet() {
      //Do something before each setter call
  }
} 
```
## Pointcuts
* Spring AOP uses subset of AspectJ's expression language for defining pointcuts
* Can create composite conditions using ||, && and !
* ```execution( <method pattern> )``` - method must match the pattern provided
* [Modifiers] ReturnType [ClassType] MethodName (Arguments) [throws ExceptionType]
  * ```execution( void com.example.*Service+.set*(..) )```
    * the pointcut above means method call of Services in com.example package starting with "set" and any number 
    of method parameters
    * ```+``` sign after a type means match that type and any subtypes - in the example above match the service class
    or any of Service's subtypes
  * the Modifiers parameter defaults to ```public``` if not specified
* More examples
  * ```execution( void find*(String) )``` - Any methods returning void, starting with "find" and having single String argument
  * ```execution(* find(*) )``` - any method named "find" having exactly one argument of any type
  * ```execution(* find(int, ..) )``` - any method named "find" where its first argument is int (.. means 0 or more)
  * ```execution( @javax.annotation.security.RolesAllowed void find(..) )``` - any method returning void, named "find",
  annotated with the specified annotation
  * ```execution(* com.*.example.*.*(..) )``` - exactly one directory between com and example
  * ```execution(* com..example.*.*(..) )``` - Zero or more directories between com and example
  * ```execution(* *..example.*.*(..) )``` - any sub-package called "Example"

### Pointcut Designators
* Spring supports the following:
  * **execution** - for matching method execution join points
  * **within** - limits matching to methods within certain types
    * ```within(com.xyz.service.*)``` any method within the service package
  * **this** - limits matching to methods where the bean reference (Spring AOP proxy) is an instance of the given type
    * means all join points where ```this instanceof AType``` is true
    * ```this(com.xyz.service.AccountService)``` - any method where the *proxy* implements the AccountService interface
  * **target** - limits matching to methods where the target object (application object being proxied) is an instance
  of the given type
    * means all join points where ```anObject instanceof AType```
    * ```target(com.xyz.service.AccountService)``` any method where the *target object* implements the AccountService interface
    * typically used as a binding form
  * **args** - limits matching to methods where the arguments are instances if the given types
    * ```args(java.io.Serializable)``` any method which takes a single parameter and where the argument passed at 
    runtime is Serializable
    * typically used as a binding form
  * **@target** - limits matching to methods where the class of the executing object has an annotation of the given type  
  * **@args** - limits matching to methods where the runtime type of the actual arguments passed have annotations of
  the given type(s)
  * **@within** - limits matching to join points within types that have the given annotation
  * **@annotation** - limits matching to join points where the method being executed has the given annotation
  * **bean(idOrNameOfBean)** - limit the matching of join points to a particular named Spring bean or to a set of named
  Spring Beans (when using wildcards)
    * ```bean(tradeService)``` any method execution on a Spring bean named tradeService
  
   
## Advice Types
### Before
* Executes before target method invocation
* If advice throws an exception, target is not called
* ```@Before("expression")```

### After Returning
* Executes **after successful target method invocation**
* If advice throws an exception, target is not called
* Return value of target method can be injected to the annotated method using ```returning``` parameter
  * @AfterReturning( value="expression", returning="paramName")
  ```java
  @AfterReturning(value="execution(* service..*.*(..))", returning="retVal")
  public void doAccessCheck(Object retVal) {
     //...
  }
  ```
  
### After Throwing
* Called after target method invocation throws an exception
* Exception object being thrown from target method can be injected into the annotated method using ```throwing``` param
* It will not stop exception from propagating but can throw another exception
* Stopping exception from propagating can be achieved using @Around advice
* @AfterThrowing( value="expression", throwing="paramName" )
```java
    @AfterThrowing(value="execution(* service..*.*(..) )", throwing="exception")
    public void doAccessCheck(Exception exception) {
    //...
    }
```

### After
* Called after target method invocation, doesn't matter if it was successful or there was an exception
* ```@After("expression")```

### Around
* ```ProceedingJoinPoint``` parameter can be injected - same as regular JoinPoint, but has ```proceed()``` method
* By calling proceed() method, target will be invoked, otherwise it will not be
```java
@Aspect
public class Audience {
  @Pointcut("execution(** concert.Performance.perform(..))")
  public void performance() {}
 
  @Around("performance()")
  public Object watchPerformance(ProceedingJoinPoint jp) {
    Object retVal = null;
    try {
      System.out.println("Silencing cell phones");
      System.out.println("Taking seats");
      retVal = jp.proceed();
      System.out.println("CLAP CLAP CLAP!!!");
    } catch (Throwable e) {
      System.out.println("Demanding a refund");
    }
    //The value returned by the around advice will be the return value seen by the caller of the method.
    return retVal;
  }
}
```
### [A] XML AOP Configuration
* Instead of AOP annotations (@Aspect, @Before, @After, ...), pure XML config can be used
* Aspects are just POJOs with no spring annotations or dependencies whatsoever
* aop namespace
```java
<aop:config>
    <aop:aspect ref="myAspect">
        <aop:before pointcut="execution(void set*(*))" method="logChange"/> 
    </aop:aspect>
    
    <bean id="myAspect" class="com.example.MyAspect" />
</aop:config>
```

### [A] Named Pointcuts
* Pointcut expressions can be named and then referenced
* Makes pointcut expressions reusable
* Pointcuts can be externalized to a separate file
* Composite pointcuts can be split into several named pointcuts
```java
//Pointcut is referenced here by its ID
@Before("setters()") 
public void logChange() {
    //...
}

//Method name is pointcut id, it is not executed
@Pointcut("execution(void set*(*))") 
public void setters() {
    //...
}
```

### [A] Context Selecting Pointcuts
* Data from ```JoinPoint``` can be injected as method parameters with type safety
* Otherwise they would need to be obtained from ```JoinPoint``` object with no type safety guaranteed
* ```@Pointcut("execution(void example.Server.start(java.util.Map)) && target(instance) && args(input")```
  * target(instance) - injects instance on which call is performed to "instance" parameter of the method
  * args(input) - injects target method parameters to "input" parameter of the method
  ```java
  public class PointcutContainer {
    //...
    @Pointcut("execution (* com.ps.services.*Service+.update*(..)) && args(id,pass) && target(service)")
    public void serviceUpdate(UserService service, Long id, String pass) {
    }
  }
  ```

Caching
=======================================================================================================================
* A Spring bean's **method results** can be cached
* Cache here means a map of cache-key, cached value
* Just like other services in the Spring Framework, the caching service is an abstraction (not a cache implementation)
and requires the use of an actual storage to store the cache data
* Multiple caches supported
* implemented under the covers using AOP
* Caching support enabled using
  * **@EnableCaching** on @Configuration class level
  * or in XML ```<cache:annotation-driven />```
  * interface based proxying is the default, can change to CGLIB via *proxyTargetClass=true*
* JCache annotation support (JSR-107)
  * available since Spring 4.1
  * enabled by default when you use @EnableCaching or <cache:annotation-driven />
* **NOTE: caching only works for methods that are guaranteed to return the same output (result) for a given
input (or arguments) no matter how many times it is being executed**

### Enabling Caching (Steps)
1. Install a caching implementation (if needed... eg. EhCache)
2. Configure a bean of type ```CacheManager``` to use the implementation
3. Configure one or more ```KeyGenerator``` (if needed)
4. Annotate a Configuration class with ```@EnableCaching``` or XML config with ```<cache:annotation-driven />```
5. Annotate methods you want to cache: eg. @Cacheable or @CachePut ...

### @Cacheable
* marks method as cacheable
* its result will be stored in the cache
* subsequent calls of the method with the same arguments will result in data being fetched from the cache
* by default **the cache key is all method parameters of the annotated method**
  * override this with the *key* attribute or *keyGenerator* attr.
* Can be used on interface methods which will then be inherited
  * **Spring recommends that you only annotate concrete classes (and methods of concrete classes) with the ```@Cache*```
  annotation, as opposed to annotating interfaces**
* requires a non-void return value - Duh!
* @Cacheable attributes:
   * ```value```- name of cache
     * multiple caches supported
       * multiple cache will be checked for the presence of the key, if key not found in any cache, then method gets executed
       and the return value is stored in all the listed caches.
       * If key is found in one cache, but not others, the other caches will get the key put into them even if the
       method does not run
   * ```key``` - key in given cache, allows you to specify how key is generated, can also use SpEL expressions
     ```java
      @Cacheable(cacheNames="books", key="#isbn.rawNumber")
      public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

      @Cacheable(cacheNames="books", key="T(someType).hash(#isbn)")
      public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
      ```
   * ```keyGenerator``` - takes a bean that implements the KeyGenerator interface
     ```java
     @Cacheable(cacheNames="books", keyGenerator="myKeyGenerator")
     public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
     ```
   * ```condition```  uses SpEL predicate, determines if a method should be cached
     * only cached if name lenghth is < 32
     ```java
     @Cacheable(cacheNames="book", condition="#name.length() < 32")
     public Book findBook(String name)
     ```
   * ```unless``` uses SpEL predicate, if it evaluates to true the method will NOT be cached
     * NOTE: unless only prevents an object from being placed in the cache. The cache is still searched and if a
     value is found, it is returned.
   * ```sync``` (added in 4.3) - allows concurrent calls on a given key to be synchronized so that the value is only
   computed once. Need to make sure your caching provider supports it
    ```java
    @Cacheable(value="mycache", key="#personName.toUpperCase()", condition="#personName.length < 50" sync="true")
    public Person getPerson(String personName) {
        //...
    }
    ```
* ```@CacheEvict(value="cacheName")```
  * on method level - clears cache before annotated method is called
  * void methods **CAN** be used with CacheEvict
  * *beforeInvocation* - default is false
    * allows you to specify if the eviction should occur before or after the method invocation
    * NOTE: "after" method invocation cache eviction may not happen if the method does not return normally
    * beforeInvocation=true - eviction always occurs
  * all you to specify if all entries should be cleared, or you can specify a *conditional*
  ```java
  @CacheEvict( cacheNames="books", allEntries=true, beforeInvocation=true )
  public void loadBooks(InputStream batch)
  ```
* ```@CachePut```
  * on method level, or class level
  * the annotated method will always execute and store the return result in the cache. Unlike @Cacheable which checks
  the cache first in order to decide if it should execute the method.
  * requires a non-void return value
* ```@Caching```
  * regroups multiple cache operations to be applied on a method
* ```@CacheConfig```
  * a class-level annotation that allows to share the cache names, the custom KeyGenerator, the custom
  CacheManager and finally the custom CacheResolver
  * When this annotation is present on a given class, it provides a set of default settings for any cache
  operation defined in that class
  * Placing this annotation on the class does not turn on any caching operation.
  ```java
  @CacheConfig("books")
  public class BookRepositoryImpl implements BookRepository {

      @Cacheable
      public Book findBook(ISBN isbn) {...}
  }
  ```

Or you can define caching in XML:
```xml
<bean id="myService" class="com.example.MyService"/>

<aop:config>
    <aop:advisor advice-ref="cacheAdvice" pointcut="execution(* *..MyService.*(..))"/>
</aop:config>

<cache:advice id="cacheAdvice" cache-manager="cacheManager">
    <cache:caching cache="myCache">
        <cache:cacheable method="myMethod" key="#id"/>
        <cache:cache-evict method="fetchData" all-entries="true" />
    </cache:caching>
</cache:advice>
```

### Cache Manager
* controls and managed caches
* Cache manager must be specified
  * ```SimpleCacheManager```
    * you explicitly define the ```org.springframework.cache.Cache``` implementations to use
      * typically you will use ```ConcurrentMapCache``` but there are others
        * Spring's ```ConcurrentMapCache``` uses Java's ```ConcurrentHashMap``` as a backing Cache store
        * null values can not be stored in ConcurrentHashMap, they are replaced with a predefined internal object
    * allows more than one cache to be defined
  * ```EHCacheCacheManager```
    * requires EHCache 2.5 or higher  http://www.ehcache.org
  * ```CompositeCacheManager```
    * lets you define a list of ```CacheManager```
    * spring will automatically iterate over them to find your cached value
  * ```ConcurrentMapCacheManager```
    * lazily builds ```ConcurrentMapCache``` instances
    * useful for testing or simple caching scenarios
  * Gemfire
  * Redis
  * GuavaCacheManager
  * JCacheCacheMananger - allows you to use JSR-107 compliant caches
  * CaffeineCacheManager
  * custom
  * ...

#### SimpleCacheManager
```java
@Bean
public CacheManager cacheManager() {
    SimpleCacheManager manager = new SimpleCacheManager();
    Set<Cache> caches = new HashSet<Cache>();
    caches.add(new ConcurrentMapCache("fooCache"));
    caches.add(new ConcurrentMapCache("barCache"));
    manager.setCaches(caches);

    return manager;
}
```

#### EHCache
1. configure a EhCacheManagerFactoryBean
2. configure a Spring CacheManager bean using EhCacheCacheManager
```java
// the CacheManager ehCache parameter is an instance of net.sf.ehcache.CacheManager
// EhCacheCacheManager implements Spring's CacheManager interface
@Bean
public CacheManager cacheManager(CacheManager ehCache) {
    EhCacheCacheManager manager = new EhCacheCacheManager();
    manager.setCacheManager(ehCache);
    return manager;
}

//creates the ehCache CacheManager used in the cacheManager method above
@Bean
public EhCacheManagerFactoryBean ehCacheManagerFactoryBean(String configLocation) {
    EhCacheManagerFactoryBean factory = new EhCacheManagerFactoryBean();
    //load the ehcache xml configuration file
    factory.setConfigLocation(context.getResource(configLocation));
    return factory;
}
```

#### Gemfire
* Distributed cache replicated across multiple nodes
* GemfireCacheManager
* Supports transaction management - GemfireTransactionManager
```xml
<gfe:cache-manager p:cache-ref="gemfire-cache"/>
<gfe:cache id="gemfire-cache"/>
<gfe:replicated-region id="foo" p:cache-ref="gemfire-cache"/>
<gfe:partitioned-region id="bar" p:cache-ref="gemfire-cache"/>
```






Data Management: JDBC, Transactions, JPA, Spring Data
===================================================================================================================
* Spring supports many data access technologies
* No matter what technology is used, consistent way of using
* JDBC, JPA, Hibernate, MyBatis, ...
* Spring manages most of the actions under the hood
  * Accessing data source
  * Establishing connections
  * Transactions - begin, rollback, commit
  * Close the connections
* Declarative transaction management (transactions through AOP proxies)

## Exception Handling
* Spring provides own set of exceptions so exceptions are not specific to underlying technology being used
* Replaces checked exceptions with unchecked
  * Checked exceptions provide a form of tight coupling between layers
  * If exception is not caught it must be declared in method signature
  * When exceptions from specific data access technologies leak to the service layer of the app, layers are no longer 
  loosely coupled
* Spring provides DataAccessException
  * Does not reveal underlying specific technology (JPA, Hibernate, JDBC,...)
  * Is not a single exception but a hierarchy of exception
    * DataAccessResourceFailureException, DataIntegrityViolationException, BadSqlGrammarException, 
    OptimisticLockingFailureException, ...
  * Unchecked (Runtime)

### DataAccessException Hierarchy
* ```DataAccessException``` family has three main sub branches:
  * ``` org.springframework.dao.NonTransientDataAccessException```
    * Exceptions considered no-transient, which means that retrying the operation will fail unless the originating 
    cause is fixed (e.g. searching for an object that doesn't exist)
  * ``` org.springframework.dao.RecoverableDataAccessException```
    * Exceptions considered recoverable - a previously failed operation might succeed if some recovery steps are 
    performed (e.g. database fails because of temporary network hiccup) 
    * Developer can remedy this by retrying the query in a new connection
  * ``` org.springframework.dao.TransientDataAccessException```
    * Exceptions considered transient - which means that retrying the operation might succeed without any intervention 
    (e.g. database becomes unavailable because of a bad network connection in the middle of the execution of a query). 
    * Developer can remedy this by retrying the query
* There is a fourth branch with only one exception type ```ScriptException``` - thrown when initialization of a test
database with DataSourceInitializer fails because of a script error
  
## In-Memory Database
* Spring can create embedded DB and run specified init scripts
* H2,HSQL,DERBY supported
* Uses ```EmbeddedDatabaseBuilder```
```java
@Bean
public DataSource dataSource() {
    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
    builder.setName("inmemorydb")
           .setType(EmbeddedDatabaseType.HSQL) 
           .addScript("classpath:/inmemorydb/schema.db") 
           .addScript("classpath:/inmemorydb/data.db");
    
           return builder.build();
}
```
Or in XML
```xml
<jdbc:embedded-database id="dataSource" type="HSQL"> 
    <jdbc:script location="classpath:schema.sql" /> 
    <jdbc:script location="classpath:data.sql" />
</jdbc:embedded-database>
```
Alternatively, existing DB can be initialized with a provided DataSource
```xml
<jdbc:initialize-database data-source="dataSource"> 
    <jdbc:script location="classpath:schema.sql" /> 
    <jdbc:script location="classpath:data.sql" />
</jdbc:initialize-database>
```
Or the same in Java:
```java
@Configuration
public class DatabaseInitializer {

    @Value("classpath:schema.sql") 
    private Resource schemaScript; 

    @Value("classpath:data.sql") 
    private Resource dataScript;

    private DatabasePopulator databasePopulator() { 
        ResourceDatabasePopulator populator = new ResourceDatabasePopulator(); 
        populator.addScript(schemaScript);
        populator.addScript(dataScript); 
        
        return populator;
    }
    
    @Bean
    public DataSourceInitializer dataSourceInitializer(DataSource dataSource) { 
        DataSourceInitializer initializer = new DataSourceInitializer(); 
        initializer.setDataSource(dataSource); 
        initializer.setDatabasePopulator(databasePopulator());
        
        return initializer; 
    }
    
}
```  



## JDBC with Spring
* Using a traditional JDBC connection has many disadvantages
  * lots of boilerplate code
  * poor exception handling (cannot really react to exceptions)
* Spring JDBC template to the rescue
  * Implements **template method** pattern
    * A template method defines the skeleton of a process. Some steps of the process are fixed and happen the same way
     every time. These common steps can be implemented by the template itself (e.g. opening/closing connections.) 
     At certain points, the process delegates some of its work to a subclass to fill in some implementation-specific details 
     (e.g. what query to perform). A template method delegates the implementation-specific portions of the process to 
     an interface. Different implementations of this interface define specific implementations of this portion of the process.
  * JDBC template takes care of boilerplate code:
    * Establishing connection
    * Closing connection
    * Transactions
    * Exception Handling
  * Handles all SQL exceptions properly
  * Transforms SQLExceptions into DataAccessExceptions
    * Spring provides a big family of DataAccessExceptions, all of which are subclasses of RuntimeException
  * Great simplification - user just performs queries
  * Create one instance and re-use it - thread safe
  * Uses runtime exceptions - no try/catch needed
  * Can query for
    * primitives, String, Date, ...
    * Generic Maps
    * Domain objects
  ```JdbcTemplate template = new JdbcTemplate( dataSource )```
  
  
### Query for Simple Object
```template.queryForObject( sqlStatement, Date.class )```
* Can bind variables represented by ```?``` in the query
* Values are provided as varargs in the queryFor* method
* Order of ```?``` matched order of vararg arguments
```
String sqlStatement = "select count(*) from ACCOUNT where balance > ? and type = ?";
accountsCount = jdbcTemplate.queryForObject(sqlStatement, Long.class, balance, type);
```

### Query for Generic Collection
* Each result row returned as map of ( Column Name -> Field Value ) pairs
* ```queryForList``` when expecting multiple results
* ```queryForMap``` when expecting single result

* Query for **single row**:
  * ```jdbcTemplate.queryForMap("select * from ACCOUNT where id=?", accountId)```
    * Results in:
     ```java
     Map<String, Object> ( ID -> 1, BALANCE -> 75000, NAME -> "Checking Account",...)
     ```
    

* Query for **multiple rows**:
  * ```jdbcTemplate.queryForList("select * from ACCOUNT);```
  * Results in: 
    ```java
    List< Map<String,Object> > { 
    Map<String, Object> {ID -> 1, BALANCE -> 75000, NAME -> "Checking account", ...} 
    Map<String, Object> {ID -> 2, BALANCE -> 15000, NAME -> "Savings account", ...} 
    ... }
    ```

### Query for Domain Object
* May consider ORM for this
#### RowMapper
* Must implement ```RowMapper``` interface if querying for DomainObject
  * RowMapper objects are **typically stateless** and thus reusable
* Use RowMapper when **each row of the ResultSet maps to a domain object**
```java
public interface RowMapper<T> {
    T mapRow(ResultSet resultSet, int rowNum) throws SQLException; 
}
```
* Query using for query for Object:
```java
//Single domain object
Account account = jdbcTemplate.queryForObject( sqlStatement, rowMapper, param1, param2 );
//Multiple domain objects
List<Account> = jdbcTemplate.query(sqlStatement, rowMapper, param1, param2);
```
Alternatively, RowMapper can be replaced by lambda expression

#### RowCallbackHandler
* Should be used when **no return object is needed**
  * Converting results to xml/json/...
  * Streaming results
  * ...
* a RowCallbackHandler object is **typically stateful**: It keeps the result state within the object, to be 
available for later inspection.
* Can be replaced by lambda expression
```java
public interface RowCallbackHandler {
    void processRow(ResultSet resultSet) throws SQLException;
}  
```
```jdbcTemplate.query(sqlStatement, rowCallbackHandler, param1, param2);```

#### ResultSetExtractor
* Can **map multiple result rows in the ResultSet to single return object**, whereas RowMapper maps one row to one object
* a ResultSetExtractor object is **typically stateless** and thus reusable, as long as it doesn't access stateful resources
* can be replaced by lambda
```java
public interface ResultSetExtractor<T> {
    T extractData(ResultSet resultSet) throws SQLException, DataAccessException;
}
```
```jdbcTemplate.query(sqlStatement, resultExtractor, param1, param2);```

### Inserts and Updates
* returns number of rows modified
* ```jdbcTemplate.update()``` used for both updates and inserts
  * ```numberOfAffectedRows = jdbcTemplate.update( sqlStatement, param1, param2)```





## Transactions
* Set of operations that take place as a single atomic unit - all or nothing
  * e.g. money transfer operations - have to deduct money from source account then transfer it to target account. If
  one part fails the other cannot be performed and everything must be rolled back
* ACID
  * Atomic - all operations are performed or none of them
  * Consistent - DB Integrity constraints are not violated
  * Isolated - Transactions are isolated from each other
  * Durable - Committed changes are permanent
* When in a transaction, one connection should be used for the entire transaction

### Java Transaction Management
* Local Transaction - One resource (e.g. a database) is involved, transaction is managed by resource itself (e.g. a 
transaction associated with a JDBC connection)
* Global Transaction - Multiple different resources involved in transaction, transaction is managed by a external 
Transaction Manager (e.g. JMS + two different database)
* Multiple modules supporting transactions - JDBC, JPA, JMS, JTA, Hibernate
* Different APIs for global transactions and local transactions
* Different modules have different transaction API
* Usually involves using programmatic transaction management
  * not declarative as in spring
  * need to call begin transaction, commit, end transaction ....
* Transaction management is a cross-cutting concern and should be centralized instead of scattered across all classes
* Transaction demarcation should be independent of actual transaction implementation
* Should be done is service layer instead of data access layer

### Spring Transaction Management
* Declarative transaction management instead of programmatic
* Transaction declaration independent of transaction implementation
* Same API for global and local transactions
* Hides implementation details
* Can easily switch transaction managers
* Can easily upgrade local transaction to global
* Does **NOT** support propagation of transaction contexts across remote calls
* To enable transaction management is Spring:
  * Java Config
    * ```@EnableTransactionManagement```
  * [A] XML
    * ```<tx:annotation driven />```
  * Declare a ```PlatformTransactionManager``` bean
    * **transactionManager** is the default bean name
    * This is the central interface in Spring's transaction infrastructure.
  * Mark methods as transactional (Java or XML)

#### Transaction Manager
* Spring provides many implementations of the PlatformTransactionManager interface
  * **JtaTransactionManager**
    * implementation for JTA, delegating to a backend JTA provider
    * This is typically used to delegate to a Java EE server's transaction coordinator, but may also be configured
    with a local JTA provider which is embedded within the application
  * **JpaTransactionManager**
    * implementation for a single JPA EntityManagerFactory
  * **HibernateTransactionManager**
    * implementation for a single Hibernate SessionFactory
  * **DataSourceTransactionManager**
    * implementation for a single JDBC DataSource
  * **JdoTransactionManager**
  * ...
```java
@Configuration
@EnableTransactionManagement
public class TransactionConfiguration {

    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }   
```

#### @Transactional
* Methods which require being run in a transaction should be annotated with @Transactional
* Object is wrapped in a proxy
  * Around advice is used
  * Before entering a method a transaction is started
  * If everything goes smoothly, there is a commit once the method is finished
  * If method throws **RuntimeException**, rollback is performed, transaction is not committed
* Can be applied to either method level or class level. 
  * If on class level, applies to **all methods declared that implement interfaces!!**. 
    * Can also be applied to interface or parent class BUT **Spring recommends that you only annotate concrete 
    classes (and methods of concrete classes) with the @Transactional annotation**, as opposed to annotating interfaces.
  * If on method level, **should be applied only to public methods.** If applied to methods with other visibilities, no
  error is raised, but method won't exhibit transactional behavior.
* Can combine method level and class level on the same class. Method level config overrides class level
* If a test method is annotated @Transactional, it is run in a transaction and rolled back after finishing
  * Can be on both class level and method level
  * Can add @Commit annotation on method level to override @Transactional on test class level
  
#### @Transactional settings
* value - optional - specify transaction manager to be used
* propagation - optional - enum: ```Propagation``` - defaults to PROPAGATION_REQUIED
* readOnly - optional - default is false - if true, serves as a hint to the underlying transaction system that
this is a read only transaction
* isolation - optional - enum: ```Isolation``` defaults to database default isolation level
* timeout - optional - given is seconds, defaults to underlying transaction system timeout
* rollbackFor - optional - array of exception Class objects that *must* cause rollback
* rollbackForClassName - optional - array of String exception Class names that *must* cause rollback
* noRollbackFor - optional - array of exception Class objects that *must not* cause rollback
* noRollbackForClassName - optional - array of String exception class names that *must not* cause rollback 

#### Isolation Levels
* 4 isolation levels available (from least strict to most strict)
  1. READ_UNCOMMITTED
  2. READ_COMMITTED
  3. REPEATABLE_READ
  4. SERIALIZABLE
* not all isolation levels may be supported for a particular database
* different Databases may implement isolation in slightly different ways

#### Isolation level definitions
* dirty read - a transaction is allowed to read data from a row that has been modified by another transaction, but
not yet committed
* non-repeatable read - during a transaction a row is retrieved twice and the values in the row differ between reads
* phantom read - during a transaction, two identical queries are executed, and the collection of rows returned by
the second query is different from the first.

##### READ_UNCOMMITTED
* ```@Transactional( isolation=Isolation.READ_UNCOMMITTED )```
* the lowest isolation level
* Dirty reads can occur - one transaction may be able to read modified data of another transaction that has not yet
been committed 
* May be viable for large transactions or frequently changing data

##### READ_COMMITTED
* ```@Transactional( isolation.Isolation.READ_COMMITTED )```
* Default isolation level for many databases
* Other transactions can see data only after it is properly committed
* Prevents dirty reads

##### REPEATABLE_READ
* ```@Transactional( isolation=Isolation.REPEATABLE_READ )```
* Prevents non-repeatable reads - when a row is read multiple times in a single transaction, its value is guaranteed
to be the same

##### SERIALIZABLE
* ```@Transactional( isolation=Isolation.SERIALIZABLE )```
* Prevents phantom reads - in the course of a transaction, two identical queries are executed, and the collection of 
rows returned by the second query is different from the first.

#### Transaction Propagation
* Happens when code from one transaction calls another transaction
* Transaction propagation says whether everything should be run in a single transaction or if nested transactions
should be used
* There are seven levels of propagation
* 2 Basic ones - **REQUIRED** and **REQUIRES_NEW**
* There is also:
  * NESTED - 
  * MANDATORY -
  * NEVER -
  * NOT_SUPPORTED - 
  * SUPPORTS - 

##### REQUIRED
* ```@Transactional( propagation=Propagation.REQUIRED )```
* This is the default if not specified otherwise
* If there is already a transaction, new @Transactional code is run in existing transaction, otherwise a new transaction
 is created
  * If an inner method causes a transaction to rollback, the outer method will fail to commit and will also rollback the transaction

##### REQUIRES_NEW
* ```@Transactional( propagation=Propagation.REQUIRES_NEW )```
* If there is no transaction, a new one is created
* If there is already a transaction, it is suspended and a new transaction is created
  * The inner transaction may commit or rollback independently of the outer transaction

##### NESTED
* ```@Transactional( propagation=Propagation.NESTED )```
* uses a single physical transaction with multiple savepoints that it can roll back to
* Such partial rollbacks allow an inner transaction scope to trigger a rollback for its scope, with the outer 
transaction being able to continue the physical transaction despite some operations having been rolled back 
* This setting is typically mapped onto JDBC savepoints, so **will only work with JDBC resource transactions** 

#### Rollback
* By default rollback is performed if RuntimeException (or any subtype)  is thrown in @Transactional method
* This can be overridden by ```rollbackFor``` ```noRollbackFor``` ```rollbackForClassName``` ```noRollbackForClassName``` 
attributes
```java
@Transactional( rollbackFor=MyCheckedException.class, noRollbackFor={FooException.class, BarException.class} )
```

#### Transaction timeout
* ```@Transactional( timeout=10 )```
* given in seconds
* defaults to the default timeout of the underlying transaction system 










## JPA with Spring
* JPA - Java Persistence API
* ORM mapping
* Domain objects are POJOs
* Common API for object relational mapping
* Several implementations of JPA
  * Hibernate EntityManager
  * EclipseLink - Glassfish
  * Apache OpenJPA - WebLogic and WebSphere
  * Data Nucleus - Google App Engine
* Can be used in spring without app server
* **As of Spring 4.0, JPA 2.0+ is required**
  * JPA 1.0 applications are still supported, however a JPA 2.0/2.1 compliant persistence provider is needed at runtime

### Spring JPA Configuration
1. Define EntityManagerFactory bean
2. Define DataSource bean
3. Define TransactionManagement bean
4. Define DAOs
5. Define metadata Mappings on entity beans

#### EntityManager
* Manages unit of work and involved persistence objects (Persistence context)
* Usually in a transaction
* EntityManager API examples:
  * ```persist( Object object )``` - adds entity to persistence context - INSERT INTO ...
  * ```remove( Object object )``` - removes entity from persistence context - DELETE FROM ...
  * ```find( Class entity, Object primaryKey )``` - Find by primary key - SELECT * FROM ... WHERE id = primaryKey
  * ```Query createQuery( String queryString )``` - used to create JPQL query
  * ```flush()``` - Current state of entity is written to DB immediately

#### EntityManagerFactory
* Provides access to new application managed Entity Managers
* Thread safe
* Represents single datasource / persistence unit

#### Setting up a JPA EntityManagerFactory
* Spring provides three ways to set up ```EntityManagerFactory```(ies)
1. ```LocalEntityManagerFactoryBean```
    * Suitable for simple deployment environments where the application only uses JPA for data access
    * Uses persistence.xml to configure one or more persistence units, data sources, and XML based mapping files
      * Expects persistence.xml to be in the default location: ```META-INF/persistence.xml```
    * Requires you to specify the persistence unit name in the bean configuration: ```setPersistenceUnitName()```
    * The most simple and most limited
      * CANNOT refer to an existing JDBC dataSource bean definition
      * No support for Global Transactions
      * Weaving of persistent classes is provider specific, often requires a specific JVM start-up option
    * Example xml config:
      ```xml
      <beans>
          <bean id="myEmf" class="org.springframework.orm.jpa.LocalEntityManagerFactoryBean">
              <property name="persistenceUnitName" value="myPersistenceUnit"/>
          </bean>
      </beans>
      ```

2. Obtain an EntityManagerFactory from JNDI:
    * Example retrieving EntityManagerFactory from JNDI:
          ```xml
          <beans>
              <jee:jndi-lookup id="myEmf" jndi-name="persistence/myPersistenceUnit"/>
          </beans>
          ```
    * Example in java config:
        ```java
        @Bean
        public JndiObjectFactoryBean entityManagerFactory() {
          JndiObjectFactoryBean jndiObjectFB = new JndiObjectFactoryBean();
          jndiObjectFB.setJndiName("jdbc/SpittrDS");
          return jndiObjectFB;
        }
        ```

3. ```LocalContainerEntityManagerFactoryBean```
    * Preferred option
    * Gives full control over ```EntityManagerFactory``` creation
    * Can work with custom dataSource
    * Can control the weaving process using LoadTimeWeaver instead of providing a JVM agent on startup
    * Can use persistence.xml file to help configure the bean, or configure strictly via bean properties
    * Allows you to scan for your @Entity classes via its **setPackagesToScan()** method
    * Configuring a LocalContainerEntityManagerFactoryBean:
      * Configure a ```JpaVendorAdapter``` bean with the specific JPA implementation you are using
        * Spring provides several implementations:
            * JpaVendorAdapter
            * HibernateJpaVendorAdapter
            * EclipseLinkJpaVendorAdapter
            * OpenJpaVendorAdapter
        * Will need to supply configuration about the Database you are using
      * Set any vendor specific Jpa Properties you require by configuring a properties object
      * Provide a dataSource bean
      * Provide the location of *persistence.xml* (files)
        * OR have it scan for entity classes via ```setPackagesToScan()```
      * set the JpaVendorAdapter
    * Example java config:
      ```java
      @Bean
      public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
          HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
          adapter.setShowSql(true);
          adapter.setGenerateDdl(true);
          adapter.setDatabase(Database.HSQL);

          Properties properties = new Properties();
          properties.setProperty("hibernate.format_sql", "true");

          LocalContainerEntityManagerFactoryBean emfb = new LocalContainerEntityManagerFactoryBean();
          emfb.setDataSource(dataSource());
          emfb.setPackagesToScan("com.example");
          emfb.setJpaProperties(properties);
          emfb.setJpaVendorAdapter(adapter);

          return emfb;
      }
      ```
    * Example xml config:
      ```xml
      <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
          <property name="dataSource" ref="dataSource"/>
          <property name="packagesToScan" value="com.example"/>
          <property name="jpaVendorAdapter">
              <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
                  <property name="showSql" value="true"/>
                  <property name="generateDdl" value="true"/>
                  <property name="database" value="HSQL"/>
              </bean>
          </property>
          <property name="jpaProperties">
              <props>
                  <prop key="hibernate.format_sql">true</prop>
              </props>
          </property>
      </bean>
      ```
* example datasource and PlatformTransactionManager configs used by EntityManagerFactoryBean configurations
```
@Bean
public DataSource dataSource() {
    //...
}

@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) { 
    return new JpaTransactionManager(emf);
}
```



#### Persistence Unit
* A JPA Persistence Unit is a logical grouping of user defined persistable classes (entity classes, embeddable classes 
and mapped superclasses) with related settings
* A persistence unit is a grouping of one or more persistent classes that correspond to a single data source
* Has a persistence provider
* Has assigned transaction type
* A Spring Application can have multiple persistence units
* In Spring configuration a persistence unit can be either in spring configuration, persistence unit itself, or a 
combination of both

#### Persistence Context
* A set of managed, unique, entity **instances**
* Associated with an EntityManager
  * EntityManager interacts with with the persistence context to manage entity instances and their lifecycle

### JPA Mapping (POJOs)
* Metadata is required to determine mapping of class fields to DB columns
* Can be done using either annotations or XML (orm.xml)
* Many defaults are provided - configuration only needed when it differs from the defaults

#### JPA Annotations on:
* Classes
  * Configuration applies to entire class
  * Class annotated @Entity
  * Table name can be overridden using ```@Table( name="TABLE_NAME" )``` , if not specified class name is used
* Fields
  * Usually mapped to columns
  * by default all fields are considered persistent
  * Can annotate ```@Transient``` if it should not be persisted
  * Accessed directly through reflection
  * @Id can be used to mark primary key field, can also be placed on getter
  * @Column( name="column_name" ) can be used to specify column name, if not specified, field name is used as default,
  can also be placed on getter
  * @Entity and @Id are mandatory, the rest are optional
  * @JoinColumn( name="foreignKey" )
  * @OneToMany, @OneToOne, @ManyToOne
* Getters
  * Maps to columns like fields
  * Alternative to annotating fields directly
  
#### @Embeddable
* One row can be mapped to multiple entities
* E.g. Person, which is one DB table contains personal info as well as address info
* Address can be embedded in the Person entity
  * Address is a separate entity annotated with @Embeddable
  * In Person class, Address field is annotated with @Embedded
  * Can use @AttributeOverride annotation to specify mappings from columns to fields
  ```java
  ...
  @Embedded
  	@AttributeOverrides({
  			@AttributeOverride(name = "area", column = @Column(name = "landmark")),
  			@AttributeOverride(name = "pincode", column = @Column(name = "postal_code")) })
  	private Address address;
    ...
  ```

### JPA Queries
#### By Primary Key
* ```entityManager.find( Person.class, personId )```
  * Returns null if no item found
  * Uses generics - no cast needed

#### JPQL Queries
* JPQL - JPA Query Language
```java
TypedQuery<Person> query = entityManager.createQuery("select p from Person p where p.firstname = :firstName", Person.class); 
query.setParameter("firstName", "John");
List<Person> persons = query.getResultList();

List<Customer> persons = entityManager.createQuery("select p from Person p where p.firstname = :firstName", Person.class)
  .setParameter("firstName", "John").getResultList();

Person person = query.getSingleResult();
```
* Criteria Queries
  * JPA criteria queries are defined by instantiation of Java objects that represent query elements
  * errors can be detected earlier, during compilation rather than at runtime
  * For dynamic queries that are built at runtime - the criteria API may be preferred
  ```java
  CriteriaBuilder cb = em.getCriteriaBuilder();
   
    CriteriaQuery<Country> q = cb.createQuery(Country.class);
    Root<Country> c = q.from(Country.class);
    q.select(c);
  ```
* Native Queries - JDBC template is preferred instead of native queries

#### JPA DAOs
* JPA DAOs in Spring
  * Transactions are handled in the Service layer - Services are wrapped in AOP proxies - Spring ```TransactionInterceptor``` 
  is invoked
  * ```TransactionInterceptor``` delegates to TransactionManager (JpaTransactionManager, JtaTransactionManager )
  * Services call DAOs with no spring dependency
  * Services are injected with AOP-proxied ```EntityManager```
  * Proxied ```EntityManager``` makes sure queries are properly joined into active transaction
* DAO implementations have no Spring dependencies
* Uses AOP for exception translation - JPA exceptions are translated to Spring DataAccessException(s)
  * Spring performs exception translation (using proxies) **through the @Repository annotation**
    * The postprocessor automatically looks for all exception translators (implementations of the 
      ```PersistenceExceptionTranslator``` interface) and advises all beans marked with the @Repository annotation so 
      the discovered translators can intercept and apply the appropriate translation on the thrown exceptions.
    * Have to explicitly configure a ```PersistenceExceptionTranslationPostProcessor``` bean
      ```java
      @Bean
      public BeanPostProcessor persistenceTranslation() {
        return new PersistenceExceptionTranslationPostProcessor();
      }
      ```
* Can use **@PersistenceContext** annotation to inject EntityManager proxy
```java
public class JpaPersonRepository implements PersonRepository { 

    private EntityManager entityManager;
    
    @PersistenceContext
    public void setEntityManager(EntityManager entityManager) { 
        this.entityManager = entityManager;
    }

    //Use EntityManager to perform queries
            
    //If this repository is not annotated as spring component (@Repository), and explicitly declared in @Configuration
    //file, there is no spring dependency in the DAO layer - @PersistenceContext is from javax.persistence
}
```

#### Enabling Support for @PersistenceUnit and @PersistenceContext
* These are not Spring annotations, provided by JPA
* Have to configure a ```PersistenceAnnotationBeanPostProcessor``` bean in order for spring to recognize
these annotations
* If using java config: (NOTE this bean is automatically declared for you if using a java config)
* In XML config the PersistenceAnnotationBeanPostProcessor is automatically registered if you use:
  * ```<context:annotation-driven />``` or ```<context:component-scan />```


JPA With Spring Data
======================================================================================================================
* Simplifies data access by reducing boilerplate code
* Consists of core project and many sub-projects for various data access technologies - JPA, Gemfire, MongoDB,Hadoop,...
1. Domain classes are annotated as usual
2. Repositories are defined just as interfaces, extending a specific Spring interface, which will be dynamically
implemented by Spring

## Spring Data domain classes
* For JPA nothing changes, classes are annotated as usual - @Entity
* Specific data stores have their own specific annotations
  * MongoDB - @Document
  * Gemfire - @Region
  * Neo4J - @NodeEntity, @GraphId

## Spring Data Repositories
* Spring searches for all interfaces extending ```Repository<DomainObjectType, DomainObjectIdType>```
* Repository is just a marker interface and has no method on its own
* Can annotate methods in the interface with ```@Query("select p from person p where ....")```
```java
package com.ps.repos;
import com.ps.ents.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.List;

public interface UserRepo extends JpaRepository<User, Long> {
  @Query("select u from User u where u.username like %?1%")
  List<User> findAllByUserName(String username);

  // using named parameters
  @Query("select u from User u where u.username= :un")
  User findOneByUsername(@Param("un") String username);
  
  @Query("select u.username from User u where u.id= :id")
  String findUsernameById(Long id);
  
  @Query("select count(u) from User u")
  long countUsers();
}
```
* Can extend ```CrudRepository``` instead of ```Repository``` which will add methods for CRUD functionality
* Can extend ```PagingAndSortingRepository``` - adds sorting and paging
* Most spring data sub-projects have their own variations of ```Repository```
  * ```JpaRepository``` for JPA
* Repositories can be injected by type of their interface
```java
public interface PersonRepository extends Repository<Person, Long> {}
    
@Service    
public class PersonService {

    @Autowired
    private PersonRepository personRepository;
            
    ...
} 
```

## Query Methods
* define these methods in your Repository interface
* Spring Data JPA parses the method names and tries to determine the methods purpose
* Method names  based on naming convention:
  * *verb* + [ subject ] + *By* + *predicate*
    * verb can be - *get* *read* *find* *count*  (get,read and find are synonymous)
      * *count* returns a count of matching objects
  * *subject* is mostly optional, its primary purpose is to allow you some flexibility in how you name your methods
    * There is one exception, if the subject starts with **Distinct**
      * ```List<Person> people = findDistinctPersonByFirstname( String name )```
    * These two methods are equivalent:
      * ```List<Person> persons = findPersonByFirstname( String name )```
      * ```List<Person> peeps = findThatThingByFirstname( String name )```
    * The type of object retrieved is determined by the Repository interface parameter e.g. ```Repository<Person,Long>```
  * ```findByFirstnameOrLastname( String firstName, String lastName )```
  * ```findByDateOfBirthGt( Date date )```
  * *predicate* can contain operations: Gt, Lt, Ne, Like, Between, OrderBy, ...
    * ```findFirstNameLike( String name )``` name would contain the SQL string performing the match: "%Sam%"
    * ```findByStartDateBetween( Date start, Date end )```
    * ```List<Pet> findPetsByBreedIn(List<String> breed)```
    * ```int countProductsByDiscontinuedTrue()```
    * ```List<Spitter> readByFirstnameOrLastnameOrderByLastnameAsc(String first, String last);```
    * ```List<Spitter> readByFirstnameOrLastnameOrderByLastnameAscFirstnameDesc( String first, String last );```
  * parameter names are irrelevant, but they must be ordered to match up with the method name’s comparators
    * ```List<Spitter> readByFirstnameOrLastnameAllIgnoresCase(String first, String last);```


## Repository Lookup
* Locations where spring should look for ```Repository``` interfaces need to be explicitly defined
* Use **@EnableJpaRepositories** on configuration class
```java
@Configuration 
@EnableJpaRepositories(basePackages="com.example.**.repository") 
public class JpaConfig {...}

@Configuration 
@EnableGemfireRepositories(basePackages="com.example.**.repository")
public class GemfireConfig {...}
  
@Configuration 
@EnableMongoRepositories(basePackages="com.example.**.repository") 
public class MongoDbConfig {...}
```
```xml
<jpa:repositories base-package="com.example.**.repository" /> 
<gfe:repositories base-package="com.example.**.repository" />
<mongo:repositories base-package="com.example.**.repository" /> 
```


Spring Web
====================================================================================================================
* Provides several web modules
  * Spring MVC - model,view,controller framework
  * Spring Web Flow - Navigation flows (wizard-like)
  * Spring Social - integration with facebook, twitter, ...
  * Spring mobile - switching between regular and mobile versions of a site
* Spring can be integrated with other web frameworks, some integration provided by spring itself
  * JSF
  * Struts 2 (Struts 1 no longer supported as of Spring 4+)
  * Wicket
  * Tapestry 5
  * ...
* Spring web layer is on top of spring application layer
* Initialized using regular servlet listener
* Can be configured either in web.xml or using AbstractContextLoaderInitializer - implements WebApplicationInitializer,
which is automatically recognized and processed by servlet container (Servlet 3.0+)

### Basic Configuration
#### AbstractContextLoaderInitializer
```java
public class WebAppInitializer extends AbstractContextLoaderInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();
        //Configure app context here
        return applicationContext;
    }
}
```
* Servlet container looks for classes implementing ServletContainerInitializer (Spring provides 
SpringServletContainerInitializer)
* SpringServletContainerInitializer looks for classes implementing WebApplicationInitializer, which specify 
configuration instead of web.xml
* Spring provides two convenience implementations
  * **AbstractContextLoaderInitializer** - only registers ContextLoaderListener
  * **AbstractAnnotationConfigDispatcherServletInitializer** - registers ContextLoaderListener and defines dispatcher
  servlet, expects a Java Config

#### web.xml
* Register spring-provided Servlet listener
* Provide spring config files as context param - contextConfigLocation
  * Defaults to WEB-INF/applicationContext.xml if not provided
* can provide spring active profiles in ```spring.profiles.active```
```xml
<context-param> 
    <param-name>contextConfigLocation</param-name> 
    <param-value>
        /WEB-INF/config.xml
        /WEB-INF/other-config.xml
    </param-value> 
</context-param>

<listener>
    <listener-class>
        org.springframework.web.context.ContextLoaderListener
    </listener-class> 
</listener>

<context-param>
    <param-name>spring.profiles.active</param-name>
    <param-value>profile1,profile2</param-value> 
</context-param>
```

### Dependency Injection in Servlets
* in init() of HttpServlet cannot access spring beans as they are not available yet
* need to use ```WebApplicationContextUtils``` - provides ApplicationContext through Servlet Context
* Only needed in Servlets - other components such as @Controllers can use regular dependency injection

```java
public class MyServlet extends HttpServlet {
    private MyService service;
    
    public void init() {
        ApplicationContext context = WebApplicationContextUtils.getRequiredWebApplicationContext(getServletContext()); 
        service = (MyService) context.getBean("myService");
    }
    
    ...
}
```

### Spring Web Flow
* Provides support for stateful navigation flows
* Handles browser back button
* Handles double submit problem
* Support custom scopes of beans - flow scope, flash scope, ...
* Configuration of flows defined in XML
  * Defines flow states - view, action, end, ...
  * Defines transition between states
  


## Spring MVC
* Spring web framework based on Model-View-Controller pattern
* Alternative to JSF, Struts, Wicket, Tapestry,...
* Components such as controllers are Spring managed beans
* Testable POJOs
* Uses spring configuration
* Supports a wide range of view technologies - JSP,Freemarker, Velocity, Thymeleaf, ...

### Dispatcher Servlet
* Front controller pattern
* Handles all incoming requests and delegates them to appropriate beans
* All the web infrastructure beans are customizable and replaceable
* DispatcherServlet is defined as servlet in web.xml or through WebApplicationInitializer interface
* Separate web application context on top of regular app context
* Web app context can access beans from root context, but NOT vice versa
```java
public class WebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    //@Configuration classes for root application context
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[]{ RootConfig.class };
    }

    //@Configuration classes for web application contet
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{ WebConfig.class };
    }

    //Urls handled by Dispatcher Servlet
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/app/*"};
    }

    ...
}
```

### Controllers
* Annotated with @Controller
* @RequestMapping annotation on controller's methods specifies which requests it should handle

#### @RequestMapping
* Can specify URL, which the annotated method should handle - @RequestMapping( "/foo" )
  * => server-url/app-context-root/servlet-mapping/request-mapping
  * can use wildcards @RequestMapping( "foo/*" )
* Can specify the HTTP method which the annotated method should handle

#### @RequestParam
* Can specify parameter from http request to be injected as method parameter
```java
@RequestMapping("/foo")
public String foo(@RequestParam("bar") String bar) {
    ...
}
```
* this will extract value "xxx" from requested URL ```/foo?bar=xxx```

#### @PathVariable
* can extract value as a method parameter from the URL
* e.g. extract "1" from /persons/1
```java
@RequestMapping("/persons/{personId}")
public String foo(@PathVariable("personId") String personId) {
    ...
}
```
* Other method parameters can be injected as well - @RequestHeader(), @Cookie,...
* Some parameters Spring injects by type - HttpServletRequest, HttpServletResponse, Principal, HttpSession, ...


### Views
* Views render output to the client based on data in the model
* Many built-in supported view technologies - Freemarker, JSP, Velocity, ...
* ViewResolver selects specific view based on logical view name returned by the controller methods

#### View Resolver
* Translates logical view name to actual view
* This way controller is not coupled to specific view technology (returns only logical view name)
* Default view resolver already configured is InternalResourceViewResolver
  * Convenient subclass of UrlBasedViewResolver that supports InternalResourceView (in effect, Servlets and JSPs) 
  and subclasses such as JstlView and TilesView
  * Configures prefix and suffix to logical view name which then results in a path to specific JSP
* Can register custom resolvers

#### View Resolution Sequence
1. Controller returns logical view name to DispatcherServlet
2. ViewResolvers are asked in sequence (based on their Order)
3. If ViewResolver matches the logical view name, then it returns which view should be used to render the output. If
not, it returns null and the chain continues to the next ViewResolver.
4. DispatcherServlet passes the model to the Resolved View and it renders the output.

### Spring MVC Quick Start
1. Register DispatcherServlet (web.xml or in Java)
2. Implement controllers
3. Register controllers with DispatcherServlet
    * can be discovered using component scan
4. Implement views
    * e.g. write your JSP pages
5. Register a ViewResolver or use the default one
    * need to set prefix (e.g. /WEB-INF/views) and suffix (e.g. .jsp)
6. Deploy

### Configuring Spring MVC
* Annotate a @Configuration class with **@EnableWebMvc** ( in xml config: <mvc:annotation-driven /> )
  * enables HttpMessageConverters, @RequestMapping, @ExceptionHandler, Number Formatting with @NumberFormat, 
  validations with @Valid, Date formatting, ....
* To customize / override the default configs
  * implement ```WebMvcConfigurer``` interface or extend ```WebMvcConfigurerAdapter``` on a @Configuration class
  



REST
=====================================================================================================================
* Representational State Transfer
* Architectural Style
* Stateless (clients maintain state, not server), makes it scalable, do not use HTTP session
* Usually over HTTP, but not necessarily
* Entities (e.g. Person) are resources represented by URIs
* HTTP methods (GET, POST, PUT, DELETE,PATCH) are actions performed on resources (like CRUD operations)
* Resource can have multiple representations (different content type)
* Request specifies desired representation using HTTP accept header, or extension in URL (.json) or parameter in
URL (format=json)
* Response states the delivered representation type by using the "Content-Type" HTTP header


## HATEOAS
* Hypertext as the Engine of Application State
* Response contains links to other items and actions --> can change behavior without changing client
* Decoupling of client and server

## JAX-RS
* Java API for RESTFUL Web Services
* Part of Java EE6
* Jersey is the reference implementation
```java
@Path("/persons/{id}")
public class PersonService {
    @GET
    public Person getPerson(@PathParam("id") String id) {
        return findPerson(id);
    }
}
```

## RestTemplate
* Can be used to simplify HTTP calls to RESTFUL api
* by default the RestTemplate relies on standard JDK facilities to establish HTTP connections
  * can switch to a different library through the 
  ```HttpAccessor.setRequestFactory(org.springframework.http.client.ClientHttpRequestFactory)``` property.
* Message converters supported - Jackson, GSON
* Automatic input/output conversion using ```HttpMessageConverters```
  * StringHttpMessageConverter
  * MarshallingHttpMessageConverter
  * GsonHttpMessageConverter
  * RssChannelHttpMessageConverter
* provides a wide set of polymorphic methods for each HTTP method
  * These methods all throw ```RestClientException```
  * DELETE
    * delete(java.lang.String url, java.util.Map<java.lang.String,?> uriVariables)
    * delete(java.lang.String url, java.lang.Object... uriVariables)
    * delete(java.net.URI url)
  * GET
    * <T> ResponseEntity<T> getForEntity( java.lang.String url, java.lang.Class<T> responseType, java.util.Map<java.lang.String,?> uriVariables)
    * <T> ResponseEntity<T> getForEntity( java.lang.String url, java.lang.Class<T> responseType, java.lang.Object... uriVariables)
    * <T> ResponseEntity<T> getForEntity( java.net.URI url, java.lang.Class<T> responseType)
    * <T> T getForObject( java.lang.String url, java.lang.Class<T> responseType, java.util.Map<java.lang.String,?> uriVariables )
    * <T> T getForObject( java.lang.String url, java.lang.Class<T> responseType, java.lang.Object... uriVariables)
    * <T> T getForObject( java.net.URI url, java.lang.Class<T> responseType)
  * HEAD
    * HttpHeaders headForHeaders( java.lang.String url, java.util.Map<java.lang.String,?> uriVariables )
    * HttpHeaders headForHeaders( java.lang.String url, java.lang.Object... uriVariables)
    * HttpHeaders headForHeaders( java.net.URI url)
  * OPTIONS
    * java.util.Set<HttpMethod> optionsForAllow( java.lang.String url, java.util.Map<java.lang.String,?> uriVariables )
    * java.util.Set<HttpMethod> optionsForAllow( java.lang.String url, java.lang.Object... uriVariables)
    * java.util.Set<HttpMethod> optionsForAllow( java.net.URI url )
  * PATCH
    * <T> T patchForObject( java.lang.String url, java.lang.Object request, java.lang.Class<T> responseType, java.util.Map<java.lang.String,?> uriVariables)
    * <T> T patchForObject( java.lang.String url, java.lang.Object request, java.lang.Class<T> responseType, java.lang.Object... uriVariables )
    * <T> T patchForObject( java.net.URI url, java.lang.Object request, java.lang.Class<T> responseType )
  * POST
    * <T> ResponseEntity<T> postForEntity( java.lang.String url, java.lang.Object request, java.lang.Class<T> responseType, java.util.Map<java.lang.String,?> uriVariables)
    * <T> ResponseEntity<T> postForEntity( java.lang.String url, java.lang.Object request, java.lang.Class<T> responseType, java.lang.Object... uriVariables) 
    * <T> ResponseEntity<T> postForEntity( java.net.URI url, java.lang.Object request, java.lang.Class<T> responseType)
    * java.net.URI 	postForLocation( java.lang.String url, java.lang.Object request, java.util.Map<java.lang.String,?> uriVariables)
    * java.net.URI 	postForLocation( java.lang.String url, java.lang.Object request, java.lang.Object... uriVariables)
    * java.net.URI 	postForLocation( java.net.URI url, java.lang.Object request )
    * <T> T postForObject( java.lang.String url, java.lang.Object request, java.lang.Class<T> responseType, java.util.Map<java.lang.String,?> uriVariables)
    * <T> T postForObject( java.lang.String url, java.lang.Object request, java.lang.Class<T> responseType, java.lang.Object... uriVariables)
    * <T> T postForObject( java.net.URI url, java.lang.Object request, java.lang.Class<T> responseType )
  * PUT
    * put( java.lang.String url, java.lang.Object request, java.util.Map<java.lang.String,?> uriVariables)
    * put( java.lang.String url, java.lang.Object request, java.lang.Object... uriVariables)
    * put( java.net.URI url, java.lang.Object request)
  * Exchange and Execute
    * generalized versions of the above methods and can be used to support additional, less frequent combinations 
    (e.g. HTTP PATCH, HTTP PUT with response body, etc.). Note however that the underlying HTTP library used must 
    also support the desired combination. 
    * can make any type of REST call, but have to pass in the HTTP method as a parameter.
    * EXCHANGE - looks like these methods "exchange" a request entity for a ResponseEntity
      * <T> ResponseEntity<T> 	exchange(RequestEntity<?> requestEntity, java.lang.Class<T> responseType)
      * <T> ResponseEntity<T> 	exchange(RequestEntity<?> requestEntity, ParameterizedTypeReference<T> responseType)
      * <T> ResponseEntity<T> 	exchange(java.lang.String url, HttpMethod method, HttpEntity<?> requestEntity, java.lang.Class<T> responseType, java.util.Map<java.lang.String,?> uriVariables)
      * <T> ResponseEntity<T> 	exchange(java.lang.String url, HttpMethod method, HttpEntity<?> requestEntity, java.lang.Class<T> responseType, java.lang.Object... uriVariables)
      * ...
    * EXECUTE - can be given a RequestCallback, and return a result using a ResponseExtractor
      * RequestCallback - Allows you to manipulate the request headers, and write to the request body, before submitting the request 
      * <T> T  execute(java.lang.String url, HttpMethod method, RequestCallback requestCallback, ResponseExtractor<T> responseExtractor, java.util.Map<java.lang.String,?> uriVariables)
      * <T> T  execute(java.lang.String url, HttpMethod method, RequestCallback requestCallback, ResponseExtractor<T> responseExtractor, java.lang.Object... uriVariables)
      * <T> T  execute(java.net.URI url, HttpMethod method, RequestCallback requestCallback, ResponseExtractor<T> responseExtractor)
* AsyncRestTemplate
  * Methods are similar to RestTemplate but return ListenableFuture
  * allows asynchronous calls
  * most methods will throw RestClientException (just like RestTemplate)


## Spring Rest Support
* In MVC Controllers, HTTP method consumed can be specified
  * ```@RequestMapping( value="/foo", method=RequestMethod.POST )```
  * New in Spring 4.3 - mappings for specific HTTP methods:
    * @GetMapping, @PostMapping, @PutMapping, @DeleteMapping, and @PatchMapping
* **@ResponseStatus** can set HTTP response status code
  * If used, void return type means no View (empty response body) and not default view!
  * 2** - success codes ( 201 Created, 204 No Content, ...)
  * 3** - redirect codes
  * 4** - client error ( 404 Not Found, 405 Method Not Allowed, 409 Conflict, ...)
  * 5** - server error
* **@ResponseBody** before controller method return type means that the response should be directly rendered to the
client and not evaluated as a logical view name
  * Uses converters to convert return type of controller method to requested content type
* **@RequestHeader** can inject value from HTTP request header as a method parameter
* **@RequestBody** - injects body of HTTP request as a method parameter, uses HttpMessageConverters to convert request
data based on content type
  * ```public void updatePerson( @RequestBody Person person, @PathVariable("id") int personId )```
  * Person can be converted from JSON,XML,...
* ```HttpEntity<?>```
  * Controller can return HttpEntity instead of view name - and directly set response body, HTTP response headers
  * ```return new HttpEntity<ReturnType>( responseBody, responseHeaders );```
* ```HttpMessageConverter```
  * Converts HTTP response (annotated by @ResponseBody) and request body data (annotated by @RequestBody)
  * ```@EnableWebMVC``` or ```<mvc:annotation-driven />``` enables it
  * XML, Forms, RSS, JSON,...
* **@RequestMapping** can have attributes "produces" and "consumes" to specify input and output content type
  * ```@RequestMapping( value= "/person/{id}", method=RequestMethod.GET, produces= {"application/json"} )```
  * ```@RequestMapping (value= "/person/{id}", method=RequestMethod.POST, consumes= { "application/json" })```
* Content Type Negotiation can be used for both view and REST endpoints
* **@RestController** - Equivalent to @Controller, but automatically adds @ResponseBody to each method
  * introduced in Spring 4.0
* ```@ResponseStatus( HttpStatus.NOT_FOUND )```
  * can be used to annotate a method or exception class to define HTTP code to be returned. Can also be used at
  controller class level and will be applied to all **@RequestMapping** handler methods.
  * The status code is applied to the HTTP response when the handler method is invoked and overrides status
  information set by other means, like ResponseEntity or "redirect:"
  * Having a controller method annotated with **@ResponseStatus** will stop the DispatcherServlet from trying to find
  a view to render
* ```@ExceptionHandler( {MyException.class } )```
  * Controller methods annotated with it are called when declared exceptions are thrown.
  * Method can also be annotated with **@ResponseStatus** to define which HTTP result code should be sent to the client
  in case of an Exception

* When determining address of child resources, ```UriTemplate``` should be used so absolute URLs are not hardcoded
```java
StringBuffer currentUrl = request.getRequestURL();
String childIdentifier = ...;//Get Child Id from url requested
UriTemplate template = new UriTemplate(currentUrl.append("/{childId}").toString());
String childUrl = template.expand(childIdentifier).toASCIIString();
```


Spring Security
==========================================================================================================
* Independent of container - does not require EE container, configured in application and not in container
  * can integrate with container managed security if need be
* Separates security from business logic
* Decoupled authorization from authentication
* In Spring Security 4, the possibility of using CSFr tokens in Spring forms to prevent cross-site
  request forgery was introduced. A configuration without a <csrf /> element configuration is invalid, and every
  login request will direct you to a 403 error page (can be disabled)

## Definitions:
* Principal
  * User, device, or system that is performing an action
* Credentials
  * identification keys that a principal uses to confirm its identity
* Authentication
  * Process of checking identity of a principal (credentials are valid)
  * basic, OAuth, Cookies, Single-sign-on, digest, form, X.509,...
  * Credentials need to be stored securely
* Authorization
  * Process of checking that a principal has privileges to perform a requested action
  * Depends on authentication
  * Often based on roles - privileges not assigned to specific users, but to groups
* Secured Item
  * any resource being secured
      
## Spring Security high-level overview
1. A user tries to access the application by making a request. The application requires the user to provide its
credentials so it can be logged in
2. The credentials are verified by the **Authentication Manager** and the user is granted access to the application.
The authorization rights for this user are loaded into the Spring Security Context
3. The user makes a resource request (view,edit,insert,delete information) and the **Security Interceptor** intercepts
the request before the user accesses a protected/secured resource
4. The Security Interceptor extracts the user authorization data from the security context and delegates the decision
to the **Access Decision Manager**
5. The Access Decision Manager polls a list of voters to return a decision regarding the rights of the authenticated
user to system resources.
6. Access is granted or denied to the resource based on the user rights and the resource attributes
  
## Major building blocks of Spring Security
* *SecurityContextHolder* - provides access to the *SecurityContext*
* *SecurityContext* - holds the *Authentication* object and possibly request-specific security information
* *Authentication* - represents the principal in a Spring security-specific manner
* *GrantedAuthority*
  * to reflect the **application-wide** permissions granted to a principal
  * are usually "roles", such as ROLE_ADMINISTRATOR or ROLE_HR_SUPERVISOR
  * GrantedAuthority objects are usually loaded by the UserDetailsService
* *UserDetails* - to provide the necessary information to build an Authentication object from your application's DAOs
or other sources of security data
* *UserDetailsService*
  * Creates a *UserDetails* when passed in a String-based username (or certificate ID, etc..)
  * It is purely a DAO for user data and performs no other function other than to supply that data to other 
  components within the framework. It does not authenticate the user, that is done by the AuthenticationManager
  * is an interface with one method - ```UserDetails loadUserByUsername( String username )```
  * Spring provides some implementations of UserDetailService:
    * InMemoryDaoImpl 
    * JdbcDaoImpl
    * Ldap
    * Most users will implement their own UserDetailsService

## Configuring Spring Security - Overview
* To configure Spring security, must do three things
  1. declare the security filter for the application
  2. define the Spring Security context
  3. configure Authentication and Authorization
  

### Spring Security Java Configuration - Overview
1. Configure the ```DelegatingFilterProxy```
    * To configure DelegatingFilterProxy in Java with a WebApplicationInitializer, then all you need to do is
    create a new class that extends ```AbstractSecurityWebApplicationInitializer```. DelegatingFilterProxy is
    automatically registered
      ```java
       import org.springframework.security.web.context.AbstractSecurityWebApplicationInitializer;

       public class SecurityWebInitializer extends AbstractSecurityWebApplicationInitializer {}
      ```
2. Annotate your @Configuration with @EnableWebSecurity
    * Your @Configuration should extend WebSecurityConfigurerAdapter
    * Override *configure(..)* methods as needed
    ```java
    @Configuration
    @EnableWebSecurity
    public class HelloWebSecurityConfiguration extends WebSecurityConfigurerAdapter {
    
      @Autowired
      public void configureGlobal(AuthenticationManagerBuilder authManagerBuilder) throws Exception {
        //Global security config here - you can define your principals here
        //Can configure inMemory authentication here, or jdbc...
      }
    
      @Override
      protected void configure(HttpSecurity httpSecurity) throws Exception {
          //http security specific config here. 
      }
    
      @Override
        protected void configure(WebSecurity web) throws Exception {
            //web security specific can be config here, For example, if you wish to ignore certain requests.
            web.ignoring().antMatchers("/resources/**","/images/**","/styles/**");
        }
    }
    ```

### DelegatingFilterProxy XML Configuration
* Need to configure the **springSecurityFilterChain** (mandatory name) to use ```DelegatingFilterProxy``` as a servlet filter 
in your web.xml
  ```xml
        <web-app ...>
          <!-- The root web application context is the security context-->
          <context-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>
              /WEB-INF/spring/security-config.xml
            </param-value>
          </context-param>

          <!-- Bootstraps the root web application context before servlet initialization -->
          <listener>
            <listener-class>
              org.springframework.web.context.ContextLoaderListener
            </listener-class>
          </listener>

          <filter>
            <filter-name>springSecurityFilterChain</filter-name>
            <filter-class>
              org.springframework.web.filter.DelegatingFilterProxy
            </filter-class>
          </filter>
          <filter-mapping>
            <filter-name>springSecurityFilterChain</filter-name>
            <url-pattern>/*</url-pattern>
          </filter-mapping>
          ...
          <!-- The front controller, the entry point for all requests -->
          <servlet>
            <servlet-name>pet-dispatcher</servlet-name>
            <servlet-class>
              org.springframework.web.servlet.DispatcherServlet
            </servlet-class>
            <init-param>
              <param-name>contextConfigLocation</param-name>
              <param-value>
                /WEB-INF/spring/mvc-config.xml
                /WEB-INF/spring/app-config.xml
              </param-value>
            </init-param>
            <load-on-startup>1</load-on-startup>
           </servlet>
        </web-app>
  ```
    


## Authorization
* Process of checking if a principal has privileges to perform requested action
* Specific URLs can have specific Role or Authentication requirements
* Can be configured using ```HttpSecurity.authorizeRequests().*```

### java config
```java
@Configuration
@EnableWebSecurity
public class HelloWebSecurityConfiguration extends WebSecurityConfigurerAdapter {

  @Override
  protected void configure(HttpSecurity httpSecurity) throws Exception {
      httpSecurity.authorizeRequests().antMatchers("/css/**","/img/**","/js/**").permitAll()
                                      .antMatchers("/admin/**").hasRole("ADMIN")
                                      .antMatchers("/user/profile").hasAnyRole("USER","ADMIN")
                                      .antMatchers("/user/**").authenticated()
                                      .antMatchers("/user/private/**").fullyAuthenticated()
                                      .antMatchers("/public/**").anonymous();
  }
}
```
* Rules are evaluated in the order listed
* Should be from the most specific to the least specific
* Options
  * ```hasRole( String role )``` - has specific role, automatically prepends "ROLE_" to the role string
  * ```hasAuthority( String role )``` - has a specific role, (does not prepend "ROLE_")
  * ```hasAnyRole( String role ...)``` - multiple roles with OR used to join roles (prepends "ROLE_" to each role string)
  * ```hasAnyAuthority( String role ...)``` - multiple roles with OR used to join roles
  * ```hasRole(FOO)``` AND ```hasRole(BAR)``` - having multiple roles
  * ```anonymous()``` - unauthenticated
  * ```authenticated()``` - not anonymous
  * ```fullyAuthenticated()``` - logged in, credentials verified, and identity confirmed and were not "remembered"
  * ```rememberMe()``` - allowed by users that have been remembered
    * Remember-me or persistent-login authentication refers to web sites being able to remember the identity of a 
    principal between sessions

### xml config
* In xml, authorization rules are defined in a separate, spring specific file which uses a special 
namespace: ```spring-security```
* ```<intercerpt-url pattern="" access=""/>``` is used to define the URLs to secure:
```xml
<beans:beans xmlns="http://www.springframework.org/schema/security"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:beans="http://www.springframework.org/schema/beans"
  xsi:schemaLocation="http://www.springframework.org/schema/security
  http://www.springframework.org/schema/security/spring-security.xsd
  http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans.xsd">

    <http>
      <intercept-url pattern="/auth*" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
      <intercept-url pattern="/users/**" access="IS_AUTHENTICATED_FULLY"/>
      <!-- define the URL for login form  -->
      <form-login login-page="/auth" />
      <!-- define the URL for logout  -->
      <logout logout-url="/logout" />
    </http>
</beans:beans>
```
* XML config also allows use of SpEL in the ```access``` attribute
  * have to enable it via ```use-expressions="true"``` in http element
  * Once the support for SpEL expression has been configured using use-expressions="true" , the previous
    syntax for access values cannot be used, so roles and configuration attributes cannot be used as values for the
    access attribute directly. So mixing the two ways is not possible.
  ```xml
  <beans:beans ...>

    <!-- enable method level security AND provide a security pointcut expression -->
    <global-method-security>
      <protect-pointcut expression="execution(* com.ps.*.*Service.findById(*))" access="hasRole(’ADMIN’)" />
    </global-method-security>

    <http use-expressions="true">
      <intercept-url pattern="/auth*" access="permitAll"/>
      <intercept-url pattern="/users/edit" access="hasRole(’ROLE_ADMIN’)"/>
      <intercept-url pattern="/users/list" access="hasRole(’ROLE_USER’)"/>
      <intercept-url pattern="/users/**" access="hasAnyRole(’ROLE_USER, ROLE_ADMIN’)"/>
    </http>

    <!-- can configure credentials and authorities as well, read into in InMemory DaoAuthenticationProvider -->
    <authentication-manager>
      <authentication-provider>
        <user-service>
            <user name="john" password="doe" authorities="ROLE_USER"/>
            <user name="jane" password="doe" authorities="ROLE_USER,ROLE_ADMIN"/>
            <user name="admin" password="admin" authorities="ROLE_ADMIN"/>
        </user-service>
      </authentication-provider>
    </authentication-manager>
  </beans:beans
  ```
* OR rather than hardcode user credentials in your XML config file, you can read them from a properties file.
```xml
<!-- spring-config.xml -->
<authentication-manager>
  <authentication-provider>
    <password-encoder hash="md5" >
     <salt-source system-wide="SpringSalt"/>
    </password-encoder>
    <user-service properties="/WEB-INF/users.properties" />
  </authentication-provider>
</authentication-manager>
```
  * The credentials property file has a specific syntax:
    * [username] = [password(encrypted)],[role1,role2...][,enabled||disabled]
    * e.g.  jane=1f533ad8d26c7bec84a291f62668a048,ROLE_USER,ROLE_ADMIN  
      * Note that the password is appended with "SpringSalt" in the above config and then stored encrypted using md5

## Authentication

### Standard Authentication Scenario (and how Spring Security handles it)
1. A user is prompted to login with a username and password
  * username and password are obtained and combined into an instance of ```UsernamePasswordAuthenticationToken``` (an 
    instance of the Authentication interface)
2. The system (successfully) verifies the password is correct for the username
  * The token is passed to an instance of ```AuthenticationManager``` for validation
3. The context information for that user us obtained (their list of roles etc..)
  * The ```AuthenticationManager``` returns a fully populated ```Authentication``` instance on successful authentication
4. A security context is established for the user
  * The ```SecurityContext``` is established by calling SecurityContextHolder.getContext().setAuthentication(..) passing
  in the returned ```Authentication``` object
  * From that point on, the user is considered to be authenticated
5. The user proceeds to interact with the system, possibly performing an operation that is protected by an access
control mechanism, which checks the required permissions for the operation against the security context information

### AuthenticationManager
* an interface that processes ```Authentication``` requests and attempts to validate them
* The default implementation is called ```ProviderManager``` and it will delegate to a list of configured 
```AuthenticationProviders```, each of which is queried in turn to see if it can perform the authentication
* each provider will either throw an exception or return a fully populated ```Authentication``` object
  * the most common approach is to load the corresponding ```UserDetails``` and check the loaded password against the one
  entered by the user. This is the approach used by ```DaoAuthenticationProvider```
  * The loaded ```UserDetails``` object and the ```GrantedAuthority```(s) it contains, will be used when building the fully populated
  ```Authentication``` object which is returned from a successful authentication and stored in the ```SecurityContext```


### AuthenticationProvider
* interface that indicates a class can process a specific ```Authentication``` implementation
* Processes authentication requests and returns a ```Authentication``` object
* Default is ```DaoAuthenticationProvider```
  * An AuthenticationProvider implementation that retrieves user details from a UserDetailsService
    * it leverages a UserDetailsService (as a DAO) in order to lookup the username,password and GrantedAuthority(s)
    * It authenticates the user simply by comparing the password submitted in a ```UsernamePasswordAuthenticationToken``` 
  against the one loaded by the ```UserDetailsService```
* **UserDetailsService**
  * an interface that provides getters that guarantee non-null provision of authentication information such as the 
  username, password, granted authorities and whether the user account is enabled or disabled
  * single method interface - ```UserDetails loadUserByUsername(String username) throws UsernameNotFoundException```
  * implemented by most AuthenticationProviders
  * Spring provides some base implementations of ```UserDetailsService```:
    * In-Memory Authentication
    * JdbcDaoImpl
      * obtains authentication information from a JDBC datasource
      * A default database schema is assumed, with two tables "users" and "authorities"
      * If you are using an existing schema you will have to set the queries usersByUsernameQuery and 
      authoritiesByUsernameQuery to match your database setup  
    * LdapUserDetailsService
    * Possible to build custom implementation
* Can provide different authentication configuration based on @Profiles - in-memory for development, JDBC for prod. etc.

#### In-memory authentication provider:
```java
@Configuration
@EnableWebSecurity
public class HelloWebSecurityConfiguration extends WebSecurityConfigurerAdapter {

  @Autowired
  public void configureGlobal(AuthenticationManagerBuilder authManagerBuilder) throws Exception  {
      authManagerBuilder.inMemoryAuthentication().withUser("alice").password("letmein").roles("USER").and()
                                                 .withUser("bob").password("12345").roles("ADMIN").and();
  }
}
```
```xml
<authentication-manager>
      <authentication-provider>
        <user-service>
            <user name="john" password="doe" authorities="ROLE_USER"/>
            <user name="jane" password="doe" authorities="ROLE_USER,ROLE_ADMIN"/>
            <user name="admin" password="admin" authorities="ROLE_ADMIN"/>
        </user-service>
      </authentication-provider>
</authentication-manager>
```

#### JDBC Authentication provider
* Authenticates against DB
* Can customize queries using ```.usersByUsernameQuery(), .authoritiesByUsernameQuery(), groupAuthoritiesByUsername()```
* Otherwise default queries will be used
```java
@Configuration
@EnableWebSecurity
public class HelloWebSecurityConfiguration extends WebSecurityConfigurerAdapter {

  @Autowired
  private DataSource dataSource;

  @Autowired
  public void configureGlobal(AuthenticationManagerBuilder authManagerBuilder) throws Exception {
      authManagerBuilder.jdbcAuthentication().dataSource(dataSource)
                                             .usersByUsernameQuery(...)
                                             .authoritiesByUsernameQuery(...)
                                             .groupAuthoritiesByUsername(...);
  }
}
```
```xml
<bean id="authDataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="org.hsqldb.jdbcDriver"/>
    <property name="url" value="jdbc:hsqldb:hsql://localhost:9001"/>
    <property name="username" value="sa"/>
    <property name="password" value=""/>
</bean>

<!-- this is in your spring security XML configuration -->
<authentication-manager>
  <authentication-provider>
    <jdbc-user-service data-source-ref="authDataSource" />
  </authentication-provider>
</authentication-manager>
```

### Password Encoding
* Supports password hashing (md5, sha, ...)
* Supports password salting - adding string to password before hashing - prevents decoding passwords using
rainbow tables
```java
@Configuration
@EnableWebSecurity
public class HelloWebSecurityConfiguration extends WebSecurityConfigurerAdapter {

  @Autowired
  private DataSource dataSource;

  @Autowired
  public void configureGlobal(AuthenticationManagerBuilder authManagerBuilder) throws Exception {
      authManagerBuilder.jdbcAuthentication().dataSource(dataSource)
                                             .passwordEncoder(new StandardPasswordEncoder("this is salt"));
  }
}
```
```xml
<authentication-manager>
  <authentication-provider>
    <password-encoder hash="md5" >
      <salt-source system-wide="SpringSalt"/>
    </password-encoder>
    <user-service properties="/WEB-INF/users.properties" />
  </authentication-provider>
</authentication-manager>
```

### Login and Logout
```java
@Configuration
@EnableWebSecurity
public class HelloWebSecurityConfiguration extends WebSecurityConfigurerAdapter {

  @Override
  protected void configure(HttpSecurity httpSecurity) throws Exception {
      httpSecurity.authorizeRequests().formLogin().loginPage("/login.jsp") .permitAll()
                                      .and()
                                      .logout().permitAll();
  }
}
```

```jsp
<c:url var="loginUrl" value="/login.jsp" />
<form:form action="${loginUrl}" method="POST">
    <input type="text" name="username"/>
    <input type="password" name="password"/>
    <input type="submit" name="submit" value="LOGIN"/>
</form:form>
```

## Spring Security Tag Libraries
* Available since Spring Security 2.0
* Add taglib to JSP
```jsp <%@ taglib prefix="security" uri="http://www.springframework.org/security/tags" %> ```
* Facelet tags for JSF also available
* Displaying properties of Authentication object - ```<security:authentication property=“principal.username”/>```
* Display content only if principal has certain role
```html
<security:authorize access="hasRole('ADMIN')">
    <p>Admin only content</p>
</security:authorize>
```
* Or inherit the role required from specific URL (roles required for specific URLs are centralized in config and not
across many JSPs)
```html
<security:authorize url="/admin">
    <p>Admin only content</p>
</security:authorize>
```

## Method Security
* Methods (e.g. service layer) can be secured (uses AOP)
  * JSR-250
  * Spring annotations
  * pre/post authorize
* Can be configured in XML using <global-method-security ..>
  * You should only declare one <global-method-security> element.
  * also allows you to define a pointcut expression that can protect multiple methods
    ```xml
    <global-method-security>
        <protect-pointcut expression="execution(* com.mycompany.*Service.*(..))" access="ROLE_USER"/>
    </global-method-security>
    ```
* @EnableGlobalMethodSecurity
  * used with @Configuration in Java configuration
    * good practice is to annotate security configuration class to keep annotations in one place
  * Enables Spring Security global method security
    * can then secure methods in other spring managed beans
  
### JSR-250 annotations
* **Only supports role based security**
* @EnableGlobalMethodSecurity( jsr250enabled=true ) on @Configuration to enable
* On method level @RolesAllowed( "ROLE_ADMIN" )
* To enable use of JSR-250 annotations via XML config use:
```xml
<global-method-security jsr250-annotations="enabled" />
```

### Spring @Secured Annotations
* @EnableGlobalMethodSecurity( securedEnabled=true ) on @Configuration to enable
* @Secured("ROLE_ADMIN") on method level
* Supports not only roles - e.g. @Secured("IS_AUTHENTICATED_FULLY")
* **SpEL not supported**
* To enable use of @Secured annotations via XML config use:
```xml
<global-method-security secured-annotations="enabled" />
```
```java
public interface BankService {

    @Secured("IS_AUTHENTICATED_ANONYMOUSLY")
    public Account readAccount(Long id);
    
    @Secured("IS_AUTHENTICATED_ANONYMOUSLY")
    public Account[] findAccounts();
    
    @Secured("ROLE_TELLER")
    public Account post(Account account, double amount);
}
```


### Pre/Post authorize
* @EnableGlobalMethodSecurity( prePostEnabled=true ) on @Configuration to enable
* Pre authorize - can use SpEL (@Secured cannot), checked before annotated method invocation
  * @PreAuthorize( "hasRole( 'ROLE_ADMIN' )")
* Post authorize - can use SpEL, checked after annotated method invocation, **can access return object of the method**
using **returnObject** variable in SpEL, If expression resolves to false, return value is not returned to the caller.
* To enable use of pre/post annotations via XML config use:
```xml
<global-method-security pre-post-annotations="enabled" />
```

PreAuthorize example - where a user is only allowed to affect a user whose username matches that of the ```user```
argument. i.e. a user can only modify their own profile
```java
@Service
@Transactional(readOnly = true, propagation = Propagation.REQUIRED)
public class UserServiceImpl implements UserService {

  @PreAuthorize("#user.username == authentication.name")
  public void modifyProfile(User user){
    //...
  }
}
```




Spring Boot
==================================================================================================================
## Basics
* Convention over configuration
  * Pre-configures Spring app with reasonable defaults, which can be overridden
* Maven and Gradle integration
* MVC is enabled by having spring-boot-starter-web as a dependency
  * Registers dispatcher servlet
  * Does the same as @EnableWebMvc + more
  * Automatically serves static resources from /static, /public, /resources or /META-INF/resources
  * Templates are served from /templates (Velocity, FreeMarker, Thymeleaf)
  * Registers BeanNameViewResolver if beans implementing View interface are found
  * Registers ContextNegotiatingViewResolver, InternalResourceViewResolver (prefix and suffix are configurable
  in application.properties)
  * Customization of auto-configured beans should be done using WebMvcConfigurerAdapter as usual

## @SpringBootApplication
* Main Class annotated with **@SpringBootApplication**, can be run in a jar with embedded application server (Tomcat by
default, can be changed to Jetty or Undertow). Can also be deployed to an application server as a .war file
* Actually consists of three Annotations: @Configuration,@EnableAutoConfiguration and @ComponentScan
* @SpringBootApplication properties:
  * *exclude* - Class[] - exclude specific auto-configuration classes 
  * *excludeName* - String[] - ""
  * *scanBasePackageClasses* - Class[] - specify packages to scan for
  * *scanBasePackages* - String[] - ""
    * * ```@SpringBootApplication( scanBasePackages = {"com.ps.web", "com.ps.start"})```
* **@EnableAutoConfiguration** 
  * configures modules based on presence of certain classes on the classpath
  * based on **@Conditional** - introduced in Spring 4.0
* Default ComponentScan package is current package plus all sub-packages
* Manually declared beans usually override beans automatically created by AutoConfiguration
  * **@ConditionalOnMissingBean** is used, usually the bean's type and not its name matters
* Can selectively exclude some AutoConfiguration classes
  * ```@EnableAutoConfiguration( exclude=DataSourceAutoConfiguration.class )```


## Dependencies
* Need to add proper maven parent and dependencies
* When using Maven:
  * Spring boot provides *starter* POMs which simplify the configuration of a project
  * must have ```spring-boot-starter-parent``` as a parent so that useful maven defaults are provided
  * version information is provided in the starter-parent and spring will automatically import compatible
  versions for the other child starters in your POM
* Using "starter" module dependencies --> using transitive dependencies bundles versions which are tested to work
well together
  * Can override version of specific artifacts in pom.xml
  ```xml
  <properties>
    <spring.version>4.2.0.RELEASE</spring.version>
  </properties>
  ```
* spring-boot-starter, spring-boot-starter-web, spring-boot-starter-test,...
* Parent pom defines all the dependencies using dependency management, specific version are not defined in or pom.xml
* Only version which needs to be specifically declared is the parent pom version
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>

<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>1.3.0.RELEASE</version>
</parent>
```

## Application Configuration
* Application configuration is externalized by default to **application.properties** file
* Alternatively, can use YAML configuration - **application.yml** by default
* Located in 
    1. *working directory/config*
    2. *working directory*
    3. *classpath/config*
    4. *classpath* root
  * this list is ordered by precedence, properties in upper positions override those in lower positions
* can provide application.properties location on the command line using ```spring.config.location```
  * ```java -jar ps-boot.jar --spring.config.location=/Users/iuliana.cosmina/temp/application.properties```
* can provide different file name too, using ```spring.config.name```
  * ```java -jar ps-boot.jar --spring.config.name=my-boot.properties```
* ```PropertySource``` automatically created based on the properties files found

## Logging
* Spring Boot uses *Commons Logging* internally by default, but leaves the underlying implementation open.
* Log4J2, Java Util and logback are supported
* By default uses Logback running over SLF4J
  * looks in  ```/src/main/resources``` for logging configuration file
  * By default logs to console, but can define a log file
  * By default, ERROR, WARN, INFO messages are logged
  * Spring will look for logging configuration files named:
    * For logback: 
      * logback-spring.xml, logback-spring.groovy, logback.xml, logback.groovy
    * For Log4j2:
      * log4j2-spring.xml, log4j2.xml
    * logging.properties for Java Util logging
* It is NOT possible to control/configure logging via @PropertySources in @Configuration classes. System properties 
and external Spring Boot configuration files should be used
* the logfile name to use be default can be configured using the ```logging.file``` Spring environment variable
```
#Logging through SLF4J
logging.level.org.springframework=DEBUG
logging.level.com.example=INFO

logging.file=logfile.log
#OR spring.log file in to configured path
logging.path=/log
```
To change logging framework from default logback
1. Exclude logback dependency ```ch.qos.logback.logback-classic```
2. Add desired logging framework dependency - eg. ```org.slf4j.slf4j-log4j12```

### DataSource
* either include spring-boot-starter-jdbc or spring-boot-starter-data-jpa
* JDBC driver required on classpath, datasource will be created automatically
* Tomcat JDBC as default pool, other connection pools can be used if present - eg. HikariCP
```
#Connection
spring.datasource.url=
spring.datasource.username=
spring.datasource.password=
spring.datasource.driver-class-name=

#Scripts to be executed
spring.datasource.schema=
spring.datasource.data=

#Connection pool
spring.datasource.initial-size=
spring.datasource.max-active=
spring.datasource.max-idle=
spring.datasource.min-idle=
```

### Web Container
```
server.port=
server.address=
server.session.timeout=
server.context-path=
server.servlet-path=
```
[A]Web container can be configured in Java dynamically by implementing EmbeddedServletContainerCustomizer interface
and registering resulting class as a @Component
```java
@Component
public class PropertiesConfBean implements EmbeddedServletContainerCustomizer {
    @Value("${app.port}")
    private Integer value;
    @Value("${app.context}")
    private String contextPath;
    
    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        container.setPort(value);
        container.setContextPath(contextPath);
    }
}
```
Or if needing more fine-grained configuration - declare bean of type ```EmbeddedServletContainerFactory```

### YAML
* YAML - Yaml Ain't a Markup Language
* Alternative to configuration in .properties file
* ```snakeyaml``` dependency must be added to the project
* Configuration can be hierarchical
```yaml
server:
    port:
    address:
    session-timeout:
    context-path:
    servlet-path:
```
* YAML can contain configuration based on spring profiles
  * Divided by ---
  ```yaml
  ---
  spring.profiles: development
  database:
    host: localhost
    user: dev
  ---
  spring.profiles: production
  database:
    host: 198.18.200.9
    user: admin
  ```
* Each profile can have its own dedicated file
  * ```application-{profilename}.yml```


### @ConfigurationProperties
* Class is annotated with ```@ConfigurationProperties( prefix="com.example" )```
* creates a bean where fields of the class are automatically injected with values from the properties
* does type conversion of the properties
* @ConfigurationProperties(prefix= "com.example") + com.example.foo → foo field injected
```java
@ConfigurationProperties(prefix="app")
public class AppSettings {
    
    private static Logger logger = LoggerFactory.getLogger(AppSettings.class);
    @NotNull
    private Integer port;  //app.port will be injected from properties file with type comversion
    
    @NotNull
    private String context; //app.context will be injected
    
    @NotNull
    private Integer sessionTimeout;  //app.session.timeout will be injected
    
    public AppSettings() { }
    
    @PostConstruct
    public void check() {
        logger.info("Initialized {} {}", port, context);
    }
    //getters & setters MUST also be defined
}
```
* Needs to be enabled on a @Configuration class with **@EnableConfigurationProperties**
  * ```@EnableConfigurationProperties( MyProperties.class )```
    * Convenience annotation to enable support for **ConfigurationProperties** annotated beans
      * Configuration properties bean will be given a name of the form: <prefix>-<fqn>
        * eg.  ```app-com.ps.MyProperties```
    * if this annotation is not used, then the AppSettings class (above) can be annotated with @Configuration
    and Spring Boot will properly create the bean and load the values from the ```Environment```
* Use the property values by injecting them where needed
    ```java
    @RestController
    @SpringBootApplication(scanBasePackages = {"com.ps.start"})
    @EnableConfigurationProperties(AppSettings.class)
    public class Application extends SpringBootServletInitializer {
        //...
        @Autowired
        private AppSettings appSettings;
      
        @Bean
        public EmbeddedServletContainerFactory servletContainer() {
            TomcatEmbeddedServletContainerFactory
            factory = new TomcatEmbeddedServletContainerFactory();
            factory.setPort(appSettings.getPort());
            factory.setSessionTimeout(
            appSettings.getSessionTimeout(), TimeUnit.MINUTES);
            factory.setContextPath(appSettings.getContext());
            return factory;
        }
    //....
    }
    ```
* Can also use **@ConfigurationProperties** on public @Bean methods. This can be particularly useful when you want 
to bind properties to third-party components that are outside of your control.
```java
@ConfigurationProperties(prefix = "bar")
@Bean
public BarComponent barComponent() {
    //...
}
```
  * Any property defined with the bar prefix will be mapped onto that BarComponent bean


## Embedded Container
* Spring Boot can run embedded application server from a jar file
* ```spring-boot-starter-web``` includes embedded Tomcat, can change to Jetty, or Undertow
  1. include ```spring-boot-starter-jetty``` as a dependency
  2. exclude spring-boot-starter-tomcat dependency from ```spring-boot-starter-web```
* Embedded app server is recommended for cloud native applications
* Spring boot can produce jar or war
  1. change packaging to ```war``` (in your pom)
  2. extend ```SpringBootServletInitializer```
  3. override configure method
      ```java
      @SpringBootApplication
      public class Application extends SpringBootServletInitializer {
        //SpringBootServletInitializer is an opinionated WebApplicationInitializer to run a 
        // SpringApplication from a traditional WAR deployment
        
        public static void main(String[] args) {
          SpringApplication.run(Application.class, args);
        }
      
        //run as war, needs to have <packaging>war</packaging> in pom.xml
        @Override
        protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
          return application.sources(Application.class);
        }
      }
      ```
  * war can still be executed ```java -jar myapp.war``` or deployed to app server
* when using spring-boot-maven-plugin, ```mvn package``` produces two jars
  * traditional jar without dependencies
  * executable jar with all dependencies included
* Both war and jar with embedded app server are valid options

## @Conditional annotations
* Enables bean instantiation only when specific condition is met
* Introduced in Spring 4.0
* Core concept of Spring Boot
* If a @Configuration class is marked with @Conditional, all of the @Bean methods, @Import annotations,
and @ComponentScan annotations associated with that class will be subject to the conditions
* @ConditonalOnBean, @ConditionalOnMissingBean
  * Can be used on class or method level
  * Using these conditions at the class level is equivalent to marking each contained @Bean method with the annotation
  * Only when specific bean is found - ```@ConditionalOnBean( type={"Datasource"} )```
    * if *type* or *name* attribute is specified, then condition matches when any of the classes specified is
    contained in the ApplicationContext
  ```java
  @Configuration
   public class MyAutoConfiguration {
  
       //defaults to the return type of this @Bean method: MyService
       //the condition will match if a bean of type MyService is already contained in the BeanFactory
       @ConditionalOnBean
       @Bean
       public MyService myService() {
           ...
       }
  
   }
  ```
  * Only when specific bean is not found - ```@ConditionalOnMissingBean```
  ```java
    @Configuration
     public class MyAutoConfiguration {
    
         //defaults to the return type of this @Bean method: MyService
         //the condition will match if a bean of type MyService is NOT already contained in the BeanFactory
         @ConditionalOnMissingBean
         @Bean
         public MyService myService() {
             ...
         }
    
     }
    ```
  * NOTE: recommended to use @ConditionalOnBean/@ConditionalOnMissingBean in auto-configuration classes as 
  those are guaranteed to load after any user-defined bean definitions
* Only when specific class is on the classpath - ```@ConditionalOnClass```
* Only when specific class in not present on classpath - ```@ConditionalOnMissingClass```
* Only when system property has certain value - ```@ConditionalOnProperty( name="server.host",havingValue="localhost" )```
* @Profile is a special case of conditional
* org.springframework.boot.autoconfigure contains a lot of conditionals for auto-configuration
  * eg. @ConditionalOnMissingBean(Datasource.class) --> creates embedded data source
  * Specifically declared beans usually disable automatically created ones
  * If needed, specific auto-configuration classes can be excluded explicitly
    * ```@EnableAutoConfiguration( exclude=DataSourceAutoConfiguration.class )```

## Testing
* **@SpringBootTest**
  * use on a test class that runs spring-boot based integration tests
  * if no *ContextLoader* is specified with ```@ContextConfiguration``` it defaults to ```SpringBootContextLoader```
  * Automatically searches for a @SpringBootConfiguration when nested @Configuration is not used, and no explicit classes are specified
  * Can add environment specific properties via the **properties** attribute
    ```java
    @RunWith( SpringRunner.class )
    @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    properties = {"app.port=9090"})
    public class CtxControllerTest {
      //...
    }
    ```
  * can define different web environment modes and start a fully running container on a random port using 
  **webEnvironment** attribute
  * registers a TestRestTemplate bean for use in web tests that use a fully running container
  
* Example controller and Spring Boot Test for the controller:
```java
@Controller
public class CtxController {
    public static final String INTRO = "Hello there dear developer, here are the beans you were looking for: </br>";

    @Autowired
    ApplicationContext ctx;
    
    @RequestMapping("/")
    @ResponseBody
    public String index() {
        StringBuilder sb = new StringBuilder("<html><body>");
        sb.append(INTRO);
        String beanNames = ctx.getBeanDefinitionNames();
        Arrays.sort(beanNames);
        for (String beanName : beanNames) {
            sb.append("</br>").append(beanName);
        }
        sb.append("</body></htm>");
        return sb.toString();
    }

    @RequestMapping("/home")
    public String home(ModelMap model) {
        model.put("bogus", "data");
        return "home";
    }
}
```
Spring Test for the controller:
```java
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.ui.ModelMap;
//...
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, properties = {"app.port=9090"})
public class CtxControllerTest {
    
    @Autowired
    CtxController ctxController;
    
    private ModelMap model = new ModelMap();
    
    @Test
    public void testIndex() {
        String result = ctxController.index();
        assertNotNull(result);
        assertTrue(result.contains(CtxController.INTRO));
    }
    
    @Test
    public void testHome() {
        String result = ctxController.home(model);
        assertEquals("home", result);
        String modelData = (String)model.get("bogus");
        assertEquals("data", modelData);
    }
}
```
* @WebMvcTest
  * Can be used when a test focuses only on Spring MVC components.


Micro-services in Spring
================================================================
## Microservices
* Classic monolithic architecture
  * Single big application
  * Single persistence (usually a relational DB)
  * Easier to build
  * Harder to scale up
  * Complex and large deployment process
* Microservices architecture
  * Each microservice has its own storage best suited for it (RDMS, Non-RelationalDB, Document Store,...)
  * Each microservice can be in a different language with different frameworks used, best suited for its purpose
  * Each microservice has separate development, testing and deployment
  * Each microservice can be developed by separate team
  * Harder to build
  * Easier to extend and maintain
  * Easier to scale up
  * Easier to deploy or update just one microservice instead of the whole monolith
* One microservice usually consits of several services in the service layer and corresponding data store
* Microservices are well suited to run in the cloud
  * Easily manage scaling - number of microservice instances
  * Scaling can be done dynamically
  * Provided load balancing across instances
  * Underlying infrastructure agnostic
  * Isolated containers
* Apps deployed to the cloud don't necessarily have to be microservices
  * Enables to gradually change existing apps to microservice architecture
  * New features of monolith can be added as microservices
  * Existing features of monolith can be refactored to microservices one by one

## Spring Cloud
* Provides building blocks for building Cloud and microservice apps
  * Intelligent routing among multiple instances
  * Service registration and discovery
  * Circuit breakers
  * Distributed configuration
  * Distributed messaging
* Independent of specific cloud environment - Supports Heroku, AWS, Cloud Foundry, but can be used without any cloud
* Built on Spring Boot
* Consists of many sub-projects
  * Spring Cloud Config
  * Spring Cloud Security
  * Spring Cloud Connectors
  * Spring Cloud Data Flow
  * Spring Cloud Bus
  * Spring Cloud for Amazon Web Services
  * Spring Cloud Netflix
  * Spring Cloud Consul
  * ...

## Building Microservices
1. Create discovery service
2. Create a microservice and register it with Discovery Service
3. Clients use microservice using "smart" RestTemplate, which manages load balancing and automatic service lookup

### Discovery Service
* Spring Cloud supports two discovery services
  * Eureka - by Netflix
  * Consul.io by Hashicorp (authors of Vagrant)
  * Both can be set up and used for service discovery easily in Spring Cloud
  * Can be switched easily later on
* On top of regular Spring Cloud and Boot dependencies (spring-cloud-starter, spring-boot-starter-web,
spring-cloud-starter-parent for parent POM), dependency for specific Discovery Service server needs to be provided (eg,
spring-cloud-starter-eureka-server)
* Annotate @SpringBootApplication with **@EnableEurekaServer**
* Provide Eureka config to application.properties (or application.yml)
```
server.port=8761
#Not a client
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

logging.level.com.netflix.eureka=OFF
logging.level.com.netflix.discovery=OFF
```

### Microservice Registration
* Annotate @SpringBootApplication with **@EnableDiscoveryClient**
* Each microservice is a spring boot app
* Fill in application.properties with app name and eureka server URL
  * ```spring.application.name``` is name of the microservice for discovery
  * ```eureka.client.serviceUrl.defaultZone``` is URL of the Eureka server
  * Config Example:
  ```properties
  # Service registers under this name
  spring.application.name=users-service

  # Discovery Server Access
  eureka.client.registerWithEureka=true
  eureka.client.fetchRegistry=false
  eureka.client.serviceUrl.defaultZone=http://localhost:3000/eureka/
  ```

### Microservice Usage
* Annotate client app using the microservice (@SpringBootApplication) with @EnableDiscoveryClient
* Clients call Microservice using "smart" RestTemplate, which manages load balancing and automatic service lookup
* Inject RestTemplate using @Autowired with additional **@LoadBalanced**
* "Ribbon" service by Netflix provides load balancing
* RestService automatically looks up microservice by logical name and provides load-balancing
```
@Autowired
@LoadBalanced
private RestTemplate restTemplate;
```
Then call the specific Microservice with restTemplate
```restTemplate.getForObject("http://persons-microservice-name/persons/{id}", Person.class, id);```




Summaries
======================================================

## Spring 4.0 Misc.
* JDK Versions:
  * minimum: JDK 6 update 18
  * recommended: Java 7 or 8
* JavaEE
  * can deploy a Spring 4 application into a servlet 2.5 container
  * recommended: javaEE 6+ or JavaEE 7, with the JPA 2.0+ and Servlet 3.0+ specifications being of particular relevance
* Unit/Integration Testing Versions
  * spring 4 supports minimum of Junit 4 as well as TestNG
  
### Improvements in Spring 4.0
* generic types can be used as a qualifier when injecting beans ```@Autowired Repository<Customer> customerRepository```
* Beans can be ordered when autowired into lists and arrays. @Ordered
* @Lazy can be used on injection points
* @Description added
* @Conditional added
* CGLIB proxy classes no longer require a default constructor
* managed timezone support across the framework
* @RestController added
* AsyncRestTemplate added
* WebSocket, SockJS, and STOMP messaging added

### 4.2
* Configuration classes may declare @Order attribute
* @AliasFor

### Improvements in 4.3
* No longer need @Autowired if target bean only has one constructor
* @Configuration classes support constructor injection
* Built on support for HTTP HEAD and HTTP OPTIONS
* @GetMapping,@PutMapping,@PostMapping,@DeleteMapping,@PatchMapping
* @ResponseStatus at class level
* JUnit support in TestContext Framework requires JUnit 4.12 or higher

  
## Annotations used in Spring Framework Testing
* @RunWith(SpringJUnit4ClassRunner.class)
  * creates a spring application context, need to load configurations with @ContextConfiguration
    * if no value provided, try to import from **${classname}-context.xml** in the same package
    * load Java configurations (use classes attribute)
        * @ContextConfiguration(classes={TestConfig.class, OtherConfig.class})
    * load XML configs
        * @ContextConfiguration({"classpath:com/example/test-config.xml","classpath:com/web/controller-config.xml"})
    * Testing with a WebApplicationContext:
        * @WebAppConfiguration("resourcePathToRootOfWebApp") - initializes a web application context
        * defaults to "file:src/main/webapp" - the base resource path for the WebApplicationContext
        * must be used with @ContextConfiguration
* Testing with transactions
  * mark class or methods with @Transactional
  * @Rollback(false) or @Commit - default policy for test methods is to rollback after method finishes 
* @ActiveProfiles({"foo","bar"}) - activates the listed profiles
* @TestPropertySource( locations={"classpath:/com/example/test.properties"}, properties={"username=foo"} )
  * overrides existing property of the same name
  * default location is **[classname].properties**
* Testing with in-memory DB
  * DB initialized with @Sql annotation
    * if no value specified, defaults to **ClassName.methodName.sql**
    * can be on class/method level
  * @Sql({"/sql/foo.sql", "/sql/bar.sql"})
  * @Sql(scripts="/sql/foo.sql", executionPhase=Sql.ExecutionPhase.AFTER_TEST_METHOD )

  
## Annotations and XML equivalents
* AOP
  * @EnableAspectJAutoProxy
  * ```<aop:aspectj-autoproxy />```
* Caching
  * @EnableCaching on @Configuration class level
  * ```<cache:annotation-driven />```
* Transactions
  * @EnableTransactionManagement
  * ```<tx:annotation-driven />```
* JPA
  * @PersistenceUnit
    * used on a field or setter method only
    * injects a ```EntityManagerFactory```
  * @PersistenceContext
    * used in a field or setter method only
    * injects a thread-safe proxy to a real ```EntityManager```
  * To enable support for these annotations you must register a ```PersistenceAnnotationBeanPostProcessor``` bean
    * In xml config this is done if you use ```<context:annotation-driven /> or <context:component-scan />```

* Spring Data JPA
  * @EnableJpaRepositories( basePackages="com.example.**.repository" )
  * ```<jpa:repositories base-package="com.example.**.repository" />```
* Spring MVC
  * @EnableWebMvc
    * introduced in Spring 3.1
    * On a @Configuration class that implements WebMvcConfigurer or you can extend WebMvcConfigurerAdapter
    * registers many infrastructure components needed for a web application 
  * ```<mvc:annotation-driven />```
* Security
  * @EnableWebSecurity on @Configuration class, config class should extend WebSecurityConfigurerAdapter
  * xml config:
    * config authorization rules, authentication provider, in an .xml file that uses the **spring-security** namespace
    * then configure your web.xml to load that file into your RootApplicationContext via **contextConfigLocation** param
* Method level security
  * @EnableGlobalMethodSecurity( jsr250enabled=true ) on @Configuration
    * ```<global-method-security jsr250-annotations="enabled" />```
  * @EnableGlobalMethodSecurity( securedEnabled=true ) on @Configuration
    * ```<global-method-security secured-annotations="enabled" />```
  * @EnableGlobalMethodSecurity( prePostEnabled=true ) on @Configuration
    * ```<global-method-security pre-post-annotations="enabled" />```
* JMS
  * @EnableJms, @JmsListener
    * ```<jms:annotation-driven />```


* Spring Boot
  * @SpringBootApplication - consists of three annotations
    * @Configuration
    * @EnableAutoConfiguration
      * to exclude classes: @EnableAutoConfiguration( exclude=DataSourceAutoConfiguration.class )
    * @ComponentScan
  * @Conditional
    * introduced in spring 4.0
    * Core concept of Spring Boot
    * Only when specific bean is found - @ConditionalOnBean( type={Datasource.class} )
    * Only when specific bean is not found - @ConditionalOnMissingBean
    * Only when specific class is on the classpath - @ConditionalOnClass
    * Only when specific class in not present on classpath - @ConditionalOnMissingClass
    * Only when system property has certain value - @ConditionalOnProperty( name="server.host",havingValue="localhost" )
  * @ConfigurationProperties
* Spring Boot Testing annotations
    * @SpringBootTest
      * used along with @RunWith(SpringRunner.class)
      * used to start a spring boot integration test, and can start an embedded application server
        * @SpringBootTest( webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, properties={"app.port=9090"}
      * can also register a TestRestTemplate
    * @SpringBootConfiguration( classes=MyApplication.class )

    * @WebMvcTest
      * can be used to test only Spring MVC components
      * does not load the full ApplicationContext
      * Typically @WebMvcTest is used in combination with @MockBean or @Import to create any collaborators required
      by your @Controller beans

* JMX
  * @ManagedResource(description = "Sample JMX managed resource", objectName="bean:name=jmxCounter")
    * class level annotation
    * *objectName* contains name of resulting MBean, if empty it is derived from fully qualified class name
  * @ManagedOperation
    * method level - marks method as a JMX operation
  * @ManagedAttribute
    * method level - marks a getter/setter as one half of a jmx attribute declaring a managed resource
  * @EnableMBeanExport
    * enables automatic exporting of all standard MBeans from the Spring Context, as well as all @ManagedResource beans
    * no need to configure an exporter bean explicitly


## Configuring JPA
1. Configure an EntityManagerFactoryBean
    * can lookup via JNDI
    * OR can be configured in Spring using:
      * LocalEntityManagerFactoryBean
        * uses persistence.xml for most of (all of) its configuration
        * only need to setPersistenceUnitName() on this bean
      * LocalContainerEntityManagerFactoryBean (preferred)
        * can figure all persistence information on the bean
        * can scan packages for @Entity classes
        * Configure a JpaVendorAdapter bean with the specific vendor implementation you are using
          * Spring provides several:
            * HibernateJpaVendorAdapter
            * EclipseLinkJpaVendorAdapter
            * OpenJpaVendorAdapter
        * Define a dataSource Bean
        * Define a PlatformTransactionManager Bean
          * usually JpaTransactionManager
          * JtaTransactionManager
          * HibernateTransactionManager

2. Declare a PersistenceExceptionTranslationPostProcessor bean so that JPA specific exceptions are automatically
translated into Spring DataAccessException(s)
    * exception translation only happens in @Repository classes
3. Define your @Entity classes
4. Define your DAOs


## Configuring Spring MVC
### General Configuration Steps
1. Define a root web application context
2. Define a web application context
    * not required, you could load all your web configurations into the root web application context
3. Define a ```DispatcherServlet```

### WebApplicationInitializer (interface)
* special spring interface that is used to configure a ServletContext programmatically
* classes that implement this interface are automatically  bootstrapped by servlet 3.0+ compliant environments

### AbstractContextLoaderInitializer
* implements ```WebApplicationInitializer```
* Registers a ```ContextLoaderListener``` which is used to initialize a **root** web application context
* requires a servlet 3.0+ environment
* Could be used in conjunction with *web.xml*

* ```SpringServletContainerInitializer``` looks for classes implementing ```WebApplicationInitializer```
  ```java
   public class WebAppInitializer extends AbstractContextLoaderInitializer {

       @Override
       protected WebApplicationContext createRootApplicationContext() {
           AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();
           //Configure app context here
           return applicationContext;
       }
   }
  ```

### AbstractDispatcherServletInitializer
* registers a DispatcherServlet in the ServletContext

### AbstractAnnotationConfigDispatcherServletInitializer (implements WebApplicationInitializer)
* Registers ```ContextLoaderListener```
* Defines ```DispatcherServlet``` (because this class extends AbstractDispatcherServletInitializer
* Expects a Java configuration
  ```java
  public class WebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

      //@Configuration classes for root application context
      @Override
      protected Class<?>[] getRootConfigClasses() {
          return new Class<?>[]{ RootConfig.class, JdbcConfig.class };
      }

      //@Configuration classes for web application context
      @Override
      protected Class<?>[] getServletConfigClasses() {
          return new Class<?>[]{ WebConfig.class };
      }

      //Urls handled by Dispatcher Servlet
      @Override
      protected String[] getServletMappings() {
          return new String[]{"/app/*"};
      }

      ...
  }
  ```


### web.xml
* **initialzer** classes can modify registrations performed in web.xml
  * if using web.xml make sure to set its version attribute to "3.0" or greater
* Default root application context config. filename is ```WEB-INF/applicationContext.xml```
* Default web application context config filename is ```WEB-INF/[servlet-name]-servlet.xml```

```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
		 http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">

    <!-- register the Spring provided servlet listener, responsible for bootstrapping the RootApplicationContext -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!--provide xml configuration files to the root context-->
    <!--defaults to WEB-INF/applicationContext.xml if not provided-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            WEB-INF/config/root-config.xml
            WEB-INF/config/jdbc-config.xml
        </param-value>
    </context-param>

    <!--define the DispatcherServlet-->
    <servlet>
        <servlet-name>my-dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!--if init-param were ommitted, Spring MVC would look for a file named
        [servlet-name]-servlet.xml in the WEB-INF dir-->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <!-- If param-value were to be left empty here, you would be configuring just one WebApplicationContext
                namely the RootApplicationContext defined above, all beans would be loaded into that. -->
            <param-value>WEB-INF/config/mvc/mvc-config.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>my-dispatcher</servlet-name>
        <url-pattern>/dispatcher/*</url-pattern>
    </servlet-mapping>

    <context-param>
        <param-name>spring.profiles.active</param-name>
        <param-value>profile1,profile2</param-value>
    </context-param>
</web-app>
```

## Configuring Caching
1. Install a cache implementation (if needed, e.g. EhCache)
2. Configure a ```CacheManager``` to use the implementation
3. Configure ```KeyGenerator```(s) (if needed)
4. Annotate a Configuration class with ```@EnableCaching``` or in XML ```<cache:annotation-driven />```
3. Annotate or configure methods you want to cache
    ```java
    @Cacheable(value="mycache", key="#personName.toUpperCase()", condition="#personName.length < 50" sync="true")
    public Person getPerson(String personName) {
        //...
    }
    ```
    ```xml
    <bean id="myService" class="com.example.MyService"/>

    <aop:config>
        <aop:advisor advice-ref="cacheAdvice" pointcut="execution(* *..MyService.*(..))"/>
    </aop:config>

    <cache:advice id="cacheAdvice" cache-manager="cacheManager">
        <cache:caching cache="myCache">
            <cache:cacheable method="myMethod" key="#id"/>
            <cache:cache-evict method="fetchData" all-entries="true" />
        </cache:caching>
    </cache:advice>
    ```
