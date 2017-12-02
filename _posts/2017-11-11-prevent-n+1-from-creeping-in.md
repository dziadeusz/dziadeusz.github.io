---
layout: post
title: How to prevent the infamous N+1 problem using Spring Boot integration testing, Spock and a DataSource proxy
---

# In this blog post I'll present  a way of writting integration tests to prevent the infamous N+1 problem from silently creeping into your project, when someone changes the underlying query and transactional configuration of your service layer. 

## Introduction
This post has been inpired by [a post by Vlad Mihalcea, Java Champion](https://vladmihalcea.com/2014/02/01/how-to-detect-the-n-plus-one-query-problem-during-testing/) and his [excellent book High-Performance Java Persistence](https://www.amazon.com/High-Performance-Java-Persistence-Vlad-Mihalcea/dp/973022823X/ref=sr_1_1?ie=UTF8&qid=1512246851&sr=8-1&keywords=high+performance+java+persistence). I aim to adapt the idea to an application written with Spring Boot and Spring Data and tested in groovy by using Spock. I also use the [ttddyy datasource-assert](https://github.com/ttddyy/datasource-assert) which builds upon [ttddyy datasource-proxy](https://github.com/ttddyy/datasource-proxy). I use the [Lombok](https://projectlombok.org/) library to reduce unnecessary biolerplate code. The import statements have been omitted for breviety. 

The full source code used in this post is available on [GitHub](https://github.com/dziadeusz/n-plus-one-integration-testing).

## The Domain Model
The test project builds upon a following entity model:
{% gist dziadeusz/bf0c6cd1349f44ef48ac808c8fad605e %}
## The Web Model
The entity model is being mapped to following Data Transfer Objects eg. for the user of a Web layer of the application. 
{% gist dziadeusz/ad6e74a94925b8078c046c45e03af9aa %}

## The "Business Logic"
For the purpose of the test the same logic of fetching and mapping the structure of a tree with its branches and leafs is implemented twice. Please bear in mind that in a production application it is not a good idea to use Entities in case of a read only operation ([checkout this post by Thorben Janssen](https://www.thoughts-on-java.org/entities-dtos-use-projection/) and [this one by Vlad Mihalcea](https://vladmihalcea.com/2016/09/13/the-best-way-to-handle-the-lazyinitializationexception/)), it is better to use a DTO projection as Entities should be used to map [state transitions to DML statements](https://vladmihalcea.com/2014/07/30/a-beginners-guide-to-jpa-hibernate-entity-state-transitions/).
{% gist dziadeusz/0dab754353a22b518ccb58694522fffb %}

The "getTreeWithSingleSelect" method delegates to a Spring Data JPA Repository by calling a HQL based method which fetches the whole Entity graph with "join fetch" clauses generating a single SQL SELECT statement. The Service layer method is not @Transactional so the mapping of Entities to Data Transfer Objects happens outside of a context of a running transaction. 

On the other hand the "getTreeWithNplusOne" method calls a Repository method which only fetches the Tree without its branches collection marked as "LAZY", which is actually exposed via a proxy.
{% highlight java %}
@OneToMany(fetch = FetchType.LAZY, mappedBy = "tree")
{% endhighlight %}
Then, while the PersistenceContext is still open, the lazy loaded branches collection is accessed, which produces another SELECT statement. Afterwards the branches collection is iterated over and the lazy loaded leafs collection of each branch is accessed, which produces n SELECT statements, one for each of the n branches.
## Wrapping the DataSOurce
To verify that the second method actually produces an N+1 problem, while the first method fetches all data using a single SELETCT, we can write an integration test using an in-memory H2 database. To intercept the actual SQL statements and verify them in the spec we need to register in the ApplicationContext a ProxyDataSource which wraps the actual DataSource and enables statement counting. Wrapping it again in a ProxyTestDataSource allows accessing the actual query execution objects in the specification. The @TestConfiguration class registers also our unit-under-test, the TreeFacade. The following test and its configuration are written in Groovy.

{% gist dziadeusz/a6ae0d67022916aaff09815fc7aad621 %}
### The SpringBoot @DataJpaTest
The @DataJpaTest by default sets up the database and replaces any explicit or auto-configured DataSource. The explicit DataSource replacement needs to be disabled by using the "replace" flag of the @AutoConfigureTestDatabase annotation. By default all @DataJpaTest annotated tests are transactional. As the tested component is not called within a transactional context of some higher-level @Component, yet it defines and encloses the transactionality on its own, the test does not need to, or even shouldn't be transactional (it might've hidden the fact that some LAZY loaded properties accessed in specification assertions are not properly fetched by the tested method and are actually fetched by the test, ofcourse in this case we do count select statements anyway). The @ActiveProfiles annotation activates the dbintegration profile, which causes the configuration from the application-dbintegration.yml file to be applied i.e. the database schema is created by Hibernate automatically from the Entities (ofcourse you should not do it in a production setup) and populated by the test data defined in the data.sql file. Apart from that, SQL statement logging, formatting and parameter logging are turned on. 

## The correct single SELECT query

The first feature method tests the "correct" method and verifies that even though the whole entity graph related to a given Tree is fetched, only one sql SELECT is performed. By accessing the PreparedExecution object intercepted by the ProxyTestDataSource, we can verify some details of the staement, including the actual query. For demonstration purposes only, the select is compared to a hardcoded expected query provided by a constant in the TestConstants class. In an actuall test scenario it is better to just verify the count and type of statements and analyze the logged statements to be sure they are optimal. 

## Detecting the N+1 problem in the spec

The second feature method verifies, that while the entire Entity graph for the test Tree is being fetched, as many as twelve SELECT statements are produced underneath: apart from the first one, created by the TreeRepository findByName method to fetch the Tree, one additional query to fetch the LAZY laoded branches collection, and then ten more, one per each lazy loaded leafs collection in each Branch (10+1). In this case I use the QueryCountHolder from the datasource-proxy library to simply verify the count of SELECT statements. 

## Conclusion
In fact you do not need such a test to verify, that the produced query is correct, you can just analyze the logs. But then, how can you be sure that no one introduces a regression in the future, maybe when adding another level domain object to the hierarchy? It is always best to automate any verification process, event though a HQL query may still "look correct".
