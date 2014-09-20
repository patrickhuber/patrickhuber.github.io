# Designing Enterprise Class Service Contracts

## Overview
Service contracts are a critical part of a service's ability to communicate its capability to consumers.  
For this reason, time and care should be given to the contract design. 

## Canonical Expression
[Canonical Expression](http://soapatterns.org/design_patterns/canonical_expression) defines a pattern meant to address the chaos and sprawl that can occur
when service contracts are designed in an inconsistent way. The whole idea behind this pattern comes down to standardizing your contracts so that when you try to perform an action against a resource, you do it in a consistent way. 

## Canonical Expression with Reading Data
There are several Verbs that can describe an operation to a resource. You can use the words "GET", "FETCH", "READ", "QUERY", etc. All with similar meeting. The most common word  you see for service operations that fetch information is "Get".

For example, there are many ways to say "Return a Product". You might write something like this:

#### A Simple Method Example
```CSharp
Product GetProduct(int id);
``` 

The pattern is {Action}{Resource}({Identifier}). So for our example above, we have:
*  {Action} = Get
*  {Resource} = Product
*  {Identifier} = id

This type of Tokenized approach to Service operations can be extended to any resource and any operation. You could define a GetPerson method with an id parameter or a GetCategory method with an id parameter. The important part is to remain consistent.

#### A More Complex Method Example
What if you want to fetch a product by Name? Time to add another method. We don't want to be unclear about how to fetch a product, so we better describe what the product method is using as criteria.

```CSharp
Product GetProductById(int id);
Product GetProductByName(string name);
```

The pattern then changes to {Action}{Resource}By{Property}({Identifier})

Product GetProductById(int id)
*  {Action} = Get
*  {Resource} = Product
*  {Property} = Id
*  {Identifier} = id

Product GetProductByName(int name)
*  {Action} = Get
*  {Resource} = Product
*  {Property} = Name
*  {Identifier} = name

#### An Example with Relationships
What if you want to fetch a product by related products or fetch products and their related products? We typically add more methods. Now we need to start representing the return types as lists. 

```CSharp
Product GetProductById(int id);
Product GetProductByName(string name);
ICollection<Product> GetProductListByRelatedProductId(int id);
Product GetProductByIdIncludeRelatedProducts(int id);
```

#### Welcome to method sprawl
At this point it becomes clear that we have a large number of methods that all do some form of query on a Product resource. We will revisit reads in a later in the post, but first lets look at Commands such as Updates, Creates and Deletes

## Canonical Expression with Modifying State
Why say "Modifying State" vs Create, Update, Delete? 
Well, all of these operations have side effects. 
The side effect of a Create operation is the Creation of a resource in the resource store. 
The side effect of a Delete operation is the Deletion of an object from the resource store. 
The resource store is the location where the state is retained.

What side effects does a Read or Get operation have? The answer is supposed to be 'None'.

Because query (read) and command(create, update, delete) operations have such a clear seperation, a pattern emerged to describe their relationship called Command Query Response Segregation [(CQRS)](http://martinfowler.com/bliki/CQRS.html)