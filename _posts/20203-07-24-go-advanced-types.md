# Go advanced types

I've been musing about type systems in go and creating libraries and patterns that I feel fill gaps in the language. 

This article serves as documentation of what I've discovered as well as a foundation of material I use in my wasm experimentation library [here](https://github.com/patrickhuber/go-wasm). 
I ran into various issues modeling the ideas of the wasm type system and wasm component model in go. Utilizing the ideas in this article, I was able to create better representations that I felt did a better job capturing the domain.

# Enums

Go does not have an explicit enum type. Instead users are guided to create a type system with constants. 

```go
type Day int
const (
    Monday Day = iota
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
    Sunday
)
```

usage in the same package

```go
var day Day = Monday
```

or 

```go
day := Monday
```

Making the enum values constants in the package has a side effect of reserving the name so it can not be used in other structs, interfaces etc in the same package. 
This often results in go developers prefixing or suffixing these constants to distinguish them from other "enums" in the same package. 

C# solves this problem by using the enumeration followed by a period(.) followed by the value. 

```csharp
public enum Day
{ 
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday,
}
```

Usage in the same namespace:

```go
var day = Day.Monday;
```

Rust has a different resolution operator, but the result is the same

```
enum Day {
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday,
}
```

Usage is scoped with the `::` operator

```
Day::Monday
```

It is often mentiond that packages are the unit of encapsulation in go, not structs. Indeed, when creating a struct, using the lower case prefix sets its visibility to private outside the package. 

Given this notion, the pattern I use for enumerations is to scope them to an isolated package. That isolated package is usually part of a sub folder within the package where I need to use the enumeration. 

An example of this can be seen in the `kind` package of the [go-wasm library](https://github.com/patrickhuber/go-wasm/blob/main/abi/kind/kind.go). I also use the stringer generator to generate string names for the type. 

When I use the type in the abi package, using the enumeration results in code like the following:

```go
switch k {
case kind.U32:  
case kind.U64:  
case kind.Float32:
case kind.Float64:
}
```

Using the package scope allows the calling code to resolve the enumerator without having to make the constant names U32Kind, KindU32 etc. 

# Tagged Unions

Go does not have a tagged 


## Result

## Option

# Tuples
