# The GraphQL Manifold

[![graphql](http://manifold.systems/images/graphql_slide_1.png)](http://manifold.systems/images/graphql.mp4)

Use the GraphQL Manifold for productive _Schema-First_ [GraphQL](https://graphql.org/) development in any Java project.
**Type-safely** build and execute queries and mutations without introducing a code generation step in your build
process. Access GraphQL types defined in standard `.graphql` schemas directly in your Java code. Build queries 
using native `.graphql` query files and immediately access changes as you make them from Java code -- without 
recompiling!  Your code is always in sync with GraphQL definitions.

> Clone the [sample GraphQL application](https://github.com/manifold-systems/manifold-sample-graphql-app) to quickly
begin experimenting with GraphQL using Manifold.

## Table of Contents
* [GraphQL Files](#graphql-files)
* [Fluent API](#fluent-api)
* [Creating Types & Queries](#building-types--queries)
* [Execute Queries](#execute-queries)
* [Execute Mutations](#execute-mutations)
* [HTTP Request Configuration](#http-request-configuration)
* [Loading a GraphQL Object](#loading-a-graphql-object)
* [Writing GraphQL Objects](#writing-graphql-objects)
* [Copying GraphQL Objects](#copying-graphql-objects)
* [Types](#type)
* [Scalar Types](#scalar-types)
* [Embedding Queries with Fragments](#embedding-queries-with-fragments)
* [IDE Support](#ide-support)
* [Setup](#setup)
* [License](#license)
* [Versioning](#versioning)
* [Author](#author)

## GraphQL Files

The GraphQL Manifold enables your Java code to directly access types and queries defined in native GraphQL schema files.
Drop a schema file into your project and start using and building queries in your code, no code generators to engage, no
recompiling between schema changes. Supports all standard file extensions including `.graphql`, `.graphqls`, and `.gql`.    

When you add a GraphQL file to your project it becomes a Java *resource*.  A resource, like a class, is rooted in a resource or
source directory, depending on your build configuration.  Maven and Gradle projects, for example, typically define a
`src/main/resources` root for all the resource files in a given module.

The path from the resource root determines the fully qualified name of the types derived from the GraphQL files.  For
example, GraphQL type definitions from file `src/main/resources/com/example/Movies.graphql` are accessible from Java
interface `com.example.Movies` where `com.example` is the package and `Movies` is the top-level interface name.  Type
definitions are inner classes defined inside the `Movies` interface.

>You can provide any number of GraphQL resource files; all files form a collective GraphQL type domain.  This means you
can organize schema types in separate files and also separate queries and mutations from schema definitions.
 
### Schema File Sample: `com/example/Movies.graphql`
```graphql
type Query {
    movies(genre: Genre!, title: String, releaseDate: Date) : [Movie!]
    reviews(genre: Genre) : [Review!]
}

type Movie {
    id: ID!
    title: String!
    genre: [Genre!]!
    releaseDate: Date!
}

type Mutation {
    createReview(movie: ID!, review: ReviewInput) : Review
}

input MovieInput {
    title: String!
    genre: Genre!
    releaseDate: Date
}

input ReviewInput {
    stars: Int!
    comment: String
}

type Review {
    id: ID!
    movie: Movie!
    stars: Int!
    comment: String
}

enum Genre {
    Action, Comedy, Drama, Fantasy, Horror, Romance, SciFi, Western
}

scalar Date
```  

### Query File Sample: `com/example/MovieQueries.graphql`
```graphql
query MovieQuery($genre: Genre!, $title: String, $releaseDate: Date) {
    movies(genre: $genre, title: $title, releaseDate: $releaseDate) {
        id
        title
        genre
        releaseDate
    }
}

query ReviewQuery($genre: Genre) {
    reviews(genre: $genre) {
        id
        stars
        comment
        movie {
            id
            title
        }
    }
}

mutation ReviewMutation($movie: ID!, $review: ReviewInput!) {
    createReview(movie: $movie, review: $review) {
        id
        stars
        comment
    }
}

extend type Query {
    reviewsByStars(stars: Int) : [Review!]!
}
```

## Fluent API

GraphQL is a language-neutral, type-safe API.  The GraphQL Manifold provides a concise, fluent mapping of the API to
Java.  For example, the `MovieQuery` type is a Java interface and provides type-safe methods to:
* **create** a `MovieQuery`
* **build** a `MovieQuery`
* **modify** properties of a `MovieQuery`  
* **load** a `MovieQuery` from a string, a file, or a URL
* **execute** a `MovieQuery` with *type-safe* response 
* **write** a `MovieQuery` as formatted JSON, YAML, or XML
* **copy** a `MovieQuery`
* **cast** to `MovieQuery` from any structurally compatible type including `Map`s, all *without proxies*

## Building Types & Queries
You create an instance of a GraphQL type using either the `create()` method or the `builder()` method.

The `create()` method defines parameters matching the `non-null` parameters declared in the query schema; if no non-null
parameters exist, `create()` has an empty parameter list.

For example, the `MovieQuery.create()` method declares one parameter corresponding with the non-null `Genre` parameter:
```java
static MovieQuery create(@NotNull Genre genre) {...}
```
You can use this to create a new `MovieQuery` with a `Genre` then modify it using _setter_ methods to change optional
properties:
```java
import com.example.MovieQueries.*;
import java.time.LocalDate;
import static com.example.Movies.Genre.Action;
...
MovieQuery query = MovieQuery.create(Action);
query.setTitle("Le Mans");
query.setReleaseDate(LocalDate.of(1971, 6, 3));
```

Alternatively, you can use `builder()` to fluently build a new instance:
```java
MovieQuery query = MovieQuery.builder(Action)
  .withTitle("Le Mans")
  .withReleaseDate(LocalDate.of(1971, 6, 3))
  .build();
```

You can initialize several properties in a chain of `with` calls in the builder. This saves a bit of typing with
heavier APIs.  After it is fully configured call the `build()` method to construct the type.

## Execute Queries
You can execute queries and mutations using a concise, fluent API.  Simply provide the endpoint as a URL, and get or
post your request. Query results are type-safe and constrained to the properties defined in the GraphQL query.

```java
import com.example.MovieQueries.*;
import java.time.LocalDate;
import static com.example.Movies.Genre.Action;
...
private static String ENDPOINT = "http://com.example/graphql";
...
var query = MovieQuery.builder(Action).build();
var result = query.request(ENDPOINT).post();
var actionMovies = result.getMovies();
for (var movie : actionMovies) {
  out.println(
    "Title: " + movie.getTitle() + "\n" +
    "Genre: " + movie.getGenre() + "\n" +
    "Year: " + movie.getReleaseDate().getYear() + "\n");
}
```

## Execute Mutations
You execute a mutation exactly as you would a query using the same API.  Note this example creates a `ReviewInput`
instance also using the same API.  

```java
// Find the movie to review ("Le Mans")
var movie = MovieQuery.builder(Action).withTitle("Le Mans").build()
  .request(ENDPOINT).post().getMovies().first();
// Submit a review for the movie
var review = ReviewInput.builder(5).withComment("Topnotch racing film.").build();
var mutation = ReviewMutation.builder(movie.getId(), review).build();
var createdReview = mutation.request(ENDPOINT).post().getCreateReview();
out.println(
  "Review for: " + movie.getTitle() + "\n" +
  "Stars: " + createdReview.getStars() + "\n" +
  "Comment: " + createdReview.getComment() + "\n"
);
```

## HTTP Request Configuration

You can configure the HTTP request to your needs.  For instance, you can use a variety of authorization options, set
header values, specify a timeout, etc.

```java
// Specify an authorization token
query.request(ENDPOINT).withAuthorization(...)

// Supply header values
query.request(ENDPOINT).withHeader(...)

// Set a timeout
query.request(ENDPOINT).withTimeout(...)
```

## Loading a GraphQL Object
In addition to creating an object from scratch with `create()` and `build()` you can also load an instance from 
a variety of existing sources using `load()`.

You can load a `MovieQuery` instance from a variety of formats including JSON, XML, and YAML:
```java
MovieQuery query = MovieQuery.load().fromYaml(
  "genre: Action\n" +
  "title: Le Mans\n" +
  "releaseDate: 1971-06-03");
```

Load from a file:
```java
User user = User.load().fromJsonFile("/path/to/MyMovieQuery.json");
```

## Writing GraphQL Objects
An instance of a GraphQL object can be written as formatted text with `write()`:
* `toJson()` - produces a JSON formatted String
* `toYaml()` - produces a YAML formatted String
* `toXml()` - produces an XML formatted String

The following example produces a JSON formatted string:
```java
MovieQuery query = MovieQuery.builder(Action)
  .withTitle("Le Mans")
  .withReleaseDate(LocalDate.of(1972, 6, 3))
  .build();

String json = query.write().toJson();
System.out.println(json);
```
Output:
```json
{
  "genre": "Action",
  "title": "Le Mans",
  "releaseDate": "1971-06-03"
}
```

## Copying GraphQL Objects
Use the `copy()` method to make a deep copy of any GraphQL object:
```java
MovieQuery query = MovieQuery.create(...);
...
MovieQuery copy = query.copy();
```
Alternatively, you can use the `copier()` static method for a richer set of features:
```java
MovieQuery copy = MovieQuery.copier(query).withGenre(Drama).copy();
```
`copier()` is a lot like `builder()` but lets you start with an already built object you can modify.


## Types

GraphQL provides several useful type abstractions these include:
* `schema`
* `type`
* `input`
* `interface`
* `enum`
* `union`
* `scalar`
* `fragment`
* `query`
* `mutation`
* `subscription`
* `extend`

The GraphQL manifold supports all type abstractions except the subscription type, which will be supported in a later
release.

### `schema`
`schema` is a simple type that lets you specify the root query type, the root mutation type, and the root subscription
type.  Without a schema type, the default root type names are `Query`, `Mutation`, and `Subscription`.

### `type`
`type` is the GraphQL foundational abstraction. The manifold API reflects a `type` as a structural _interface_. 
As the basis of the GraphQL manifold API, interfaces hide implementation detail that may otherwise complicate
the evolution of the API e.g., as new features are added to the GraphQL specification.

### `input`
An `input` is basically a `type` intended for use as a mutation constraint. It is identical to `type` in terms of
representation in the manifold API.  

### `interface`
`interface` abstractions are structural interfaces in the manifold API.  They are structured just like `type`
abstractions, but do not have creation methods.

### `enum`
The `enum` abstraction maps directly to a Java enum in the manifold API.

### `union`
Since the JVM does not provide a union type the manifold API approximates it as an interface extending the least
upper bound (LUB) `interface` abstraction of the union component types, or extends nothing if no LUB `interface` exists.
Its declared properties consist of the intersection of all the properties of the union component types.  As such a
property outside the intersection must be accessed by casting a union to a union component type declaring the
property.  Note a GraphQL query can provide a discriminator in terms of the `__typename` property to facilitate
conditional access to union properties.

### `scalar`
The manifold API fully supports GraphQL scalars and also provides a host of non-standard but commonly used types. See
_Scalar Types_ below.
 
### `fragment`
Not to be confused with [Manifold Fragments](#embedding-queries-with-fragments), a GraphQL fragment is generally a query
you can directly reference inside other queries so you don't have to copy and paste the same set of fields. Instead you
simply reference the name of the fragment. This not only helps reduce the size of queries, but also prevents copy/paste
errors and makes your queries more readable.
 
### `query`
Similar to the `type` abstraction, the manifold API exposes a `query` as a structural interface. Non-null query
parameters translate to parameters in the `create` and `builder` methods, and the nullable parameters are _getter_/_setter_
methods and _with_ methods in the builder.  Additionally the _structural_ interfaces allow the query implementation to be free
of POJOs, marshalling, and other mapping code present in conventional API tooling.  As such a query structural interface
*directly* overlays a raw GraphQL query response; there is absolutely *zero* processing of query results after a query
HTTP request.  The only processing involved happens when a scalar value must be coerced to a type-safe value; this
happens lazily on a per call-site basis.

### `mutation`
The manifold API treatment of mutations is identical to queries. See `query` above.
 
### `subscription`
_not implemented_

### `extend`
You can add properties, interfaces, and annotations to existing types using the `extend` construct.  The manifold API
fully supports all type extensions.


## Scalar Types

GraphQL specifies several standard scalar types, in addition to these Manifold provides several other non-standard, but
commonly used types.  These include:

| Name             | Persists&nbsp;As  | Java Type                                     |
|------------------|--------------|-----------------------------------------------|
| **Byte**         | _byte_       | `byte` or `java.lang.Byte` if nullable        |
| **Char**         | _char_       | `char` or `java.lang.Character` if nullable   |
| **Character**    | _char_       | `char` or `java.lang.Character` if nullable   |
| **Int**          | _integer_    | `int` or `java.lang.Integer` if nullable      |
| **Integer**      | _integer_    | `int` or `java.lang.Integer` if nullable      |
| **Long**         | _long_       | `long` or `java.lang.Long` if nullable        |
| **Float**        | _double_     | `double` or `java.lang.Double` if nullable    |
| **Double**       | _double_     | `double` or `java.lang.Double` if nullable    |
| **Boolean**      | _boolean_    | `boolean` or `java.lang.Boolean` if nullable  |
| **String**       | _string_     | `java.lang.String`                            |
| **ID**           | _string_     | `java.lang.String`                            |
| **Date**         | _string_     | `java.time.LocalDate`                         |
| **LocalDate**    | _string_     | `java.time.LocalDate`                         |
| **Time**         | _string_     | `java.time.LocalTime`                         |
| **LocalTime**    | _string_     | `java.time.LocalTime`                         |
| **DateTime**     | _string_     | `java.time.LocalDateTime`                     |
| **LocalDateTime**| _string_     | `java.time.LocalDateTime`                     |
| **Instant**      | _integer_    | `java.time.Instant`                           |
| **BigInteger**   | _string_     | `java.math.BigInteger`                        |
| **BigDecimal**   | _string_     | `java.math.BigDecimal`                        |
| **Binary**       | _string_     | `manifold.api.json.codegen.schema.OctetEncoding`      |
| **Octet**        | _string_     | `manifold.api.json.codegen.schema.OctetEncoding`      |
| **Base64**       | _string_     | `manifold.api.json.codegen.schema.Base64Encoding`     | 

Additionally, Manifold includes an API you can implement to provide your own custom scalar types.  Implement the 
`manifold.api.json.codegen.schema.IJsonFormatTypeResolver` interface as a 
[service provider](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html#register-service-providers).

>If you've implemented format type resolvers for JSON Schema, you can share them with your GraphQL APIs.  No need to
reinvent the wheel!

# Embedding Queries with Fragments

<small>
(Note this is a completely separate feature from GraphQL fragments and does not involve the `fragment` keyword)
</small><br>

>Note fragments are an experimental feature

You can now *embed* resource content such as GraphQL directly in Java source as a type-safe resource _**fragment**_. This
means you can embed a type-safe GraphQL query exactly where you use it in your Java code -- no need to create a separate
resource file.

A fragment can be either a *declaration* or an *expression*.  A fragment declaration is embedded in a multi-line
comment like this:
```java
/*[>MyQuery.graphql<]
query Movies($title: String, $genre: Genre, $releaseDate: Date) {
    movies(title: $title, genre: $genre, releaseDate: $releaseDate) {
        id
        title
        genre
        releaseDate
    }
}
*/

var query = MyQuery.Movies.builder().withGenre(Action).build();
out.println(query.toString());
```
>**IntelliJ users...**
>
>Get the [JS GraphQL plugin](https://plugins.jetbrains.com/plugin/8097-js-graphql) for rich editing of embedded
>GraphQL fragments, it pairs exceptionally well with the [Manifold plugin](https://plugins.jetbrains.com/plugin/10057-manifold).
 
A fragment *expression* is embedded in a String literal:
```java
var query = "[>.graphql<] query MovieQuery($genre: Genre){ movies(genre: $genre){ genre } }";
var result = query.builder().build().request("").post();
result.getMovies().forEach( e -> e.getGenre() );
```

With Java 13 [text block](https://openjdk.java.net/jeps/355) String literals you can easily author multi-line fragment
expressions like this:
```java
var query = """
  [>.graphql<]
  query Movies($genre: Genre!, $title: String, $releaseDate: Date) {
    movies(genre: $genre, title: $title, releaseDate: $releaseDate) {
      id
      title
      genre
      releaseDate
    }
  }
  """;
var result = query.create(Action).request(ENDPOINT).post();
```  
 
Read more about fragments in the [core Manifold docs](https://github.com/manifold-systems/manifold/tree/master/manifold-core-parent/manifold#embedding-with-fragments-experimental).

# IDE Support 

Manifold is best experienced using [IntelliJ IDEA](https://www.jetbrains.com/idea/download).

## Install

Get the [Manifold plugin](https://plugins.jetbrains.com/plugin/10057-manifold) for IntelliJ IDEA directly from IntelliJ
via:

<kbd>Settings</kbd> ➜ <kbd>Plugins</kbd> ➜ <kbd>Marketplace</kbd> ➜ search: `Manifold`

<p><img src="http://manifold.systems/images/ManifoldPlugin.png" alt="echo method" width="60%" height="60%"/></p>

## Sample Project

Experiment with the [Manifold GraphQL Sample Project](https://github.com/manifold-systems/manifold-sample-graphql-app)
via:

<kbd>File</kbd> ➜ <kbd>New</kbd> ➜ <kbd>Project from Version Control</kbd> ➜ <kbd>Git</kbd>

<p><img src="http://manifold.systems/images/OpenSampleProjectMenu.png" alt="echo method" width="60%" height="60%"/></p>

Enter: <kbd>*https://github.com/manifold-systems/manifold-sample-graphql-app.git</kbd>

<p><img src="http://manifold.systems/images/OpenSampleProject_graphql.png" alt="echo method" width="60%" height="60%"/></p>

Use the [plugin](https://plugins.jetbrains.com/plugin/10057-manifold) to really boost your productivity. Use code
completion to conveniently build queries and discover the schema's API.  Navigate to/from call-sites and GraphQL schema
file elements.  Make changes to your query schema files and use the changes immediately, no compilation!  Find usages of
any element in your schema files. Perform rename refactors to quickly and safely make project-wide changes.

>**Note:** Don't forget to install the [JS GraphQL](https://plugins.jetbrains.com/plugin/8097-js-graphql) plugin
for superb GraphQL file editing support in your project. It pairs well with the Manifold plugin.

# Setup

## Building this project

The `manifold-graphql` project is defined with Maven.  To build it install Maven and a Java 8 JDK and run the following
command.
```
mvn compile
```

## Using this project

The `manifold-graphql` dependency works with all build tooling, including Maven and Gradle. It fully supports Java
versions 8 - 13.

>Note you can replace the `manifold-graphql` dependency with [`manifold-all`](https://github.com/manifold-systems/manifold/tree/master/manifold-all)
as a quick way to gain access to all of Manifold's features.  But `manifold-graphql` already brings in a lot of
Manifold including [Extension Methods](http://manifold.systems/docs.html#extension-classes),
[String Templates](http://manifold.systems/docs.html#templating), and more.

## Binaries

If you are *not* using Maven or Gradle, you can download the latest binaries [here](http://manifold.systems/docs.html#download).


## Gradle

Here is a sample `build.gradle` script. Change `targetCompatibility` and `sourceCompatibility` to your desired Java
version (8 - 13), the script takes care of the rest. 
```groovy
plugins {
    id 'java'
}

group 'systems.manifold'
version '1.0-SNAPSHOT'

targetCompatibility = 11
sourceCompatibility = 11

repositories {
    jcenter()
    maven { url 'https://oss.sonatype.org/content/repositories/snapshots/' }
}

dependencies {
    compile group: 'systems.manifold', name: 'manifold-graphql', version: '2019.1.29'
    testCompile group: 'junit', name: 'junit', version: '4.12'

    // Add manifold to -processorpath for javac
    annotationProcessor group: 'systems.manifold', name: 'manifold-graphql', version: '2019.1.29'
}

if (JavaVersion.current() != JavaVersion.VERSION_1_8 &&
    sourceSets.main.allJava.files.any {it.name == "module-info.java"}) {
    tasks.withType(JavaCompile) {
        // if you DO define a module-info.java file:
        options.compilerArgs += ['-Xplugin:Manifold', '--module-path', it.classpath.asPath]
    }
} else {
    tasks.withType(JavaCompile) {
        // If you DO NOT define a module-info.java file:
        options.compilerArgs += ['-Xplugin:Manifold']
    }
}

tasks.compileJava {
    classpath += files(sourceSets.main.output.resourcesDir) //adds build/resources/main to javac's classpath
    dependsOn processResources
}
tasks.compileTestJava {
    classpath += files(sourceSets.test.output.resourcesDir) //adds build/resources/test to test javac's classpath
    dependsOn processTestResources
}
```
Use with accompanying `settings.gradle` file:
```groovy
rootProject.name = 'MyProject'
```

## Maven

### Java 8

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-graphql-app</artifactId>
    <version>0.1-SNAPSHOT</version>

    <name>My GraphQL App</name>

    <properties>
        <!-- set latest manifold version here --> 
        <manifold.version>2019.1.29</manifold.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>systems.manifold</groupId>
            <artifactId>manifold-graphql</artifactId>
            <version>${manifold.version}</version>
        </dependency>
    </dependencies>

    <!--Add the -Xplugin:Manifold argument for the javac compiler-->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.0</version>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                    <encoding>UTF-8</encoding>
                    <compilerArgs>
                        <!-- Configure manifold plugin-->
                        <arg>-Xplugin:Manifold</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Java 9 or later
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-graphql-app</artifactId>
    <version>0.1-SNAPSHOT</version>

    <name>My GraphQL App</name>

    <properties>
        <!-- set latest manifold version here --> 
        <manifold.version>2019.1.29</manifold.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>systems.manifold</groupId>
            <artifactId>manifold-graphql</artifactId>
            <version>${manifold.version}</version>
        </dependency>
    </dependencies>

    <!--Add the -Xplugin:Manifold argument for the javac compiler-->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.0</version>
                <configuration>
                    <source>11</source>
                    <target>11</target>
                    <encoding>UTF-8</encoding>
                    <compilerArgs>
                        <!-- Configure manifold plugin-->
                        <arg>-Xplugin:Manifold</arg>
                    </compilerArgs>
                    <!-- Add the processor path for the plugin (required for Java 9+) -->
                    <annotationProcessorPaths>
                        <path>
                            <groupId>systems.manifold</groupId>
                            <artifactId>manifold-graphql</artifactId>
                            <version>${manifold.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

# License

## Open Source
Open source Manifold is free and licensed under the [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0) license.  

## Commercial
Commercial licenses for this work are available. These replace the above ASL 2.0 and offer 
limited warranties, support, maintenance, and commercial server integrations.

For more information, please visit: http://manifold.systems//licenses

Contact: admin@manifold.systems

# Versioning

For the versions available, see the [tags on this repository](https://github.com/manifold-systems/manifold/tags).

# Author

* [Scott McKinney](mailto:scott@manifold.systems)
