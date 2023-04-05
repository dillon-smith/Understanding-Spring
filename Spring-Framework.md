# Introduction
Spring is one of the most widely used Java frameworks. This 2022 LinkedIn article indicates that over 60% of surveyed Java developers are using Spring Boot.

The Spring ecosystem currently consists of 21 active projects such as Spring Boot, Spring Data, Spring Batch, Spring Security, etc. All of these projects are built on top of Spring's core project: Spring Framework. Without fully understanding the core Spring Framework, utilizing Spring's other projects such as Spring Boot will prove difficult.

The Spring Framework is an Inversion of Control and Dependency Injection container. The remainder of this document provides a more comprehensive understanding of this definition.

# What is a Dependency?
Imagine you are writing a Java class that accesses a Movies table in a database. The class might look something like this:

