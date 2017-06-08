---
layout: post
title:  "Asynchronous Query Methods with Spring Data & CompletableFuture"
author: karl
date: 2017-06-08
comment: true
categories: [Articles]
published: true
noindex: false
---
[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), introduced in Java 8, provides an easy way to write asynchronous, non-blocking, multithreaded code.
Since Spring 4.2 it's now possible to write asynchronous code by returning `CompletableFuture` from non-private methods annotated with `@Async`.

Spring Data has taken advantage of this advancement and now allows you to to write non-blocking, asynchronous [Repository queries using CompletableFuture](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-async).
This is done by returning `CompletableFuture` from a query method and annotating the method with the `@Async` annotation.

When invoked the query method will return immediately with the actual query execution taking place in a task submitted to a [Spring TaskExecutor](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/scheduling.html).

You can find the source code for this blog post [here](https://github.com/karlkyck/spring-boot-completablefuture/blob/master/src/main/java/com/humansreadcode/example/repository/UserRepository.java).

####Repository
Creating asynchronous query methods in a Repository is as simple as returning CompletableFuture from your query method and annotating it with `@Async`:

```java
public interface UserRepository extends MongoRepository<User, String> {

    @Async(AsyncConfiguration.TASK_EXECUTOR_REPOSITORY)
    CompletableFuture<Page<User>> findAllBy(final Pageable pageable);

    @Async(AsyncConfiguration.TASK_EXECUTOR_REPOSITORY)
    CompletableFuture<User> findOneById(final String id);
}
```

Spring even allows you to specify the `TaskExecutor` to run the query on.
In this case the query methods are using a `TaskExecutor` configured in the custom `AsyncConfiguration` file.

####Task Executor
When configuring a `TaskExecutor` to use with asynchronous query method it is often advantageous to use a bounded [ThreadPoolTaskExecutor](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/concurrent/ThreadPoolTaskExecutor.html).
  
A bounded thread pool is better for performance as spawning a new thread for each request can be costly. 
It is also useful to bound a thread pool as it makes you to consider the nature of your application and the resource and tuning requirements it will need in production.

Here's an example of how to create a `ThreadPoolTaskExecutor`, the source code for which can be found [here](https://github.com/karlkyck/spring-boot-completablefuture/blob/master/src/main/java/com/humansreadcode/example/config/AsyncConfiguration.java):
 
```java
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {

    private static final String TASK_EXECUTOR_DEFAULT = "taskExecutor";
    private static final String TASK_EXECUTOR_NAME_PREFIX_DEFAULT = "taskExecutor-";
    private static final String TASK_EXECUTOR_NAME_PREFIX_REPOSITORY = "serviceTaskExecutor-";
    ...

    public static final String TASK_EXECUTOR_REPOSITORY = "repositoryTaskExecutor";
    ...

    private final ApplicationProperties applicationProperties;

    public AsyncConfiguration(final ApplicationProperties applicationProperties) {
        this.applicationProperties = applicationProperties;
    }

    @Override
    @Bean(name = TASK_EXECUTOR_DEFAULT)
    public Executor getAsyncExecutor() {
        return newTaskExecutor(TASK_EXECUTOR_NAME_PREFIX_DEFAULT);
    }

    @Bean(name = TASK_EXECUTOR_REPOSITORY)
    public Executor getRepositoryAsyncExecutor() {
        return newTaskExecutor(TASK_EXECUTOR_NAME_PREFIX_REPOSITORY);
    }
    
    ...

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new SimpleAsyncUncaughtExceptionHandler();
    }

    private Executor newTaskExecutor(final String taskExecutorNamePrefix) {
        final ApplicationProperties.Async asyncProperties = applicationProperties.getAsync();
        final ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(asyncProperties.getCorePoolSize());
        executor.setMaxPoolSize(asyncProperties.getMaxPoolSize());
        executor.setQueueCapacity(asyncProperties.getQueueCapacity());
        executor.setThreadNamePrefix(taskExecutorNamePrefix);
        return executor;
    }
}
```

The thread pool configuration parameters are being supplied by the custom configuration properties singleton that is injected into the constructor.

The [SimpleAsyncUncaughtExceptionHandler](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/interceptor/SimpleAsyncUncaughtExceptionHandler.html) simply logs any exceptions that occur during the execution of a task. 

Hopefully this gives you an idea of the powerful asynchronous features Spring with it's support for Completable brings. 
