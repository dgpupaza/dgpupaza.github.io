---
layout: post
title:  "Quarkus - Use REST Data with Panache to implement CRUD applications"
date:   2020-06-04 16:38:14 +0300
categories: quarkus
tags: [Quarkus, Panache, CRUD, REST]
---

While Panache simplifies the interaction with SQL and Non-SQL (MongoDB) databases, `rest-data-panache` extension introduced in [Quarkus](https://quarkus.io/) 1.5.0 reduces at minimum the code a developer has to write in order to expose CRUD operations as REST endpoints.

To be able to implement the basic functionalities we have to:

### 1. Create a Maven project

Create a new Maven project using the following command:

```sh
mvn io.quarkus:quarkus-maven-plugin:1.5.0.Final:create \
    -DprojectGroupId=ro.appvalue.quarkus \
    -DprojectArtifactId=quarkus-rest-panache \
    -Dextensions="quarkus-hibernate-orm-rest-data-panache, resteasy-jsonb, quarkus-jdbc-h2"
```

Using the specified extensions the following dependencies were added in `pom.xml`:

```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-resteasy</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-resteasy-jsonb</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-hibernate-orm-rest-data-panache</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-jdbc-h2</artifactId>
</dependency>
```

### 2. Implement an entity bean

We will consider the `Member` entity that maps to `members` table into the database.

```java
@Table
@Entity(name = "members")
public class Member extends PanacheEntity {
    
    public String name;
    public String email;
    public String phone;

}
```

### 3. Configure the datasource

For this demo, the H2 database is used. The corresponding datasource is defined in `application.properties` configuration file.

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url =jdbc:h2:mem:members_demo
quarkus.datasource.username = sa
quarkus.datasource.password =
quarkus.hibernate-orm.dialect = org.hibernate.dialect.H2Dialect
quarkus.hibernate-orm.sql-load-script = import-h2.sql
quarkus.hibernate-orm.database.generation = drop-and-create
```
The definition includes the `import-h2.sql` script and is used for the database initialization.

```sql
insert into members(id, name, email, phone) values(nextval('hibernate_sequence'), 'Timmy Stafford', 'Timmy.Stafford@demo.io', '11111111');
insert into members(id, name, email, phone) values(nextval('hibernate_sequence'), 'Anna Brennan', 'Anna.Brennan@demo.io', '11111112');
insert into members(id, name, email, phone) values(nextval('hibernate_sequence'), 'Alexis Silva', 'Alexis.Silva@demo.io', '11111113');
insert into members(id, name, email, phone) values(nextval('hibernate_sequence'), 'Kathrine Stuart', 'Kathrine.Stuart@demo.io', '11111114');
```

### 4. Define the REST endpoints

In order to model the interaction with the database, we can use the _active record_ or _repository_ pattern. According to the chosen solution, the defined interface will extend `PanacheEntityResource` or `PanacheRepositoryResource`, and it is going to be used by REST Data with Panache to generate the JAX-RS resource. In the default implementation, the generated resource will include endpoints for CRUD operations, but it can be easily customized.

```java
public interface MemberResource extends PanacheEntityResource<Member, Long> {
}
```

Based on the defined interface(s) REST Data with Panache automatically generates JAX-RS resources - `MemberResourceImpl` in this case, which includes the following public methods:

| CRUD Operation | Method | Path | HTTP Method | HTTP Status |
| -------------- | ----------- | ----------- | ----------- |
| Create         | `Response add(Long id)` | `/member/` | `POST` | `201`|
| Read (all elements) | `List<Member> list()` | `/member/` | `GET` | `200` |
| Read (one element) | `Member get(Long id)` | `/member/<ID>` | `GET` | `200` or `404` |
| Update         | `Response update(Long id, Member member)` | `/member/<ID>` | `PUT` | `204` or `201` |
| Delete         | `void delete(Long id)` | `/member/<ID>` | `DELETE` | `204` or `404` |

> **[Note]** JAX-RS annotations where intentionally omitted.

For _Read one element_ and _Delete_ operations, if the specified element having the `<ID>` is not found the `404` HTTP status is returned. In case of _Update_ operation if `<ID>` does not exist a create operation is performed instead of update and the `201` HTTP status is returned.

The default path is generated from the defined interface by converting its name to lowercase and removing the `resource` or `controller` suffix. Due to this reason, when the interface is defined, the following rule can be used for the name - `Entity[Resource|Controller]` - in our case `MemberResource` and `MemberController` are valid options.

> **[NOTE]** Case the interface's name does not follow the rule, the words that compose the name are separated by a dash and converted to lowercase - example: for `MbrDemo` the generated path is `mbr-demo`.

## Customizing endpoints

REST Data with Panache offers mechanisms (annotations) that can be used to customize the generation of the class that implements the defined interface, and implicit the exposed endpoints:

`@ResourceProperties` - Applicable at the interface level
* `path` - used to redefine the default generated path (default value is an empty string)
* `hal` - when set to `true` generates additional methods used to expose endpoints that can return `application/hal+json` (default value is `false`) 

`@MethodProperties` - Used at the method level
* `exposed` - if set to `false` the endpoint associated with the annotated method is not defined (exposed). The method is implemented but throws `RuntimeException("'<METHOD>' method is not exposed")`.
* `path` - when defined, the specified value appends to the basic path (default value is an empty string) 

### Redefine the generated path

As we've seen, the auto-generated path (based on the interface's name) is `/member`. In order to follow REST good practices `@ResourceProperties` annotation can be used in order to define the new base path as being `/members`.

```java
@ResourceProperties(path = "/members")
public interface MemberResource extends PanacheEntityResource<Member, Long> {
}
```

If in the default configuration base URI for the endpoints was `http://127.0.0.1:8080/member`, now the base URI is `http://127.0.0.1:8080/members`.

### Override the default implementation for _Read (all elements)_ endpoint

The read all elements endpoint will be redefined in order to return the members ordered by _name_. Thus the default implementation has to be marked as disabled and then a new one has to be defined.

#### 1. Disable the default endpoint

In order to disable the default implementation for the endpoint, the associated method will be defined in the `MemberResource` interface and annotated with `@MethodProperties`.

```java
@ResourceProperties(path = "/members")
public interface MemberResource extends PanacheEntityResource<Member, Long> {

    @MethodProperties(exposed = false)
    public List<Member> list();

}
```

#### 2. Define the new implementation

```java
@Path("/members")
public class CustomMemberResource {
    
    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public List<Member> list() {
        return Member.listAll(Sort.ascending("name"));
    }
}
```

> **[Note]** Read (all elements) endpoint is now defined in `CustomMemberResource` class, while for the other endpoints the default implementation is used.

The source code is available on [GitHub](https://github.com/dgpupaza/quarkus-rest-panache). 

## Testing the application 

The application exposes the functionalities (CRUD operations in this case) as REST endpoints, so there are many possibilities to test them - web browsers, command line, etc. The test process is presented from developer perspective and the scope was to keep it as simple as possible.

### 1. Start the Quarkus application

Start the application in development mode:

```java
cd quarkus-rest-panache
./mvnw quarkus:dev
```

### 2. Test enpoints

The [HTTPie](https://httpie.org/) command line HTTP client is used to test the endpoints. It is simple to use, powerful and can be used from IDE (terminal).

#### Get all members

```sh
http :8080/members

HTTP/1.1 200 OK
Content-Length: 335
Content-Type: application/json

[
    {
        "email": "Alexis.Silva@demo.io",
        "id": 3,
        "name": "Alexis Silva",
        "phone": "11111113"
    },
    ...
]
```

#### Create (add) a new member

```sh
http POST :8080/members name="Silvia Maddox" email=Silvia.Maddox@test.io phone=111111110

HTTP/1.1 201 Created
Content-Length: 83
Content-Type: application/json
Location: http://localhost:8080/members/5

{
    "email": "Silvia.Maddox@test.io",
    "id": 5,
    "name": "Silvia Maddox",
    "phone": "111111110"
}
```

> **[NOTE]** Due to a bug in [RESTEasy](https://github.com/quarkusio/quarkus/issues/9510), in dev mode, the Create and Update operation fails. These functionalities can be tested building the application and running it in terminal: `java -jar quarkus-rest-panache-1.0.0-runner.jar`.

### 3. Running the unit tests

The following command can be used to run the unit tests:

```sh
./mvnw verify
```



