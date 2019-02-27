Container, Dependency, and IOC
==============================
### What is dependency injection and what are the advantages? 

Dependency injection (DI) is where a needed dependency is injected by another object. The class being injected has no
responsibility instantiating the object being injected.

Advantages of DI:
* classes are loosely coupled
* can easily switch dependencies based on current needs 
  * local development vs production environment
  * makes mocking tests easier
  * in memory datasource vs production database
* promotes programming to interfaces


### What is meant by interface and what are the advantages of making use of them in Java? 

In Java, an interface is similar to a java class except that interfaces can only contain public method signatures,
not actual implementations of the methods. The implementation of these methods is left to other classes that must 
declare they implement the interface.

Advantages of using interfaces:
* enables polymorphism
* promotes code reuse
* enables you to swap in different implementation of an interface without changing your code


### What is meant by “application-context” and how do you create one? 

The ApplicationContext (sometimes called the "container") is a Java interface that you use to create, configure, and 
launch your Spring Application. The ApplicationContext is your "interface" to your spring application and the beans 
you have configured.

The application context is initialized with one or more configuration files (mainly java or XML), 
which contains setting for the framework, DI, persistence, transactions,...

Spring provides a number of ways to load configurations into the ApplicationContext, from a variety of resource
 locations. Spring will even let you mix and match configs from XML and Java.

* XML Configs
  * specify the location of one or more XML configuration files
  * can be loaded using pre-defined classes for loading from classpath or filesystem
    * ```ApplicationContext context = new ClassPathXmlApplicationContext ("classpath:spring/application-config.xml");```
    * ```ApplicationContext context = new FileSystemXmlApplicationContext("c:/knight.xml")```
  * can also override the default resource location and explicitly define a resource location with prefix:
    * ```ApplicationContext context = new ClassPathXmlApplicationContext ("file:///spring/application-config.xml");```
    * ```ApplicationContext context = new FileSystemXmlApplicationContext("classpath:config/knight.xml")```
  * in a web application you would use:
    * XmlWebApplicationContext() - loads context definitions from one or more XML files contained in a Web Application
* Java configs
  * pass in one or more configuration class files
  * methods to load these class files typically begin with ```Annotation```
    * ```ApplicationContext ctx = new AnnotationConfigApplicationContext(ApplicationConfig.class);```
  * For a web application using java config:
    * ```AnnotationConfigWebApplicationContext``` 
  

### What is the concept of a “container” and what is its lifecycle? 

The Spring container manages the beans you have configured for your application. It is the central part of a Spring
 Application. It "contains" your application beans that you have configured. It is responsible for reading 
 configuration files, managing beans, dependency injection, and more. The ApplicationContext is your interface 
 to the Spring "container".
 
Lifecycle:
* Initialization - bean definitions are read, beans are created, dependencies are injected, resources are allocated.
When this phase is complete, the app can be used.
* Use - the application is up and running. In this phase for 99% of the lifecycle
* Destruction - context is being shut-down, resources are released, beans are handed over to garbage collector

Lifecycle independent of configuration style used (Java/XML)


### Dependency injection using Java configuration

Java configuration allows DI via constructor,setter method, or field

Couple ways to wire beans together in a java configuration
* Within a @Bean method, refer to the referenced bean’s method directly (both beans **should** be in same @Configuration
 class)
```java
@Bean
public CDPlayer cdPlayer() {
return new CDPlayer(sgtPeppers());
}
```
* specify the dependant bean as a parameter to an @Bean method and let Spring resolve the required dependency (preferred)
```java
@Bean
public CDPlayer cdPlayer(CompactDisc compactDisc) {
return new CDPlayer(compactDisc);
}
```
* use @Autowired on constructor,setter,field of a dependant class and let the spring container inject the dependency


### Dependency injection in XML, using constructor or setter injection

* constructor injection (uses the ```<constructor-arg>``` element)
  * uses the **ref** attribute to refer to another bean id to inject as a constructor parameter.
  * There is also the **value** attribute that can be used when the value to inject is a scalar
    ```xml
    <beans ...>
      <bean id="complexBean" class="com.ps.beans.ctr.ComplexBeanImpl">
            <constructor-arg ref="simpleBean"/>
            <constructor-arg value="true"/>
      </bean>
    </beans>
    ```
  * There is also the **index** attribute which should be used when a constructor has more parameters of the same type
  ```
  // ComplexBean2Impl.java constructor
   public ComplexBean2Impl(SimpleBean simpleBean1, SimpleBean simpleBean2) {
          this.simpleBean1 = simpleBean1;
          this.simpleBean2 = simpleBean2;
  }
  
  <beans ...>
      <bean id="simpleBean0" class="com.ps.beans.SimpleBeanImpl"/>
      <bean id="simpleBean1" class="com.ps.beans.SimpleBeanImpl"/>
      <bean id="complexBean2" class="com.ps.beans.ctr.ComplexBean2Impl">
          <constructor-arg ref="simpleBean0" index="0"/>
          <constructor-arg ref="simpleBean1" index="1"/>
      </bean>
  </beans>
  ```
  * OR instead of index, used the **name** attribute to specify the (name of the parameter in the constructor) 
    bean name to inject
   ```
   public ComplexBean2Impl(SimpleBean simpleBean1, SimpleBean simpleBean2) {
     this.simpleBean1 = simpleBean1;
     this.simpleBean2 = simpleBean2;
   }
   <beans ...>
       <bean id="simpleBean0" class="com.ps.beans.SimpleBeanImpl"/>
       <bean id="simpleBean1" class="com.ps.beans.SimpleBeanImpl"/>
       <bean id="complexBean2" class="com.ps.beans.ctr.ComplexBean2Impl">
           <constructor-arg ref="simpleBean0" name="simpleBean1"/>
           <constructor-arg ref="simpleBean1" name="simpleBean2"/>
       </bean>
   </beans>
   ```
* setter injection (uses the ```property``` element):
  * The ```<property />``` element defines the property to be set and the value to be set with and does so using
    a pair of attributes: [name, ref] or [name,value]. **name** attribute is mandatory. **ref** tells the container 
    the value of this attribute is another bean. **value** indicates setting a scalar value.
  ```
  <beans ...>
      <bean id="simpleBean0" class="com.ps.beans.SimpleBeanImpl"/>
      <bean id="complexBean" class="com.ps.beans.set.ComplexBeanImpl">
          <property name="simpleBean" ref="simpleBean"/>
      </bean>
  </beans>
  ```
   

### Dependency injection using annotations ( @Component , @Autowired )

* @Component
  * Used at the class level to mark the whole class as a spring managed bean
  * Class is eligible for component scanning
  * class name becomes bean name (first letter of class is lower-cased) unless name explicitly given
    * e.g. @Component("someBeanName")

* @Autowired
  * used to inject spring managed beans into fields, arbitrary method params (including setters), or constructor params
  * constructor level
     * in spring version 4.3, no longer need to specify @Autowired if class only has one constructor
  ```java
  public class MovieRecommender {
  
      private final CustomerPreferenceDao customerPreferenceDao;
  
      @Autowired
      public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
          this.customerPreferenceDao = customerPreferenceDao;
      }
  
      // ...
  }
  ```
  * arbitrary method level
  ```java
  public class MovieRecommender {
  
      private MovieCatalog movieCatalog;
  
      private CustomerPreferenceDao customerPreferenceDao;
  
      @Autowired
      public void prepare(MovieCatalog movieCatalog,
              CustomerPreferenceDao customerPreferenceDao) {
          this.movieCatalog = movieCatalog;
          this.customerPreferenceDao = customerPreferenceDao;
      }
  
      // ...
  }
  ```
    
  * field level
    * use @Autowired above the field declaration, can even be used on private fields (considered bad practice)
  ```java
  public class MovieRecommender {
  
      private final CustomerPreferenceDao customerPreferenceDao;
  
      @Autowired
      private MovieCatalog movieCatalog;
  
      // ...
  }
  ``` 
* Setter vs Constuctor vs Field Injection
  * all three types can be used and combined
  * usage should be consistent
  * constructors are preferred for mandatory, immutable dependencies
  * setters are preferred for optional, changeable dependencies
  * only way to resolve circular dependencies is using setter injection, constructors cannot be used
  * Field injection can be used, but mostly discouraged

* Autowire conflicts
    * if no bean of given type for @Autowired dependency is found and it is not optional, exception is thrown
    * If more than one bean found that satisfies the dependency, ```NoSuchBeanDefinitionException``` is thrown as well
    * Can use @Qualifier to inject bean by name. The name is set in @Component annotation ```@Component("myName")```
    or name is set in @Bean("someBeanNane")
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
     
     
* Autowired resolution sequence
    1. Try to inject bean by type, if there is just one
    2. Try to inject by @Qualifier, if present
    3. Try to inject by bean name matching name of property being set
      * bean 'foo' for 'foo' field in field injection
      * bean 'foo' for 'setFoo' setter in setter injection
      * bean 'foo' for constructor param named 'foo' in constructor injection
    * When a bean name is not specified, one is auto-generated: De-capitalized non-qualified class name



### Component scanning, Stereotypes and Meta-Annotations

Component Scan
* used on an @Configuration class, @ComponentScan specifies packages to scan for @Component annotation
* these @Component classes are automatically turned into spring managed beans
* applies to @Component classes those meta-annotated with @Components (@Service,@Controller,@Repository,@Configuration)
* Beans declare their dependencies using @Autowired - automatically injected by Spring
  * field level - even private
  * method level
  * constructor level
* Autowired dependencies are required by default and results in an exception when no matching beans are found
* can be made optional using @Autowired( required=false )
* Dependencies of type ```Optional<T>``` are automatically considered optional
* can also be used for scanning jar dependencies
* recommended to explicitly specify packages to scan in order speed up scan time
```java
@Configuration
@ComponentScan( { "org.foo.myapp", "com.bar.anotherapp" })
public class ApplicationConfiguration {
    //Configuration here
}
```  

* Java config: 
  * use @ComponentScan on an @Configuration class. By default it will scan the package it is currently in for 
  @Component classes (and @Component sub-types), else you can specify list of packages to scan
  ```java
    @Configuration
    @ComponentScan( { "org.foo.myapp", "com.bar.anotherapp" })
    public class ApplicationConfiguration {
        //Configuration here
    }
    ```
* XML config:
  * use ```<context:component-scan base-package="com.example.foo, com.example.bar"/>```
    * this will trigger a scan for classes annotated with @Component and its sub-annotations
    
 

Spring stereo-type annotations are all meta-annotated with @Component:
* @Component
* @Configuration
* @Repository
* @Service
* @Controller
* ... more in other spring projects

Meta-annotations
* used to annotate other annotations
* e.g. you could create a custom annotation, which combines @Service and @Transactional
```java
  //creates a Custom Transaction annotation named CustomTx built on top of @Transactional
  @Target({ElementType.METHOD, ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Transactional("customTransactionManager", timeout="90")
  public @interface CustomTx {
    boolean readOnly() default false;
  }
  ```
  


### Scopes for Spring beans. What is the default? 

Spring bean scopes:
* Singleton (default) - one instance of bean is created for the entire application
* Prototype - one instance of bean is created every-time the bean is injected into or received from the Spring
application context. In the case of prototypes, configured destruction lifecycle callbacks are not called 
e.g. (@PreDestroy, destroyMethod)
* [A]Global-Session - one global session shared among all portlets - only in web portlet environment
* Application - in a web application, one instance of bean is created per servlet context
* Session - in a web application, one instance of bean is created for each session
* Request - in a web application, one instance of bean is created for each request
* WebSocket - scopes a single bean definition to the lifecycle of a WebSocket. Only valid in a web-aware App.Context
* custom scope - developers can define their own custom scopes

Example of setting scope in Java config
```java
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) //or @Scope("prototype")
public Notepad notepad() {
return new Notepad();
}
```
Setting scope in XML config:
```<bean id="fooBeanName" class="com.example.foo.Foo" scope="prototype" />```



