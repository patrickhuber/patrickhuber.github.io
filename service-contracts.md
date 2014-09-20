# Designing Enterprise Class Service Contracts

## Overview
Service contracts are a critical part of a service's ability to communicate its capability to consumers.  
For this reason, time and care should be given to the contract design. 

## Canonical Expression
[Canonical Expression](http://soapatterns.org/design_patterns/canonical_expression) defines a pattern meant to address the chaos and sprawl that can occur
when service contracts are designed in an inconsistent way. 

For example, there are many ways to say "Return a Product". You might write something like this:

```
Product GetProduct(int id)
``` 

What if you want to fetch a product by Name? Time to add another method. We don't want to be unclear about how to fetch a product, so we better describe what the product method is using as criteria.
```
Product GetProductById(int id);
Product GetProductByName(string name);
```

What if you want to fetch a product by related sub products or fetch products and their related products? We typically add more methods.
