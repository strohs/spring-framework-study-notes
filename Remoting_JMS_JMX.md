Remoting
=====================================================================================================================
## Middleware
* allows software components that were developed independently and run on different network platforms to interact
* Three types:
  * Remote Procedure Call (RPC) - allows one application to call procedures on a remote application as if they were
  local calls
  * Object Request Broker (ORB) - enables an application's objects to be distributed and shared across heterogeneous
  networks. (Remoting fails in this regard)
  * Message Oriented Middleware (MOM) - allows distributed applications to communicate and exchange data by sending
  and receiving messages

## Remoting Overview
* way of communicating between applications on same computer, different computers, different networks, with diff. languages
* applications communicating know about each other
* there is a server application and a client application
  * supports state management
  * server can correlate multiple calls from the same client
  * can support callbacks
* Remoting client
  * creates a proxy of the server target object and uses that to access the object on the server
* Can be used across any protocol but **has trouble with firewalls**
* Relies on existence of common language runtime assemblies that contain information about data types
  * limits information that must be passed about an object and allows pass by value or by reference
  * communication done using a binary,XML, or JSON format
* All these limitations make remoting quite deprecated and Web-Services have replaced them

## Remoting Technologies Provided by Spring
* **Remote Method Invocation** (RMI)
  * used for accessing and exposing Java based services when network constraints like firewalls aren't a factor
* **Hessian** or **Burlap**
  * used for accessing and exposing Java based services over HTTP when network constraints aren't a factor
  * Hessian is a binary protocol
  * Burlap is XML-based
    * burlap is deprecated as of Spring 4
* **Spring's HTTP-based remoting**
  * used for accessing and exposing **Spring-based** services when network constraints aren't a factor and you
  desire Java serialization over HTTP rather than proprietary serialization
* Web-Services with JAX-RPC and JAX-WS
  * accessing/exposing platform-neutral, SOAP based web services


### RMI (general information)
* RMI (Remote Method Invocation) is Java's version of RPC
  * introduced in JDK 1.1
  * Often consists of two processes: a server and a client
  * Server creates objects that will be accessed remotely and exposes a *skeleton* (interface of the remote object)
  * "Client Service" invokes methods on a stub (proxy object) and stub works with skeleton which will in turn call
  the actual "Remote Service"
* Client and Server run in different JVMs
* Disadvantages:
  * Client and Server are coupled to the RMI framework
    * server must extend ```java.rmi.remote```
    * client must catch the checked exception: ```java.rmi.RemoteException```
  * RMI stub and skeleton have to be generated with **rmic** compiler (in JAVA_HOME/bin)
  * An RMI registry must be started to host the services
  * Extra code must be written to handle binding and retrieving objects from the remote server

### Using RMI in Spring
* Uses **JRI** (Java Remote Invocation) to allow objects in one JVM to access objects on a remote JVM
* Uses marshalling to pass objects between client and server
  * **objects must implement ```java.io.Serializable``` **
* Uses JRMP (Java Remote Method Protocol) for remote Java object communication
* Spring provides two utility classes for writing remote application that are designed to hide the "plumbing" and help
developers write remote applications quickly and easily
  * ```org.springframework.remoting.rmi.RmiServiceExporter```
    * allows you to expose a "service" (a Java POJO) interface as a RMI Service
    * Wraps your service bean (POJO) in an adapter class that proxies requests to your actual service bean
    * allows you to bind to a registry or expose an endpoint (spring will do this for you)
    * existing beans can be exposed as RMI services without modifying their code
        * Thus, the service interface is not required to implement the RMI Remote interface
  * ```org.springframework.remoting.rmi.RmiProxyFactoryBean```
    * allows you to access a "service" interface via proxies that are created to communicate with the server-side endpoint
    * Remoting exceptions are unchecked (there is a hierarchy of remoting exceptions all extending
      org.springframework.remoting.**RemoteAccessException**), so the code to catch and handle them does not have
      to be written.
* Spring also provides different protocol versions of the above two classes
  * RMI over HTTP
  * RMI using Hessian
  * RMI using Burlap (Deprecated in Spring 4)

#### Spring Remoting Advantages
* Working with remote services in Spring is purely a matter of configuration
  * You don't have to write any Java code to support remoting
  * Your service beans don't have to be aware they are involved in a RPC
* Proxy beans created via Spring remoting will catch ```jmi.rmi.RemoteException``` and throw it as an unchecked
exception under the hierarchy ```org.springframework.remoting.RemoteAccessException```

### Spring RMI Configuration (Overview)
1. Configure your rmi "service" (the server side)
    * configure a RmiServiceExporter bean
2. Configure your rmi "client"  (the client side)
    * configure a RmiProxyFactoryBean
3. Start your rmi service
    * start a spring application and load your rmi service bean(s) and other beans into the ApplicationContext
4. Use your rmi client to connect to the service (one easy way to do this is in a test class)
    * inject the client bean that implements RmiProxyFactoryBean and start calling the methods on it

