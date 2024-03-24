---
layout: post
title: Declarative SQL with JOOQ
categories: [Kotlin, Database, SQL, JOOQ]
---

# Introduction
I believe every developer working with technologies such as Java or Kotlin had to deal with databases and implementation
of specific interaction layer with at some point in their career. Usually probably the most common tool which firstly comes
to mind is `JDBC` - Java Database Connectivity Driver. It is a standard API for connecting to relational databases and 
executing SQL queries. This is usually the first layer (the closest to database), then we have usually some kind of
abstraction layer on top of it, which is responsible for translating database records into objects and vice versa. This
layer is usually implemented using `ORM` (Object-Relational Mapping) libraries like `Hibernate`, `JPA`, `Ebean` etc. Going
even further, in ecosystems where we would use Spring Framework, we could use `Spring Data JPA` which under the hood uses
`Hibernate` but hides its complexity behind simple interfaces and annotations.

# Problem statement

## Data model vs application logic
When it comes to creating complex system nowadays we should be asking ourselves a question: "Will my data model drive the
application logic or will the application logic drive the data model?". It usually comes down to realization if our database
can outlive the data model. If there is possibility that application can come and go, it will be rewritten, its codebase
totally changes, but data model will stay consistent and database is somehow priority, then we should take into consideration
clear separation between data model and application logic which is usually not the case when using `ORM's`.

## Complex reads vs writes
Another common problem which we can observe in systems is the complexity of reads and writes. Usually, writes are simple
and straightforward - we just insert, update or delete records in the database. But when it comes to reads, especially
when we have created complicated schema with many relations between tables, the complexity of the queries can grow exponentially.
Usually `ORM's` in such cases add just more complexity coming directly from it's internal specifications based on the
multiple annotations and configurations which we have to conceptually understand.

# Imperative vs declarative approach
Let's look at two paradigms in which our applications can interact with the database.

## Imperative approach
Generally `ORMs` are considered more imperative in nature because they focus on how to execute an operation by mapping
Java objects to database tables. ORMs abstract away much of the direct handling of SQL queries, allowing developers to work
with Java objects instead of direct database operations. The imperative approach is more evident in how ORMs
manage entity state transitions, caching, and transactional logic, even though it allows for declarative transaction
management through annotations.

## Declarative approach
On the other hand, when we want to interact with the database in a more declarative way, we can use directly `SQL` queries
from the level of our code. There are multiple ways how we can achieve it and implement, for example by implementing `Active Record`
pattern and specifying specific components to interact with database using raw `SQL` statements. There is also another way
which is using `JOOQ` - Java Object Oriented Querying library which allows us to write `SQL` queries in a declarative, type-safe
and fluent way.

# JOOQ (Java Object Oriented Querying)

## What is JOOQ?
> jOOQ generates Java code from your database and lets you build type safe SQL queries through its fluent API.

In a nutshell, `JOOQ` is a open-source library which allows us to write `SQL` queries in a type-safe and fluent way.
It generates Java classes based on the database schema which we can use to interact with the database. It is database 
agnostic in the sense that it is able to transform subtle differences between multiple dialects and allows to generate
queries which are compatible with multiple databases. 

## JOOQ vs Hibernate
You may ask, why I should even consider migrating from `Hibernate` to `JOOQ`? If your application is simple, and you
mostly do simple read and writes, then `Hibernate` is probably a good choice. But if your data model is complex, and you
have to deal with many complex read queries, have a clear and strict separation between data model and application logic,
then `JOOQ` is probably a better choice. `JOOQ` is SQL-centric, it does not come with complex configuration and immense amount
of multiple mechanisms and annotations we have to learn in order to use it effectively. 

# Real-world example

Let's build something functional, since we already know what `JOOQ` is, let's see how we can use it in practice to build
a simple application which interacts with the database.

## Setting up the project
We will use `Kotlin` as our programming language, `JOOQ` as our database interaction library and `MySQL` as database.
First, we need to add `JOOQ` to our project. We can do it by adding following dependencies to our `build.gradle.kts` file:

