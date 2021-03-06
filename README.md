# Norm 

NORM is acronym for Not an ORM.

Norm is purpose built for use with Kotlin and PostgreSQL. While most ORMs and Database access libraries try to hide SQL, Norm works on just the opposite principle. SQL is powerful and battle tested. Norm gives SQL back the control it deserves. 

Norm has two major components, a compile time code generator and a very lighteight runtime module. 

1. Norm’s code generator connects with our development database. Using JDBC, it infers just enough information from the SQL query/command to generate the Kotlin code. For each of the given SQL query it generate required code to execute the query and map the result-set to kotlin data classes. As a result, we get type safe database access layer with zero boilerplate code. 

2. Norm's Runtime provides extension methods to execute the generated code with ease. 

## Norm's Design Choices:

- Provide ability to execute aribitraily complex and performant SQL with ease
- Reduce the boilerplate code to zero, by generated all the required code in a separate source set
- Provide type safe database access, whenever database model changes, you get compilation error rather than runtime exceptions.
- No annotation soup, No bytecode magic, you can see and check-in all the generated code in git
- Differentiate between Query v/s Commands. At a high level, queries return results without modifying database state, commands modify database state but do not return anything.

> Norm is not meant to be a generic framework that tries to cover all the JVM languages or all SQL/NoSQL databases.

## How Norm works

Norm requires us to have an active database connection to precompile queries for typesafe data access. Codegen module uses this connection to generate the code.
    