### What is an initialization method and how is it declared in a Spring bean? 

Initialization method(s) are methods called by the spring application context to perform post processing on a bean.
Developers have several options:


* Post process a bean **after dependencies have been injected** by using an **initializer**
  * Using Java Config:
    * @PostConstruct - (recommended) identifies a **single** method of your bean that should be called to perform initialization
      * @PostConstruct methods must have no arguments, return void, and they can have any access right
    * @Bean( initMethod="<methodToCallForInit>" ) - (also recommended) equivalent to PostConstruct
  * Using XML config:
    * use the ```init-method``` attribute  e.g. 
      * ```<bean id="barBeanName" class="com.example.bar.Bar" init-method="setup" destroy-method="teardown" />```
    * enable @PostConstruct methods to be called from XML config:
      * must add either ```<context:annotation-config />``` to your XML config  OR
      * ```<context:component-scan base-package="{package names}""/>``` (recommended) since it activates detection
        for JPA and EJB annotations plus Spring stereotype annotations
* Implement the ```org.springframework.beans.factory.InitializingBean``` interface and providing an implementation 
    for the method afterPropertiesSet (**not recommended**, since it couples the application code with Spring infrastructure).
* Implement the ```BeanPostProcesessor``` interface
  * The BeanPostProcessor interface defines callback methods that you can implement to provide your own 
    (or override the container’s default) instantiation logic, dependency-resolution logic, and so forth. If you want 
    to implement some custom logic after the Spring container finishes instantiating, configuring, and initializing a bean, 
    you can plug in one or more BeanPostProcessor implementations.
    * To create custom bean post processor, implement interface BeanPostProcessor, and add the bean to your config
      ```java
      public interface BeanPostProcessor {
          public Object postProcessAfterInitialization(Object bean, String beanName); 
          public Object postProcessBeforeInitialization(Object bean,String beanName);
      }
      ```
  
  

### What is a destroy method, how is it declared and when is it called?

A destroy method is used to perform custom bean clean-up code before a bean is destroyed during the destruction phase
 of the Spring container.
Two options, annotate a method with @PreDestroy, or configure a ```destroyMethod``` in your bean configuration

* @PreDestroy
  * invoked before the bean is destroyed
  * after application context is closed
  * only if JVM exits normally
  * method must be void, no-args, any access modifier
  * PreDestroy **not called for prototype scoped beans** 
  * Applies to both beans discovered by component scan and declared by @Bean in @Configuration
* @Bean( destroyMethod="someDestroyMethod" ) 
  * in XML config: ```<bean id="barBeanName" class="com.example.bar.Bar" destroy-method="teardown" />```
  
There is also the ```org.springframework.beans.factory.DisposableBean.destroy()``` interface that you could implement
in your bean, but it is **not recommended** as it couples your code to Spring. Use the above mentioned destruction
callbacks.
  

### What is a BeanFactoryPostProcessor and what is it used for? 

A BeanFactoryPostProcessor is an interface you can implement in order to post-process bean **configuration** meta-data
The ApplicationContext will automatically detect beans that implement this interface and make sure they run before 
other beans are instantiated. e.g. - PropertySourcesPlaceholderConfigurer is a BeanFactoryPostProcessor.

* Developers can choose to post process **bean configuration metadata** at runtime by implementing the 
```BeanFactoryPostProcessor.postProcessBeanFactory(BeanFactory bf)``` interface in their bean configurations. 
This will allow you to potentially change configuration data before the container instantiates any other beans
  * @Bean methods that return BeanFactoryPostProcessor **should be declared static**


### What is a BeanPostProcessor and how is it different to a BeanFactoryPostProcessor? What do they do? When are they called? 

The BeanPostProcessor interface defines callback methods that you can implement to provide your own 
(or override the container’s default) instantiation logic, dependency-resolution logic, and so forth. If you want 
to implement some custom logic after the Spring container finishes instantiating, configuring, and initializing a bean, 
you can plug in one or more BeanPostProcessor implementations.
 
* Define your own BeanPostProcessor by implementing the BeanPostProcessor interface
    * provides two call back methods that you must implement:
      * ```public Object postProcessAfterInitialization(Object bean, String beanName);```
        * Apply this BeanPostProcessor to the given new bean instance after any bean initialization callbacks
      * ```public Object postProcessBeforeInitialization(Object bean,String beanName);```
        * Apply this BeanPostProcessor to the given new bean instance before any bean initialization callbacks
    * the return value of these methods is a post processed bean. It could be the same bean with altered state or
    an entirely new object. **BeanPostProcessor can return a proxy of the bean**. Commonly used to add security and
    transactional behavior
    

### Are beans lazily or eagerly instantiated by default? How do you alter this behavior? 

Beans are eagerly instantiated by default. To change this behavior use @Lazy annotation (XML: ```lazy-init```)

* @Lazy
  * typically used with @Bean, can be used with @Autowired or @Inject
* In XML config: 
  * ```<bean id="barBeanName" class="com.example.bar.Bar" lazy-init="true" init-method="setup"/>```


### What does component-scanning do 
  
Component scanning scans one or more packages (or packages in .jars) for classes annotated with @Component 
(or its sub-stereotypes). These classes are then automatically turned into spring-managed beans that are then 
eligible for dependency-injection via @Autowired or @Inject or explicit wiring


### What is the behavior of the annotation @Autowired with regards to field injection, constructor injection and method injection 

* Autowired dependencies are required by default and result in an exception if no matching bean is found
* Dependencies can be made optional by using @Autowired( required=false )
* Dependencies of type ```Optional<T>``` are automatically considered optional

* Autowired conflicts
  * If no bean of a given type for @Autowired dependency injection is found and it is not optional, an exception is thrown
  * If more than one bean is found that satisfies the dependency, NoSuchBeanDefinitionException is thrown
  * Can use @Qualifier to inject a bean by name. The name is set in @Component("beanName") or in the @Bean("beanName") 
  ```java
    @Component("myRegularDependency")
    public class MyRegularDependency implements MyDependency {...}
    
    @Component("myOtherDependency")
    public class MyOtherDependency implements MyDependency {...}
    
    @Component
    public class MyService {
        @Autowired
        public MyService(@Qualifier("myOtherDependency") MyDependency myDependency) {...}
    }
  ```

* Autowired resolution sequence
  1. Try to inject bean by type, if there is just one
  2. Try to inject by @Qualifier, if present
  3. Try to inject bean by name that matches the property being set
    * bean 'foo' for 'foo' field in field injection
    * bean 'foo' for 'setFoo' setter in setter injection
    * bean 'foo' for constructor param named 'foo' in constructor injection

When a bean name is not specified:
* When using @Component - one is auto-generated: De-capitalized non-qualified class name
* When using @Bean - bean name is method name



### How does the @Qualifier annotation complement the use of @Autowired ? 

@Qualifier allows you to narrow the set of type matches so that a specific Autowired bean is chosen

Example java class with @Qualifier
```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(@Qualifier("main")MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```
Corresponding XML class
```xml
  <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="main"/>

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="action"/>

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>
```


### What is the role of the @PostConstruct and @PreDestroy annotations? When will they get called?

@PostConstruct and @PreDestroy are used to annotate callback methods of a bean, so that it can be post-processed after
dependency injection (@PostConstruct); as well as perform any freeing of resources before being destroyed by the 
ApplicationContext (@PreDestroy)

@PostConstruct / @PreDestroy
* Can be used on @Component methods
* Methods can have any access modifier (private recommended), but must be void and take no arguments
* Applies to beans discovered by component scan and declared by @Bean in @Configuration
* Defined by JSR-250 in javax.annotation.package, they are also used by EJB3

@PostConstruct
* Method is invoked by Spring after the beans dependencies are injected

@PreDestroy
* Method is invoked before bean is destroyed
* **not called for prototype scoped beans**
* After application context is closed
* Only if JVM exits normally


### What is a proxy object and what are the two different types of proxies Spring can create 

A proxy object is an object that "wraps" another object (the target) in order to provide additional 
functionality to the target object.  Objects that call methods of the target object will have no idea they are going 
through a proxy object. Spring can create two types of proxies:
* JDK Dynamic proxies
* CGLib proxies


### What is the power of a proxy object and where are the disadvantages 

Proxy objects can transparently add functionality to the proxied object.
May be a slight performance hit.
Method calls between methods of the same object instance are not proxied.
  


### What are the limitations of these proxies (per type) 

* JDK Dynamic Proxies
  * Proxied beans must implement at least one java interface
  * collaborators must work with the proxied bean via the methods of the interface
  * proxying can only be applied to Spring managed beans
  * proxy is bypassed when one method of the class calls a method of the same class
* CGLib Proxies
  * cannot be applied to final classes or methods as they cannot be overridden
  * **CGLib proxies only intercept public method calls**
  * proxying can only be applied to Spring managed beans
  * proxy is bypassed when one method of the class calls a method of the same class
  


### How do you inject scalar/literal values into Spring beans 

Java Config:
* Use @Value
  * Can be on fields, constructor, or setters
  * On constructors and setters **must be combined with @Autowired**
  * can inject value from system properties using ${} or SpEL using #{}
  * Can specify default values:
    * ```"${minAmount:100}"```  the : acts like a default separator. if minAmount not present then default to 100
    * ```#{environment['minAmount'] ?: 100}```  ?: is elvis operator. if minAmount is null then default to 100

XML Config:
* Use the ```value``` attribute with the constructors element or property element
  * Constructor element
    * ```xml
        <bean id="exampleBean" class="examples.ExampleBean">
             <constructor-arg type="int" value="7500000"/>
             <constructor-arg type="java.lang.String" value="42"/>
         </bean>
      ```
      ```xml
      <bean id="exampleBean" class="examples.ExampleBean">
          <constructor-arg index="0" value="7500000"/>
          <constructor-arg index="1" value="42"/>
      </bean>
      ```
      ```xml
      <bean id="exampleBean" class="examples.ExampleBean">
          <constructor-arg name="years" value="7500000"/>
          <constructor-arg name="ultimateAnswer" value="42"/>
      </bean>
      ```
  * Property element
    * ```<property name="tensionAmount" value="100"/>```
    * OR use ```<value/>``` tag inside of property
      ```
      <property name="someOtherProperty">
         <value>42</value>
      </property>
      ```
* use the c-namespace or p-namespace with the bean element
  * c-namespace - ```c:[name-of-constructor-property]```
    * ```<bean id="complexBean1" class="com.ps.beans.ctr.ComplexBeanImpl" c:simpleBean-ref="simpleBean0" c:complex="true"/>```
  * p-namespace - ```p:[name-of-setter-property]```
    * ```<bean id="complexBean" class="com.ps.beans.set.ComplexBeanImpl" p:simpleBean-ref="simpleBean" p:complex="true"/>```
  


### How are you going to create a new instance of an ApplicationContext 

Depends on if you are creating a Spring web application or a Spring "stand-alone" application. In addition you may have
 to read XML configuration files OR java configuration files OR perhaps you want to mix and match java configs and XML
 configs (spring will let you mix and match them.)
 
Spring web applications will have to use a web aware application context:
* AnnotationConfigWebApplicationContext - loads Spring Web Application context from one or more Java based
configuration classes.
* XmlWebApplicationContext - loads context definitions from one or more XML files contained in a Web Application


For stand alone applications, Spring provides a number of convenience Constructors for loading configurations from XML
or Java config (you can mix and match). If using XML config you can use a different constructor depending on if your XML 
files are loaded from the classpath or file-system. If using Java config you can use a constructor that takes Java 
classes as arguments, OR you can also use Spring's resource loaders to force loading configuration data from 
different resource locations (see prefix question below)