```kotlin
buildscript {
	repositories {
		google()
		mavenCentral()
	}
	dependencies {
		classpath("org.jooq:jooq-codegen:3.17.5")
		classpath("org.postgresql:postgresql:42.5.0")
		classpath("org.jooq:jooq-meta-extensions:3.17.5")
	}
}

dependencies {
    implementation("org.jooq:jooq:3.17.5")
    implementation("org.jooq:jooq-codegen:3.17.5")
    implementation("org.jooq:jooq-meta-extensions:3.17.5")

    implementation("org.jooq:jooq-meta:3.17.5")
    implementation("org.jooq:jooq-kotlin-coroutines:3.17.5")
    implementation("org.jooq:jooq-kotlin:3.17.5")
}


fun generateJooq() {
    Configuration().apply {
        generator = Generator().apply {
            name = "org.jooq.codegen.KotlinGenerator"
            database = Database().apply {
                name = "org.jooq.meta.extensions.ddl.DDLDatabase"
                properties = listOf(
                    Property()
                        .withKey("scripts")
                        .withValue("$projectDir/src/main/resources/db/changelog.sql"),
                )
            }
            generate = Generate().apply {
                pojosAsKotlinDataClasses = true
                isDeprecationOnUnknownTypes = false
                isKotlinNotNullPojoAttributes = false
                isKotlinNotNullRecordAttributes = false
                isKotlinNotNullInterfaceAttributes = false
                isKotlinDefaultedNullablePojoAttributes = true
                isKotlinDefaultedNullableRecordAttributes = true
            }
            target = Target().apply {
                packageName = "database.schema"
                directory = jooqGenerationTargetDir
            }
        }
    }.also {
        generate(it)
    }
}
```

What we have done above, is we have added `JOOQ` dependencies to our project, and we have also added a function which
will generate `JOOQ` classes based on the database schema. Since our codebase is in `Kotlin`, we have also specified
a dedicated `KotlinGenerator` which will generate `Kotlin` classes for us and provide us with a fluent API to interact
with the database using `Kotlin` specific features like data classes, named parameters etc.

## Database schema
Now its time to create a database schema. We will create a simple schema with two tables: `payments` and `data`. The
`payments` table will store information about payments, and the `data` table will store some additional data which we
will use to demonstrate how we can join tables using `JOOQ`.

```sql
CREATE TABLE IF NOT EXISTS payment
(
    id      UUID          NOT NULL PRIMARY KEY,
    amount  DECIMAL(14,2) NOT NULL,
    data_id UUID          NOT NULL
);

CREATE TABLE IF NOT EXISTS data
(
    id          UUID         NOT NULL PRIMARY KEY,
    information VARCHAR(255) NOT NULL
);

ALTER TABLE payment ADD CONSTRAINT fk_data FOREIGN KEY (data_id) REFERENCES data (id) ON UPDATE RESTRICT ON DELETE CASCADE;
```

## Writing queries
Now, let's write some queries using `JOOQ`. We will write a simple query which will fetch `payment` by id with its metadata
from the database using generated `JOOQ` classes and type safe `SQL` query. Let's look at the example:

```kotlin
private val dslContext = DSL.using(
    ConnectionFactories.get("r2dbc:pool:postgresql://user:pass@127.0.0.1:5432/db")
)

class PaymentRepository {

    fun fetchPaymentByIdBlocking(id: UUID): Payment? {
        return dslContext
            .select()
            .from(PAYMENT)
            .join(DATA).onKey(Keys.FK_DATA)
            .where(PAYMENT.ID.eq(id))
            .fetchOne()
            ?.toDomain()
    }
    
    suspend fun fetchPaymentByIdNonBlocking(id: UUID): Payment? {
        return dslContext
            .select()
            .from(PAYMENT)
            .join(DATA).onKey(Keys.FK_DATA)
            .where(PAYMENT.ID.eq(uuid))
            .awaitFirstOrNull()
            ?.let { record: PaymentRecord ->
                record.into(PAYMENT).toDomain(data = it.into(DATA))
            }
    }

    private fun Record.toDomain() = Payment(
        id = this[PAYMENT.ID],
        amount = this[PAYMENT.AMOUNT],
        data = Payment.Data(
            id = this[DATA.ID],
            information = this[DATA.INFORMATION]
        )
    )

    private fun PaymentRecord.toDomain(data: DataRecord) = Payment(
        id = id,
        amount = amount,
        data = data.toDomain()
    )

    private fun DataRecord.toDomain() = Payment.Data(
        id = id,
        information = information
    )
}
```