### Spring RMI Configuration
1. configure a server bean that extends **RmiServiceExporter** with the following properties:
    * *registryPort* - OPTIONAL - exported RMI service port, default value is 1099
    * *alwaysCreateRegistry* - OPTIONAL - defaults to false. will attempt to locate an existing registry first, if
    not found, one will be created. If set to true, no lookup is performed, and registry is created per client request
    * **serviceName** - REQUIRED - the exported RMI service will be available at **rmi://host:port/serviceName**
    * **serviceInterface** - REQUIRED - this property is used to set the interface of the bean type that will be exposed
    as an RMI service.
    * **service** - REQUIRED - sets the bean that will be exposed as an RMI service
    ```xml
    <!-- rmi-server-config.xml -->
    <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:p="http://www.springframework.org/schema/p"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans.xsd">

      <bean class="org.springframework.remoting.rmi.RmiServiceExporter"
        p:registryPort="1099"
        p:alwaysCreateRegistry="true"
        p:serviceName="userService"
        p:serviceInterface="com.ps.services.UserService"
        p:service-ref="userServiceImpl"/>
      </beans>
    ```
    Java Config:
    ```java
    //RmiServerConfig.java
    import org.springframework.remoting.rmi.RmiServiceExporter;
    //...
    @Configuration
    public class RmiServerConfig {

        @Autowired
        @Qualifier("userServiceImpl")
        UserService userService;

        @Bean
        public RmiServiceExporter userService() {
            RmiServiceExporter exporter = new RmiServiceExporter();
            exporter.setRegistryPort( 1099 );
            exporter.setAlwaysCreateRegistry(true);
            exporter.setServiceName("userService");
            exporter.setServiceInterface(UserService.class);
            exporter.setService(userService);
            return exporter;
        }
    }
    ```
2. Create a server application that will contain the remote exported bean and the application beans that need to be
accessed remotely
    * XML Config
        ```java
        import org.springframework.context.support.ClassPathXmlApplicationContext;
        public class RmiExporterBootstrap {

            public static void main(String args) throws Exception {
                ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext(
                    "spring/rmi-server-config.xml",
                    "spring/app-config.xml");
                System.out.println("RMI server started.");
                System.in.read();
                ctx.close();
            }
        }
        ```
    * app-config.xml contains all the service and repository beans needed on the remote server
        ```xml
        <beans ...">
            <!-- import service configurations -->
            <bean class="com.ps.config.ServiceConfig"/>
            <context:annotation-config/>
        </beans>
        ```
    * Java Config: (uses AnnotationConfigApplicationContext to load RmiServerConfig.class and ServiceConfig.class)
        ```java
        import com.ps.config.ServiceConfig;
        import org.springframework.context.annotation.AnnotationConfigApplicationContext;

        public class RmiExporterBootstrap {
            public static void main(String args) throws Exception {
                AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(
                    RmiServerConfig.class, ServiceConfig.class);
                System.out.println("RMI reward network server started.");
                System.in.read();
                ctx.close();
            }
        }
        ```
      * When you run the RmiExporterBootstrapClass a java process will start that will provide an RMI service named
      *UserService* on port 1099
3. Configure a **RmiProxyFactoryBean** that will take care of creating the proxy objects needed on the client side
to access the RMI service
    * XML Config:
        ```xml
        <!-- rmi-client-config.xml -->
        <beans ...>
            <bean id="userService" class="org.springframework.remoting.rmi.RmiProxyFactoryBean"
              p:serviceInterface="com.ps.services.UserService"
              p:serviceUrl="rmi://localhost:1099/userService"/>
        </beans>
        ```
    * Java Config:
        ```java
        //RmiClientConfig.java
        import org.springframework.remoting.rmi.RmiProxyFactoryBean;
        @Configuration
        public class RmiClientConfig {

            @Bean
            public UserService userService() {
                RmiProxyFactoryBean factoryBean = new RmiProxyFactoryBean();
                factoryBean.setServiceInterface(UserService.class);
                factoryBean.setServiceUrl( "rmi://localhost:1099/userService" );
                //must be explicitly called
                factoryBean.afterPropertiesSet();
                return (UserService) factoryBean.getObject();
            }
        }
        ```
      * afterPropertiesSet() must be called explicitly so that the factory bean is initialized before creating
      the proxy bean
4. Create a client application that uses the client factory bean. Easiest way to do this is to create a test class
and inject the client proxy bean created by RmiProxyFactoryBean
    ```java
    //RmiTests.java
    @ContextConfiguration( locations = {"classpath:spring/rmi-client-config.xml"} )
    @RunWith(SpringJUnit4ClassRunner.class)
    public class RmiTests {

        @Autowired
        private UserService userService;

        public void setUp() {
            assertNotNull(userService);
        }

        @Test
        public void testRmiAll() {
            List<User> users = userService.findAll();
            assertEquals( 5, users.size() );
        }

        @Test
        public void testRmiJohn() {
            User user = userService.findByEmail( "john.cusack@pet.com" );
            assertNotNull(user);
        }
    }
    ```

### Spring Remoting using HTTP Invoker
* bridges the gap between RMI services and HTTP-based services (like Hessian and Burlap)
* uses a lightweight binary HTTP-based protocol
  * makes firewalls happy
* uses java serialization to serialize objects
  * your objects must be serializable
* **HttpInvokerProxyFactoryBean**
  * creates a proxy of the remote service object
  * your code thinks its working with a local version of the remote service, but all calls go through the proxy
    * this proxy does the actual remoting using a Spring specific HTTP-based protocol
* **HttpInvokerServiceExporter**
  * It’s a Spring MVC controller that receives requests from a client through DispatcherServlet and translates those
    requests into method calls on the service implementation POJO
* RMI methods will be converted to HTTP POST methods and the result of these methods will be returned
as a HTTP result.
* Downsides
  * both client and server must be spring enabled applications
  * both client and server must have the same version of the classes and same version of Java runtime
  * server side (eg. the ServiceExporter) must be run in a container, such as Tomcat
* To use Spring HTTP Invoker
  * Configure your HttpInvokerServiceExporter bean (the server)
  * Configure Your HttpInvokerProxyFactoryBean (the client)
  * Make sure your DispatcherServlet (or HttpRequestHandlerServlet) can route requests to your service