Examples:
* new AnnotationConfigApplicationContext(AppConfig.class); - load java configuration
* new ClassPathXmlApplicationContext(“com/example/app-config.xml”); - load xml config from classpath
* new FileSystemXmlApplicationContext(“/Users/vojtech/app-config.xml”); - load xml config from filesystem
* new FileSystemXmlApplicationContext(“./app-config.xml”);




### What is a prefix 

A prefix is a string you specify in an ApplicationContext constructor during initialization that specifies the
location of configuration metadata. It can override the constructors default resource loader and force the loading of 
configuration data from a different resource location.

List of Spring prefixes:
* classpath:
  * resource is loaded from classpath
  * e.g.   classpath:com/myapp/config.xml
* file:
  * Loaded as a URL, from the filesystem.
  * e.g. file:///data/config.xml 
* http:
  * Loaded as a URL
  * e.g.  http://myserver/logo.png
 


### What is the lifecycle on an ApplicationContext 

Three main phases:
* Initialization
  * bean definitions are read, beans are created, dependencies are injected, resources are allocated
* Use
  * the application is up an running. Beans are retrieved and used to do work. This is 99% of the lifecycle
* Destruction 
  * The context is being shut down, resources are released, beans are handed over to the garbage collector

Lifecycle is independent of the configuration style used (Java/XML)



### What Does the @Bean annotation do 

It specifies that a method declares an explicit bean definition in a Spring Java configuration.
* A method is explicitly marked @Bean as spring managed bean in a @Configuration class
* All settings of the bean are present in the @Configuration class in the bean declaration
* Spring config is completely in the @Configuration, bean is just a POJO with no Spring dependency.
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
  
  
### How are you going to create an ApplicationContext in an integration test or a JUnit test?*   

Import the artifact spring-test.jar into your spring project and use SpringJUnit4ClassRunner to create 
an ApplicationContext for your tests.

* SpringJunit4ClassRunner - application context is cached among test methods - one one instance of app context is 
created per test class
* Test class needs to be annotated with @RunWith( SpringJUnit4ClassRunner.class )
* Spring configuration classes to be loaded are specified in @ContextConfiguration annotation
  * If no value is provided to @ContextConfiguration, config file ${classname}-context.xml in the same package is imported
  * XML files are loaded by providing string value to annotation 
    * @ContextConfiguration("classpath:/com/example/test-config.xml")
  * Java @Configuration files are loaded from classes attribute
    * @ContextConfiguration( classes={ TestConfig.class, OtherConfig.class } )
  * Spring **caches application contexts by locations attribute** so if the same locations appear for the second time, 
  Spring uses the same context rather than creating a new one
* Setting active profiles for your test
  * @ActiveProfiles({ "foo","bar"})
* Overriding property sources for your tests
  * @TestPropertySource overrides any existing property of the same name
  * Default location is "[classname].properties"
  * @TestPropertySource( properties={"username=foo"} )
  
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
  @Configuration inner classes (must be static) are automatically detected and loaded in tests
```java
  @RunWith(SpringJUnit4ClassRunner.class)
  @ContextConfiguration(...)
  public final class FooTest  {
      @Configuration
      public static class TestConfiguration {...}
  }
```


### What do you have to do, if you would like to inject something into a private field 

Only available when using java configuration
* If injecting another bean, annotate the field with @Autowired or @Inject (JSR-330)
* If injecting a scalar type, use @Value



### What are the advantages of JavaConfig? What are the limitations 

Advantages
* Java is type safe, can use the compiler to catch errors at compile time
* Easier to refactor / debug (assuming you are using an IDE)
* Shorter and more concise configuration than XML
* field injection only possible in java config

Limitations
* have to recompile when configuration changes
* code is coupled with configuration
* some argue that configuration becomes decentralized and harder to control



### What is the default bean id if you only use @Bean ?

Default bean id is the method name that @Bean is used on.



### Can you use @Bean together with @Profile ?

Yes. Beans with an associated @Profile will only be loaded into the AppContext of the profile is active

* Can use @Profile("profileName") either with the @Bean or with @Configuration class level (applies to all beans in config)
* Beans with no profile are always active
* Beans with a specific profile are only loaded when the given profile(s) are active
* Multiple profiles can be active



### What is Spring Expression Language (SpEL for short) ?

SpEL is a powerful expression language that supports querying and manipulating and "object graph" at runtime.
SpEL provides:
* method invocation
* access to properties and indexed collections
* collection filtering
* accessing arrays
* boolean and relational operators
* mathematical operations
* can be used in @Value annotations
* SpEL expressions are enclosed in #{}
* ${} is for accessing values loaded from a properties file via PropertySourcesPlaceholderConfigurer
  * to access property of a bean use #{beanName.property}
* Can reference systemProperties and systemEnvironment
  * ```#{systemProperties['user.name']}```
  * ```#{systemEnvironment['JAVA_HOME']}```



### What is the environment abstraction in Spring ?

The environment is an object integrated into the spring container and models **profiles** and **properties**

* A profile is a named, logical group of bean definitions to be registered with the container only if the given profile 
is active

* The role of the environment object with regards to profiles is in determining which profiles (if any) are active,
and which profiles (if any) should be active be default
* The role of the environment object with regards to properties is to provide the user with a convenient service
interface for configuring property sources and resolving properties from them
* Environment can be injected with @Autowired
* Properties are obtained using  ```environment.getProperty("propertyName")```



### What can you reference using SpEL? 

see above.

### How do you configure a profile. What are possible use cases where they might be useful ?

* Use @Profile({"profileName1", "profileName2"}) either with the @Bean level or @Configuration class level 
(will apply to all beans in the config)
* In XML: ```<beans profile="development" ...```
* beans with no profile are always active, they automatically inherit the "default" profile
  * The "default" profile represents the profile that is enabled by default.
  * THe name "default" can be changed via Environment.setDefaultProfiles() 
* Active profiles can be set in several different ways:
  * @ActiveProfiles in spring integration tests
  * In web.xml deployment descriptor as a context param. "spring.profiles.active"
  * By setting "spring.profiles.active" system property ```-Dspring.profiles.active="profile1,profile2"```
  * Active profiles are comma-separated
  * programmatically:
   ```java
       //setting active profile programatically
       AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
       ctx.getEnvironment().setActiveProfiles("development","mock");
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
   
Most common use-case for Profiles is to easily switch your configuration and properties files for different
environments like "dev" or "qa" or "production". Could also be used to switch different Database implementations



### How many profiles can you have? 

As many as you need. Technically, the profile names are stored as String(s) in a LinkedHashSet.


### How do you enable JSR-250 annotations like @PostConstruct ? 

If using Java config, that are enabled out of the gate (assuming you used an AnnotationConfigApplicationContext to load
your java classes)
If using XML config you have to do one of two things:
* add ```<context:annotation-config />``` into one of your configuration files
  * activates detection for @PostConstruct, @PreDestroy, @Resource, @Autowired, and @Required and other JPA and 
  EJB 3 annotations
* add ```<context:component-scan base-package="{package names}""/>``` (recommended)
  * does the same as annotation-config but also adds detection for spring Stereotype annotations
  * should be faster (if you specify base-package names)



### Why are you not allowed to annotate a final class with @Configuration*

All @Configuration classes are subclassed at startup-time with CGLIB. CGLIB cannot proxy final classes as it cannot 
subclass them


### Why must you have a default constructor in your @Configuration annotated class 

Prior to Spring 4, CGLIB-based proxy classes require a default constructor. And this is not the limitation of CGLIB 
library, but Spring itself. Fortunately, as of Spring 4 this is no longer an issue. CGLIB-based proxy classes no 
longer require a default constructor.


### Why are you not allowed to annotate final methods with @Bean  

Final methods cannot be proxied by CGLIB 


### What is the preferred way to close an application context 

register a shutdown hook with your application context by calling ctx.registerShutdownHook(). It will ensure that it 
shuts down gracefully and all relevant destroy methods on your singleton beans are called prior to your app shutting down. 
**Spring’s web-based ApplicationContext implementations already have code in place to shut down the Spring IoC 
container gracefully when the relevant web application is shut down.**


### How can you create a shared application context in a JUnit test?* 

Spring caches application contexts by locations attribute so if the same locations appear for the second time, 
 Spring uses the same context rather than creating a new one


### What does a static @Bean method do 

You may declare @Bean methods as static, allowing for them to be called without creating their containing configuration 
class as an instance. This makes particular sense when defining post-processor beans, e.g. of type 
BeanFactoryPostProcessor or BeanPostProcessor, since such beans will get initialized early in the container lifecycle 
and should avoid triggering other parts of the configuration at that point.


### What is a PropertySourcesPlaceholderConfigurer used for 

It's a BeanFactoryPostProcessor that you must have configured in your application if you plan to use a custom 
properties file to obtain values for your configuration. Essentially, it is used to fill in property
placeholders (e.g. ${}) with values from a properties file

* If custom property sources are used, and you want to use the @Value annotation to inject a property, you need to 
declare PropertySourcesPlaceholderConfigurer as a static bean


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

### What is a namespace used for in XML configuration?

They are used to simplify configuration and hide detailed bean configuration. They also make the XML configuration
less verbose. 

* c-namspace - simplifies constructor injection for a bean:
  * ```<bean id="complexBean1" class="com.ps.ctr.ComplexBeanImpl" c:simpleBean-ref="simpleBean0" c:complex="true"/>```
* p-namespace - simplifies property injection
  * ```<bean id="complexBean" class="com.ps.beans.ComplexBeanImpl" p:simpleBean-ref="simpleBean" p:complex="true"/>```
* context
  * <context:property-placeholder location="config.properties" /> instead of declaring 
  PropertySourcesPlaceholderConfigurer bean manually
* aop
  * <aop:aspectj-autoproxy /> Enables AOP (5+ beans)
* tx (transactions)
  * <tx:annotation-driven /> Enables Transactions (15+ beans)
* util
* jms
* ...


### If you saw one of the < context/ > elements covered in the course, would you know what it does?

* ```<context:property-placeholder location="config.properties" />``` instead of declaring 
  PropertySourcesPlaceholderConfigurer bean manually  
* ```<context:annotation-config />```
  * activates detection for @PostConstruct, @PreDestroy, @Resource, @Autowired, and @Required and other JPA and 
  EJB 3 annotations
* ```<context:component-scan base-package="{package names}""/>``` (recommended)
  * does the same as annotation-config but also adds detection for spring Stereotype annotations
  * should be faster (if you specify base-package names)
  * example of component-scan using filters:
    ```
    <context:component-scan base-package="org.example">
            <context:include-filter type="regex"
                    expression=".*Stub.*Repository"/>
            <context:exclude-filter type="annotation"
                    expression="org.springframework.stereotype.Repository"/>
    </context:component-scan>
    ```  


### What is @Value used for? 

It's used to inject scalar values into fields and/or method parameters when using Java configuration.
* Can be used with property placeholders to set values retrieved from a properties file 
* Can be used on fields and method parameters.
* @Value("${propertyName}") can be used as an alternative to Environment object



### What is the difference between $ and # in @Value expressions ?

* ```${}``` - property placeholder - used to set a value from the properties associated with the ApplicationContext
* ```#{}``` - expression resolver - For @Value annotations, an expression resolver is preconfigured to look for bean 
names when resolving expression text. Basically it's used to access property of bean e.g. #{beanName.property}


  
  
  
Aspect Oriented Programming
================================================================================================================
### What is the concept of AOP? What problems does it solve? 

The concept of AOP is to take common functionality that is scattered about your classes (but not the core 
functionality of a class) and modularize it into separate classes called Aspects. AOP calls this common functionality 
cross-cutting concerns. AOP keeps good separation of concerns and does not violate the Single Responsibility Principle
AOP also helps to solve two main problems:
* Code Tangling - coupling of different concerns in one class
* Code scattering - same code scattered among multiple classes

### What is a pointcut, a join point, an advice, an aspect, weaving? 

* join point - a point in the execution of an app. where AOP code will be applied
* pointcut - an expression that matches one or more join points
* advice - code to be executed at each matched join point
* aspect - module that encapsulates pointcut(s) and advice
* weaving - technique by which aspects are integrated into main code

