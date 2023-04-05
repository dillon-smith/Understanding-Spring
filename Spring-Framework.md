# Introduction
Spring is one of the most widely used Java frameworks. This 2022 LinkedIn article indicates that over 60% of surveyed Java developers are using Spring Boot.

The Spring ecosystem currently consists of 21 active projects such as Spring Boot, Spring Data, Spring Batch, Spring Security, etc. All of these projects are built on top of Spring's core project: Spring Framework. Without fully understanding the core Spring Framework, utilizing Spring's other projects such as Spring Boot will prove difficult.

The Spring Framework is an Inversion of Control and Dependency Injection container. The remainder of this document provides a more comprehensive understanding of this definition.

# What is a Dependency?
Imagine you are writing a Java class that accesses a Movies table in a database. The class might look something like this:

```
public class MovieRepository {

    public Movie findById(Long id) {
        // execute SQL against the database
    }
}
```

Our ```MovieRepository``` has a ```findById``` method which executes some SQL to locate a movie by its ID.

In order to execute our query, this class ***depends on*** a connection to our database. In Java, database connections are usually represented with some type of data source. Our code would look like this:

```
public class MovieRepository {

    public Movie findById(Long id) {
        try(Connection conn = dataSource.getConnection()) {
            var select = connection.prepareStatement("select * from movies where id = ?");
        }
    }
}
```
## Resolving Dependencies
Our ```findById``` method above is using some ```dataSource``` object to obtain the connection to the database. Where does this ```dataSource``` object come from?

One way of obtaining this dataSource could be to create it ourselves:
```
public class MovieRepository {

    public Movie findById(Long id) {
        OracleDataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");

        try(Connection conn = dataSource.getConnection()) {
            var select = connection.prepareStatement("select * from movies where id = ?");
        }
    }
}
```

We have updated our ```findById``` method to create an OracleDataSource object, and configured it with our URL, user, and password.

What if later we needed to update this repository with another method of locating Movies? Your code might look something like this:
```
public class MovieRepository {

    public Movie findById(Long id) {
        OracleDataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");

        try(Connection conn = dataSource.getConnection()) {
            var select = connection.prepareStatement("select * from movies where id = ?");
        }
    }

    public Movie findByName(String name) {
        OracleDataSource dataSource = new OracleDataSource();
        dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
        dataSource.setUser("dillon");
        dataSource.setPassword("myPassword");

        try(Connection conn = dataSource.getConnection()) {
            var select = connection.prepareStatement("select * from movies where name = ?");
        }
    }

}
```
We have a few notable drawbacks by obtaining our data sources this way:
1. We are creating a new ```OracleDataSource``` for each executed query. Opening and closing data sources is an expensive operation. It makes sense for us to pay the cost of creating and closing these connections as little as necessary.
2. It is extremely unlikely that our database contains just movies. What if we also had a ```ProducerRepository```, ```DirectorRepository```, and a ```StudioRepository```? We would need to create and manage an ```OracleDataSource``` in each of these repositories.
3. Creating and managing this data source crowds the code and makes our business logic less apparent.

How could we solve these problems? One way would be to write a singleton ```Application``` class which is responsible for creating an ```OracleDataSource```.
```
public class Application {

    public static Application INSTANCE = new Application();

    private DataSource dataSource;

    public DataSource getDataSource() {
        if( dataSource == null ) {
            OracleDataSource dataSource = new OracleDataSource();
            dataSource.setURL("jdbc:oracle:thin:@localhost:8080:orcl");
            dataSource.setUser("dillon");
            dataSource.setPassword("myPassword");
            this.dataSource = dataSource;
        }
        return dataSource;
    }

}

```
And our MovieRepository would look like this:
```
public class MovieRepository {

    public Movie findById(Long id) {
        try(Connection conn = Application.INSTANCE.getDataSource().getConnection()) {
            var select = connection.prepareStatement("select * from movies where id = ?");
        }
    }

    public Movie findByName(String name) {
        try(Connection conn = Application.INSTANCE.getDataSource().getConnection()) {
            var select = connection.prepareStatement("select * from movies where name = ?");
        }
    }

}
```
- This approach solves the problems above. The ```Application``` class is a singleton which enforces that our ```DataSource``` is created only once.

- Our ```MovieRepository```, and any other repositories we have in our application, no longer create the data source.

Unfortunately we still have several drawbacks:
1. Our ```MovieRepository``` still needs to know how to resolve its ```DataSource``` dependency. In this case it has to know to obtain the datasource from the ```Application``` class.
2. This approach does not scale well. Applications frequently have lots of dependencies that need to be managed. The ```Application``` class would grow into a huge unmaintainable mess of a class.

## A Better Approach

### Inversion of Control
 Instead of actively obtaining the ```DataSource``` from the ```Application``` class, we could instead have the ```MovieRepository``` declare to the outside world that it depends on a ```DataSource``` and *remove* its ability to resolve the dependency.

 This process is called ***inversion of control*** because we are inverting which piece of code controls the resolution of ```MovieRepository```'s dependencies. One way to achieve inversion of control is to *inject* the dependency into the ```MovieRepository```. The code would look something like this:
 ```
 public class MovieRepository {

    private DataSource dataSource;

    public MovieRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Movie findById(Long id) {
        try(Connection conn = dataSource.getConnection()) {
            var select = connection.prepareStatement("select * from movies where id = ?");
        }
    }

    public Movie findByName(String name) {
        try(Connection conn = dataSource.getConnection()) {
            var select = connection.prepareStatement("select * from movies where name = ?");
        }
    }

}
```
Whenever someone wants an instance of our ```MovieRepository``` class, they **must** inject the ```DataSource``` via the constructor. The ```findBy``` methods will use the injected ```DataSource```.

This offers a few improvements:

1. ```MovieRepository``` does not need to care about how the ```DataSource``` is created or who creates it.
2. The ```MovieRepository``` can be instantiated with a variety of different database connections like an ```OracleDataSource``` or a ```MysqlDataSource```.
3. Writing unit tests for this ```MovieRepository``` just got a lot easier. We no longer have to have a full-fledged database connection, and can instead instantiate the ```MovieRepository``` with a dummy database connection.

Yet we *still* have problems. You are still manually wiring up the dependencies *somewhere*. All you accomplished was moving the complexity to a different part of the code. The perfect solution would be some type of paradigm that allows us to declare ```MovieRepository```'s dependencies, and avoid the manual injection of the dependencies.

*Dependency Injection Containers* are containers which construct and manage dependencies *and* inject these dependencies into the dependent objects.

# Dependency Injection with Spring