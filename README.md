# Designing a Domain Model to enforce No Duplicate Names

Some design approaches to enforcing a business rule requiring no duplicates.

## The Problem

You have some entity in your domain model. This entity has a `name` property. You need to ensure that this name is unique within your application. How do you solve this problem?

## The Database

This simplest approach is to ignore the rule in your domain model, and just rely on some infrastructure to take care of it for you. Typically this will take the form of a unique constraint on a SQL database which will throw an exception if an attempt is made to insert or update a duplicate value in the entity's table. This works, but doesn't allow for changes in persistence (to a system that doesn't support unique constraints or doesn't do so in a cost or performance acceptable fashion) nor does it keep business rules in the domain model itself.

## Domain Service

One option is to detect if the name is duplicated in a domain service, and force updates to the name to flow through the service. This keeps the logic in the domain model, but leads to anemic entities.

## Pass Necessary Data Into Entity Method

One option is to give the entity method all of the data it needs to perform the check. In this case you would need to pass in a list of every name that it's supposed to be different from. And of course don't include the entity's current name, since naturally it's allowed to be what it already is. This probably doesn't scale well when you could have millions or more entities.

## Pass a Service for Checking Uniqueness Into Entity Method

With this option, you perform dependency injection via method injection, and pass the necessary dependency into the entity. Unfortunately, this means the caller will need to figure out how to get this dependency in order to call the method. They also could pass in the wrong service, or perhaps no service at all, in which case the domain rule would be bypassed.

## Pass a Function for Checking Uniqueness Into Entity Method

This is just like the previous option, but instead of passing a type we just pass a function. Unfortunately, the function needs to have all the necessary dependencies and/or data to perform the work, which otherwise would have been encapsulated in the entity method. There's also nothing in the design that requires the calling code to pass in the appropriate function or even any useful function at all (a no-op function could easily be supplied). The lack of encapsulation means the validation of the business rule is not enforced at all by our domain model, but only by the attentiveness and discipline of the client application developer.

## Reference

This project was inspired by [this exchange on twitter between Kamil and me](https://twitter.com/kamgrzybek/status/1280868055627763713). Kamil's ideas for approaching this problem included:

1. Pass all the current names to the update method. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/03_PassDataToMethod)
2. Pass all the names matching the proposed name to the update method. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/06_PassFilteredDataToMethod)
3. Pass an `IUniquenessChecker` into the update method which returns a count of entities with that name. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/04_MethodInjectionService)
4. Pass a function that performs the same logic as #3. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/05_MethodInjectionFunction)
5. Check the invariant in an Aggregate.
6. Create a unique constraint in the database. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/01_Database)

My own two approaches include:

7. Use a domain service (which will lead to an anemic entity) [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/02_DomainService)
8. Use domain events and a handler to perform the logic

### Learn More

[Learn Domain-Driven Design Fundamentals](https://www.pluralsight.com/courses/domain-driven-design-fundamentals)