### How does Spring solve (implement) a cross cutting concern? 

To implement a cross cutting concern using Spring AOP you have to:
1. write your main application logic
2. write aspects to address cross cutting concerns
3. weave aspects into the application to be applied in the right places

Spring will then use proxy objects (either JDK Dynamic, or CGLIB) to "weave" your aspects into your code

### Which are limitations of the two proxy types? 

Spring AOP will either use JDK Dynamic proxies, preferred and is the default if the proxied class implements at least 
one interface. OR you can use CGLIB proxies if proxied class does not implement any interface. 

Limitations of the proxy types:
* can only apply advice to methods
* only applied to Spring managed beans
* AOP is not applied when one method of the class calls a method of the same class (proxy is bypassed)

* JDK Dynamic Proxies
  * only interfaces are proxied, so you must work with the class via its interface
* CGLIB Proxies
  * can intercept **public,protected,package-visible** methods but best practice is to use public methods
  * final classes and methods **cannot** be advised as they cannot be overridden
 
###  How many advice types does Spring support. What are they used for 

Spring supports 5 advice types:
* Before 
  * advice executes before target method invocation, if **advice** throws an exception the target is not called
* After Returning 
  * advice executes after method successfully returns, if **target** throws an exception the advice is not called
* After Throwing 
  * advice executes if target method throws an exception
* After (finally) 
  * advice executes after target method invocation, doesn't matter if it was successful or if there was an exception
* Around
  * this advice encapsulates the target method and surrounds the join point. Around advice has control over its 
  target method execution via ```ProceedingJoinPoint```   

  
###  What do you have to do to enable the detection of the @Aspect annotation? 

* In java config - You have to add the @EnableAspectJAutoProxy annotation to a configuration class
* In XML config - ``` <aop:aspectj-autoproxy />```


###  Name three typical cross cutting concerns?

Logging, Transactions, Security


###  What two problems arise if you don't solve a cross cutting concern via AOP? 

Code scattering, and code tangling


###  What does @EnableAspectJAutoProxy do? 

Enables support for handling components marked with AspectJ's @Aspect annotation. Spring can then "autoproxy" beans 
based on whether or not they are advised by those aspects. By autoproxy we mean that if Spring determines that a 
bean is advised by one or more aspects, it will automatically generate a proxy for that bean to intercept method 
invocations and ensure that advice is executed as needed.


###  What is a named pointcut? 

* It's a pointcut that has been given a name so that it can be referenced in other pointcut expressions
  * the method name that the pointcut is defined on becomes the pointcut name
  ```
  //Method name is pointcut id, it is not executed
  @Pointcut("execution(void set*(*))") 
  public void setters() {
      //...
  }
  
  //Pointcut is referenced here by its ID
  @Before("setters()") 
  public void logChange() {
      //...
  }
  ```


###  How do you externalize pointcuts? What is the advantage of doing this? 

You define named pointcut expressions in a separate class (annotated with @Aspect) and then import that class where it
is needed.

* it makes the pointcut expressions reusable in multiple aspects
* pointcut definitions are in one place, makes use of single responsibility principle
* it allows you to combine several named pointcuts into a composite pointcut


###  What is the JoinPoint argument used for? 

JoinPoint is an AspectJ interface that "Provides reflective access to both the state available at a join point and 
static information about it"

The JoinPoint argument can be passed into any advice method as its first parameter ( NOTE: around advice must declare
ProceedingJoinPoint as its first parameter). 

The JoinPoint contains many useful methods, such as:
* ```getArgs()``` - to get a list of arguments passed into the target method
* ```getThis()``` - returns the proxy object
* ```getTarget()``` - returns the target object
* ```getSignature()``` - returns a description of the method being advised
* ...


###  What is a ProceedingJoinPoint? 

ProceedingJoinPoint is a sub-interface of JoinPoint. It must be passed in as the first parameter of an @Around advised
method. In addition to the methods provided by JoinPoint, ProceedingJoinPoint contains the ```proceed()``` method which
must be called in order for the advised method to run.


###  What are the five advice types called? 

* @Before
* @AfterThrowing
* @AfterReturning
* @After
* @Around


###  Which advice do you have to use if you would like to try and catch exceptions ?

If you want to actually catch the exception and stop it from propagating you will need to use ```@Around``` advice.
If you want to do some processing on a thrown exception and don't care that the exception will keep propagating you
should use ```@AfterThrowing``` advice. @AfterThrowing can be configured with the ```throwing``` attribute which
will give you access to thrown exception.
```java
    @AfterThrowing(value="execution(* service..*.*(..) )", throwing="exception")
    public void doAccessCheck(Exception exception) {
    //...
    }
```


###  What is the difference between @EnableAspectJAutoProxy and ```<aop:aspectj-autoproxy >```  ?
They both enable enable Spring support for configuring Spring AOP based on @AspectJ aspects, and auto-proxying beans
 based on whether or not they are advised by those aspects. 