#### Codegen
- Norm opens a connection to development database. 
- It reads the SQL file and analyzes query by creating a [PreparedStatement](https://docs.oracle.com/javase/7/docs/api/java/sql/PreparedStatement.html)
- If query is syntactically correct, it creates a [SqlModel](https://github.com/medly/norm/blob/master/codegen/src/main/kotlin/norm/SqlAnalyzer.kt) which is an intermidiary data model that acts as an input to CodeGenerator
- Finally, generates Kotlin files with classes of Query or Command, ParamSetter and RowMapper

#### Runtime     

- Provides generic extension functions and interfaces which can be used without code-gen. These simplify usage of [Connection](https://docs.oracle.com/javase/7/docs/api/java/sql/Connection.html) or [ResultSet](https://docs.oracle.com/javase/7/docs/api/java/sql/ResultSet.html) or [PreparedStatement](https://docs.oracle.com/javase/7/docs/api/java/sql/PreparedStatement.html) classes.
- Adds extentions on these interfaces to be able to execute query/command, map result to a list, execute batch commands and queries etc.

---

## Getting Started

### 1. Set up the schema

Start postgres server locally. Create schemas and tables needed for your application or repository.

> It is highly recommended to manage the database schema migrations (DDL statements) with a tool like Liquibase or Flyway.

Let's take an example of `person table` and create it in the default `public schema` of default `postgres database`. 

```SQL
create table persons(
    id SERIAL PRIMARY KEY,
    name VARCHAR,
    age INT, 
    occupation VARCHAR,
    address VARCHAR
);
```
> we can use the `psql -p 5432 -d postgres` to run the following query: 


All the migrations need to be run before using norm codegen to generate classes.


### 2. Write the sql queries

norm needs two directories defined in the project:
1. input directory - contains all sql files
2. output directory - would contain all files generated by norm


Write our queries and commands in input dir. Continuing our example, lets write a query that fetches all persons whose age is greater than some number.
add a sql file in this sql's source dir with path, say eg `sql/person/get-all-persons-above-given-age.sql`
        
```SQL
SELECT * FROM persons WHERE AGE > :age;
```

### 3. Generate the code 


#### Command Line Interface

Norm CLI can be used to generate Kotlin files corresponding to SQL files. 

we can provide multiple of files using  `-f some/path/a.sql -f some/path/b.sql`. This will generate Kotlin files
at `some.path.A.kt` & `some.path.A.kt`. If we want to exclude `some` from package name then we must use `-b` option 
with the base dir `-f some/path/a.sql -f some/path/b.sql -b some/`. Now the kotlin files will be generated in package
`path.A.kt` & `path.A.kt` inside the `output-dir`.

If option `--in-dir` is used, all the `*.sql` files will be used for code generation.

```terminal
$ norm-codegen --help

Usage: norm-codegen [OPTIONS] [SQLFILES]...

  Generates Kotlin Source files for given SQL files using the Postgres
  database connection

Options:
  -j, --jdbc-url TEXT        JDBC connection URL (can use env var PG_JDBC_URL)
  -u, --username TEXT        Username (can use env var PG_USERNAME)
  -p, --password TEXT        Password (can use env var PG_PASSWORD)
  -b, --base-path DIRECTORY  relative path from this dir will be used to infer
                             package name
  -f, --file FILE            [Multiple] SQL files, the file path relative to
                             base path (-b) will be used to infer package name
  -d, --in-dir DIRECTORY     Dir containing .sql files, relative path from
                             this dir will be used to infer package name
  -o, --out-dir DIRECTORY    Output dir where source should be generated
  -h, --help                 Show this message and exit

```

#### Using a custom gradle task:

We can optionally add a task in gradle which would execute norm's code generation
```gradle

configurations { norm }

dependencies {
    norm "org.postgresql:postgresql:$postgresVersion" 
    norm "com.medly.norm:runtime:$normVersion"
    norm "com.medly.norm:cli:$normVersion"
} 

task compileNorm(type: JavaExec) {
    classpath = configurations.norm  
    main = "norm.cli.NormCliKt"
    args "${rootProject.rootDir}/sql"                //input dir
    args "${rootProject.rootDir}/gen"                //output dir
    args "jdbc:postgresql://localhost/postgres"      //postgres connection string with db name
    args "postgres"                                  //db username
    args ""                                          //db password (optional for local) 
}
```     

Run `./gradlew compileNorm`. It will generate a folder within `gen` (output dir) with the same name as folder inside `sql` (input dir), `person`.
All the generated classes would be within a file named - ```GetAllPersonsAboveGivenAge``` (title case name of sql file)

#### Peek at the generated code

The content of generated file would look like:
```kotlin 
package person

import java.sql.PreparedStatement
import java.sql.ResultSet
import kotlin.Int
import kotlin.String
import norm.ParamSetter
import norm.Query
import norm.RowMapper

data class GetAllPersonsAboveGivenAgeParams(
  val age: Int?
)

class GetAllPersonsAboveGivenAgeParamSetter : ParamSetter<GetAllPersonsAboveGivenAgeParams> {
  override fun map(ps: PreparedStatement, params: GetAllPersonsAboveGivenAgeParams) {
    ps.setObject(1, params.age)
  }
}

data class GetAllPersonsAboveGivenAgeResult(
  val id: Int,
  val name: String?,
  val age: Int?,
  val occupation: String?,
  val address: String?
)

class GetAllPersonsAboveGivenAgeRowMapper : RowMapper<GetAllPersonsAboveGivenAgeResult> {
  override fun map(rs: ResultSet): GetAllPersonsAboveGivenAgeResult =
      GetAllPersonsAboveGivenAgeResult(
  id = rs.getObject("id") as kotlin.Int,
    name = rs.getObject("name") as kotlin.String?,
    age = rs.getObject("age") as kotlin.Int?,
    occupation = rs.getObject("occupation") as kotlin.String?,
    address = rs.getObject("address") as kotlin.String?)
}

class GetAllPersonsAboveGivenAgeQuery : Query<GetAllPersonsAboveGivenAgeParams,
    GetAllPersonsAboveGivenAgeResult> {
  override val sql: String = """
      |SELECT * FROM persons WHERE AGE > ?;
      |""".trimMargin()

  override val mapper: RowMapper<GetAllPersonsAboveGivenAgeResult> =
      GetAllPersonsAboveGivenAgeRowMapper()

  override val paramSetter: ParamSetter<GetAllPersonsAboveGivenAgeParams> =
      GetAllPersonsAboveGivenAgeParamSetter()
}
```            

### 4. Calling queries/commands generated by norm

Add norm runtime dependencies

```gradle

// add jitpack repo
repositories {		
    maven { url 'https://jitpack.io' }
}

dependencies {
    // add the postgres driver
    implementation "org.postgresql:postgresql:$postgresVersion" 

    // add norm runtime dependency
    implementation "com.medly.norm:runtime:$normVersion"
} 
```      

> To run any query/command, a DataSource connection of postgres is required.

Create an instance of DataSource using the postgresql driver(already added in dependency) methods
```kotlin
  val dataSource = PGSimpleDataSource().also {
          it.setUrl("jdbc:postgresql://localhost/postgres")
          it.user = "postgres"
          it.password = ""
  }
``` 

Finally we can execute the query:

```kotlin
  val result = dataSource.connection.use { connection -> 
      GetAllPersonsAboveGivenAgeQuery().query(connection, GetAllPersonsAboveGivenAgeParams(20))
  }
  result.forEach { println(it.toString()) }
```  

Have fun :)