What we have done above is we have created a `PaymentRepository` class which is responsible for fetching `Payment` by id.
We have created two methods: `fetchPaymentByIdBlocking` and `fetchPaymentByIdNonBlocking`. The first one is blocking and
shows how we can use type-safe `SQL` queries to fetch data from the database. Let's go through each step of execution to
better understand what is happening here:

1. We are creating a `DSLContext` object which is responsible for executing queries against the database. It allows to
configure JOOQ's behaviour when executing queries. Here we configured our context to using specific connection factory
based on the database URL.
2. `select().from(PAYMENT)` - we are creating a `SELECT` query which will fetch data from the `PAYMENT` table. `PAYMENT` 
table is expressed as static reference within JOOQ's generated classes.
```java
public class Payment extends TableImpl<PaymentRecord> {

    private static final long serialVersionUID = 1L;

    /**
     * The reference instance of <code>payment</code>
     */
    public static final Payment PAYMENT = new Payment();
```
3. `join(DATA).onKey()` - we are joining `PAYMENT` table with `DATA` table on the foreign key. This is a type-safe way
to express join between two tables.
4. `where(PAYMENT.ID.eq(id))` - we are specifying a condition which will filter out only records which have `ID` equal
to the one we are passing as a parameter. As we can see one more time, we are using type-safe way to express this condition,
the UUID type is inferred from the database schema and matched against the UUID parameter passed as method argument.
5. `fetchOne()` - we are executing the query and fetching the first record which matches the condition. This method returns
`Record` object which is a representation of the fetched record.
6. `toDomain()` - we are converting fetched `Record` object into our domain object `Payment` which is a data class 
representing the payment. For that we are using one more time type-safe way to access fields from the `Record` object
using generated classes.
```java
    // JOOQ generated source
    public final TableField<PaymentRecord, UUID> ID = createField(DSL.name("id"), SQLDataType.UUID.nullable(false), this, "");

    /**
     * The column <code>payment.amount</code>.
     */
    public final TableField<PaymentRecord, String> AMOUNT = createField(DSL.name("amount"), SQLDataType.VARCHAR(12), this, "");

    /**
     * The column <code>payment.data_id</code>.
     */
    public final TableField<PaymentRecord, UUID> DATA_ID = createField(DSL.name("data_id"), SQLDataType.UUID, this, "");
```

The second method `fetchPaymentByIdNonBlocking` is a non-blocking version of the first method. It is using `R2DBC` driver
to interact with the database in a non-blocking way. The rest of the logic is the same as in the blocking version, but we
are using `awaitFirstOrNull()` method to fetch the first record from the database in a non-blocking fashion which nicely 
bridges the reactive implementation of `R2DBC` with `Kotlin` coroutines in order to provide easy and
convenient way to write non-blocking code. We also used dedicated `<R extends Record> R into(Table<R> table)` method to
convert fetched `Record` object into `PaymentRecord` object:
```java
// JOOQ generated source
public class PaymentRecord extends UpdatableRecordImpl<PaymentRecord> implements Record3<UUID, String, UUID> {
```
which gives more declarative way to express the conversion.

# Conclusion
In this article we have seen how we can use `JOOQ` to interact with the database in a more declarative way. In order to
achieve that, we took advantage of `JOOQ`'s type-safe and fluent API to write `SQL` queries as we would normally do in
`SQL` editor. We also took a glance at how we can combine Kotlin coroutines, R2DBC specification as reactive driver and
JOOQ to write non-blocking code which is easy to read and scale. 

The link to the full source code can be found [Github repository](https://github.com/stosik/cockroach-r2dbc/tree/main)