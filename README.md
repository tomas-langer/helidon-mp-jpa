# Helidon Quickstart MP Example

This example implements a simple Hello World REST service using MicroProfile.

## Add JPA on top of the quickstart
The JPA example will use h2 database. If you use docker, use the following
command to start an h2 database:

`docker run -d -p 9092:9082 -p 8082:8082 --name=h2 nemerosa/h2`

Once the database is running, we can add JPA related artifacts to our project.

### Dependencies
This example will use Hibernate JPA provider. 

Update the `dependencies` section in your `pom.xml`, add the following:
```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>jakarta.persistence</groupId>
    <artifactId>jakarta.persistence-api</artifactId>
    <version>2.2.2</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>javax.transaction</groupId>
    <artifactId>javax.transaction-api</artifactId>
    <version>1.2</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>io.helidon.integrations.cdi</groupId>
    <artifactId>helidon-integrations-cdi-hibernate</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.helidon.integrations.cdi</groupId>
    <artifactId>helidon-integrations-cdi-jpa</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.helidon.integrations.cdi</groupId>
    <artifactId>helidon-integrations-cdi-jta-weld</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.helidon.integrations.cdi</groupId>
    <artifactId>helidon-integrations-cdi-datasource-hikaricp</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Required resources
JPA requires a few resources to be added to your project:
`src/main/resoures/META-INF/persistence.xml` - this file defines the entity class(es) to be used and configuration
options for Hibernate (scuh as the database dialect to use - in our case H2).
```xml
<persistence version="2.2"
             xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
                                 http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd">
  <persistence-unit name="test" transaction-type="JTA">
    <jta-data-source>test</jta-data-source>
    <class>io.helidon.examples.quickstart.mp.Greeting</class>
    <properties>
      <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
      <property name="hibernate.hbm2ddl.auto" value="create-drop" />
      <property name="show_sql" value="true"/>
      <property name="hibernate.temp.use_jdbc_metadata_defaults" value="false"/>
    </properties>
  </persistence-unit>
</persistence>
```

We also need to configure the datasource - this is done in configuration.
We can use the `src/main/resources/META-INF/microprofile-config.properties`:

```properties
# there are some existing properties - just add the following:
javax.sql.DataSource.test.dataSourceClassName=org.h2.jdbcx.JdbcDataSource
javax.sql.DataSource.test.dataSource.url=jdbc:h2:mem:test;INIT=CREATE TABLE IF NOT EXISTS GREETING (FIRSTPART VARCHAR NOT NULL, SECONDPART VARCHAR NOT NULL, PRIMARY KEY (FIRSTPART))\\;MERGE INTO GREETING (FIRSTPART, SECONDPART) VALUES ('hello', 'world')
javax.sql.DataSource.test.dataSource.user=sa
javax.sql.DataSource.test.dataSource.password=
```

### Source code changes
We need an entity class that maps to the table we created in H2 (see the `dataSource.url` - it contains initial create table and insert):
Class `Greeting`:
```java
@Access(AccessType.FIELD)
@Entity(name = "Greeting")
@Table(name = "GREETING")
public class Greeting {

    @Id
    @Column(name = "FIRSTPART", insertable = true, nullable = false, updatable = false)
    private String firstPart;

    @Basic(optional = false)
    @Column(name = "SECONDPART", insertable = true, nullable = false, updatable = true)
    private String secondPart;

    // add required default constructor (JPA requirement), constructor with properties and getter and setters
    // see sources of this project
}
```

Next we want to use this entity in our JAX-RS resource. We will modify the `GreetResource` for this purpose.

Inject entity manager and transaction into the resource:
```java
@PersistenceContext
private EntityManager entityManager;

@Inject
private Transaction transaction;
```

Then we modify the method `createResponse(String)` to try to find the message in our database:
```java
private JsonObject createResponse(String who) {
    Greeting greeting = this.entityManager.find(Greeting.class, who);
    String message;
    if (null == greeting) {
        // not in database
        message = greetingProvider.getMessage();
    } else {
        message = greeting.secondPart();
    }
    String msg = String.format("%s %s!", message, who);

    return JSON.createObjectBuilder()
            .add("message", msg)
            .build();
}
```

### Add an insert method
Let's add a method that creates a new mapping in the database. 
We will (once again) update the `GreetResource` by adding a JAX-RS resource method to it:

```java
@Path("/db/{firstPart}")
@POST
@Consumes(MediaType.TEXT_PLAIN)
@Produces(MediaType.TEXT_PLAIN)
@Transactional(Transactional.TxType.REQUIRED)
public Response dbCreateMapping(@PathParam("firstPart") String firstPart, String secondPart) {
    Greeting greeting = new Greeting(firstPart, secondPart);
    this.entityManager.persist(greeting);

    return Response.created(URI.create("/greet/" + firstPart)).build();
}
```

Now we can create a new mapping:

`curl -i -X POST -H 'Content-Type:text/plain' -d 'Use' http://localhost:8080/greet/db/helidon`

And test it:

`curl -i http://localhost:8080/greet/helidon`

### Add an update method
Let's add the final method to `GreetResource`:

```java
@Path("/db/{firstPart}")
@PUT
@Consumes(MediaType.TEXT_PLAIN)
@Produces(MediaType.TEXT_PLAIN)
@Transactional(Transactional.TxType.REQUIRED)
public Response dbUpdateMapping(@PathParam("firstPart") String firstPart, String secondPart) {
    try {
        Greeting greeting = this.entityManager.getReference(Greeting.class, firstPart);
        greeting.setSecondPart(secondPart);
    } catch (EntityNotFoundException e) {
        return Response.status(404).entity("Mapping for " + firstPart + " not found").build();
    }

    return Response.ok(firstPart).build();
}
``` 

Now we can update the previous mapping:

`curl -i -X PUT -H 'Content-Type:text/plain' -d 'I am using' http://localhost:8080/greet/db/helidon`

And test that it was updated:

`curl -i http://localhost:8080/greet/helidon`

## Build and run

With JDK8+
```bash
mvn package
java -jar target/helidon-quickstart-mp.jar
```

## Exercise the application

```
curl -X GET http://localhost:8080/greet
{"message":"Hello World!"}

curl -X GET http://localhost:8080/greet/Joe
{"message":"Hello Joe!"}

curl -X PUT -H "Content-Type: application/json" -d '{"greeting" : "Hola"}' http://localhost:8080/greet/greeting

curl -X GET http://localhost:8080/greet/Jose
{"message":"Hola Jose!"}
```

## Try health and metrics

```
curl -s -X GET http://localhost:8080/health
{"outcome":"UP",...
. . .

# Prometheus Format
curl -s -X GET http://localhost:8080/metrics
# TYPE base:gc_g1_young_generation_count gauge
. . .

# JSON Format
curl -H 'Accept: application/json' -X GET http://localhost:8080/metrics
{"base":...
. . .

```

## Build the Docker Image

```
docker build -t helidon-quickstart-mp .
```

## Start the application with Docker

```
docker run --rm -p 8080:8080 helidon-quickstart-mp:latest
```

Exercise the application as described above

## Deploy the application to Kubernetes

```
kubectl cluster-info                         # Verify which cluster
kubectl get pods                             # Verify connectivity to cluster
kubectl create -f app.yaml               # Deploy application
kubectl get service helidon-quickstart-mp  # Verify deployed service
```
