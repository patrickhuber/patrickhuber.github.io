# Go advanced types

I've been musing about type systems in go and creating libraries and patterns that I feel fill gaps in the language. 

This article serves as documentation of what I've discovered as well as a foundation of material I use in my wasm experimentation library [here](https://github.com/patrickhuber/go-wasm). 
I ran into various issues modeling the ideas of the wasm type system and wasm component model in go. Utilizing the ideas in this article, I was able to create better representations that I felt did a better job capturing the domain.

# Enums

Go does not have an explicit enum type. Instead developers are guided to create a type system with constants. 

```go
type Kind int

const (
    U32 Kind = iota
    U64
    Float32
    Float64
)
```

usage in the same package

```go
var k Kind = U32
```

or 

```go
k := U32
```

Making the enum values constants in the package has a side effect of reserving the name so it can not be used in other structs, interfaces etc in the same package. 
This often results in go developers prefixing or suffixing these constants to distinguish them from other "enums" in the same package. 

C# solves this problem by using the enumeration followed by a period(.) followed by the value. 

```csharp
public enum Kind
{ 
    U32,
    U64,
    F32,
    F64,
}
```

Usage in the same namespace:

```go
var kind = Kind.U32;
```

Rust has a different resolution operator, but the result is the same

```rust
enum Kind {
    U32,
    U64,
    F32,
    F64,
}
```

Usage is scoped with the `::` operator

```rust
Kind::U32
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

Applying this to our example above results in the following:

```go
package kind      // put the enum in its own package

type Kind int

const (
    U32 Kind = iota
    U64
    Float32
    Float64
)
```

We would then us the type in other packages by utilizing the import

```go
kind.U32
```

# Tagged Unions

Go does not have a tagged union type, but it does have structs and interface. The general consensus is to favor composition over inheritance, but it is difficult to transition a clear type system like the wasm spec without some kind of type hierarchy. You can see the scope of what I'm talking about here: https://webassembly.github.io/spec/core/syntax/types.html.

```ebnf
numtype ::= i32 | i64 | f32 | f64
vectype ::= v128
reftype ::= funcref | externref
valtype ::= numtype | vectype | reftype
```

At first I tried to model hierarchies with composition, but ended up with a lot of null fields on structs. For example, to model the wasm number types, vec types and ref types with composition, you end up with a huge struct with a lot of pointer fields which result in added space to the struct. For small use cases this may not matter, but it is a lot of wasted memory created for a struct that is mostly nil pointers.

```go
type I32 struct{}
type I64 struct{}
type F32 struct{}
type F64 struct{}
type VecType struct{}
type NumType struct{
   I32 *I32
   I64 *I64
   F32 *F32
   F64 *F64
}
type RefType struct{
    FuncRef *FuncRef
    ExternRef *ExternRef
}
type ValType struct{
   NumType *NumType
   RefType *RefType
   VecType *VecType
}
```

I also found using the types very difficult. There was a lot of nil checking required. For example, to see if a ValType is a F32 you need to do the following:

```go
func SomeFunction(v *ValType) error {
   if v.NumType == nil {
       return fmt.Errorf("ValType is not a NumType")
   }
   nt := v.NumType
   if nt.F32 == nil {
       return fmt.Errorf("ValType is not a F32")
   }
   // do something with the f32
}
```

In other languages, you can do a type check to see if the type is of a specific sub type. For example, rust has [pattern matching](https://doc.rust-lang.org/book/ch18-03-pattern-syntax.html#destructuring-enums) and C# has the [is operator](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/is). 

Go also has type matching, but you need to model the system in a different way in order for it to work. For example, using struct embedding is just another version of what we have above and has all the same issues. 

The go authors themselves even run into challenges where a type system hierarchy makes the programming task much easier https://cs.opensource.google/go/go/+/master:src/go/ast/ast.go;l=14 and are the source of the sealed interface pattern.

```go
// All node types implement the Node interface.
type Node interface {
	Pos() token.Pos // position of first character belonging to the node
	End() token.Pos // position of first character immediately after the node
}

// All expression nodes implement the Expr interface.
type Expr interface {
	Node
	exprNode()
}

// All statement nodes implement the Stmt interface.
type Stmt interface {
	Node
	stmtNode()
}

// All declaration nodes implement the Decl interface.
type Decl interface {
	Node
	declNode()
}
```

The sealed interface pattern defines a private member function on a public interface. Structs of the same package are the only structs that can implement the interface member so you end up with the ability to model inheritance without exposing nonsense methods as public memthods of the struct. 

Applying this pattern to our wasm type example above, we end up with the following structure:

```go
type Value interface {
    value()
}

type Number interface {
    Value()
    number()
}

type Vec interface{
    Value()
    vec()
}

type Ref interface{
    Value()
    ref()
}

// Number types
type I32 struct{}

func (I32) number() {}
func (I32) value()  {}

type I64 struct{}

func (I64) number() {}
func (I64) value()  {}

type F32 struct{}

func (F32) number() {}
func (F32) value()  {}

type F64 struct{}

func (F64) number() {}
func (F64) value()  {}

// Vec types
type V128 struct{}
func (V128) value() {}
func (V128) vec() {}

// Ref Types
type FuncRef struct{}
func (FuncRef) value(){}
func (FuncRef) ref() {}

type ExternRef struct{}
func (ExternRef) value() {}
func (ExternRef) ref() {}
```

With the above model, we have eliminated the need for null checks and removed the wasted space taken by unused pointers. In our function example above, we can change it to use a type switch or type assertion:

```go
func SomeFunction(v Value) error {
   f32, ok := v.(F32)
   if !ok{
      return fmt.Errorf("ValType is not a F32")
   }
   // do something with the f32
}
```

We only represented a tagged union above using a simple hierarchy, when using type assertion or type switches you get access to the underlying type of the assertion. So we could also access member fields. For example, if we created a List struct, we could add a Type field to represent a list of items of a specific type.

```go
type Collection interface{
   Value
   collection()
}
type List struct{
   Type Value
}
func (List) value() {}
func (List) collection(){}

func IsList(v Value) bool{
   _, ok := v.(List)
   return ok
}
```

## Result

## Option

# Tuples
