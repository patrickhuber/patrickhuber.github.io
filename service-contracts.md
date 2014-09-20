# Designing Enterprise Class Service Contracts

## Overview
Service contracts are a critical part of a service's ability to communicate its capability to consumers.  
For this reason, time and care should be given to the contract design. 

## Canonical Expression
[Canonical Expression](http://soapatterns.org/design_patterns/canonical_expression) defines a pattern meant to address the chaos and sprawl that can occur
when service contracts are designed in an inconsistent way. The whole idea behind this pattern comes down to standardizing your contracts so that when you try to perform an action against a resource, you do it in a consistent way. 

There are several Verbs that can describe an operation to a resource. You can use the words "GET", "FETCH", "READ", "QUERY", etc. All with similar meeting. The most common word  you see for service operations that fetch information is "Get".

For example, there are many ways to say "Return a Product". You might write something like this:

### Canonical Expression Simple Example
```CSharp
Product GetProduct(int id);
``` 

The pattern is {Action}{Resource}({Identifier}). So for our example above, we have:
*  {Action} = Get
*  {Resource} = Product
*  {Identifier} = id

This type of Tokenized approach to Service operations can be extended to any resource and any operation. You could define a GetPerson method with an id parameter or a GetCategory method with an id parameter. The important part is to remain consistent.

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

What if you want to fetch a product by related products or fetch products and their related products? We typically add more methods.

```CSharp
Product GetProductById(int id);
Product GetProductByName(string name);
ICollection<Product> GetProductListByRelatedProductId(int id);
Product GetProductByIdIncludeRelatedProducts(int id);
```

