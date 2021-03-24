# KumuluzEE Migrations sample using Liquibase

> Build a simple REST service that uses Liquibase for database migrations.

The aim of this sample is to demonstrate the usage of KumuluzEE Migrations using Liquibase. 
The tutorial guides you through the development of migrations at application startup and shows you how migrations 
can be achieved while application is already running.
Required knowledge: basic familiarity with JPA, CDI and basic concepts of REST and JSON.

## Requirements

In order to run this example you will need the following:

1. Java 8 (or newer), you can use any implementation:
    * If you have installed Java, you can check the version by typing the following in command line:
    ```bash
    java -version   
    ```
2. Maven 3.2.1 (or newer):
    * If you have installed Maven, you can check the version by typing the following in a command line:
    ```bash
    mvn -version
    ```
3. Git:
    * If you have installed Git, you can check the version by typing the following in a command line:
    ```bash
    git --version
    ```

## Prerequisites

In order to run this sample you will have to setup a local PostgreSQL database:

+ **database host:** localhost:5432
+ **database name:** customers
+ **user:** postgres
+ **password:** postgres

You can run the database inside docker:
```
docker run -d --name books-db -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=postgres -p 5432:5432 postgres:latest
```

The required tables will be created automatically upon running the sample.

## Usage

The example uses maven to build and run the microservice.

1. Build the sample using maven:
    ```bash
    cd kumuluzee-migrations-liquibase
    mvn clean package
    ```
2. Start local PostgreSQL DB:
    ```bash
    docker run -d --name postgres -e POSTGRES_DB=books -e POSTGRES_PASSWORD=postgres -p 5432:5432 postgres:latest
    ```
3. Run the sample:
    * Uber-jar:
    ```bash
    java -jar target/${project.build.finalName}.jar
    ```

    * Exploded:
    ```bash
    java -cp target/classes:target/dependency/* com.kumuluz.ee.EeApplication
    ```

