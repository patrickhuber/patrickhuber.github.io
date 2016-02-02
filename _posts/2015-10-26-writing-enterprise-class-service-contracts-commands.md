---
layout: default
title: Designing Enterprise Class Service Contracts - Commands
---

## Overview
In a previous blog post I covered creating enterprise class serivce contracts for reading data. 
In this blog post, I will build off of the previous examples and continue with writing service contracts
for modifying state. 

The goal is to follow the CQRS pattern of separating commands and queries in order to keep a clean 
canonical data model and allow for the commands and queries to be in charge of modifying state. 

We will take some liberties from a purist CQRS implementation in the fact that the contracts will 
utilize a request response pattern instead of a command/correlation and query/response pattern. 
Whenever such liberties are taken, a comment in the code or a short description will point out 
options for the reader. 

## Recap

In the previous example, we created the following data model.

```csharp
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

## A first attempt.

When presented with a simple domain model like above, 
the first reaction is to create Update, Delete and Create methods for the domain models.

```csharp
void UpdateProduct(Product product);
void CreateProduct(Product product);
void DeleteProduct(Product product);
void UpdateProductManufacturer(Product product, Manufacturer newManufacturer);
```

```csharp
void UpdateManufacturer(Manufacturer manufacturer);
void CreateManufacturer(Manufacturer manufacturer);
void DeleteManufacturer(Manufacturer manufacturer);
```

### Whats wrong with these methods?

These simple Update, Create and Delete methods work in very simple domain cases, 
but there are a few problems that are not addressed by these methods.

* How are batch operations handled?
* Relationships need to be managed through alternative methods ex: 
	* Update a relationship
* How are fields changed when a criteria is met instead of doing the change by Id?
	
When using the domain model as the input for the create, update and delte methods directly
the flexibility of the method to perform customized operations is limited. 

The consumer of the service is often required to do a complex query before performing 
a update or delete operation. This causes the user to perform an extra round trip for every operation. 
 
During a create operation, they have to somehow know which fields are defaulted and computed from 
the domain model. If, for example, the domain model had FirstName, LastName and FullName fields, 
the consumer of the service wouldn't know what to put in the FullName field. 

### What to do?

As a first step, we will make sure that the clean domain model remains clean by avoiding the tempation
to assign action or behavior to it. The domain model should represent the data of the domain
separated from the commands that modify state and the queries that project it to consumers. 

## A Create Canonical Expression

### The ID

When creating the canonical expression for a create method, first look at the ID assignment
process. 

* Is the ID assigned by the caller? If so, provide a ID field for the caller to put the ID.
* Is the ID assigned by the service? If so, omit the ID field, but return it in the response.

For this example, we'll assume that the service is responsible for creating the ID, so it will be omitted.

### The Fields
The second thing to consider is any fields that are required as input. 
For the product creation, we want the user to provide a name and a description, so 
we should add those fields to the CreateProductRequest. 

### The Relationships
The final consideration will be relationships. In order to keep the request object small
and avoid duplicating data, relationships will be represented with a simple named ID assignment. 

### The Product Contracts

```csharp

public class CreateProductRequest
{
	public IEnumerable<CreateProduct> Products { get; set; }
}

public class CreateProduct
{
	public string Name { get; set; }
	public string Description { get; set; }
	public int ManufacturerID { get; set; }
}

public class ProductService
{
	public CreateProductResponse Create(CreateProductRequest request);
}

/// The implementation could return the entire Product, just set the Id property
/// or return a total count. 
public class CreateProductResponse
{
	public IEnumerable<Product> Products { get;set; }
}

```

```XML
<CreateProductRequest>
	<Products>
		<CreateProduct name="Door" description="A solid wood door" manufacturerId="1" />
		<CreateProduct name="Chair" description="A solid wood chair with soft seat" manufacturerId="2" />
		<CreateProduct name="Spoon]" description="A silver spoon" manufacturerId="2" />
		<CreateProduct name="Plate" description="A nice plastic plate" manufacturerId="3" />
	</Products>
</CreateProductRequest>
```

### Compare this to SQL

```SQL
INSERT INTO PRODUCT ( Name, Description, ManufacturerID )
VALUES
	('Door',  'A solid wood door', 1),
	('Chair', 'A solid wood chair with soft seat', 2),
	('Spoon', 'A silver spoon', 2),
	('Plate', 'A nice plastic plate', 3)
```

### Compare this to OData

```
POST http://services.example.org/v4/Products HTTP/1.1
OData-Version: 4.0
OData-MaxVersion: 4.0
Content-Length: 364
Content-Type: application/json
```

```JSON
[	{	"name" : "Door",
		"description" : "A solid wood door",
		"manufacturerId": 1},
	{	"name" : "Chair",
		"description" : "A solid wood chair with soft seat",
		"manufacturerId": 2},
	{	"name" : "Spoon",
		"description" : "A silver spoon",
		"manufacturerId": 2},
	{	"name" : "Plate",
		"description" : "A nice plastic plate",
		"manufacturerId": 3}
]
```

## A Delete Canonical Expression

The delete expression presents an interesting challenge. We don't simply want to delete
by Id. A delete expression provides the same capability as a simple Id Delete, and also 
provides the ability to delete filtered sets. 


## A Update Canonical Expression

Similar to the delete expression, the update expression will provide an expression based
contract. This give the consumer the ability to update multiple records with a simple clause. 

The delete differs from the Delete contract through the data fields that need updates. 
The delete contract should only present fields that should be updated so the user doesn't need
to echo back data that was queried from the data source and saves on implementation and on the wire cost. 
