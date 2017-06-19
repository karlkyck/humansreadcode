---
layout: post
title:  "Non-blocking Web Apps with Spring Boot & CompletableFuture"
author: karl
date: 2017-06-19
comment: true
categories: [Articles]
published: true
noindex: false
---
It's easy to write non-blocking, asynchronous multi-layered web applications using Spring 4.2+ and Java 8's CompletableFuture.
This blog post shows you how.

CompletableFuture, introduced in Java 8, provides an easy way to write asynchronous, non-blocking, multithreaded code.
Since Spring 4.2 it's now possible to have Controllers, Services, and Repositories return CompletableFuture from non-private methods annotated with @Async.
In this blog post we will see how we can take advantage of this to write non-blocking, asynchronous code across multiple layers within our application.

You can find the source code for the sample project for this post [here](https://github.com/karlkyck/spring-boot-completablefuture)

#### Interface Driven Multilayered Architecture
Our application will consist of 3 layers of responsibility with each layer performing a specific role in the application:

* The Controller layer for presentation (JSON)
* The Service layer for business logic
* The Repository layer for persistence

Using a multilayered architecture pattern allows us to write modular code. 
With the Interface Driven Design architecture pattern we can establish a well known contract at the boundary of each module.
This allows us to encapsulate the implementation details of each module and prevent those implementation details from leaking and causing coupling between modules.
  
With this approach we can substitute the implementation of a module for an alternative implementation using the same well known contract as defined by that module's interface 
e.g. an RDBMS implementation for a MongoDB implementation, or a mock implementation.
  
This approach makes it much easier unit test our code by being able to mock each module by virtue of it's interface.

#### AsyncConfiguration
To achieve non-blocking with CompletableFuture in our application we will need to configure `ThreadPoolTaskExecutor` instances for each layer within our application.
The reason for this is to ensure that one layer cannot starve another of threads and cause deadlock.

A bounded thread pool is better for performance as spawning a new thread for each request can be costly. 
It is also useful to bound a thread pool as it makes you to consider the nature of your application and the resource and tuning requirements it will need in production.

We make use of custom configuration properties, `ApplicationProperties`, to configure each `ThreadPoolExecutor`.
 
```java
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {

    ...

    @Override
    @Bean(name = TASK_EXECUTOR_DEFAULT)
    public Executor getAsyncExecutor() {
        return newTaskExecutor(TASK_EXECUTOR_NAME_PREFIX_DEFAULT);
    }

    @Bean(name = TASK_EXECUTOR_REPOSITORY)
    public Executor getRepositoryAsyncExecutor() {
        return newTaskExecutor(TASK_EXECUTOR_NAME_PREFIX_REPOSITORY);
    }

    @Bean(name = TASK_EXECUTOR_SERVICE)
    public Executor getServiceAsyncExecutor() {
        return newTaskExecutor(TASK_EXECUTOR_NAME_PREFIX_SERVICE);
    }

    @Bean(name = TASK_EXECUTOR_CONTROLLER)
    public Executor getControllerAsyncExecutor() {
        return newTaskExecutor(TASK_EXECUTOR_NAME_PREFIX_CONTROLLER);
    }

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

#### Repository Layer
Here is our repository interface. 
As we are using MongoDB for this example we extend `MongoRepository`:
  
```java
public interface UserRepository extends MongoRepository<User, String> {

    @Async(AsyncConfiguration.TASK_EXECUTOR_REPOSITORY)
    CompletableFuture<Page<User>> findAllBy(final Pageable pageable);

    @Async(AsyncConfiguration.TASK_EXECUTOR_REPOSITORY)
    CompletableFuture<User> findOneById(final String id);
}

```

We specify our `User` domain model as the type this `Repository` will manage.
The `id` type for the User entity is of type `String`.
 
Spring Data allows us to create [query methods](http://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#repositories.query-methods), methods whose very signature defines a database query.
When Spring Data creates a proxy bean for a `Repository` it will use the query method signatures to implement the query you wish to execute. 
The `findBy` convention allows us to define `query methods` and return `CompletableFuture` instances from our query methods thus making them asynchronous and non-blocking.

Check out my [blog post](http://humansreadcode.com/spring-data-completablefuture/) on creating asynchronous query methods with Spring Data and `CompletableFuture`.

##### Find All Users

With the following query method we can run a query to find all users in our database:

```java
CompletableFuture<Page<User>> findAllBy(final Pageable pageable);
```

Spring Data supports pagination out of the box and provides the `Page` type that represents a page of entity objects.
The `CompletableFuture<Page<User>>` being returned by our method is yielding a page of `User` entity objects.
To specify the pagination criteria to use in the query the `Pageable` parameter is used.
`Pageable` is another type provided by Spring for this very purpose.

##### Find One User
The query method to find one user should return only one result so we use the prefix `findOne`. 
We are also searching by the id of the user so we specify that by naming the method `findOneById`.
The parameter `id` allows us to pass the id of the user we are searching for.
To make the method asynchronous we give it a return type of `CompletableFuture<User>`.

```java
CompletableFuture<User> findOneById(final String id);
```

If no user is found the `CompletableFuture` will yield a `null` result.
Spring Data supports returning `Optional` from standard query methods, but unfortunately there is no support for returning `CompletableFuture<Optional>`.

#### Service Layer
Now to create our service layer. First we create an interface to define the contract for our service.
Other than providing a contract on how you expose the functionality of your service, interface driven development is great for testing.

```java
public interface UserService {

    CompletableFuture<Page<User>> findAll(final Pageable pageable);

    CompletableFuture<Optional<User>> findOneById(final String id);
}
```

The implementation for our service is very simple:

```java
@Service
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    public UserServiceImpl(final UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    @Async(AsyncConfiguration.TASK_EXECUTOR_SERVICE)
    public CompletableFuture<Page<User>> findAll(final Pageable pageable) {
        return userRepository.findAllBy(pageable);
    }

    @Override
    @Async(AsyncConfiguration.TASK_EXECUTOR_SERVICE)
    public CompletableFuture<Optional<User>> findOneById(final String id) {
        return userRepository
                .findOneById(id)
                .thenApply(Optional::ofNullable);
    }
}
```

To let Spring and maintainers of your code (read you) know that we are creating a bean that will be used as a service we annotate the class with `@Service`.
`@Service` doesn't provide any additional behaviour over `@Component` but it may do so at some point in the future and it helps to be explicit in your intentions for the bean. 

We are also using the recommended injection method of constructor based injection.
If there is only one constructor in the bean then Spring doesn't require the constructor be annotated with `@Inject`.
 
To enable asynchronous execution using Spring we annotate our method implementations with `@Async` and also provide the name of the executor we want the work to be dispatched to.
Here we are using the `TASK_EXECUTOR_SERVICE` executor.

In the method `findOneById` we are transforming our response object of `User` to `Optional<User>` as the requested user may not exist.
By wrapping the potentially null returned value in `Optional` we are explicitly stating that the returned element may or may not be present.

#### RestController
Here's what our RESTful controller looks like:

```java
@RestController
@RequestMapping(value = UserController.REQUEST_PATH_API_USERS)
class UserController {

    static final String REQUEST_PATH_API_USERS = "/api/users";
    private static final Logger log = LoggerFactory.getLogger(UserController.class);
    private static final String REQUEST_PATH_API_USERS_INDIVIDUAL_USER = "/{userId}";

    private final UserService userService;

    public UserController(final UserService userService) {
        this.userService = userService;
    }

    @Async(AsyncConfiguration.TASK_EXECUTOR_CONTROLLER)
    @GetMapping(produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public CompletableFuture<ResponseEntity> getUsers(final Pageable paging) {
        return userService
                .findAll(paging)
                .<ResponseEntity>thenApply(ResponseEntity::ok)
                .exceptionally(handleGetUsersFailure);
    }

    @Async(AsyncConfiguration.TASK_EXECUTOR_CONTROLLER)
    @GetMapping(value = REQUEST_PATH_API_USERS_INDIVIDUAL_USER,
                produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public CompletableFuture<ResponseEntity> getUser(@PathVariable final String userId) {
        return userService
                .findOneById(userId)
                .thenApply(mapMaybeUserToResponse)
                .exceptionally(handleGetUserFailure.apply(userId));
    }

    private static Function<Throwable, ResponseEntity> handleGetUsersFailure = throwable -> {
        log.error("Unable to retrieve users", throwable);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
    };

    private static Function<Optional<User>, ResponseEntity> mapMaybeUserToResponse = maybeUser -> maybeUser
            .<ResponseEntity>map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());

    private static Function<String, Function<Throwable, ResponseEntity>> handleGetUserFailure = userId -> throwable -> {
        log.error(String.format("Unable to retrieve user for id: %s", userId), throwable);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
    };
}
```

To make our request methods asynchronous and non-blocking we annotate them with the `@Async` annotation and specify the executor we want to dispatch the work on.
 
##### Get All Users
Here's the method we use to return all users when an HTTP `GET` request is received:
 
```java
@Async(AsyncConfiguration.TASK_EXECUTOR_CONTROLLER)
@GetMapping(produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
public CompletableFuture<ResponseEntity> getUsers(final Pageable paging) {
    return userService
            .findAll(paging)
            .<ResponseEntity>thenApply(ResponseEntity::ok)
            .exceptionally(handleGetUsersFailure);
}

private static Function<Throwable, ResponseEntity> handleGetUsersFailure = throwable -> {
    log.error("Unable to retrieve users", throwable);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
};
```

The return type from our method is `CompletableFuture<ResponseEntity>` and we annotate the method with `@Async` and specify the task executor we want to dispatch the work on.
In this case we are using the `TASK_EXECUTOR_CONTROLLER` task executor.

It is possible to specify the type returned in your `ResponseEntity` however in order to return error codes and error information of our own leaving this specificity out is necessary.
  
The method takes a `Pageable` parameter. 
This is a type provided by Spring and allows the caller to specify paging parameters e.g.

```
http://localhost:8080/api/users?page=2&size=10
```

With this URL we are requesting the `users` resource and specifying the page offset and the number of results to return.

Our service and repository methods for finding all users take this `Pageable` parameter so all we need to do is pass it on.

When we invoke our service method `findAll` we get a `CompleteableFuture<Page<User>>` back.
Our job is to map this type to the expected type `CompletableFuture<ResponseEntity>`.
By using the method `thenApply` on `CompletableFuture` we can do something with the return value in this case `Page<User>`.
All we need to do is wrap the `Page<User>` object in a `ResponseEntity` and give it a HTTP response code of 200.
Spring provides a handy way of instantiating just such a `ResponseEntity` with the `ResponseEntity.ok()` method.
  
```java
.<ResponseEntity>thenApply(ResponseEntity::ok)
```

We have to explicitly define the return type when invoking the `thenApply` method to tell Java that we are creating a `ResponseEntity<Page<User>>` object but we want it to be treated as a plain `ResponseEntity` object.
This helps us when we define our error recovery:
 
```java
.exceptionally(handleGetUsersFailure);
```

By specifying error handling code to be run in the event of an exception we can create an appropriate response to the client making the HTTP request.
In this case we want to recover from any error and return a status code of 500:

```java
ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
```

##### Get One User
This is the method we use to retrieve one user:

```java
@Async(AsyncConfiguration.TASK_EXECUTOR_CONTROLLER)
@GetMapping(value = REQUEST_PATH_API_USERS_INDIVIDUAL_USER,
            produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
public CompletableFuture<ResponseEntity> getUser(@PathVariable final String userId) {
    return userService
            .findOneById(userId)
            .thenApply(mapMaybeUserToResponse)
            .exceptionally(handleGetUserFailure.apply(userId));
}

private static Function<Optional<User>, ResponseEntity> mapMaybeUserToResponse = maybeUser -> maybeUser
        .<ResponseEntity>map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());

private static Function<String, Function<Throwable, ResponseEntity>> handleGetUserFailure = userId -> throwable -> {
    log.error(String.format("Unable to retrieve user for id: %s", userId), throwable);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
};
```

Again to make this method asynchronous and non-blocking we annotate it with `@Async` and return `CompletableFuture<ResponseEntity>`.

We need to bind the `{userId}` path variable to the `userId` parameter.
This is done with `@PathVariable`.
Issuing a GET request to the following URL will invoke this method:

```
http://localhost:8080/api/users/1234
```  

As with the `getAllUsers` method we need to transform the `CompletableFuture<Optional<User>>` response we get from the `UserService` to `ResponseEntity`.
To do this we again use the `CompletableFuture.thenApply` method.
This time we create a [pure function](http://alvinalexander.com/scala/how-to-create-scala-methods-no-side-effects-pure-functions#pure-functions) to transform the response:

```java
private static Function<Optional<User>, ResponseEntity> mapMaybeUserToResponse = maybeUser -> maybeUser
        .<ResponseEntity>map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
```
  
In the case where the user exists we are mapping the `Optional<User>` parameter to a `ResponseEntity` with the `User` object as the body and a status code of 200 (OK).
If the `Optional<User>` is empty we instead return a `ResponseEntity` with no body and a status code of 404 (Not Found).

If there an exception is thrown we recover by sending a response with a status code of 500 (Internal Server Error):
 
```java
private static Function<String, Function<Throwable, ResponseEntity>> handleGetUserFailure = userId -> throwable -> {
    log.error(String.format("Unable to retrieve user for id: %s", userId), throwable);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
};
```

#### Wrapping Up
By using this technique you can split up your application into separate layers and have each layer perform tasks asynchronously.
This separation of concerns makes it easier for you to conceptualise your logic, combine and recombine your code and logic, and make it easier to tune, and test your code from unit to integration to system testing.

Take a look at this code in action by cloning the [source code](https://github.com/karlkyck/spring-boot-completablefuture).
