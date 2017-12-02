---
layout: post
title: How to prevent N+1 from creeping in with Spring Boot integration testing
---

In this blog post I'll present ways to prevent the infamous N+1 problem from creeping into your project silently, when someone changes the underlying query and transactional configuration of your service layer. 

The full source code used in this post is available on [GitHub](https://github.com/dziadeusz/n-plus-one-integration-testing).

Test5
{% highlight java %}
@MappedSuperclass
@Getter
@EqualsAndHashCode(of="uuid")
@FieldDefaults(level = AccessLevel.PRIVATE)
class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    String uuid = UUID.randomUUID().toString();

    @Version
    Long version;

}
{% endhighlight %}


