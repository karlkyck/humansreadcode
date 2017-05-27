---
layout: post
title:  "Non-blocking with Spring Boot & CompletableFuture"
author: karl
date: 2017-05-22
comment: true
categories: [Articles]
published: true
noindex: false
---
Non-blocking Spring Boot & CompletableFuture
==================
CompletableFuture, introduced in Java 8, provides an easy way to write asynchronous, non-blocking, multithreaded code.
With Spring 4.2 it's possible to have Controllers, Services, and Repositories return CompletableFuture from non-private methods annotated with @Async.
In this blog post we will see how we can take advantage of this to write non-blocking, asynchronous code across multiple layers within our application.

Gradle
------
We'll begin by setting up our project using Gradle.

``

Gradle Wrapper
-- distributionSha256Sum

Spring Boot gradle project
-- https://docs.spring.io/spring-boot/docs/current/reference/html/getting-started-installing-spring-boot.html#getting-started-gradle-installation

Dependencies
------------
compile("org.springframework.boot:spring-boot-starter-data-mongodb")

AsyncConfiguration
------------------
ApplicationProperties - to specify the executor props
AsyncConfiguration
AsyncUncaughtExceptionHandler - explain why

DatabaseConfiguration
---------------------
DbMigrations with Mongobee

Repository
----------
readAllBy required by Spring for query methods return CompletableFuture

Service
-------
Constructor based injection (no need for @Autowired because there is only one constructor)
Interface driven for testing

RestController
----------
Request mapping method Annotated with @Async
Optional User object mapped to ResponseEntity.ok or else ResponseEntity.notFound
Explicit cast .<ResponseEntity>map(ResponseEntity::ok)
Some functional programming and currying

Spring Test & MockMvc
---------------------
MockMvc & Pageable -> .setCustomArgumentResolvers(pageableArgumentResolver)
Embedded mongo for testing -> testCompile 'de.flapdoodle.embed:de.flapdoodle.embed.mongo' (auto imported by spring boot)

Immutables.io
-------------
Value Types
Custom Converters for MongoDB