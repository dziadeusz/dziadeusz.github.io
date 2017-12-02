---
layout: post
title: How to prevent the infamous N+1 problem using Spring Boot integration testing, Spock and a DataSource proxy
---

In this blog post I'll present  a way of writting integration tests to prevent the infamous N+1 problem from silently creeping into your project, when someone changes the underlying query and transactional configuration of your service layer. 

This post has been inpired by [a post by Vlad Mihalcea, Java Champion](https://vladmihalcea.com/2014/02/01/how-to-detect-the-n-plus-one-query-problem-during-testing/) and his [excellent book High-Performance Java Persistence](https://www.amazon.com/High-Performance-Java-Persistence-Vlad-Mihalcea/dp/973022823X/ref=sr_1_1?ie=UTF8&qid=1512246851&sr=8-1&keywords=high+performance+java+persistence). I aim to adapt the idea to an application written with Spring Boot and Spring Data and tested in groovy by using Spock. I also use the [ttddyy datasource-assert](https://github.com/ttddyy/datasource-assert) which builds upon [ttddyy datasource-proxy](https://github.com/ttddyy/datasource-proxy). I use the [Lombok](https://projectlombok.org/) library to reduce unnecessary biolerplate code. The import statements have been omitted for breviety. 

The full source code used in this post is available on [GitHub](https://github.com/dziadeusz/n-plus-one-integration-testing).

The test project builds upon a following entity model:
{% gist dziadeusz/bf0c6cd1349f44ef48ac808c8fad605e %}

The entity model is being mapped to following Data Transfer Objects eg. for the user of a Web layer of the application. 
{% gist dziadeusz/ad6e74a94925b8078c046c45e03af9aa %}

For the purpose of the test the same logic of fetching and mapping the structure of a tree with its branches and leafs is implemented twice. Please bear in mind that in a production application it is not a good idea to use Entities in case of a read only operation ([checkout this post by Thorben Janssen](https://www.thoughts-on-java.org/entities-dtos-use-projection/) and [this one by Vlad Mihalcea](https://vladmihalcea.com/2016/09/13/the-best-way-to-handle-the-lazyinitializationexception/)), it is better to use a DTO projection as Entities should be used to map [state transitions to DML statements](https://vladmihalcea.com/2014/07/30/a-beginners-guide-to-jpa-hibernate-entity-state-transitions/).
{% gist dziadeusz/0dab754353a22b518ccb58694522fffb %}

The "getTreeWithSingleSelect" method delegates to a Spring Data JPA Repository by calling a HQL based method which fetches the whole Entity graph with "join fetch" clauses generating a single SQL SELECT statement. The Service layer method is not @Transactional so the mapping of Entities to Data Transfer Objects happens outside of a context of a running transaction. 

On the other hand the "getTreeWithNplusOne" method calls a Repository method which only fetches the Tree without its branches collection marked as "LAZY", which is actually a proxy.
```java
@OneToMany(fetch = FetchType.LAZY, mappedBy = "tree")
```
Then, while the PersistenceContext is still open, the lazy loaded branches collection is accessed, which produces another SELECT statement. Afterwards the branches collection is iterated over and the lazy loaded leafs collection of each branch is accessed, which produces n SELECT statements, one for each of the n branches.

{% gist dziadeusz/a6ae0d67022916aaff09815fc7aad621 %}
