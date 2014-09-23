# Designing Enterprise Class Service Contracts

## Overview
Service contracts are a critical part of a service's ability to communicate its capability to consumers.  
For this reason, time and care should be given to the contract design. 

## Canonical Expression
[Canonical Expression](http://soapatterns.org/design_patterns/canonical_expression) defines 
a pattern meant to address the chaos and sprawl that can occur when service contracts are designed in an inconsistent way. 
The whole idea behind this pattern comes down to standardizing your contracts so that when you try to perform an action against a resource, you do it in a consistent way. 

## Canonical Expression with Reading Data
There are several Verbs that can describe an operation to a resource. You can use the words "GET", "FETCH", "READ", "QUERY", etc. 
All with similar meeting. The most common word  you see for service operations that fetch information is "Get".

For example, there are many ways to say "Return a Product". You might write something like this:

## First, Our Model
```CSharp
public class Product
{
    public int Id {get;set;}
    public string Name {get;set;}
    public string Description {get;set;}  
	public Manufacturer Manufacturer {get;set;}
}

public class Manufacturer
{
	public int Id {get;set;}
	public string Name{get;set;}
	public ICollection<Product> Products{get;set;}
}
```

#### A Simple Method Example
```CSharp
Product GetProduct(int id);
``` 

The pattern is {Action}{Resource}({Identifier}). So for our example above, we have:
*  {Action} = Get
*  {Resource} = Product
*  {Identifier} = id

This type of Tokenized approach to Service operations can be extended to any resource and any operation. 
We could define a GetPerson method with an id parameter or a GetCategory method with an id parameter. 
The important part is to remain consistent.

#### A More Complex Method Example
What if you want to fetch a product by Name? Time to add another method. 
We don't want to be unclear about how to fetch a product, so we better describe what the product method is using as criteria.

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
What if you want to fetch a product by related products or fetch products and their related products? 
We typically add more methods. Now we need to start representing the return types as lists. 

```CSharp
Product GetProductById(int id);
Product GetProductByName(string name);
ICollection<Product> GetProductListByManufacturerId(int id);
Product GetProductByIdIncludeManufacturer(int id);
```

#### Welcome to method sprawl
At this point it becomes clear that we have a large number of methods that all do some form of 
query on a Product resource. When we add additional expressions, we have to add additional methods. 
We will revisit reads in a later in the post, but first lets look at Commands such as Updates, Creates and Deletes

## Canonical Expression with Modifying State
Why say "Modifying State" vs Create, Update, Delete? 
Well, all of these operations have side effects. 
The side effect of a Create operation is the Creation of a resource in the resource store. 
The side effect of a Delete operation is the Deletion of an object from the resource store. 
The resource store is the location where the state is retained.

What side effects does a Read or Get operation have? The answer is supposed to be 'None'.

Because query (read) and command(create, update, delete) operations have such a clear seperation, 
a pattern emerged to describe their relationship called 
Command Query Response Segregation [(CQRS)](http://martinfowler.com/bliki/CQRS.html)

### Simple Create, Update and Delete examples
Lets take a simple approach to modifying a Product may take the form of the following.

```CSharp
void CreateProduct(Product product);
void DeleteProduct(int id);
void UpdateProduct(Product product);
```

In the examples above, all of the methods take in the same product object with the exception of the Delete operation. 
One glaring issue with this approach is the fact that creating an Entity and updating an Entity can sometimes take different data contracts. 

What will occur if the Description field on the Product object is nullable? If we want to update the product, but not change the description, I need to echo back every field. 
For small objects this can seem trivial, but for large object sets sending back a large message when we only want to change a few fields is a bit overkill. 
Also, what do we do for batch operations? With the current contracts, several calls to the same method must be called over and over again.

## Canonical Expression Inspiration
So we expressed a few capabilities above which are summarized here:
* Read operations should handle multiple criteria
* Read operations should fetch related entities
* Read operations could return one or more entities (a set of one is still a set)

### What can we learn from databases?
A simple T-SQL example will show that the expressions above can be expressed in relative ease. For example, assume we have the following DDL.

```SQL
CREATE TABLE PRODUCTS
(
	Id int not null identity(1,1),
	Name varchar(100) not null,
	Description varchar(1000) not null,
	ManufacturerId int not null
)

CREATE TABLE MANUFACTURERS
(
	Id int not null identity(1,1),
	Name varchar(100) not null,
	Description varchar(1000) not null
)
```

For simplicity sake, the schema above represents the same structure as our domain model. Running over the list of capabilities, lets see what we can accomplish with sql select statements.

```SQL
-- select by id
-- returns single entity
SELECT *
FROM PRODUCTS
WHERE Id = 12345

-- select by name
SELECT *
FROM PRODUCTS
WHERE Name = 'grapes'

-- select multiple criteria
-- returns multiple entities
SELECT *
FROM PRODUCTS
WHERE Name = 'grapes' OR Name = 'apples'

-- select entity, expand relationship
SELECT *
FROM PRODUCTS p
INNER JOIN MANUFACTURERS m
	on m.Id = p.ManufacturerId
WHERE Id = 12345
```

SQL is very expressive and allows us to accomplish the operations with relative ease and with very terse expressions. 
The functional and set based nature of SQL can be an inspiration when we are creating a more robust data contract. 
The important piece is to think about resources as sets.

### What can we learn from the OData standard?
[OData](http://www.odata.org/) is an OASIS standard for data representation over HTTP. 
It takes the concept of resources, and applies standardized expressions over those resources.
Think of it like REST with a very strict standard.
 
Much like SQL, OData is very expressive and is functional in nature.

Using the capabilities above, and assume the Product and Manufacturer class models are now applied as serialized JSON objects.

select by id
```
GET svcroot/Products(12345)
```

select by name
```
GET svcroot/Products?$filter=Name eq 'grapes'
```

select multiple criteria
```
GET svcroot/Products?$filter=Name eq 'grapes' or Name eq 'apples'
```

select entity expand relationship
```
GET svcroot/Products(12345)?$expand=Manufacturer
```

### Pros and cons of SQL and OData
OData and SQL are very expressive, however there is a cost to this expressiveness. 
Allowing any expression to hit the repository can cause performance impacts. 
Stored procedures and Strong unitized data contracts have always been a great way to isolate paths into the database to allow for better tuning and optimization.

So there are tradeoffs:

1. You can have a very expressive data contract and attempt to blacklist (exclude) expressions that are not allowed
OR
2. You can have a limited expressive data contract and whitelist (create) expressions that are allowed.

### The approach for WCF
In the section above we expressed the tradeoffs between expression and control. 
WCF and SOAP services by nature tend to fall into the second category of limited expressiveness. 
Because of this categorization, we will need to standardize a format expressiveness.

### Request / Reply
Following a 'one model in, one model out' philosophy manifests as a Request object and a Response object.
Because the action on the resource is a Read operation (CRUD), we will name the request and reply with this information as well.

Data Contracts
```CSharp
public class ProductReadRequest
{
	public ICollection<ProductReadCriteria> Where{get;set;}
	public ProductExpand Expand {get;set;}
}

public class ProductReadCriteria
{
	public int? Id{get;set;}
	public string Name{get;set;}
}

public class ProductExpand
{
	public bool Manufacturer{get;set;}
}

public class ProductReadResponse
{
	public ICollection<Product> Products{get;set;}
}
```

Service Interface
```CSharp
public interface ProductService
{
	ProductReadResponse Read(ProductReadRequest request);
}
```

