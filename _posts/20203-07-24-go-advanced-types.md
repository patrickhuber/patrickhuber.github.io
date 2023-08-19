# Go advanced types

I've been musing about type systems in go and creating libraries and patterns that I feel fill gaps in the language. 

This article serves as documentation of what I've discovered as well as a foundation of material I use in my wasm experimentation library [here](https://github.com/patrickhuber/go-wasm). 
I ran into various issues modeling the ideas of the wasm type system and wasm component model in go. Utilizing the ideas in this article, I was able to create representations that I feel do a better job capturing the domain.

# Enums

Go does not have an explicit enum type. Instead developers are guided to create a type system with constants. 

```go
type Kind int

const (
    I32 Kind = iota
    I64
    Float32
    Float64
)
```

There are a few issues with this pattern, namely you can assign any integer to a Kind value and the compiler won't throw an error. This is due to `type Kind int` defining an alias for Kind, but not making it a new type. You can see this live here https://go.dev/play/p/fkiGltPxXn1

```go
package main

type Color int

const (
	Red   Color = 0
	Green Color = 1
	Blue  Color = 2
)

func main() {
	var carColor Color = 10
	fmt.Printf("color : %v", carColor)
}
```

One enhancement can be made to the color type to prevent this issue. We can change around our Color type from an alias to a public interface with a single private member function. This pattern enforces the type constraint of Color and prevents us from assigning unspecified values to the color type. Because the interface is sealed, the only implementation of Color can be supplied by the package. You can see the compiler error here https://go.dev/play/p/HRPjEo82SXE

```go
package main

import "fmt"

type Color interface {
	color()
}
type color int

func (color) color() {}

const (
	Red   color = 0
	Green color = 1
	Blue  color = 2
)

func main() {
	var carColor Color = 10
	fmt.Printf("color : %v", carColor)
}
```

Another issue, more difficult to solve, is the lack of exhaustive checking for all enum values. For example, nothing is preventing us from completely missing the `Blue` case in the following type switch. 

```go
var c Color = Blue
switch c{
case Red:
  fmt.Println("Red")
case Green:
  fmt.Println("Green")
}
```

In this case, nothing is printed because we forgot to put a type switch in for the `Blue` case. Some languages have detection mechanisms to check switches like this for exhaustive checking. 

Fortunately go has made it's AST available as a library in the https://pkg.go.dev/golang.org/x/tools/go/analysis package. The exaustive static check can be written as an analyzer and an examle can be found here https://github.com/nishanths/exhaustive. This analyzer will check for enum values in switch statements as well as in maps. 

At the time of writing, the `exhaustive` analyzer doesn't support the pattern above. I did write a test using the idiomatic approach and it indeed works as expected.

```go
package main

import "fmt"

type Vehicle int

const (
	Car   Vehicle = 0
	Truck Vehicle = 1
	Van   Vehicle = 2
)

func main() {
	var v Vehicle = Car
	switch v {
	case Truck:
		fmt.Println("Truck")
	case Van:
		fmt.Println("Van")
	}
}
```

```bash
exhaustive .
main.go:36:2: missing cases in switch of type main.Vehicle: main.Car
```

# Tagged Unions

Go does not have a tagged union type, but it does have structs and interface. The general consensus is to favor composition over inheritance, but it is difficult to transition a clear type system like the wasm spec without some kind of type hierarchy. You can see the scope of what I'm talking about here: https://webassembly.github.io/spec/core/syntax/types.html.

```ebnf
numtype ::= i32 | i64 | f32 | f64
vectype ::= v128
reftype ::= funcref | externref
valtype ::= numtype | vectype | reftype
```

At first I tried to model hierarchies with composition, but ended up with a lot of null fields on structs. For example, to model the wasm number types, vec types and ref types with composition, you end up with a struct with a lot of pointer fields which result in added space to the struct. For small use cases this may not matter, but it is a lot of wasted memory created for a struct that is mostly nil pointers.

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

So what can be done? Again, the sealed interface pattern can be used to constrain the types. We will instead use the interface over a struct type which allows us to model a tagged union.

```go
type Value interface {
    value()
}

type Number interface {
    Value
    number()
}

type Vec interface{
    Value
    vec()
}

type Ref interface{
    Value
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

func IsListI32(v Value) bool{
   l, ok := v.(List)
   if !ok{
       return false
   }
   _, ok := l.Type.(I32)
   return ok
}
```

## Result

We now have the ability to model tagged unions so we can apply this to common tagged unions in other languages. First on the list is the Result[TValue, TError] type used to represent if a value is Ok or Error. This tagged union is part of the standard library in rust, so if you have used that language you have probably used it frequently. 

Idiomatic go represents a result using Tuple semantics but without first class Tuple support. For example, to return an value or an error we can do the following:

```go
func ReturnsSomethingOrError() (int, error) {
   return 0, nil
}
```

The tuple here can be used at the call site with another tuple deconstructing the function return values

```go
func CallReturnSomethingOrError() {
   v, err := ReturnSomethingOrError()
   if err != nil{
      fmt.Println(err)
   }else{
      fmt.Println(v)
   }
}
```

This is the extent of the result type in go. You can use it as a signature of a function or in multiple assignments, but you can't pass it around as a value. What if we want to use the output in a channel? We would need two channels, one for the error and one for the success. Another option would be to create a struct of channels or a channel of a struct. 

Furthermore, return tuples do not really represent the concept of "Error OR Value". Instead they model "Error AND Value". The developer expects that the function will return either value, but there is nothing stopping the implementer of the function from returning both. These cases are often exposed via documentation and the type system of go does not enforce this in any way. When the contract of a function requires deep understanding of implementation internals, consumers will need to be extra careful to handle the different cases. 

What can be done? Using go generics and the tagged union concepts from above, we can create a tagged union of types. One type will be called Ok, and represents a success. The other will be called Error and represents a failure. In go there is an idiomatic error type defined as an interface so we don't necessarily need to represent a result as Result[TOk, TError], but we could. 

```go
type Result[T any] interface{
    result(t T)
    IsError() bool
    IsOk() bool
}
```

The Ok type implements the result type but only exposes a successful value.

```go
type Ok[T any] struct{
    Value T
}

func (Ok[T]) result(t T){}

func (Ok[T]) IsOk() bool{
   return true
}

func (Ok[T]) IsError() bool {
   return false
}
```

The Error type implements the result type but only exposes an error.

```go
type Error[T any] struct{
    Error error
}

func (Error[T]) result(t T){}

func (Error[T]) IsOk() bool {
    return false
}

func (Error[T]) IsError() bool {
    return true
}
```

Using a Result type avoids the issues of Tuple return where there is either a Value or an Error, not both. This provides a clear contract to the consumer that they will not be receiving both values and need to handle ambiguity when both values are returned. 

A full example of the Result type can be found in my [types library](https://github.com/patrickhuber/go-types) 

## Option

# Tuples
