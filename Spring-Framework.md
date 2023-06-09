# Introduction
Spring is one of the most widely used Java frameworks. This [2019 survey by JetBrains](https://www.jetbrains.com/lp/devecosystem-2019/java/)[^1]article indicates that over 60% of surveyed Java developers are using Spring Boot. Despite this, many developers feel like Spring is 'magic', consisting of lots of annotations and unexpected behavior that leave them scratching their heads.

[^1]: https://www.jetbrains.com/lp/devecosystem-2019/java/

The Spring ecosystem currently consists of 21 active projects such as Spring Boot, Spring Data, Spring Batch, Spring Security, etc. All of these projects are built on top of Spring's core project: Spring Framework. 

Without fully understanding the core Spring Framework, attempting to utilize Spring's other projects such as Spring Boot will lead to that feeling of interacting with 'magic scrolls'.

This guide is meant to provide a more intuitive understanding of what the Spring framework is.

# What is a Dependency?
Imagine you are writing a Java class that accesses a Movies table in a database. The class might look something like this:

```
public class MovieRepository {

    public Movie findById(Long id) {
        // execute SQL against the database
    }
}
```

Our ```MovieRepository``` has a ```findById``` method which executes SQL to locate a movie by its ID.

In order to execute our query, this class ***depends on*** a connection to our database. In Java, a database connection is usually represented with an instance of some type of `DataSource` object. Our code would look like this:

```
public class MovieRepository {

    public Movie findById(Long id) {
        try(Connection conn = dataSource.getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where id = ?");
        }
    }

}
```
## Resolving Dependencies
Our ```findById``` method above is using a ```dataSource``` object to obtain the connection to the database. Where does the ```dataSource``` object come from?

One way of obtaining a dataSource could be to create it ourselves:
```
public class MovieRepository {

    public Movie findById(Long id) {
        DataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");

        try(Connection conn = dataSource.getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where id = ?");
        }
    }
}
```

We have updated our ```findById``` method to create a DataSource object, and configured it with our URL, user, and password.

What if later we needed to update this repository with another method of locating Movies? Your code might look something like this:
```
public class MovieRepository {

    public Movie findById(Long id) {
        DataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");

        try(Connection conn = dataSource.getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where id = ?");
        }
    }

    public Movie findByName(String name) {
        DataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");

        try(Connection conn = dataSource.getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where name = ?");
        }
    }

}
```
We have a few notable drawbacks by obtaining our data sources this way:
1. We are creating a new ```DataSource``` for each executed query. Opening and closing data sources is an expensive operation. It would be ideal if we could limit how often we had to pay for this operation.
2. It is extremely unlikely that our database contains just movies. What if we also had a ```ProducerRepository```, ```DirectorRepository```, and a ```StudioRepository```? We would need to create and manage an ```DataSource``` in each of these repositories.
3. Creating and managing this data source crowds the code and makes our business logic less apparent.

How could we solve these problems? One way would be to write a singleton ```DataSourceManager``` class which is responsible for creating the ```DataSource```.
```
public class DataSourceManager {

    private static DataSource dataSource = null;

    public static DataSource getDataSource() {
        if( dataSource == null ) {
            DataSource dataSource = new OracleDataSource();
            dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
            dataSource.setUser("dillon");
            dataSource.setPassword("myPassword");
            this.dataSource = dataSource;
        }
        return dataSource;
    }

}

```
And our `MovieRepository` would look like this:
```
public class MovieRepository {

    public Movie findById(Long id) {
        try(Connection conn = DataSourceManager.getDataSource().getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where id = ?");
        }
    }

    public Movie findByName(String name) {
        try(Connection conn = DataSourceManager.getDataSource().getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where name = ?");
        }
    }

}
```
- This approach solves the problems above. The ```DataSourceManager``` class is a singleton which enforces that our ```DataSource``` is created only once.

- Our ```MovieRepository```, and any other repositories we have in our application, no longer create the data source.

Unfortunately we still have several drawbacks:
1. Our ```MovieRepository``` still needs to know how to resolve its ```DataSource``` dependency. In this case it has to know to obtain the `DataSource` from the ```DataSourceManager``` class.
2. This approach does not scale well. Applications frequently have lots of dependencies that need to be managed. Our code base would end up littered with similar `Manager` classes, or even one big `Application` class that manages all the dependencies. 

## A Better Approach

### Inversion of Control
 Instead of actively obtaining the ```DataSource``` from the ```DataSourceManager``` class, we could instead have the ```MovieRepository``` declare to the outside world that it depends on a ```DataSource``` and *remove* its ability to resolve the dependency.

 This process is called ***inversion of control*** because we are inverting which piece of code controls the resolution of ```MovieRepository```'s dependencies. One way to achieve inversion of control is to *inject* the dependency into the ```MovieRepository```. The code would look something like this:
 ```
 public class MovieRepository {

    private DataSource dataSource;

    public MovieRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Movie findById(Long id) {
        try(Connection conn = dataSource.getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where id = ?");
        }
    }

    public Movie findByName(String name) {
        try(Connection conn = dataSource.getConnection()) {
            PreparedStatement select = connection.prepareStatement("select * from movies where name = ?");
        }
    }

}
```
Whenever someone wants an instance of our ```MovieRepository``` class, they **must** inject the ```DataSource``` via the constructor. The ```findBy``` methods will then use the injected ```DataSource```.

This offers a few improvements:

1. ```MovieRepository``` does not need to care about how the ```DataSource``` is created or who creates it.
2. The ```MovieRepository``` can be instantiated with a variety of different database connections like an ```OracleDataSource``` or a ```MysqlDataSource```.
3. Writing unit tests for this ```MovieRepository``` just got a lot easier. We no longer have to have a full-fledged database connection, and can instead instantiate the ```MovieRepository``` with a variety of database connections suitable for testing. We could even test our ```MovieRepository``` with a broken data source, a valid data source, a data source that does not have data, etc.

Yet we *still* have problems. We are still manually wiring up the dependencies *somewhere*. All we accomplished was moving the complexity to a different part of the code. The perfect solution would be some type of paradigm that allows us to declare ```MovieRepository```'s dependencies, and avoid the manual injection of the dependencies.

*Dependency Injection Containers* are containers which create managed objects, taking care of where and how the objects are created.

# Dependency Injection with Spring
A Dependency Injection Container (DI Container), or Inversion of Control (IoC) container, is a container that manages the classes we write, and configures their dependencies for us. At its core, the Spring Framework is an IoC container.

**Note**: DI and IoC are often used interchangably.

## Spring Application Context
Within the Spring IoC container, the element that manages our dependencies is called the *Application Context*.
```
public class MovieApplication {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(configClass);

        MovieRepository movieRepo = context.getBean(MovieRepository.class);
        Movie movie101 = movieRepo.findById(101);
        Movie titanic = movieRepo.findByName("Titanic");

        DataSource dataSource = context.getBean(DataSource.class);

    }

}
```
In the code snippet above, we constructed the Spring `ApplicationContext`.

- The `ApplicationContext` gives us a fully constructed `MovieRepository`, with its `DataSource` dependency provided by the `ApplicationContext`.
- We can also retrieve a DataSource object. This DataSource object is the *same* object as the one provided to the `MovieRepository`.
- We no longer have to manually provide dependencies, nor do our objects have to resolve their dependencies themself. All we need to do is ask the `ApplicationContext` for an instance of our object.

The `ApplicationContext` understands how to construct our objects and provide their dependencies through what the Spring framework calls a `Configuration`. Recall that in the code above we provided a `configuration` variable to the `ApplicationContext`.

### Application Configurations
What we really passed into the constructor above, is something like this:
```
@Configuration
public class DataSourceConfiguration {

    @Bean
    public DataSource dataSource() {
        DataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");
        return dataSource;
    }

    @Bean
    public MovieRepository movieRepository() {
        return new MovieRepository(dataSource());
    }

}
```
- We have a dedicated configuration class. This class is annotated with `@Configuration`
- We have a method that returns a `DataSource`. This method is annotated with `@Bean`
- We have a `@Bean` annotated method that returns a `MovieRepository`, constructing it with the `DataSource` obtained via the `dataSource` method.

This Configuration class was all we needed to run our application, and have Spring manage and resolve our object dependencies.

## Beans and The `@Bean` Annotation
A Bean is an object that is created and managed by the Spring framework, and whose dependencies are injected by Spring.

The `@Bean` annotation is an annotation applied to a method and indicates to the Spring framework that it returns a Bean that should be managed by Spring.

### Bean Scopes
You might have the following questions:
- How does Spring know how many instances of a bean to create?
- When does Spring create a bean?

Spring manages this concept through something called a *bean scope*. By default, all Spring beans are singletons. These beans are created when the Spring container is started.

Out of the box, the Spring framework supports six scopes. It is also possible to create your own scope, but that is not covered in this guide.
| Scope | Description |
| ------------- | ------------- |
| Singleton  | (Default) Scopes a sikngle bean definition to a single object instance for each Spring IoC container.  |
| Prototype  | Scopes a single bean definition to any number of object instances. |
| Request  | Scopes a single bean definition to the lifecycle of a single HTTP request. That is, each HTTP request has its own instance of a bean created off the back of a single bean definition. Only valid in the context of a web-aware Spring ApplicationContext.|
| Session  | Scopes a single bean definition to the lifecycle of an HTTP Session. Only valid in the context of a web-aware Spring ApplicationContext.  |
| Application  | Scopes a single bean definition to the lifecycle of a ServletContext. Only valid in the context of a web-aware Spring ApplicationContext.  |
| Websocket  | Scopes a single bean definition to the lifecycle of a WebSocket. Only valid in the context of a web-aware Spring ApplicationContext. |

A scope can be applied to a Bean like-so:
```
@Configuration
public class DataSourceConfiguration {

    @Bean
    @Scope("singleton")
    public DataSource dataSource() {
        DataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");
        return dataSource;
    }

    @Bean
    @Scope("prototype")
    public MovieRepository movieRepository() {
        return new MovieRepository(dataSource());
    }

}
```

Most applications consist of singleton beans.

### Component Scanning
Lets take a look at our `Configuration` class again.
```
@Configuration
public class DataSourceConfiguration {

    @Bean
    @Scope("singleton")
    public DataSource dataSource() {
        DataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");
        return dataSource;
    }

    @Bean
    @Scope("prototype")
    public MovieRepository movieRepository() {
        return new MovieRepository(dataSource());
    }

}
```
In order to provide the `MovieRepository` with a `DataSource`, our `movieRepository` method is manually invoking the `dataSource` method. Why do we have to manually do this? Isn't this a problem we were trying to solve?

This is where the `@ComponentScan` annotation comes into play. It turns out that if we added this annotation to our configuration, we can entirely remove the `movieRepository` method and let Spring figure out how to create the `MovieRepository`.
```
@Configuration
@ComponentScan
public class DataSourceConfiguration {

    @Bean
    @Scope("singleton")
    public DataSource dataSource() {
        DataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");
        return dataSource;
    }

}
```

The `@ComponentScan` annotation tells Spring to search for beans by scanning *all* packages *and* subpackages in the same package as the context configuration.

How does Spring know if something is a bean? It looks for the `@Component` in packages being searched.

We can add the `@Component` annotation to our `MovieRepository` like-so.
 ```
 @Component
 public class MovieRepository {

    private DataSource dataSource;

    public MovieRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

}
```

The `MovieRepository` class now tells Spring: 'I want to be a Spring bean managed by the Spring IoC container!' Spring has all the information it needs to configure both our `DataSource`, and our `MovieRepository`.

How does Spring know where to get the `DataSource`? In earlier versions of Spring, it would not have been. You would need to add the `@Autowired` annotation like this:
 ```
 @Component
 public class MovieRepository {

    private DataSource dataSource;

    public MovieRepository(@Autowired DataSource dataSource) {
        this.dataSource = dataSource;
    }

}
```
Newer versions of Spring are actually smart enough to understand your component's dependencies without the `@Autowired` annotation. Explicitly adding `@Autowired` can help to make your code more understandable.

### Component Stereotypes
The Spring ecosystem features numerous *stereotype* annotations that are all specialized versions of the `@Component` annotation:
| Stereotype | Description |
| ------------- | ------------- |
| `@Service`| Indicates that an annotated class is a "Service", originally defined by Domain-Driven Design (Evans, 2003) as "an operation offered as an interface that stands alone in the model, with no encapsulated state."|
| `@Repository`| Indicates that an annotated class is a "Repository", originally defined by Domain-Driven Design (Evans, 2003) as "a mechanism for encapsulating storage, retrieval, and search behavior which emulates a collection of objects".  |
| `@Controller`| Indicates that an annotated class is a "Controller" (e.g. a web controller). |
| `@RestController`| A convenience annotation that is itself annotated with `@Controller` and `@ResponseBody`.  |

If you inspect these annotations in your IDE, you will find that they are all also annotated with `@Component`.

**Note**: The `@Component` annotation is itself considered a stereotype.


### Field Injection and Setter Injection
Spring is not required to go through a constructor to inject dependencies.

#### Field Injection
```
 @Component
 public class MovieRepository {

    @Autowired
    private DataSource dataSource;

    public MovieRepository() {
        // empty constructor
        // Spring cannot provide the dependency here
    }

}
```

#### Setter Injection
```
 @Component
 public class MovieRepository {

    private DataSource dataSource;

    public MovieRepository() {
        // empty constructor
        // Spring cannot provide the dependency here
    }

    @Autowired
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }

}
```

Constructor Injection, Field Injection, and Setter Injection are all different roads to the same destination. In each scenario Spring injects the `DataSource` dependency, and our `MovieRepository` is fully constructed and managed by Spring.

Which injection method to choose? The [Spring framework's documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-setter-injection) recommends using constructor injection for all mandatory dependencies, and setter injection for optional dependencies.

# Spring Ecosystem
Some of the most common projects within the Spring ecosystem are as follows:
| Project | Description | 
| ------------- | ------------- |
|[Spring Framework](https://spring.io/projects/spring-framework)|Provides core support for dependency injection, transaction management, web apps, data access, messaging, and more.|
|[Spring Boot](https://spring.io/projects/spring-boot)|Takes an opinionated view of building Spring applications.|
|[Spring Data](https://spring.io/projects/spring-data)|Provides a consistent approach to data access.|
|[Spring Security](https://spring.io/projects/spring-security)|Provides extensible authentication and authorizatino support.|
|[Spring Kafka](https://spring.io/projects/spring-kafka)|Provides Spring abstractions for Apache Kafka|

A full list of Spring projects can be found [here](https://spring.io/projects).

# Glossary
## Dependency
An object that another object depends on for critical functionality.

## Inversion of Control (IoC)
A design pattern frequently used for decoupling the resolution of an object's dependencies from the object's behavior.

## Dependency Injection (DI)
A particular method of implementing Inversion of Control in which an object receives dependencies from an external source. 

## Spring Application Context
A container that is responsible for creating, configuring, and managing a Spring Bean. The container uses metadata like Configuration classes to understand the creation, configuration, and management of a bean.

 Also commonly referred to as:
 - Application context
 - Spring container
 - Spring IoC container
 - IoC container
 - DI container

## Stereotype Annotation
An annotation for specializations of Spring managed beans.

## Constructor Injection
A particular method of implementing DI in which an object receives dependencies from an external source through an object's constructor.

## Field Injection
A particular method of implementing DI in which an object receives dependencies when an external source directly mutates the object's fields. 

## Setter Injection
A particular method of implementing DI in which an object receives dependencies by an external source invoking the object's setter methods.

## `@Configuration`
Indicates that a class contains `@Bean` annotated methods, and that the Spring Application Context can use this class as metadata for creating, configuring, and managing beans.

## `@ComponentScan`
An annotation applied to a `@Configuration` that allows Spring to autodetect and register metadata within the Application Context for `@Component` and other sterotype annotated classes.

## `@Bean`
An annotation that indicates that a method produces a bean to be managed by the Spring Application Context.

## `@Scope`
An annotation that can be attached to a `@Component` or `@Bean` and provides the Spring Application Context instruction on how to manage the creation and lifecycle of a bean.

## `@Component`
An annotation that indicates that a class is a candidate for component scanning autodetection.

## `@Repository`
An annotation that indicates that a class is a candidate for component scanning autodetection, and indicates that an annotated class is a "Repository".

## `@Service`
An annotation that indicates that a class is a candidate for component scanning autodetection, and indicates that an annotated class is a "Service".

## `@Controller`
An annotation that indicates that a class is a candidate for component scanning autodetection, and indicates that an annotated class is a "Controller".

## `@RestController`
An annotation that indicates that a class is a candidate for component scanning autodetection, and indicates that an annotated class is a "Controller" within a REST architecture.