#### HttpInvokerServiceExporter (server side config)
    ```xml
    <!-- httpinvoker-server-config.xml -->
    <beans ...>
        <bean name="/httpInvoker/userService"
            class="org.springframework.remoting.httpinvoker.HttpInvokerServiceExporter"
            p:service-ref="userServiceImpl"
            p:serviceInterface="com.ps.services.UserService"/>
    </beans>
    ```
    ```java
    // HttpInvokerConfig.java
    @Configuration
    public class HttpInvokerConfig {
        @Autowired
        @Qualifier("userServiceImpl")
        UserService userService;

        @Bean(name = "/httpInvoker/userService")
        HttpInvokerServiceExporter httpInvokerServiceExporter() {
            HttpInvokerServiceExporter invokerService = new HttpInvokerServiceExporter();
            invokerService.setService( userService );
            invokerService.setServiceInterface( UserService.class );
            return invokerService;
        }
    }
    ```
    * NOTE: **the bean name "/httpInvoker/userService" is the URL where the HTTP Service is exposed**

#### HttpInvokerProxyFactoryBean (client side config)
    ```xml
    <!-- httpinvoker-client-config.xml -->
    <beans ...>
        <bean id="userService"
            class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean"
            p:serviceInterface="com.ps.services.UserService"
            p:serviceUrl="http://localhost:8080/invoker/httpInvoker/userService"/>
    </beans>
    ```
    ```java
    //HttpInvokerClientConfig.java
    @Configuration
    public class HttpInvokerClientConfig {

        @Bean
        public UserService userService() {
            HttpInvokerProxyFactoryBean factoryBean = new HttpInvokerProxyFactoryBean();
            factoryBean.setServiceInterface(UserService.class);
            factoryBean.setServiceUrl("http://localhost:8080/invoker/httpInvoker/userService");
            factoryBean.afterPropertiesSet();
            return (UserService) factoryBean.getObject();
        }
    }
    ```
    * NOTE: the service URL is not an RMI URL, but an HTTP URL made from the web application
    URL [http://hostname:port/context] concatenated with the name of the invoker service bean

#### Invoker Service Web Application configuration
  * does not **require** a full-blown DispatcherServlet configuration
  * ```org.springframework.web.context.support.HttpRequestHandlerServlet``` is suitable
    ```xml
    <servlet>
        <servlet-name>invoker</servlet-name>
        <servlet-class>
            org.springframework.web.context.support.HttpRequestHandlerServlet
        </servlet-class>
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>
                org.springframework.web.context.support.AnnotationConfigWebApplicationContext
            </param-value>
        </init-param>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>
                com.ps.remoting.config.HttpInvokerConfig
            </param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>invoker</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
    ```

### Hessian protocol
* Still in use in Spring 4
* As of Spring 4.0, this exporter requires Hessian 4.0 or above
* **binary**, **cross-platform**, **HTTP-based** remoting protocol provided by Caucho
  * proprietary serialization mechanism
  * binary messages are lightweight and more bandwidth friendly (than Burlap)
    * portable to languages other than Java
* Downsides
  * If your data model is complex, the Hessian/Burlap serialization model may not be sufficient
  * not (very) readable compared to XML (if you need to debug)


#### HessianServiceExporter configuration
  * implemented as a Spring MVC controller
  1. Must configure a DispatcherServlet (web.xml, or Java Config etc..) and deploy your application as a web app
  2. Must configure a URL handler to dispatch Hessian service URLs to the appropriate Hessian Service Bean
      * example AbstractAnnotationConfigDispatcherInitializer
        ```java
        //...
        @Override
        protected String[] getServletMappings() {
          return new String[] { "/", "*.service" };
        }
        ```
      * When configured as above, any request whose URL ends with .service will be given to DispatcherServlet, which
      will in turn hand off the request to the Controller that’s mapped to the URL
  3. Must configure your HessianServiceExporter Bean
      * **the bean name contains the URL that exposes the hessian service**
         ```java
             @Bean(name = "/hessianInvoker/user.service")
             public HessianServiceExporter hessianInvokerServiceExporter() {
                 HessianServiceExporter invokerService = new HessianServiceExporter();
                 invokerService.setService(userService);
                 invokerService.setServiceInterface(UserService.class);
                 return invokerService;
             }
         ```
         ```xml
         <!-- hessian-service-config.xml -->
           <beans ...>
             <bean name="/hessianInvoker/userService" class="org.springframework.remoting.caucho.HessianServiceExporter"
                 p:service-ref="userServiceImpl"
                 p:serviceInterface="com.ps.services.UserService"/>
           </beans>
         ```
     * note that there is no need to set ```serviceName=``` property as hessian doesn't use a service registry


#### HessianProxyFactoryBean configuration
* no different than configuring RmiProxyFactoryBean
  ```java
  @Bean
  public HessianProxyFactoryBean userService() {
    HessianProxyFactoryBean proxy = new HessianProxyFactoryBean();
    proxy.setServiceUrl("http://localhost:8080/hessianInvoker/user.service");
    proxy.setServiceInterface(UserService.class);
    return proxy;
  }
  ```
  ```xml
  <!-- hessian-client-config.xml -->
  <beans ...>
    <bean id="userServiceHessian" class="org.springframework.remoting.caucho.HessianProxyFactoryBean"
        p:serviceInterface="com.ps.services.UserService"
        p:serviceUrl="http://localhost:8080/invoker/hessianInvoker/user.service"/>
  </beans>
  ```

### Burlap protocol
* Deprecated in Spring 4
* XML-based over HTTP, alternative to hessian
  * simple message structure (doesn't require external definition language)
  * XML messages more readable than a binary format
* sacrifices performance for better interoperability
  * portable to any language that can parse XML
* **BurlapProxyFactoryBean**
* **BurlapServiceExporter**
  * Configuration is identical to Hessian config above







Spring JMS
====================================================================================================================
## Message Oriented Middleware (Overview)
* handles the producer-consumer problem
* An application produces messages and a consumer consumes them asynchronously
* Application sending the message has no knowledge of the consumer(s)
* Form of loosely coupled distributed communication
* Advantages:
  * Relaxed coupling between producer and consumer
  * Ability to integrate heterogeneous platforms
  * reduce system bottlenecks
  * increase scalability
    * messaging is asynchronous (no waiting)
  * respond more quickly to change
* Spring provides support for JMS (Java Messaging Service) - an abstraction for accessing MOM middleware
  * avoids vendor lock-in
  * increases portability
* Spring supports AMQP

## spring-messaging module


## Basic Building blocks of JMS
* connection factories
  * encapsulates a set of connection configuration parameters that has been defined by an administrator
  * create connections
* connections
  * a client's active (and open) connection to its JMS provider
  * support concurrent use
  * create sessions
* sessions
  * a single-threaded context for producing and consuming messages
  * creates messages, creates message producers, creates message consumers
* messages
* message producers (publisher)
* message consumers (subscriber)
* destinations
  * encapsulates a provider-specific address
    * eg. ```ActiveMQQueue("com.queue.user")```
  * support concurrent use

### JMS Connections and Sessions
* Connections are obtained from a **ConnectionFactory** and injected into the client
* ConnectionFactories can be looked-up via JNDI and managed by an enterprise application server or declared locally
in your app as a Bean
* Xml config for ConnectionFactory
    ```xml
    //XML standalone example
    <bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
      <property name="brokerURL" value="tcp://localhost:60006"/>
    </bean>
    ```
    ```xml
    //XML connection factory retrieved from JNDI
    <jee:jndi-lookup id="connectionFactory" jndi-name="jms/ConnectionFactory" />
    ```
* Java config for ConnectionFactory
  ```java
  // standalone example
  @Bean ConnectionFactory connectionFactory(){
    ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory();
    connectionFactory.setBrokerURL("tcp://localhost:60006");
    return connectionFactory;
  }
  ```
  ```java
  // using Java Configurations to declare a connection factory
  //with JNDI name of jms/ConnectionFactory
  @Resource(lookup = "jms/ConnectionFactory")
  private static ConnectionFactory connectionFactory;
  ```
* Once a **ConnectionFactory** is obtained, it is used to get a ```javax.jms.Connection``` and the connection is used
to start a ```javax.jms.Session```
    ```java
    Connection connection = factory.createConnection();
    connection.start();
    Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
    // process messages
    //...
    ```
* **Session**
  * Sessions created via ```connection.createSession( boolean transacted, int autoAcknowledgeMode )```
    * first parameter indicates if session should be transacted or not
    * second parameter is used when first parameter is false
      * used to determine whether the session automatically acknowledges messages when they have been received successfully
  * Represents a unit of work
      * when processing messages is done the session must be committed or rolled-back
      * ```session.commit();  //all is OK```
      * ```session.rollback();//something went wrong```
  * Session also responsible for creating messages that will be sent to a client


## JMS Messages
* JMS API defines the standard form of a **Message** which should be portable across all JMS providers
* Created by a **Session** object
* Has a standard structure composed of:
  * **header**
    * contains system level information common to all messages
      * time sent
      * destination
      * application specific information stored as Key/Value pairs
  * **body**
    * Contains the effective application data
    * JMS defines five distinct message body types (java interfaces in ```javax.jms.Message```)
      * **javax.jms.TextMessage** - java String object
        ```java
        TextMessage message = session.createTextMessage();
        message.setText("this is a text message");
        ```
      * **javax.jms.MapMessage** - K/V pairs with String keys and primitive values, without a fixed order
        ```java
        MapMessage mm = session.createMapMessage();
        mm.set("key", "value");
        ```
      * **javax.jms.BytesMessage** - stream of uninterpreted bytes
        ```java
        session.createBytesMessage("bytes array".getBytes());
        ```
      * **javax.jms.StreamMessage** - stream of Java primitive values, filled and read sequentially
        ```java
        streamMessage = session.createStreamMessage();
        //writing primitives on it
        streamMessage.writeBoolean(false);
        streamMessage.writeInt(223344);
        streamMessage.writeChar('q');
        //emptying the stream
        streamMessage.reset();
        ```
      * **javax.jms.ObjectMessage** - a serializable Java object
        ```java
        //user must be serializable
        session.createObjectMessage(user);
        ```

## JMS Destinations
* ```javax.jms.Destination``` interface used to specify target of messages produced by the client and the source
of messages consumed by the client
* Can be a **Queue** or a **Topic**
* **Queue**
  * in point-to-point domains, each message has one producer and one consumer
* **Topic**
  * in publisher/subscriber domains, each message that is published is consumed by multiple subscribers
* Destinations can be stand-alone or retrieved via JNDI
  * JNDI
    ```java
    // using Java Configurations to declare a queue
    //with JNDI name of jms/UserQueue
    @Resource(lookup = "jms/UserQueue")
    private static Queue userQueue;
    ```
    ```java
    // using Java Configurations to declare a topic
    //with JNDI name of jms/Topic
    @Resource(lookup = "jms/Topic")
    private static Topic topic;
    ```
    ```xml
    // using XML to declare a queue
    //with JNDI name of jms/Queue
    <jee:jndi-lookup id="userQueue" jndi-name="jms/UserQueue" />
    ```
  * Standalone
    ```xml
    //XML standalone queue
    <bean id="userQueue" class="org.apache.activemq.command.ActiveMQQueue">
      <constructor-arg value="queue.users"/>
    </bean>
    ```
    ```java
    //Java Configuration standalone Queue
    @Bean
    public Queue userQueue(){
      return new ActiveMQQueue("queues.users");
    }
    ```
* Session object is responsible for creating producers and consumers using destination objects. Producer objects
  send messages, and consumer objects receive them
  ```java
  MessageProducer producer = session.createProducer( userQueue );
  ObjectMessage userMessage = session.createMessage( user );
  producer.send( userMessage );
  //...
  MessageConsumer consumer = session.createConsumer( userQueue );
  Message message = consumer.receive();
  ```

## Apache ActiveMQ
* most popular and powerful open source messaging and Integrations Pattern server
* accepts non-java clients
* can be used standalone in production environments
* supports pluggable transport protocols: in-VM,TCP,SSL,NIO,UDP,multicast...
* can be used as in-memory JMS provider, using an embedded broker, avoiding overhead when unit testing JMS
* can be embedded in an application
* can be configured using ActiveMQ or Spring configuration( XML or Java )
* provides advanced messaging features such as: message groups,wildcards, virtual and composite destinations
* supports Enterprise Integration Patterns
* ...

### Configuring ActiveMQ in Spring
* ActiveMQConnectionFactory
  * have to explicitly whitelist packages via ```trustedPackages``` property that can exchange ```javax.jms.ObjectMessage```
  * or can whitelist all packages via ```trustAllPackages``` property
  ```xml
  <!-- XML configuration -->
  <beans ...>
    <bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
      <property name="brokerURL" value="tcp://localhost:61616"/>
      <property name="trustedPackages">
        <!-- List<String> argument-->
        <list>
          <value>com.ps</value>
        </list>
      </property>
      <!-- or general and unsafe -->
      <property name="trustAllPackages" value="true"/>
    </bean>
  </beans>
  ```
  ```java
  //Java Configuration
  @Configuration
  public class JmsCommonConfig {
    List<String> packagesList = ...;
    @Bean
    public ConnectionFactory nativeConnectionFactory(){
      ActiveMQConnectionFactory cf = new ActiveMQConnectionFactory();
      cf.setBrokerURL("tcp://localhost:61616");
      cf.setTrustedPackages(packagesList);
      // or or general and unsafe
      cf.setTrustAllPackages(true);
      return cf;
    }
  //...
  }
  ```
* ActiveMQ destination queues must be declared as beans of type ```org.apache.activemq.command.ActiveMQQueue```
* ActiveMQ destination topics must be beans of type ```org.apache.activemq.command.ActiveMQTopic```
* need to import the ```spring-jms``` artifact into your project
* need to import the ```org.apache.activemq:activemq-spring``` artifact into your project

## Spring JmsTemplate
* enables decoupling of the application from backend infrastructure, but app. must be coded against the API
* central class in the Spring JMS core package
* simplifies **synchronous** message sending and receiving
* transparent creation and release of resources
* reduces boilerplate code
* exposes a basic request-reply operation that makes it possible to send a message and wait for a reply on a
temporary queue (created as part of the operation)
* practical exception handling
  * converts JMS exceptions into Spring specific unchecked exceptions **JmsException**
    * DestinationResolutionException
    * ListenerExecutionFailedException
    * MessageConversionException
    * SyncedLocalTransactionFailedException
    * UncategorizedJmsException
    * ...
* supports custom message converters and destination resolvers
* provides convenience methods and callbacks
* Can receive messages directly through its ```receive(..)``` method, but that only works synchronously,
meaning it will block. Recommended to use ```DefaultMessageListenerContainer``` instead

### JmsTemplate methods
* all methods are **synchronous**
* some methods will use the configured default Destination, some will let you supply the default Destination
* some methods let you pass in a MessageCreator callback that lets you pre-configure the Message
  * some methods will let you pass in the message as an ```Object``` and use the pre-configured MessageConverter to convert it
* some *convertAndSend()* methods take a MessagePostProcessor which lets you post-process a message after it has been
converted by the pre-configured MessageConverter
* sending a message
  * ```void send( Destination destination, MessageCreator messageCreator )```
  * ```void send( MessageCreator messageCreator )```
  * ```void send( String destinationName, MessageCreator messageCreator )```
* sending a message and receiving a reply from the specified destination
  * ```Message sendAndReceive( Destination destination, MessageCreator messageCreator )```
  * ```Message sendAndReceive( MessageCreator messageCreator )```
  * ```Message sendAndReceive( String destinationName, MessageCreator messageCreator )```
* sending a message using the configured default MessageConverter
  * ```void convertAndSend( final Object message )```
  * ```void convertAndSend( Destination destination, final Object message )```
  * ```void convertAndSend( Destination destination, final Object message, MessagePostProcessor postProcessor )```
  * ```void convertAndSend( String destinationName, final Object message )```
  * ```void convertAndSend( String destinationName, Object message, MessagePostProcessor postProcessor)
* receive a message (uses configured default destination, or you can specify)
  * ```Message receive()```
  * ```Message receive( Destination destination )```
  * ```Message receive( String destination )```
* receive a message and convert it (using the default MessageConverter) returns Object
  * ```Object receiveAndConvert()```
  * ```Object receiveAndConvert( Destination destination )```
  * ```Object receiveAndConvert( String destination )```
* receive selected message by specifying a JMS Message Selector
  * ```Message receiveSelected( String messageSelector )```
  * ```Message receiveSelected( Destination destination, String messageSelector )
  * ```Message receiveSelected( String destinationName, String messageSelector )
* receive selected message and use default MessageConverter to convert it
  * ```Object receiveSelectedAndConvert( Destination destination, String messageSelector )```
  * ```Object receiveSelectedAndConvert( String messageSelector )```
  * ```Object receiveSelectedAndConvert( String destinationName, String messageSelector )```
* execute methods that perform multiple operations on a Session, they take either:
   * interface ```ProducerCallback<T>```  ```doInJms(Session session, MessageProducer producer)```
   * interface ```SessionCallback<T>``` ```doInJms(Session session)```
     * ```<T> T execute( Destination destination, ProducerCallback<T> action )```
     * ```<T> T execute( ProducerCallback<T> action )```
     * ```<T> T execute( SessionCallback<T> action )```
     * ```<T> T execute( SessionCallback<T> action, boolean startConnection )```
     * ```<T> T execute(String destinationName, ProducerCallback<T> action)```


### JmsMessagingTemplate
* introduced in Spring 4.1
* built on JmsTemplate to provide integration with Spring's messaging abstraction:
  * ```org.springframework.messaging.Message<T>```
* allows *generic* creation and sending of messages

### MessageConverter
* interface in ```org.springframework.jms.support.converter.MessageConverter```
  * used by **JmsTemplate** to convert java object to/from ```javax.jms.Message```
  ```java
  public interface MessageConverter {
      Message toMessage(Object object, Session session) throws JMSException, MessageConversionException;
      Object fromMessage(Message message) throws JMSException, MessageConversionException;
  }
  ```

#### Spring provided MessageConverters
* **SimpleMessageConverter**
  * ```org.springframework.jms.support.converter.SimpleMessageConverter```
  * implements ```MessageConverter```
  * Used as **default conversion strategy by JmsTemplate**, for ```convertAndSend``` and ```receiveAndConvert``` operations
  * converts basic types
    * text, serializable objects, maps, byte[]
* **MarshallingMessageConverter**
  * uses JAXB to convert messages to/from XML
* **MappingJackson2MessageConverter**
  * uses Jackson2 library to convert messages to/from JSON
* **MappingJacksonMessageConverter**
  * uses Jackson JSON library to convert messages to/from JSON
* If a developer needs to send a more complex object, they must implement their own ```MessageConverter```
  * Or an alternative is to delegate to an **OXM** marshaller

### Configuring JmsTemplate
* needs a ```ConnectionFactory``` and a ```Destination```(eg. queue or topic)
  * can both be injected into JmsTemplate using their Bean id's
* Xml Config:
  ```xml
  <!-- XML configuration -->
  <beans ...>
    <bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
      <property name="brokerURL" value="tcp://localhost:61616"/>
      <property name="trustAllPackages" value="true"/>
    </bean>

    <bean id="confirmationQueue" class="org.apache.activemq.command.ActiveMQQueue">
      <constructor-arg value="com.queue.confirmation"/>
    </bean>

    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
      <property name="connectionFactory" ref="connectionFactory"/>
      <property name="defaultDestination" ref="confirmationQueue"/>
      <property name="pubSubNoLocal" value="false"/>
    </bean>
  </beans>
  ```
* Java Config:
  ```java
  //Java Configuration
  import org.apache.activemq.ActiveMQConnectionFactory;
  import org.apache.activemq.command.ActiveMQQueue;
  //...
  @Bean
  public ConnectionFactory connectionFactory() {
    ActiveMQConnectionFactory cf = new ActiveMQConnectionFactory();
    cf.setBrokerURL("tcp://localhost:61616");
    cf.setTrustAllPackages(true);
    return cf;
  }

  @Bean
  public ActiveMQQueue confirmationQueue() {
    return new ActiveMQQueue("com.queue.confirmation");
  }

  @Bean
  public JmsTemplate jmsTemplate() {
      JmsTemplate jmsTemplate = new JmsTemplate();
      jmsTemplate.setConnectionFactory( connectionFactory() );
      jmsTemplate.setDefaultDestination( confirmationQueue() );
      //whether to inhibit the delivery of messages published by its own connection - default is false
      jmsTemplate.setPubSubNoLocal( false );
      return jmsTemplate;
  }
  ```

### MessageListener (interface)
* interface specified at ```javax.jms.MessageListener```
  * single method ```void onMessage(Message message)```
* message listeners listen for a message and do something with it

#### MessageListenerContainer
* Spring provides interface ```org.springframework.jms.listener.MessageListenerContainer```
* Lets you create Message Driven POJO(s)
* Spring provides two implementations:
  * **SimpleMessageListenerContainer**
    * creates a fixed number of JMS sessions and consumers at startup
    * leaves it up the JMS provider to perform listener callbacks
    * supports native transactions
    * cannot participate in externally managed transactions
* Spring default implementation is **DefaultMessageListenerContainer**
  * Manages the JMS Sessions with which the listener will be invoked
  * allows for the asynchronous receipt of messages as well as caching sessions and message consumers
  * able to participate in externally managed transactions and native transactions
  * can dynamically adapt to runtime demands:
    * can be configured to set its consumer's min/max concurrency limit, by setting **concurrency** property
      * it's a min-max String interval (e.g. "5-10"  5 is minimum, 10 is maximum )

##### Message Listener example (consumer side):
```java
import javax.jms.JMSException;
import javax.jms.Message;
import javax.jms.MessageListener;

//Listens for a message and sends a confirmation that a message was successfully received
public class UserReceiver implements MessageListener {

    private Logger logger = LoggerFactory.getLogger(UserReceiver.class);

    //used to generate unique IDs for Confirmation objects
    private static AtomicInteger id = new AtomicInteger();

    //automatically provided by Spring JMS
    @Autowired
    MessageConverter messageConverter;

    @Autowired
    ConfirmationSender confirmationSender;

    @Override
    public void onMessage(Message message) {
        try {
            User receivedUser = (User) messageConverter.fromMessage(message);
            logger.info(" >> Received user: " + receivedUser);
            confirmationSender.sendMessage(new Confirmation(id.incrementAndGet(), "User "
                + receivedUser.getEmail() + " received."));
        } catch (JMSException e) {
            logger.error("Something went wrong ...", e);
        }
    }
}
```
```xml
<!-- XML configuration for DefaultMessageListenerContainer -->
<bean id="containerListener" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="userQueue"/>
    <property name="messageListener" ref="userReceiver"/>
    <property name="concurrency" value="5-10"/>
</bean>
<bean id="userReceiver" class="com.ps.jms.UserReceiver"/>
```
```java
//Java Configuration
@Bean
public UserReceiver userReceiver(){
    return new UserReceiver();
}

@Bean
public DefaultMessageListenerContainer containerListener() {
    DefaultMessageListenerContainer listener = new DefaultMessageListenerContainer();
    listener.setConnectionFactory( connectionFactory() );
    listener.setDestination( userQueue() );
    //UserReceiver implements the MessageListener interface
    listener.setMessageListener( userReceiver() );
    listener.setConcurrency("5-10");
    return listener;
}

```
#### Producer side (sender) example and configuration:
```java
// uses JmsTemplate to send messages to the "userQueue"
@Component
public class UserSender {
  @Autowired
  JmsTemplate jmsTemplate;

  public void sendMessage(final User user) {
    //using java lambda to send a user message
    jmsTemplate.send ( (Session session) -> session.createObjectMessage( user ));
  }
}

  <!-- user queue configuration -->
  @Bean
  public ActiveMQQueue userQueue(){
    return new ActiveMQQueue("com.queue.user");
  }

    <!-- JmsTemplate configured to use "userQueue" -->
    @Bean
    public JmsTemplate jmsTemplate(){
      JmsTemplate jmsTemplate = new JmsTemplate();
      jmsTemplate.setConnectionFactory( connectionFactory() );
      jmsTemplate.setDefaultDestination( userQueue() );
      jmsTemplate.setPubSubNoLocal( false );
      return jmsTemplate;
    }
```

### CachingConnectionFactory
* JmsTemplate aggressively opens and closes resources such as connections and sessions. It assumes that they are cached
by the connectionFactory. So, instead of using a ActiveMQConnectionFactory class as a type for the connectionFactory
bean, the Spring-specific **```CachingConnectionFactory```** class should be used to wrap up the native implementation.
```java
import org.springframework.jms.connection.CachingConnectionFactory;
//...
@Configuration
public class JmsCommonConfig {
    @Bean
    public ConnectionFactory nativeConnectionFactory(){
        ActiveMQConnectionFactory cf = new ActiveMQConnectionFactory();
        cf.setBrokerURL("tcp://localhost:61616");
        cf.setTrustAllPackages(true);
        return cf;
    }

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory cf = new CachingConnectionFactory();
        cf.setTargetConnectionFactory( nativeConnectionFactory() );
        return cf;
    }
//...
}
```

## JMS improvements in Spring 4.1
* **@EnableJms**
  * enables the bean post processor that processes @JmsListener annotations and creates the message listener
  container "under the hood"
  * used at the class level on @Configuration classes
  * XML equivalent is ```<jms:annotation-driven />```
* **@JmsListener**
  * Used on methods. Marks the method as a target of a JMS Listener
  * Added in Spring 4.1
  * Methods annotated with it have flexible signatures and can have arguments that the Spring container injects
  automatically, such as ```Session```, the raw ```Message``` object, JMS ```Headers```, and more
    * Can also use annotate @JmsListener methods with:
      * @Payload, @Header, @Headers, and @SendTo
  * Two important properties:
    * **destination** - REQUIRED - the jms destination to listen to
    * **containerFactory** - bean name of the **JmsListenerContainerFactory** to use to create the message listener container
    responsible for serving this endpoint
      * If not specified, the default container factory is used, if any
      ```java
      // UserReceiver.java
      @Component
      public class UserReceiver{
        private Logger logger = LoggerFactory.getLogger(UserReceiver.class);
        private static AtomicInteger id = new AtomicInteger();

        @Autowired
        ConfirmationSender confirmationSender;

        @JmsListener(destination = "userQueue", containerFactory = "connectionFactory")
        public void receiveMessage(User receivedUser) {
          logger.info(" >> Received user: " + receivedUser);
          confirmationSender.sendMessage( new Confirmation(id.incrementAndGet(), "User "
            + receivedUser.getEmail() + " received.") );
        }
      }
      ```
  * **JmsListenableContainerFactory** - is responsible to create the listener container responsible for a
  particular endpoint
    * ```org.springframework.boot.autoconfigure.jms.DefaultJmsListenerContainerFactoryConfigurer``` - spring boot class
    used to configure DefaultMessageListenerContainer(s), message converters, automatically creating a JmsTemplate
     ```JmsListenerContainerFactory``` a bean that creates MessageListenerContainers that are then used in @JmsListener
     endpoints
      * The *containerFactory="connectionFactory"* in the above example references the "connectionFactory" Bean created below
           ```java
           @Bean
           public JmsListenerContainerFactory<?> connectionFactory(ConnectionFactory connectionFactory,
                                                                       DefaultJmsListenerContainerFactoryConfigurer configurer) {
             DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
             // This provides all of boot's default to this factory, including the message converter
             configurer.configure(factory, connectionFactory);
             // You could still override some of Boot's defaults if necessary
             return factory;
           }
           ```



Web Services
=====================================================================================================================
## Web Services Overview
* Cross-platform inter-process communication method using common standards
* able to work through firewalls
* works via messages an not objects
  * Client sends a message and a reply is returned
* Stateless
  * Each message results in a new object created to service the request
* Supports interoperability across platforms
* Good for homogeneous environments
* Expose their set of operations via:
  * WSDL (Web Services Description Language)
  * SOAP (Simple Object Access Protocol)

### Spring-WS
* not covered on exam
* Spring has ```WebTemplate``` to simplify working with web services

JMX
=====================================================================================================================
## JMX (Java Management Extensions)
* Supplies tools for managing and monitoring resources: system objects, devices (eg. printers) and service-oriented networks
* Managed resources represented by objects called **Mbeans** (Managed Bean)
* Spring JMX support provides features to integrate a Spring application into a JMX infrastructure easily and transparently

## JMX Architecture
* Structured in three layers
  * **Instrumentation Layer** - where resources are wrapped in MBeans
  * **Agent Layer** - management infrastructure consisting of:
    * *MBean Server*
    * *Agents* - provide the following services:
      * Monitoring
      * Event notification
      * Timers
  * **Management Layer** - defines how external management application can interact with the other layers in terms
  of protocols, APIs and so on.
    * uses JSR-077 - distributed services specification

## JMX Connectors

## MBean
* A managed java object that follows the design patterns set forth in the JMX spec.
* Can represent a device, application, or any resource that needs to be managed
* Expose a management interface that consists of:
  * attributes
  * operations
  * self description
* 5 types of MBeans defined in JMX spec.
  * **Standard MBeans** (aka Simple MBeans)
  * **Dynamic MBeans**
  * **Open MBeans**
  * **Model MBeans**
  * **MXBeans**
* Management meta-data can be automatically generated at runtime in two ways:
  * implementing a java interface
  * annotating the implementation

## JMX MBean interface
### Plain JMX
* By convention, **an MBean interface takes the name of the Java class that implements it, with the suffix MBean added**
```java
package com.ps.jmx;

public interface JmxCounterMBean {
  int getCount(); // attribute "count"
  void add(); // operation "add
}

public class JmxCounter implements JmxCounterMBean{
  public int getCount() {
    //...
  }
  public void void add() {
    //...
  }
}
```
* Once a resource has been instrumented by MBeans, management of the resource is performed by a JMX agent
```java
MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
ObjectName name = new ObjectName("com.ps.jmx:type=UserCounter");
UserCounter mbean = new UserCounter();
mbs.registerMBean(mbean, name);
```

### Spring JMX
* makes developers life easier by providing a practical way to integrate a Spring application into a JMX infrastructure
* provides four core features:
  * automatic registration of any Spring Bean as a JMX MBean
  * flexible mechanism for controlling the management interface of your beans
  * declarative exposure of MBeans over remote JSR-160 connectors
  * simple proxying of both local and remote MBean resources
* does not couple your application to Spring or JMX classes/interfaces
* JMX Infrastructure can be configured using the ```context``` namespace or java configuration
* spring beans can be exposed as MBeans using annotations or XML

#### MBeanExporter
* Core class in Spring's JMX framework
* Takes your Spring beans and registers them with a JMX MBeanServer
  * you don't have to write the registration code
  * Spring beans must be annotated with the proper annotations, such as:
     * **@ManagedResource**
     * **@ManagedOperation**
     * **@ManagedAttribute**
* Declaring the server and exporter beans in XML:
  ```xml
   <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

       <!-- Configuration for JMX exposure in the application -->
       <context:mbean-server />
       <context:mbean-export />
   </beans>
  ```
  * ```<context:mbean-server />``` - declares a **MBeanServerFactoryBean** which obtains a reference to the
  MBeanServer from the MBeanServerFactory API
  * ```<context:mbean-export />``` - declares a **MBeanExporter** bean which allows exposing a Spring managed bean to a
  MBeanServer without the need to define any specific JMX information in the bean classes
* Declaring server and exporter beans in Java Config:
  ```java
  import org.springframework.jmx.export.MBeanExporter;
  import org.springframework.jmx.support.MBeanServerFactoryBean;
  //...

  @Configuration
  public class JmxConfig(){
    // equivalent of <context:mbean-server />
    @Bean
    MBeanServerFactoryBean mbeanServer(){
      return new MBeanServerFactoryBean();
    }

    // equivalent of <context:mbean-export />
    @Bean
    MBeanExporter exporter(){
      MBeanExporter exporter = new MBeanExporter();
      exporter.setAutodetect(true);
      <!-- managed beans passed to exporter in a Map -->
      exporter.setBeans(map);
      return exporter;
    }
  }
  ```
* **@ManagedResource**
  * Class-level annotation that indicates instances of a class should be registered with a JMX server
  * *objectName* attribute contains name of resulting MBean
    * if left empty then it is derived from the fully qualified class name
    * ```@ManagedResource(description = "Sample JMX managed resource", objectName="bean:name=jmxCounter")```
* **@ManagedOperation**
  * method level annotation that marks the method as a JMX operation
* **@ManagedAttribute**
  * method level annotation
  * marks a getter/setter as one half of a jmx attribute
* Declaring a ManagedResource
```java
import org.springframework.jmx.export.annotation.ManagedAttribute;
import org.springframework.jmx.export.annotation.ManagedOperation;
import org.springframework.jmx.export.annotation.ManagedResource;
import javax.annotation.ManagedBean;

@ManagedResource(description = "sample JMX managed resource", objectName="bean:name=jmxCounter")
public class JmxCounterImpl implements JmxCounter {
    private int counter = 0;

    @ManagedOperation(description = "Increment the counter")
    @Override
    public int add() {
        return ++counter;
    }

    @ManagedAttribute(description = "The counter")
    @Override
    public int getCount() {
        return counter;
    }
}
```
The MBeanExporter must then be configured to receive the managed resources as arguments in a Map:
```java
@Configuration
public class JmxConfig {
  @Bean
  MBeanExporter exporter(){
    MBeanExporter exporter = new MBeanExporter();
    Map<String, Object> map = new HashMap<>();
    map.put("bean:name=jmxCounter1", jmxCounter());
    exporter.setBeans(map);
    return exporter;
  }
  //...
}
```
* OR... MBeanExporter can be configured to detect all JMX beans in the BeanFactory that this exporter runs in:
  * set the ```autodetect``` property to true
    *  ```exporter.setAutodetect(true);```
  ```java
  @Configuration
  public class JmxConfig {

      @Bean
      MBeanExporter exporter(){
        MBeanExporter exporter = new MBeanExporter();
        exporter.setAutodetect(true);
        return exporter;
      }
      //...
  }
  ```




* **@EnableMBeanExport**
  * used on @Configuration class (typically used with @SpringBootApplication)
  * enables automatic exporting of all standard MBeans from the Spring Context
  * enables automatic exporting of all **@ManagedResource** beans
  * no need to configure an MBeanExporter bean explicitly