The application/service can be accessed on the following URLs:
+ Book endpoints - [http://localhost:8080/v1/books](http://localhost:8080/v1/books)
+ Reset database - [http://localhost:8080/v1/migrations/reset](http://localhost:8080/v1/migrations/reset)
+ Populate database - [http://localhost:8080/v1/migrations/populate](http://localhost:8080/v1/migrations/populate)

To shut down the example simply stop the processes in the foreground.

## Tutorial

This tutorial will guide you through the steps required to use a Liquibase extension in KumuluzEE microservice.
Since JPA and CDI parts are not explained in this tutorial, it is recommended to complete the existing [KumuluzEE JPA and CDI sample](https://github.com/kumuluz/kumuluzee-samples/tree/master/jpa)
before continuing with this one.

We will follow these steps:
+ Add Maven dependencies
+ Create Liquibase changelog
+ Add Liquibase configuration
+ Implement REST service to trigger migrations in runtime
+ Build the microservice
+ Run it

### Add Maven dependencies

We will need the following dependencies in our microservice:
+ `kumuluzee-core`
+ `kumuluzee-servlet-jetty`
+ `kumuluzee-jax-rs-jersey`
+ `kumuluzee-cdi-weld`
+ `kumuluzee-jpa-eclipselink`
+ `kumuluzee-migrations-liquibase`
+ `postgresql`

Add the following Maven dependencies into `pom.xml`:
```xml
<dependency>
   <groupId>com.kumuluz.ee</groupId>
   <artifactId>kumuluzee-core</artifactId>
</dependency>
<dependency>
   <groupId>com.kumuluz.ee</groupId>
   <artifactId>kumuluzee-servlet-jetty</artifactId>
</dependency>
<dependency>
   <groupId>com.kumuluz.ee</groupId>
   <artifactId>kumuluzee-jax-rs-jersey</artifactId>
</dependency>
<dependency>
   <groupId>com.kumuluz.ee</groupId>
   <artifactId>kumuluzee-cdi-weld</artifactId>
</dependency>
<dependency>
   <groupId>com.kumuluz.ee</groupId>
   <artifactId>kumuluzee-jpa-eclipselink</artifactId>
</dependency>
<dependency>
   <groupId>com.kumuluz.ee.migrations</groupId>
   <artifactId>kumuluzee-migrations-liquibase</artifactId>
   <version>${kumuluzee-migrations.version}</version>
</dependency>

<!-- Only if using PostgreSQL-->
<dependency>
   <groupId>org.postgresql</groupId>
   <artifactId>postgresql</artifactId>
   <version>${postgresql.version}</version>
</dependency>
```

Add the `kumuluzee-maven-plugin` build plugin to package microservice as uber-jar:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.kumuluz.ee</groupId>
            <artifactId>kumuluzee-maven-plugin</artifactId>
            <version>${kumuluzee.version}</version>
            <executions>
                <execution>
                    <id>package</id>
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

or exploded:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.kumuluz.ee</groupId>
            <artifactId>kumuluzee-maven-plugin</artifactId>
            <version>${kumuluzee.version}</version>
            <executions>
                <execution>
                    <id>package</id>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Create Liquibase changelog

This sample already contains a simple `Book` entity for which we will create a Liquibase changelog. 
Changelog will contain two changeSets, one for updating database table and the other one for populating it.

Changelog file will be named `books-changelog.xml` and will be placed into `resources/db` directory.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.3.xsd">

    <changeSet author="KumuluzEE" id="create_table_book" context="init">
        <createTable tableName="book">
            <column name="id" type="varchar(128)"/>
            <column name="title" type="varchar(64)"/>
            <column name="author" type="varchar(64)"/>
        </createTable>
    </changeSet>

    <changeSet author="KumuluzEE" id="populate_table_book" context="populate">
        <insert tableName="book">
            <column name="id">2465c7c0-4e43-4dd9-8257-0542d4661b94</column>
            <column name="title">KumuluzEE in action</column>
            <column name="author">KumuluzEE</column>
        </insert>
        <insert tableName="book">
            <column name="id">452aa339-6481-49d4-9024-5796fa6ac633</column>
            <column name="title">KumuluzEE migrations</column>
            <column name="author">KumuluzEE</column>
        </insert>
        <insert tableName="book">
            <column name="id">9c3bb6ce-3906-4a37-b807-229e6687346d</column>
            <column name="title">KumuluzEE tips and tricks</column>
            <column name="author">KumuluzEE</column>
        </insert>
        <insert tableName="book">
            <column name="id">f7c5deb1-1602-41a5-b75d-d23d80f547fc</column>
            <column name="title">KumuluzEE best practices</column>
            <column name="author">KumuluzEE</column>
        </insert>
    </changeSet>

</databaseChangeLog>
```

### Add Liquibase configuration

In order to trigger Liquibase migration at application startup, Liquibase configuration needs to be placed in KumuluzEE
config file. Configuration contains data source JNDI name, Liquibase changelog file location, actions to be done on 
startup, Liquibase contexts and Liquibase labels.

Add the following configuration:
```yaml
kumuluzee:
  # Data source configurations
  datasources:
    - jndi-name: jdbc/BooksDS
      connection-url: jdbc:postgresql://localhost:5432/postgres
      username: postgres
      password: postgres
      pool:
        max-size: 20
  # Liquibase configurations
  migrations:
    enabled: true
    liquibase:
      changelogs:
        - jndi-name: jdbc/BooksDS
          file: db/books-changelog.xml
          contexts: "init"
          startup:
            drop-all: true
            update: true
```

### Implement REST service

In order to trigger Liquibase migrations in runtime, we need to inject `LiquibaseContainer` object. Liquibase container 
holds appropriate Liquibase object that is created based on provided `jndiName` in `@LiquibaseChangelog` annotation.

*Note: If only one Liquibase configuration is provided in KumuluzEE config file, 
`jndiName` parameter or whole `@LiquibaseChangelog` annotation can be omitted.* 

Sample service:
```java
@RequestScoped
public class LiquibaseService {

    private static final Logger LOG = Logger.getLogger(LiquibaseService.class.getName());

    @Inject
    @LiquibaseChangelog(jndiName = "jdbc/BooksDS")
    private LiquibaseContainer liquibaseContainer;

    public void reset() {

        Liquibase liquibase = liquibaseContainer.createLiquibase();

        // Retrieves contexts and labels from Liquibase configuration in KumuluzEE config file
        Contexts contexts = liquibaseContainer.getContexts();
        LabelExpression labels = liquibaseContainer.getLabels();

        try {
            liquibase.dropAll();
            liquibase.update(contexts, labels);
            liquibase.validate();

        } catch (Exception e) {
            LOG.info("Error while resetting database: " + e.getMessage());
        }
    }

    public void populate() {

        Liquibase liquibase = liquibaseContainer.createLiquibase();

        try {
            liquibase.update("populate");
        } catch (Exception e) {
            LOG.info("Error while populating database: " + e.getMessage());
        }
    }
}
```

Sample resource:
```java
@Path("migrations")
@RequestScoped
public class LiquibaseResource {

    @Inject
    private LiquibaseService liquibaseService;

    @GET
    @Path("reset")
    public Response reset() {
        liquibaseService.reset();
        return Response.noContent().build();
    }

    @GET
    @Path("populate")
    public Response populate1() {
        liquibaseService.populate();
        return Response.noContent().build();
    }
}
```

### Build the microservice and run it

To build the microservice and run the example, use the commands as described in previous sections.
