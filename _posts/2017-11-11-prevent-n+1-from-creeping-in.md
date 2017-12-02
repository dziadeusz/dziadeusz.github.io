---
layout: post
title: How to prevent the infamous N+1 problem using Spring Boot integration testing, Spock and a DataSource proxy
---

In this blog post I'll present  a way of writting integration tests to prevent the infamous N+1 problem from silently creeping into your project, when someone changes the underlying query and transactional configuration of your service layer. 

The full source code used in this post is available on [GitHub](https://github.com/dziadeusz/n-plus-one-integration-testing).

I use the [Lombok](https://projectlombok.org/) library to reduce unnecessary biolerplate code. The import statements have been ommited for breviety. The test project builds upon a following entity model:
{% gist dziadeusz/bf0c6cd1349f44ef48ac808c8fad605e %}

The entity model is being mapped to following Data Transfer Objects eg. for the Web layer of the application.
{% gist dziadeusz/ad6e74a94925b8078c046c45e03af9aa %}

For the purpose of the experiment the same logic of fetching and mapping the structure of a tree with its branches and leafs is implemented twice. 
{% gist dziadeusz/0dab754353a22b518ccb58694522fffb %}

The "getTreeWithSingleSelect" method delegates to a Spring Data JPA Repository HQL based method which fetches the whole structure with "join fetch" clauses generating a single SQL SELECT statement. The Service layer method is not @Transactional so the mapping of Entity to Data Transfer Object happens outside of a context of any running transaction (Only the Repository method is implicitly transactional).

On the other hand the "getTreeWithNplusOne" method calls a Repository method which only fetches the Tree without its branches collection marked as "LAZY", which is actually a proxy.
```java
@OneToMany(fetch = FetchType.LAZY, mappedBy = "tree")
```
Then, while the PersistenceContext is still open, the lazy loaded branches collection is accessed, which produces another SELECT statement. Afterwards the branches collection is iterated over and the lazy loaded leafs collection of each branch is accessed, which produces n SELECT statements, one for each of the n branches.

{% gist dziadeusz/a6ae0d67022916aaff09815fc7aad621 %}