@EnableAspectJAutoProxy is used to enable AspectJ annotations in a java configuration. It can be used on spring components
  ( @Configuration or @Component classes ) to make sure that @Aspect annotated beans will properly advise the beans that
  match their pointcut expressions. You can also use @EnableAspectJAutoProxy along with @ComponentScan in which case
  you must annotate any @Aspect beans with @Component (or one of @Component's subclasses)

```<aop:aspectj-autoproxy >``` is for XML based configuration.
```xml
<aop:aspectj-autoproxy>
  <aop:include name="userRepoMonitor"/>
</aop:aspectj-autoproxy>
```
it has an optional child <aop:include name="..."/> element where you can explicitly list the beans you want to be auto-proxied.

Data Management: JDBC, Transactions, JPA, Spring Data
=====================================================================================================================
### What is the difference between checked and unchecked exceptions? 

Checked exceptions must be handled in a try/catch block or, if not caught, you must declare that a method will throw a checked 
exception. Unchecked exceptions do not have to be caught or declared as being thrown by a method.

### Why does Spring prefer unchecked exceptions? 

Checked exceptions provide a form of tight coupling between layers (unchecked do not). When exceptions from specific  
data access technologies leak to the service layer of the app, layers are no longer loosely coupled.

### What is the data access exception hierarchy? 

It is a hierarchy of RuntimeExceptions that provide a practical way for the developer to handle database access 
exceptions without knowing the details of the specific data access API (JPA, Hibernate, JDBC...). DataAccessException 
has three main sub-branches:
* NonTransientDataAccessException - exceptions that will fail if retried, unless the originating cause is fixed.
  * e.g. searching for an object that does not exist
* RecoverableDataAccessException - exceptions that might succeed if retried, if some recovery steps are performed
  * e.g. a temporary network hiccup causes a transaction to fail - can fix by retrying the query in a new connection
* TransientDataAccessException - exceptions might succeed if retried, without any intervention by the developer
  * bad connection in the middle of a query execution, can remedy this by retrying the query
  
### How do you configure a datasource in Spring? 

Spring offers several options for configuring Datasource beans:

* Data sources looked up via JNDI
  * use a JndiObjectFactoryBean to look up the datasource from JNDI
  ```java
  @Bean
  public JndiObjectFactoryBean dataSource() {
    JndiObjectFactoryBean jndiObjectFB = new JndiObjectFactoryBean();
    jndiObjectFB.setJndiName("jdbc/SpittrDS");
    jndiObjectFB.setResourceRef(true);
    jndiObjectFB.setProxyInterface(javax.sql.DataSource.class);
    return jndiObjectFB;
  }
  ```

* Data sources defined by a JDBC Driver
  * Spring offers three datasources classes to choose from:
    * DriverManagerDataSource - returns a new connection every time a connection is requested (these are not pooled)
    ```java
    @Bean
    public DataSource dataSource() {
      DriverManagerDataSource ds = new DriverManagerDataSource();
      ds.setDriverClassName("org.h2.Driver");
      ds.setUrl("jdbc:h2:tcp://localhost/~/spitter");
      ds.setUsername("sa");
      ds.setPassword("");
      return ds;
    }
    ```
    * SimpleDriverDataSource - similar to DriverManagerDataSource, except it works with JDBC driver directly to overcome 
    class loading issues that may arise in certain environments
    * SingleConnectionDataSource - returns the same connection every time a connection is requested  

* Data sources that pool connections
```java
// Example java config for Apache DBCP pooled datasource:
@Bean
public BasicDataSource dataSource() {
    BasicDataSource ds = new BasicDataSource();
    ds.setDriverClassName("org.h2.Driver");
    ds.setUrl("jdbc:h2:tcp://localhost/~/spitter");
    ds.setUsername("sa");
    ds.setPassword("");
    ds.setInitialSize(5);
    ds.setMaxActive(10);
    return ds;
  }
}
``` 

When using a JDBC Driver dataSource or pooled connection datasource you will typically need:
* A database driver
* a connection URL that is the entry point for DB communication
* database credentials (usually username and password)

Using the previous information, a javax.sql.Datasource object is created, which is the interface that must be implemented
by every datasource driver library

For development and testing purposes, Spring provides an in-memory database (H2,HSQL,or Derby) that can be created using 
the EmbeddedDatabaseBuilder() bean. Which can be configured to load schema/data scripts when the bean is instantiated

### What is the template design pattern and what is JDBC template? 

Template design pattern is a behavioral design pattern that defines the "skeleton" of an algorithm. 
Some steps of the algorithm are fixed and happen the same way every time. These fixed steps can be implemented by the 
template itself (e.g. opening/closing connections.) At certain points, the algorithm delegates some of its work to a 
subclass to fill in some implementation-specific details (e.g. executing a query). A template method delegates the 
implementation-specific portions of the algorithm to an interface. Different implementations of this interface define 
specific implementations of this portion of the algorithm.

JDBCTemplate is a class that takes care of the boilerplate code that is required when working with JDBC and allows the
developer to simply run queries.

JdbcTemplate handles
* opening/closing connections
* transactions
* exception handling
* transforms SQLExceptions into DataAccessExceptions (no try/catch needed)
* jdbcTemplate is thread-safe

### What is a callback? What are the three JdbcTemplate callback interfaces described in the notes?
### What are they used for?
#### (You would not have to remember the interface names in the exam, but you should know what they do if you see them in a code sample)
    
A callback is a piece of code that you pass as an argument to some other code so that it executes it. JdbcTemplate 
provides three callback interfaces you can use to process data returned by a query:
* RowMapper - when each row of a ResultSet maps to a single domain object
* RowCallbackHandler - when no return object is needed (e.g. converting results to XML/JSON)
* ResultSetExtractor - when you need to map multiple ResultSet rows to a single return object

### Can you execute a plain SQL statement with the JDBC template?

Yes, JdbcTemplate provides ```execute()``` methods that can run arbitrary SQL or DDL statements.
It is heavily overloaded with variants taking callback interfaces, binding variable arrays, and so on.
There are also the various query* methods

### Does the JDBC template acquire (and release) a connection for every method called or once per template? 

Basically, every time a JdbcTemplate method is called, a connection is automatically opened for query
execution, and then the same connection is released and all the magic is done by the Spring utility class
org.springframework.jdbc.datasource.DataSourceUtils

### Is the JDBC template able to participate in an existing transaction? 

Yes it is, assuming you have configured a platformTransactionManager, and enabled transaction management
(@EnableTransactionManagement), and if you annotate a method containing jdbcTemplate code as @Transactional

### What is a transaction? What is the difference between a local and a global transaction? 

A transaction is a set of operations that take place as a single atomic unit - all or nothing. SQL transactions obey 
ACID properties
* Atomic - all sql operations are successfully committed else they are all rolled back
* Consistent - DB constraints are not violated
* Isolated - transactions are isolated from each other
* Durable - committed changes are permanent 

In a local transaction, one resource (e.g. database) is involved and the transaction is managed by the resource itself.
In a global transaction, multiple resources are involved and the transaction is managed by an external transaction
manager (e.g. JMS message must be successfully put in a queue + two different databases must commit SQL queries in order
for the transaction to be considered successful)

### Is a transaction a cross cutting concern? How is it implemented in Spring? 

Yes a transaction is a cross-cutting concern. Spring provides declarative transaction management support as well as
 programmatic transaction support (preferably using Spring's ```TransactionTemplate```).  Declarative is preferred, 
 which allows you to use the @Transactional annotation to declare classes/methods that should participate in a 
 transaction. Spring wraps these transactional classes in an AOP proxy using *Around* advice. Before entering a 
 transactional method a transaction is started, if everything goes smoothly the transaction is committed once the 
 method is finished. If the method throws a RuntimeException, a rollback is performed and the transaction is not 
 committed.

### How are you going to set up a transaction in Spring? 

There are a couple of steps to set-up transaction management.
1.  You need to configure a bean that implements the PlatformTransactionManager interface. Spring provides
several implementations of this interface that support various transactional technologies:
    1. **JtaTransactionManager** - used to delegate to a backend JTA provider
    2. **JpaTransactionManager** - implementation for a single JPA EntityManagerFactory
    3. **HibernateTransactionManager** - implementation for a single Hibernate SessionFactory
    4. **DatasourceTransactionManager** - implementation for a single JDBC DataSource
    5. ...
2. You need to annotate a Configuration class with @EnableTransactionManagement (or ```<tx:annotation driven />``` in XML)
3. Mark methods/classes as @Transactional (java config or xml config)

### What does @Transactional do? What is the PlatformTransactionManager? 

@Transactional declares that a class/method should be wrapped in a transaction.
PlatformTransactionManager is the core Spring interface that acts as an abstraction layer for different transaction
providers (see question above.) 

### What is the TransactionTemplate? Why would you use it? 

TransactionTemplate is a class that facilitates programmatic transaction management. It uses a callback approach, to 
free application code from having to do the boilerplate acquisition and release of transactional resources, and results 
in code that is intention driven (like JdbcTemplate). TransactionTemplate is the recommended approach for programmatic
transaction management versus working with the PlatformTransactionManager directly. 
* Programmatic transaction management is usually a good idea only if you have a small number of transactional operations
* If you want to set the transaction name explicitly, you have to use the programmatic approach to transaction management 
* Or perhaps you need functionality not available with Spring's declarative transaction management. The downside 
is you are coupled to Spring's transaction infrastructure

### What is a transaction isolation level? How many do we have and how are they ordered?

Transaction isolation level defines how data modified in a transaction affects other simultaneous transactions.
Generally we want transactions to be isolated. Technically there are five isolation levels defined in the
org.springframework.transaction.annotation.Isolation enumeration, but really there are four we care about

* **DEFAULT** - the default isolation level of the DBMS (typically READ_COMMITTED, but depends on DBMS)

from least strict to most strict:
* **READ_UNCOMMITTED** - dirty reads can occur - one transaction can read uncommitted data from another transaction.
* **READ_COMMITTED**  - other transactions can see data only after it is properly committed. Non-repeatable reads could
occur 
* **REPEATABLE_READ** - no dirty reads, prevents non-repeatable reads - a repeatable read is when a row is read multiple
times in a single transaction, its value is guaranteed to be the same
* **SERIALIZABLE** - no dirty reads, no repeatable reads, and no phantom reads are possible. A phantom read happens when
in the course of a transaction, execution of identical queries can lead to different result sets being returned.

* Definitions
  * dirty read - a transaction is allowed to read data from a row that has been modified by another transaction, but
  not yet committed
  * non-repeatable read - during a transaction a row is retrieved twice and the values in the row differ between reads
  * phantom read - during a transaction, two identical queries are executed, and the collection of rows returned by
  the second query is different from the first.
  

### What is the difference between @EnableTransactionManagement and <tx:annotation-driven >

The XML ```<tx:annotation-driven >``` can be declared without the attribute ```transaction-manager```. In this case
Spring is hardwired to look for a bean named *transactionManager* by default, and if it is not found the application will not
start. ```@EnableTransactionManagement``` is more flexible, it looks for a bean of any type that implements 
PlatformTransactionManager so the name is not important

### How does the JdbcTemplate support generic queries? How does it return objects and lists/maps of objects? 

JdbcTemplate provides a multitude of methods that allow you to query for entities (domain objects), generic maps, lists
and simple types (Date, Integer, String). There are also overloaded versions of these methods that allow you to write 
queries using '?' placeholders, and then provide values for these placeholders using a vararg parameter.
The JdbcTemplate query methods can be broken down into three different types based on what you expect a query to return:

* Queries for a Simple Object type
  * you are expecting a single type of object to be returned (Long,Integer,String,...)
  * use ```queryForObject( sqlStatement, Class<T> requiredType)```
  * ```
    String sql = "select count(*) from p_user";
    Integer count = jdbcTemplate.queryForObject(sql, Integer.class);
    ```
* Queries for a generic collection
  * use ```queryForMap()``` when you want to extract a single row and have the ResultSet's column and values mapped to 
  keys and values in the returned map
    * ```jdbcTemplate.queryForMap("select * from ACCOUNT where id=?", accountId)``` Results in 
    Map<String, Object> ( ID -> 1, BALANCE -> 75000, NAME -> "Checking Account",...)
  * use ```queryForList()``` methods when you expect a ResultSet to return return multiple rows. These methods can
  return either a List<Map<String,Object>> or List<T>
    * ```return jdbcTemplate.queryForList("select * from ACCOUNT);```
      Results in: 
      ```
      List< Map<String,Object> > { 
      Map<String, Object> {ID -> 1, BALANCE -> 75000, NAME -> "Checking account", ...} 
      Map<String, Object> {ID -> 2, BALANCE -> 15000, NAME -> "Savings account", ...} 
      ... }
      ```

* Query for a Domain Object
  * Use the ```queryForObject()``` with an implementation of ```RowMapper``` callback interface when you expect a single row
  of the result set to map to a single domain object
  ```java
  public interface RowMapper<T> {
      T mapRow(ResultSet resultSet, int rowNum) throws SQLException; 
  }
  ```
    * e.g. ```Account account = jdbcTemplate.queryForObject( sqlStatement, rowMapper, param1, param2 );```
  * If you want to map a ResultSet to multiple domain objects (of the same type) you can use a ```query()``` method with
  RowMapper implementation:
    * ```List<Account> = jdbcTemplate.query(sqlStatement, rowMapper, param1, param2);```
  * There is also the ```RowCallbackHandler``` interface
    * Should be used when **no return object is needed** e.g. streaming results, or building an XML object
    ```java
    public interface RowCallbackHandler {
        void processRow(ResultSet resultSet) throws SQLException;
    }  
    ```
  * There is also the ```ResultSetExtractor``` callback interface
    * Should be used when you need to process multiple ResultSet rows into a single return object
    ```java
    public interface ResultSetExtractor<T> {
        T extractData(ResultSet resultSet) throws SQLException, DataAccessException;
    }
    ```
    
      
### What does transaction propagation mean?

Transaction propagation deals with whether or not an existing transaction should be extended into another transaction
or if nested transactions should be used. It can happen when code from one transaction calls another transaction.

### What happens if one @Transactional annotated method is calling another @Transactional annotated method on the same object instance?

The second method being called will not obey any @Transactional properties you have configured for it. However, the
first method will still obey its @Transactional settings which will propagate into the second method.
Spring AOP proxies cannot intercept method calls between methods of the same object instance.

### Where can the @Transactional annotation be used? What is a typical usage if you put it at class level?

@Transactional can be used at the method or class level. If used on a method, the method must be public or else the 
@Transactional declaration will have no effect. If used at the class level, and the class implements at least one 
 interface method, then it **applies to all methods declared that implement interfaces.** @Transactional can also be 
 applied to the interface or parent class, but spring recommends that you only annotate concrete classes and methods of 
 concrete classes. Can also combine class level and method level on the same class. Typically, you define default 
  transactional behavior (for all methods of the class) at the class level, and then define specific transactional 
  behavior for each method, as needed. Method level overrides class level. 

### What does declarative transaction management mean?

Declarative transaction management allows you to manage a transaction with the help of configuration instead of hard 
coding implementation details in your source code. By using annotations, you specify where/when transactions should
occur and leave the "how to do it" to Spring. This allows you to separate transaction management logic from your business
logic.

### What is the default rollback policy? How can you override it?

The default rollback policy is to rollback a transaction if a RuntimeException (or any subtype) is thrown. This can be
overridden by using the following attributes
* ```rollbackFor```
* ```noRollbackFor```
* ```rollbackForClassName```
* ```noRollbackForClassName``` 

```java
@Transactional( rollbackFor=MyCheckedException.class, noRollbackFor={FooException.class, BarException.class} )
```

### What is the default rollback policy in a JUnit test, when you use the SpringJUnit4ClassRunner and annotate your @Test annotated method with @Transactional?

The default rollback policy for a JUnit test is to rollback after every test method. This can be overridden by using
the @Commit annotation on a test method OR @Rollback(false).

### Why is the term "unit of work" so important and why does JDBC AutoCommit violate this pattern?

With regards to transactions, a unit of work is a series of interactions with one or more resources that must complete
"all or nothing". If any interaction fails to complete, it must be as if none of the interactions ever happened.
JDBC autocommit violates this pattern because it treats every query as a single "unit of work". 





### What does JPA mean - what is ORM? What is the idea behind an ORM?

JPA is the Java Persistence API. It is an interface for persistence providers to implement.
ORM is an Object Relational Mapping. An ORM specifies how to map an object-oriented domain model to a relational 
database. The general idea behind an ORM is that rather than working directly with database tables and SQL for a 
particular database technology; a developer uses Java objects to store their domain model, and calls out to an ORM
implementation to map their objects into rows/columns of the whatever Relational Database they are using.

### What is a PersistenceContext and what is an EntityManager. What is the relationship between both?

Each EntityManager instance is associated with a persistence context: a set of managed entity instances that exist
in a particular data store. A persistence context defines the scope under which particular entity instances are
created, persisted, and removed. The EntityManager interface defines the methods that are used to interact with
the persistence context.

An EntityManager, manages entities. It takes care of creation,deletion,querying, and updating a set of entities. The
EntityManager manipulates the persistence context via its API:
* perist( Object object ) - adds entity to persistence context
* remove( Object object ) - removes entity from the persistence context
* find( Class entity, Object primaryKey ) - find by primary key
* Query createQuery( String queryString ) - used to create JPQL query
* flush() - current state of persistence context is written to the database
* ...

### Why do you need the @Entity annotation. Where can it be placed?

@Entity can only be placed at the class level. It marks a java class as a JPA entity and enables it to be mapped and
persisted in a dataStore by your ORM provider.

### What do you need to do in Spring if you would like to work with JPA?

1. Define an EntityManagerFactoryBean
    * this creates EntityManagerFactories which are in turn used to create EntityManagers
    * can create application managed EntityManagerFactory - ```LocalEntityManagerFactoryBean```
    * or can create container managed EntityManagerFactory (preferred) - ```LocalContainerEntityManagerFactoryBean```
      * you will not need to configure a persistence.xml file, configuration is done programmatically, on the bean
      * allows you to scan for your @Entity classes via its **setPackagesToScan()** method
    * you must also configure the EntityManagerFactoryBean with your JpaVendorAdapter
      * HibernateJpaVendorAdapter(), EclipseLinkJpaVendorAdapter(), OpenJpa, ...
2. Define a DataSource bean - required by the EntityManagerFactoryBean
3. Define a PlatformTransactionManager bean with a **JpaTransactionManager**
4. Define your DAOs
5. Define the metadata mappings on your entity beans (POJOs)
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

@Bean
public DataSource dataSource() {
    ...
}

@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) { 
    return new JpaTransactionManager(emf);
}
```
Or in XML:
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

### Are you able to participate in a given transaction in Spring while working with JPA?

Yes. If you inject a AOP-proxied EntityManager into your code (using @PersistenceContext), the proxied EntityManager
will make sure JPA queries are properly joined into active transactions.

### What is the PlatformTransactionManager?

PlatformTransactionManager is the core Spring interface that acts as an abstraction layer for different transaction
providers. When using JPA, for example, you would  want to configure a JpaTransactionManager. If using JDBC, you would
 want to configure a DataSourceTransactionManager, Hibernate would use a HibernateTransactionManager

### What does @PersistenceContext do?

@PersistenceContext is used to inject a "shared entity manager" into your object. This EntityManager is a proxy to the
actual entity manager. This proxy provides thread-safety and it is able to participate in transactions.   
**It can only be injected via field and setter injection.**

### What are disadvantages of ORM? What are the benefits?

ORM Benefits:
* not tied to a particular database implementation
* less code to write, reduces development time
* developers only work with java objects
* provides concurrency support
* provides management of cache, connection pool

ORM Disadvantages
* have to learn the ORM api
* may be difficult/impossible to use if you have to conform to an existing database schema
* could perform worse than writing your own queries / using JdbcTemplate

### What is an "instant repository"?

When using Spring Data, an instant repository abstracts away common repository functionality.  The common functionality 
 is provided "out of the gate" and reduces boilerplate code. The developer can customize/override the instant 
 repository, as needed, to provide specific functionality.
 
### How do you define an “instant” repository?

You declare an interface that extends the core spring data repository interface 
```Repository<T,ID extends Serializable>``` or extend one of its sub-interfaces:
* CrudRepository
* PageAndSortingRepository
* JpaRepository

Then you must use the @EnableJpaRepositories(basePackages={"pkgName"}) annotation to enable scanning for Repository interfaces


### What is @Query used for?

In spring data JPA, @Query is used to annotate a method of a repository interface with a query that should be run when
that method is called. It allows you to override the default methods query behavior or perhaps provide your own query
 if spring data's default method queries do not meet your needs.
```java
@Query("select u from User u where u.emailAddress = ?1")
  User findByEmailAddress(String emailAddress);
```



Spring MVC and the Web Layer
=====================================================================================================================

### MVC is an abbreviation for a design pattern. What does it stand for and what is the idea behind it?

MVC stands for: Model, View, Controller. The main idea is that there are three decoupled components. Each of them can
be easily swapped with a different implementation, and together they provide desired functionality. The ```View``` is
the interface with which the user interacts. It displays data and passes it to the Controller. The ```Controller```
calls business components and sends the data to the ```Model```. The Model content is displayed by the View.


### Do you need spring-mvc.jar in your classpath or is it part of spring-core?

Yes you need it in your classpath if you want to use spring mvc functionality.
The jar is called "spring-webmvc-[VERSION].jar"


### What is the DispatcherServlet and what is it used for?

The DispatcherServlet is a "front-controller" servlet. It is the central piece of Spring MVC and is the entry point
for every Spring Web application. It dispatches requests to handlers, with configurable handler mappings, view
resolution, locale, timezone, and support for uploading files. The DispatcherServlet converts HTTP requests into
commands for controller components and manages rendered data as well.


### Is the DispatcherServlet instantiated via an application context?

No. It is a servlet that must be configured either in ```web.xml``` or a configuration class that extends
```AbstractDispatcherServletInitializer``` or ```AbstractAnnotationConfigDispatcherServletInitializer```. These are
two specialized classes that implement ```WebApplicationInitializer``` which is a special interface. Objects implementing
this interface are detected automatically by ```SpringServletContainerInitializer``` which are bootstrapped
automatically by every Servlet 3.0+ environment. The DispatcherServlet creates a separate "servlet" application context
containing all specific web beans (controllers, views, view resolvers.) This context is also called the *web context*
or the *DispatcherServletContext*, it will inherit all beans defined in the "RootApplicationContext"



### What is the root application context? How is it loaded?

The *RootApplicationContext* or *root WebApplicationContext*. It contains all non-web
beans and is instantiated by a bean of type ```org.springframework.web.context.ContextLoaderListener```. The
RootApplicationContext is a parent of the WebContext, thus beans in the WebContext can access beans in the
RootApplicationContext but not vice versa.

* Can be configured either in web.xml or using AbstractContextLoaderInitializer - implements WebApplicationInitializer,
which is automatically recognized and processed by servlet container (Servlets 3.0+)
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
  * AbstractContextLoaderInitializer - only Registers ContextLoaderListener
  * AbstractAnnotationConfigDispatcherServletInitializer - Registers ContextLoaderListener and defines Dispatcher
  Servlet, expects JavaConfig

web.xml
* Register spring-provided Servlet listener
* Provide spring config files as context parameters - parameter name is "contextConfigLocation"
* Defaults to WEB-INF/applicationContext.xml if not provided
* Can provide spring active profiles in spring.profiles.active
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


### What is the @Controller annotation used for? How can you create a controller without an annotation 

@Controller is a Spring stereotype annotation used on classes. It is used to indicate that the class functions as a
web controller. It is also a specialization of @Component and is eligible for component scanning.

To create a controller without an annotation you need to implement the
```org.springframework.web.servlet.mvc.Controller``` interface or extend a class that implements it, such as
```org.springframework.web.servlet.mvc.AbstractController ``` and then expose it as a bean in the DispatcherServletContext.
You will also need to map request URLs to your controller using a class that implements the ```HandlerInterface```
such as ```SimpleUrlHandlerMapping``` and then call its setMappings() method to Map URL paths to handler bean names.
The DispatcherServlet will find then automatically find your "HandlerInterface" bean(s) and use them to call your
controller.
```xml
<mvc:annotation-driven/>
<bean id="testController" class="com.test.web.TestController"/>
<bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter"/>
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
  <property name="mappings">
      <value>
      /test=testController
      </value>
  </property>
  <property name="order" value="0"/>
</bean>
```

```java
public class TestController implements Controller {

    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        PrintWriter responseWriter = response.getWriter();
        responseWriter.write("test");
        responseWriter.flush();
        responseWriter.close();
        return null;
    }
}
```


### What is the ContextLoaderListener and what does it do? 

It's a bootstrap listener that implements ServletContextListener. It starts up and shuts down Spring's root
WebApplicationContext. The root WebApplicationContext should contain all the infrastructure beans that should be
shared between your other contexts and Servlet instances. (e.g. Services and Repositories)



### What are you going to do in the web.xml. Where do you place it? 

By convention, the web.xml resides in the WEB-INF directory.
If you are not configuring your application using a pure java config, you have to use the web.xml file to configure
your RootApplicationContext and tell it where to find your bean configuration(s). You also need to use web.xml to
 load the DispatcherServlet and configure it with the location of your web specific beans as well as
 configuring it with the URL mappings it should handle.

Upon initialization of a DispatcherServlet, Spring MVC looks for a file named [servlet-name]-servlet.xml in the WEB-INF
directory of your web application and creates the beans defined there, overriding the definitions of any beans
defined with the same name in the global scope.

```xml
<web-app>

    <!-- set the location of your bean configurations for the RootApplicationContext
          Defaults to WEB-INF/applicationContext.xml if not provided -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            /WEB-INF/config.xml
            /WEB-INF/other-config.xml
        </param-value>
    </context-param>

    <!-- configure the dispatcher servlet -->
    <servlet>
        <servlet-name>exampleDispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

        <!-- explicitly specifying the location of web specific bean configurations 
             If <init-param> were omitted, Spring would look for configs in default location of  [servlet-name]-servlet.xml -->
        <init-param>
          <param-name>contextConfigLocation</param-name>
          <!-- If param-value were to be left empty here, you would be configuring just one WebApplicationContext
                namely the RootApplicationContext defined above, all beans would be loaded into that. -->
          <param-value>
            /WEB-INF/spring/mvc-config.xml
          </param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!-- configure the mappings the DispatcherServlet should handle -->
    <servlet-mapping>
        <servlet-name>exampleDispatcher</servlet-name>
        <url-pattern>/example/*</url-pattern>
    </servlet-mapping>

    <!-- register the Spring provided servlet listener, responsible for bootstrapping
           the RootApplicationContext -->
    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <!-- set the active profiles if you need them -->
    <context-param>
        <param-name>spring.profiles.active</param-name>
        <param-value>profile1,profile2</param-value>
    </context-param>

</web-app>
```


### How is an incoming request mapped to a controller and mapped to a method 

@RequestMapping annotation on controller's methods specifies which requests they should handle as well as the type
of HTTP request (GET,POST,PUT...). The @RequestMapping can also be used at class level, to simplify mappings at
method level when the mappings have common elements.
More specifically, the DispatcherServlet is responsible for mapping incoming requests to controllers. It uses some
infrastructure beans called "Handler Mappings" to identify a controller method to call
and "Handler Adapters" to call it.


### What is the @RequestParam used for 

@RequestParam is used to extract a parameter from an HTTP request and inject it into an argument of a controller
method. Parameter values are converted to the declared method argument type
```java
//this would extract value "xxx" from requested url: /foo?bar=xxx
@RequestMapping("/foo")
public String foo(@RequestParam("bar") String bar) {
    ...
}
```

### What are the differences between @RequestParam and @PathVariable  

@RequestParam extracts the values of query parameters (sometimes called the URL query string)
@PathVariable extracts the value of a placeholder from the URI, ( Spring URI template )



### What are some of the valid return types of a controller method 

* ModelAndView -
* Model -
* Map -
* View -
* String - interpreted as the logical view name
* void - if the method handles the response itself
* HttpEntity<?> -
* ResponseEntity<?> -
* HttpHeaders -
* ...




### What is a View and what's the idea behind supporting different types of View  

A View renders output to the client based on data in the Model. It is the interface with which the user interacts. It
  displays data and passes requests to the Controller.
The idea behind supporting different types of View is to keep Spring decoupled from any specific view technology.


### How is the right View chosen when it comes to the rendering phase 

Spring uses the ```ViewResolver``` and ```View``` interfaces to choose the right view for rendering.
The ```ViewResolver``` interface maps logical view names to actual views. It has a method
```View resolveViewName( String viewName, Locale locale ) throws Exception;``` that returns a ```View``` when given a
 logical view name and locale. The View interface's job is to take the model, as well as the servlet request and
 response objects, and render output into the response. Spring provides 13 implementations of the ViewResolver
 interface.

* View Resolution Sequence
  1. Controller returns logical view name to DispatcherServlet
  2. ViewResolvers are asked in sequence (based on their Order)
  3. If a ViewResolver matches the logical view name it then returns which View should be used to render the output.
  If not, it returns null and the chain continues to the next ViewResolver
  4. DispatcherServlet passes the model to the resolved view and it renders the output.


### What is the Model? 

The Model contains data used to populate a view. In spring it is the name of the interface that defines a holder for
Model attributes. This interface allows you to access the Model as a java.util.Map and is designed to abstract the
view technology.


### Why do you have access to the model in your View? Where does it come from? 

You have access to the Model in your View because the view typically needs to display data from the model. The data
in a Model is (typically) retrieved from business components (services) that are called by the Controller. The
Controller puts the data into the Model (which is essentially a Java Map) and sends it back to the DispatcherServlet
 which then sends it to the view to be rendered. The model Map is simply transformed into an appropriate format,
 such as JSP request attributes, or a Velocity template model, etc.... so that it can be accessed in your view.


### What is the purpose of session scope 

To keep a Spring managed bean scoped to the lifetime of an HTTP Session.



### What is the default scope in the web context? 

Singleton


### Why are controllers testable artifacts? 
First, controllers are Spring Beans.
Spring provides the *Spring MVC Test Framework* which allows you to test with JUnit, TestNG or any other testing
framework. It's built on **Servlet API Mock Objects** from the spring-test module and hence does not use a running
Servlet Container. It uses the DispatcherServlet to provide full Spring MVC runtime behavior and provides support
for loading actual Spring Configuration with the TestContext framework in addition to a standalone mode in which
controllers may be instantiated manually and tested one at a time.



###  What does the InternalResourceViewResolver do? 

This is the default ViewResolver in Spring MVC and comes pre-configured (it can be overridden)
It resolves views as resources internal to the web application (typically JSPs ).
It supports ```InternalResourceView``` (in effect, Servlets and JSPs) and subclasses such as JstlView and TilesView 





Security
=================================================================================================================
###  What is the delegating filter proxy?

It's the entry point for Spring Security.

Specifically, it's a bean in the class ```org.springframework.web.filter.DelegatingFilterProxy```. It intercepts
 secured requests and delegates calls to a list of chained security filter beans.



###  What is the security filter chain?

It's a chain of Spring managed beans that have the following responsibilities:
* driving authentication
* enforcing authorization
* managing logout
* maintaining *SecurityContext* in HttpSession

These beans all implement the javax.servlet.Filter interface.





###  In the notes several predefined filters were shown. Do you recall what they did and what order they occurred in?

* ChannelProcessingFilter - used if redirection to another protocol is necessary
* SecurityContextPersistenceFilter - used to set up a security context and copy changes from it to HttpSession
* ConcurrentSessionFilter - Used for concurrent session handling package
* LogoutFilter - used to log a principal out
* BasicAuthenticationFilter - used to store a basic authentication token in the security context
* JaasApiIntegrationFilter - This bean attempts to obtain a JAAS Subject and continue the FilterChain running as that Subject
* RememberMeAuthenticationFilter - used to store a valid Authentication and use it if security context did not change
* AnonymousAuthenticationFilter - stores an anonymous Authentication and uses it if the security context did not change
* ExceptionTranslationFilter - used to translate Spring Security exceptions into HTTP corresponding error responses
* FilterSecurityInterceptor - used to protect web URIs and raise access denied exceptions

Every time an HTTP request is received by the server, each of these filters performs its action if the situation
applies.



###  Are you able to add and/or replace individual filters 

Yes, in your security configuration:
* In xml config, using the *custom-filter* element
  * the ```position``` attribute takes a pre-defined enum value that indicates what order you want your filter to
  run in. Your custom filter will override the default filter in that position.
```xml
...
<http>
  <!-- position uses an ENUM in spring-security.xsd to define the position the filter should run in -->
  <custom-filter position="CONCURRENT_SESSION_FILTER" ref="customConcurrencyFilter" />
  <!-- custom-filter also has a 'before' and 'after' attribute -->
  <custom-filter after="CONCURRENT_SESSION_FILTER" ref="customTokenCheckerFilter" />
  
  <beans:bean id="customConcurrencyFilter" class="com.ps.web.session.CustomConcurrentSessionFilter"/>
  <beans:bean id="customTokenCheckerFilter" class="com.ps.web.myFilter.TokenCheckerFilter"/>
</http>

```
* In java config, configure the ```HttpSecurity``` object using either:
  * ```addFilterAfter( Filter filter, Class<? extends Filter> afterFilter )```
  * ```addFilterAt( Filter filter, Class<? extends javax.servlet.Filter> atFilter)```
  * ```addFilterBefore( Filter filter, Class<? extends Filter> beforeFilter)```
  ```java
   @Configuration
   @EnableWebSecurity
   @EnableGlobalMethodSecurity(jsr250Enabled = true)
   public class SecurityConfig extends WebSecurityConfigurerAdapter {

   @Override
   protected void configure(HttpSecurity http) throws Exception {
     
     http.addFilterAfter( SecurityContextPersistenceFilter.class, customConcurrencyFilter() );

     http
     .authorizeRequests()
      //...
     .logout()
     .logoutUrl("/logout")
     .logoutSuccessUrl("/");
    }
   //...

   @Bean
   CustomConcurrentSessionFilter customConcurrencyFilter() {
      return new CustomConcurrentSessionFilter();
   }
   }
  ```


###  Is it enough to hide sections of my output (e.g. JSP-Page) 

No. You should always configure url authorization rules in your security configuration as well. This will
prevent someone from entering a "hidden" URL directly into a browser. Additionally, you may want
to use method level security as well, especially if you have implemented web services you want to secure.



###  Why do you need the intercept-url  

In an xml configuration, it is used to secure URL(s) by defining what authorities a principal needs to access the
given URL. ```<intercept-url pattern="/**" access="hasRole('USER')" />```



###  Why do you need method security? What type of object is typically secured at the method level? (think of its purpose not its Java type)

It's an extra layer of security applied at the lowest level (currently) available. In a properly designed system, the
front-end (web-layer) and back-end should know nothing about each other. Your server-side code should not rely on
the front-end to perform all security. In fact, you may need method level security to help secure any web-services
that have been written (e.g. a REST API)

Service layer objects that access sensitive data should be secured at the method level.



###  Is security a cross cutting concern? How is it implemented internally 

Yes, security is a cross-cutting concern. Spring uses AOP to implement Authentication and Authorization with the help
of a few infrastructure beans.

Spring Security high-level overview:
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



###  What do @Secured and @RolesAllowed do? What is the difference between them 

* @Secured - is a Spring framework specific annotation for method level security
  * supports role based security as well as using AuthenticateVoter checking - e.g. @Secured("IS_AUTHENTICATED_FULLY")
  * cannot use SpEL in the annotation
* @RolesAllowed - is defined by JSR-250 annotation for method level security
  * only supports role based security - @RolesAllowed("ROLE_ADMIN")

  

###  What is a security context 

The security context contains details about the principal currently using the application. Of note, it contains the
```Authentication``` object (principal plus their authorities.)




###  In which order do you have to write multiple intercept-url's 

The most restrictive rule must be on top, otherwise a more relaxed rule will apply and some URL will be accessible to
users that should not have access to them.



###  How is a Principal defined

A principal is a user,device, or system that is performing an action.

* ```UserDetails```, to provide the necessary information to build an Authentication object from your application’s
DAOs or other source of security data.
* UserDetailsService, to create the UserDetails object when passed in a String-based username (or certificate ID or the like).
UserDetailsService is purely a DAO for user data and performs no other function other than to supply that data to
other components within the framework.

Configuring a principal can be done in a number of different ways. Spring supports XML configs and Java configs
* Can be defined directly in your security-config.xml or a properties file. these will be read and stored in a
in-memory ```AuthenticationProvider```
  * security-config.xml
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
    ```xml
    <!-- spring-config.xml -->
    <!-- reading from a properties file. The password is stored encrypted within the properties file -->
    <authentication-manager>
      <authentication-provider>
        <password-encoder hash="md5" >
         <salt-source system-wide="SpringSalt"/>
        </password-encoder>
        <user-service properties="/WEB-INF/users.properties" />
      </authentication-provider>
    </authentication-manager>
    ```
  * properties file format:   [username] = [password(encrypted)],[role1,role2...][,enabled||disabled]
* Spring also provides support for retrieving credentials from LDAP or a jdbc data store

Once your principals and their authorities are defined somewhere, Spring uses an ```AuthenticationProvider``` to access them
* ```AuthenticationProvider``` interface
  * processes authentication requests and returns ```Authentication``` object
  * the default provider is ```DaoAuthenticationProvider```
    * a ```UserDetailsService``` implementation is needed to provide credentials and authorities
    * spring provides some built-in implementations
      * LDAP
      * in-memory
      * JDBC
        * uses two tables named *users* and *authorities* - must have a specific schema (see ref. docs for info)
      * can build a custom authentication provider

The authentication provider can be used to access the UserDetailsService which can then be used to get the UserDetails.
```UserDetails``` is a core interface in Spring security. It represents a principal, but in an extensible and
application-specific way.
e.g. ```UserDetails userDetails =  daoAuthProvider.getUserDetailsService().loadUserByUsername("someUserName");```

On successful authentication, Spring uses the UserDetails to build the ```Authentication``` object that is
stored in the SecurityContext. You can then use the Authentication object to get details on the principal
currently interacting with the application.

* Obtaining information about the current user
  * You can use the following code block - from anywhere in your application - to obtain the name of the currently
  authenticated user:
    ```java
    Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

    if (principal instanceof UserDetails) {
      String username = ((UserDetails)principal).getUsername();
    } else {
      String username = principal.toString();
    }
    ```

   


###  What is authentication and authorization? Which must come first 

* Authentication - the process of checking the identity of a principal. (Are you who you say you are?)
* Authorization - the process of checking if a principal has privileges to perform requested action (Are you allowed to do that?)

Authentication must happen before authorization.



###  In which security annotation are you allowed to use SpEL 

You can use SpEL in the Pre/Post authorize annotations
* @EnableGlobalMethodSecurity(prePostEnabled=true) on @Configuration to enable
* Pre authorize - can use SpEL (@Secured cannot), checked before annotated method invocation
  * @PreAuthorize("hasRole('ROLE_ADMIN')")
* Post authorize - can use SpEL, checked after annotated method invocation, can access return object of the method
using ```returnObject``` variable in SpEL; If expression resolves to false, return value is not returned to caller




###  Does Spring Security support password hashing? What is salting 

Yes, using the ```org.springframework.security.crypto.password.PasswordEncoder``` interface, Spring provides a number
of implementations of hashing algorithms (md5, sha, bcrypt, ...). Salting is adding a well-known string to a password
string in order to increase the security of a passwords (salting prevents rainbow table attacks)






REST
=====================================================================================================================

### What does REST stand for? 

REpresentational State Transfer



### What is a resource? 

A Resource is an object with a type, associated data, relationships to other resources, and a set of methods that
operate on it. It is similar to an object instance in object-oriented languages, with the important difference that
only a few standard methods are defined for that resource (e.g. HTTP GET,POST,PUT,DELETE...)



### What are safe REST operations? 

Safe operations are operations that do not modify **resources**.
* GET
* OPTIONS
* HEAD



### What are idempotent operations? Why is idempotency important? 

Idempotent operations are operations that can be called many times **without** different outcomes. The **result** of calling
an idempotent operation should always be the same. Idempotent operations produce the same result on the server (no
 side effects.) Idempotency is important because it helps in building a fault tolerant API. If a fault occurs in the
 middle of an idempotent method, you can safely retry it until you get back a result.

* GET
* PUT 
  * idempotent, (unless subsequent calls of the same PUT request increments a counter in the resource, for example)
  * PUTting the same resource twice should not modify the resource
* DELETE - considered idempotent because requests to delete a resource that no longer exist will always return 404
* OPTIONS
* HEAD

### Is REST scalable and/or interoperable? 

Yes, a RESTful architecture should be a stateless client-server architecture, so the system is loosely coupled.
 The stateless nature of REST increases scalability since the server does not have to maintain, update or
communicate session state. Since HTTP is supported by every platform/language, REST communication can be done between
quite a wide range of different systems, and thus it is very scalable and interoperable.



### What are the advantages of REST template? 

* Simplifies calls to a RESTful API
* Provides a wide set of methods for each HTTP method that can be used to access RESTful services
* Message converters are supported - Jackson, GSON,
* Automatic input/output conversion - using HttpMessageConverters
  * StringHttpMessageConverter - text/plain
  * MappingJackson2HttpMessageConverter - application/*+json
  * AtomFeedHttpMessageConverter - application/atom+xml
  * MappingJackson2XmlHttpMessageConverter - application/*+xml
  * ....



### Which HTTP methods does REST use? 

GET,POST,PUT,PATCH,DELETE,HEAD,OPTIONS



### What is an HttpMessageConverter?  

They are used to convert resources to/from different representations formats.



### Is REST normally stateless? 

Ideally...yes. RE**ST** transfers the state of a resource to the client. Servers do not store state about a client (
e.g. client session)



### What does @RequestMapping do? 

It maps request URLs to a Controller class and/or method.



### Is @Controller a stereotype? Is @RestController a stereotype? 

@Controller is a stereotype. @RestController is not.


### What is the difference between @Controller and @RestController 

@RestController is meta-annotated with @Controller. In addition to regular @Controller functionality it automatically
adds @ResponseBody to the return types of all @RequestMapping methods in the controller


### When do you need @ResponseBody?  

Whenever you need to write a return type (e.g. a resource representation) directly to the HTTP response body and NOT
placed in a Model or interpreted as a view name.


### What does @PathVariable do? 

@PathVariable extracts the value of a placeholder from the URI in a @RequestMapping, ( Spring calls this URI 
the Spring URI template )



### What is the HTTP status return code for a successful DELETE statement? 

* 204 NO Content - the server successfully processed the delete request, but is not returning any content



### What does CRUD mean 

* Create --> POST
* Read   --> GET
* Update --> PUT
* Delete --> DELETE



### Is REST secure?  What can you do to secure it? 

Not by default but it can be made secure. Since REST is stateless, credentials have to be embedded in every
request header. Basic authentication is the easiest to implement but it guarantees the lowest level of security.
Should always use TLS (SSL) encryption.
You can use Spring Method Security to secure the methods of your REST application.



### Where do you need @EnableWebMVC ?  

In a @Configuration class. The configuration class has to be annotated with the @Configuration and @EnableWebMvc
annotation and has to either implement WebMvcConfigurer or extend an implementation of this interface, for example
WebMvcConfigurerAdapter, which gives the developer the option to override only the methods he or she is interested in.
Annotating a configuration class with @EnableWebMvc has the result of importing the Spring MVC configuration
implemented in the WebMvcConfigurationSupport class and is equivalent to ```<mvc:annotation-driven />```.


### Name some common HTTP Response Codes. When do you need @ResponseStatus ? 

* 200 - OK
* 201 - CREATED
* 204 - NO_CONTENT
* 404 - NOT_FOUND
* 403 - FORBIDDEN
* 405 - METHOD_NOT_ALLOWED
* 409 - CONFLICT
* 415 - UNSUPPORTED_MEDIA_TYPE

You need @ResponseStatus whenever you want to explicitly set the status code of a response. You can use it on a
@RequestMapping method, on an Exception class, or at the Controller class level - will be applied to all 
@RequestMapping handler methods. Having a controller method annotated with @ResponseStatus will stop the
DispatcherServlet from trying to find a view to render


### Does REST work with Transport Layer Security (TLS) ?

Yes.



### Do you need Spring MVC in your classpath?

Yes you do, if you want to use Spring's REST classes.



Spring Boot
===================================
###  What is Spring Boot 

It's a set of pre-configured frameworks/technologies designed to reduce boilerplate configuration and provide a quick
way to have a spring web application up and running.



###  What are the advantages of using Spring Boot 

In no particular order:
* Convention over Configuration:
  * Spring Boot will pre-configure your application with reasonable defaults, which you can override if needed
  * dependencies are curated so Spring Boot will automatically select known working versions of spring jars to use
  * reduces the amount of configuration code you have to write
  * default location for externalized configuration (application.properties) file(s), automatically read into a PropertySource
  * defaults for common functionality auto-configured (logging, datasource, conn. pools, security, transactions, ...)
* Comes with an embedded app. server (Tomcat is default, can use Jetty, or Undertow)
  * makes it easy to test your web app on a real application server versus having to deploy to a remote
* Can build a web application that runs as a standalone jar with embedded application server
  * recommended for Cloud / microservice deployments

In short, Spring Boot helps you get an application up and running in a short time




###  Why is it "opinionated" 

When you start a new Spring Boot project, it will include common frameworks/technologies that are known to work together
 and will pre-configure/wire them for you. This is designed to get you up and running with minimal-fuss, but it also
 allows you to override any configurations with your own.




###  How does it work? How does it know what to configure 

It's triggered by @EnableAutoConfiguration annotation which is part of @SpringBootApplication.
Spring Boot will then attempt to configure beans for you with sensible default values, by examining
your starter poms as well as other jar dependencies.

Spring Boot makes used of the various @Conditional annotations located in ```org.springframework.boot.autoconfigure```
 to help it auto-configure beans.
eg. @ConditionalOnMissingBean(Datasource.class) will create a dataSource bean if you have a database driver as a
dependency.

For example if Boot sees a dependency on Thymeleaf, 3 beans that Thymeleaf uses are automatically initialized:
* ThymeleafViewResolver
* SpringTemplateEngine
* TemplateResolver

Another example: If HSQLDB is on your classpath, and you have not manually configured any database connection beans,
then Boot will auto-configure an in-memory database. If spring-boot-starter-data-jpa is listed as a dependency then
a Jpa entity manager will be configured as well, using whatever database jar you have included as a dependency.



###  What things affect what Spring Boot sets up 

Spring Boot examines the "starters" you have listed as dependencies as well as other jar dependencies.



###  How are properties defined? Where 

By default, Spring boot will look for properties in a file named ```application.properties``` (or application.yml)
Spring will look for this application.properties in the following locations (in order of precedence):
* A /config subdirectory of the current directory.
* The current directory
* A classpath /config package
* The classpath root



###  Would you recognize common Spring Boot annotations and configuration properties if you saw them in the exam 

* @SpringBootApplication -
* @EnableAutoConfiguration
* @ConfigurationProperties( prefix="com.example" )
  * Needs to be enabled on @Configuration class - ```@EnableConfigurationProperties( MyProperties.class )```
* @Conditional
    * Only when specific bean is found - ```@ConditionalOnBean( type={Datasource.class} )```
    * Only when specific bean is not found - ```@ConditionalOnMissingBean```
    * Only when specific class is on the classpath - ```@ConditionalOnClass```
    * Only when specific class in not present on classpath - ```@ConditionalOnMissingClass```
    * Only when system property has certain value - ```@ConditionalOnProperty( name="server.host",havingValue="localhost" )```

###  What is the difference between an embedded container and a WAR 

An embedded container is included in the build of your jar file.  Running the jar will start the embedded container with
 your application pre-loaded.

A WAR typically only contains your application code plus its dependencies. You must manually deploy a WAR file into
an application server.

NOTE: Artifacts built with Spring Boot, whether jars or wars, are executable and can be executed with java -jar.



###  What embedded containers does Spring Boot support 

Out of the box: Tomcat 7.X, Tomcat 8.X, Jetty 8.X, Jetty 9.2+, Undertow 1.3



###  What does ```@EnableAutoConfiguration``` do? What about ```@SpringBootApplication``` 

* @EnableAutoConfiguration - triggers spring boot's auto-configuration. It will examine classpath (looks at your jar
dependencies)
* @SpringBootApplication - it's made up of three annotations:
  * @Configuration
  * @ComponentScan
  * @EnableAutoConfiguration



###  What is a Spring Boot starter POM? Why is it useful 

The starters contain a lot of the dependencies that you need to get a project up and running quickly and with a
consistent, supported set of managed transitive dependencies.
Example starters: spring-boot-starter, spring-boot-starter-web, spring-boot-starter-test,...

* Parent pom defines all the dependencies using dependency management, specific version are not defined in our pom.xml
* Only version which needs to be specifically declared is the parent pom version
```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>1.3.0.RELEASE</version>
</parent>
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
</dependencies>
```



###  Spring Boot supports both Java properties files and YML files. Would you recognize and understand them if you saw them 

###  Can you control logging with Spring Boot? How 

Yes. logback over SLF4J is the default. You can change logging levels, log file names, logging paths in
```application.properties```
```properties
#Logging through SLF4J
logging.level.org.springframework=DEBUG
logging.level.com.example=INFO

logging.file=logfile.log
#OR spring.log file in to configured path
logging.path=/log
```
* To change logging framework from default logback
    * Exclude logback dependency ```ch.qos.logback.logback-classic```
    * Add desired logging framework dependency - eg. ```org.slf4j.slf4j-log4j12```

Can also define logback configuration using the traditional logback.xml file. Typically located at /src/main/resources




Microservices
====================================
###  What is a microservices architecture?

It's a type of Service Oriented Architecture that requires services to be broken down into highly specialized instances
of functionality (a microservice) that communicate with each other through agnostic communication protocols (like REST).
These services then work together to accomplish a common business goal. Each microservice is a really small unit of
stateless functionality. It has no idea of what the big picture is. It does not care where input is coming from and does
 not care where it is going to.

###  What are the advantages and disadvantages of microservices?

* Advantages
  * increased granularity
  * increased scalability
  * easy to automate deployment and testing
  * increased decoupling
  * enhanced cohesion
  * suitable for continuous refactoring, integration and delivery
  * improved agility and velocity - each microservice can be developed and deployed independently and in parallel with others
  * each service is elastic (can scale up or down independent of others)
  * improved fault isolation
  * not committed to any particular tech. stack because microservices can be written in different programming languages
* Disadvantages
  * introduce additional complexity and the necessity of careful handling of requests between modules
  * handling multiple databases and transactions can be painful
  * testing can be cumbersome
  * deployment may require coordination among multiple modules



###  What sub-projects of Spring Cloud did we cover in the course? Spring cloud is a large umbrella project -- only what was covered in the course will be tested.

* Spring Cloud Config - provides centralized external configuration backed by a Git repository
* Spring Cloud Security -
* Spring Cloud Connectors - simplifies the process of connecting to services and gaining operating environment awareness in cloud platforms
* Spring Cloud Data Flow - a toolkit for building data integration and real-time data processing pipelines.
* Spring Cloud Bus - provides a messaging system that can be used to monitor and manage components within the framework
* Spring Cloud for Amazon Web Services - eases the integration with hosted Amazon Web Services
* Spring Cloud Netflix - provides netflix OSS integrations for Spring Boot apps
* Spring Cloud Consul - provides Consul integrations for Spring Boot apps through auto-configuration
and binding to the Spring Environment and other Spring programming model idioms


###  Would you recognize the Spring Cloud annotations and configuration we used in the course if you saw it in the exam?
* @EnableEurekaServer
* @EnableDiscoveryClient
* @LoadBalanced - used on a restTemplate along with @Autowired

###  What Netflix projects did we use?
* *Eureka* - an OSS discovery service (aka a registration server)
* *Ribbon*
  * Ribbon is a client side IPC library that is battle-tested in cloud. It provides the following features:
    * Load balancing
    * Fault tolerance
    * Multiple protocol (HTTP, TCP, UDP) support in an asynchronous and reactive model
    * Caching and batching
  * if you are using Eureka in your clients, then it is used in your ```restTemplate``` to do micro-service
lookup and load balance across them

###  What is Service Discovery? How is it related to Eureka?

Service discovery is the process by which microservices become aware of other microservices. Eureka is Netflix's
implementation of a "discovery server". A discovery server keeps a registration of the microservices instances in your
application. When a microservice starts-up, it has to register itself with the discovery server so other microservices
can find it.

###  How do you setup Service Discovery?

1. Include the appropriate starter as a dependency: e.g.  spring-cloud-starter-eureka-server
2. Create a spring boot application that is annotated ```@EnableEurekaServer```
3. Provide it with Eureka configurations in your application.properties (or application.yml)
```properties
server.port=8761
#Not a client
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

logging.level.com.netflix.eureka=OFF
logging.level.com.netflix.discovery=OFF
```

###  How do you access a RESTful microservice?

You use a special "smart" version of restTemplate which manages load balancing and automatic service lookup

1. Annotate client app using the microservice (@SpringBootApplication) with @EnableDiscoveryClient
2. call the microservice using restTemplate:
    * Inject RestTemplate using @Autowired with additional **@LoadBalanced**
    * "Ribbon" service by netflix provides load balancing
    ```java
    @Autowired
    @LoadBalanced
    private RestTemplate restTemplate;
    //
    //
    restTemplate.getForObject("http://persons-microservice-name/persons/{id}", Person.class, id);
    ```

    
