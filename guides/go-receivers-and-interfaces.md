# Go Receivers and interfaces

- In this document what are receivers and how can we attach methods to our custom types using them and then how interfaces work.

## Custom User Types

- We can make types out of anything in Go!
- To make a type, say User, with multiple fields we simply create a struct:

```go
// the internal fields are private and it being public is out of scope of the context
type User struct{
    firstName string
    lastName *string // '*' to handle nil cases
    age int
    email string
    password string

}
```

```txt
Q. What does this even mean?
It means that there's a type User with 5 fields in it with underlying types defined as string, *string and int, i.e., simple types which the language already understands.
```

```go
u:=User // represents the collection of those types
```

Similarly, this definition can be extended to simple types like `int` or `string` etc.

```go
type Age int
```

Now the above code snippet means that `Age` is a type which stores `int` just like User stored multiple fields, here it itself is the value but is of type `Age` and NOT `int`.

## Attaching methods to custom types

We can attach methods to any type we define ourselves in our own package. This includes types built on top of primitives (like type Age int), structs, function types, slices, maps — anything we've given a name to with type. We cannot attach methods directly to built-in types like int or string, or to types defined in other packages.

Example:

```go
import "fmt"
type Age int

func (a Age) IsAdult() bool{
    return a >= 18
}

func main(){
    var a Age = 23
    fmt.Println(a.IsAdult())
}
```

The above code snippet attaches a method to any type with the help of receivers! The `(a Age)` just before the function name is the receiver, it means that the `IsAdult` function can be called via the dot operator on any variable whose type is `Age`.

- **Interfaces are types that describe a method contract. You cannot declare new methods on an interface using receiver syntax — implementations always live on concrete types. A concrete type satisfies an interface automatically when its method set includes everything the interface requires.**

- `IsAdult` can only be invoked on a value (or pointer to a value) of type Age. It cannot be called as a standalone function."

This mechanism is the foundation of Go's interface system: a type satisfies an interface simply by having the right set of methods attached to it.

```go
// Anonymous calling
import "fmt"
type Age int

func (a Age) IsAdult() bool{
    return a >= 18
}

func main(){
    fmt.Println(Age(20).IsAdult()) // true

    // IsAdult() // illegal
}
```

In the same fashion, we can do so with functions, here's an example:

```go
import "fmt"

func add(a, b int) int{
    return a + b
}

func subtract (a, b int) int{
    return a - b
}

type BinaryOp func(int, int) int
// The underlying 'value' of BinaryOp is just a function with two params of type 'int' and it returns an 'int'

func main(){
    f := add // give 'add' function another label, functions are treated as values in go

    fmt.Println(f(2,3)) // 5
    // The above code does the same thing as add(2,3), its just that we changed the label

    var op BinaryOp = add // matches the underlying signature

    fmt.Println(op(2,3)) // 5

    op = subtract // relabel it to subtract
    fmt.Println(op(2, 3)) // -1
    fmt.Println(op(5, 2)) // 3
}
```

Now lets attach a method to a function(yes, a method(function) to another function!):

```go
import "fmt"
type BinaryOp func(int, int) int

func add(a, b int) int{
    return a + b
}

func subtract(a, b int) int{
    return a - b
}

func (op BinaryOp) Describe() string{
    return "I am a binary operation!"
}

func main(){
    var op BinaryOp = add
    fmt.Println(op.Describe()) // "I am a binary operation!"
}

```

Now, let's go one step further and **call the receiver itself inside the method**, as its a function too!

```go
import "fmt"

type BinaryOp func(int, int) int

func add(a, b int) int{
    return a + b
}

func subtract(a, b int) int{
    return a - b
}

func (op BinaryOp) Describe() string{
    return "I am a binary operation!"
}

func (op BinaryOp) CallTwice(a, b int) string{
    first := op(a, b) // calling the receiver itself as its a function!
    second := op(first, b)
    return second
}


func main(){
    var op BinaryOp = add
    fmt.Println(op.CallTwice(2,3)) // 8
    op = subtract
    fmt.Println(op.CallTwice(2, 3)) // -4
}

```

## Final Code

```go
package main

import "fmt"

type Age int

func (a Age) IsAdult() bool {
	return a >= 18
}

func add(a int, b int) int {
	return a + b
}

func subtract(a, b int) int {
	return a - b
}

type BinaryOp func(int, int) int

func (op BinaryOp) Describe() string {
	return "Hey there, I am a binary operation!"
}

func (op BinaryOp) CallTwice(a, b int) int {
	first := op(a, b)      // called once
	second := op(first, b) // called once
	return second
}

func main() {
	var age Age = 23
	fmt.Println(age.IsAdult())

	var op BinaryOp = add
	fmt.Println(op(2, 3))

	// relabel op
	op = subtract
	fmt.Println(op(10, 3))

	fmt.Println(op.Describe())

	fmt.Println(op.CallTwice(2, 3)) // ans should be -4 because we relabeled op as subtract

	op = add
	fmt.Println(op.CallTwice(2, 3)) // ans should be 8
}

```

## Interfaces

- They are contracts enforced over a _type_ that a set of particular function(s) must be defined over it.

- You cannot define methods on interface types using receiver syntax. Methods must be defined on concrete types — structs, named types over primitives, function types, slices, maps, etc.

Q. Why this restriction exists?

- An interface is a contract — it describes what methods a type must have, but it doesn't have its own data or implementation. Asking "what does the interface itself do?" is a category error. The interface is just a label; the implementations live on concrete types.

```go
import "fmt"

type BinaryOpStore interface{
    Describe(string) string
    CallTwice(a, b int) int
}

type BinaryOp func(int, int) int

func add(a, b int) int {
    return a + b
}

func subtract(a, b int) int {
    return a - b
}

func (op BinaryOp) Describe(operation string) string {
    if operation == "addition"{
        return fmt.Sprintf("I am an %s operation", operation)
    }else{
        return fmt.Sprintf("I am a %s operation", operation)
    }
}

func (op BinaryOp) CallTwice(a, b int) int{
    first := op(a, b)
    second := op(first, b)
    return second
}

func GiveDescriptionAndCallTwiceResult(op BinaryOpStore, operation string, a, b int) (string, int) {
    return op.Describe(operation), op.CallTwice(a, b)
}

func main(){
    var op BinaryOp = add
    GiveDescriptionAndCallTwiceResult(op, "addition", 2, 3) // ("I am an addition operation", 8)
}
```

- Note: Go's idiomatic style is usually "accept interfaces, return structs" — return concrete types when you can, accept interface types in parameters. But returning an interface is fine when you genuinely want to hide the implementation.

### Use cases of interfaces

1. As `any`

```go
func someFn(i interface{}) any{
    return i
}

// `interface{}` is same as `any`

// Since the `interface{}` doesn't have any methods so every `type` satisfies it!

// Loss of type checking though
```

2. Type checking via `switch`

```go
func checkType(i interface{}){
    switch v:= i.(type){
        case int: fmt.Println("int"),
        case float64: fmt.Println("float64")
    }
}
```

- Note that `.(type)` is only allowed inside `switch` statements and nowhere else.
- In GoLang to assert types, we use the `.(actual type)` syntax, for example:

```go
var i any = 10
j := i.(int) // if the `type` matches - great!

// if it doesn't then the program panics
```

- To avoid `panics` we can use `val, ok` syntax:

```go
var i interface{} = "hello"
f, ok := i.(float64)

if !ok{
    fmt.Println("Type assertion failed")
}

// Now the program won't panic and 'f' will be the default 0 value according to the asserted type
```

- For type conversions, we can do the following:

```go
    var f float64 = 420.69
    var i int32 = int32(420.69)

    // We use the Type(value) syntax for this.
```

#### What You Can Convert

- Numeric to Numeric: Between any integers and floats (e.g., int32 to float64).
- String to Byte/Rune Slice: Converting string to []byte or []rune, and vice versa.- Identical Underlying Types: Between a named custom type and its literal type (e.g., converting type Duration int64 back to int64).
- Pointers with Same Base: Pointers to types that share identical underlying structures.

#### What You Cannot Convert

- Boolean to Integer: You cannot convert bool to int (e.g., int(true) fails).
- Incompatible Slices: You cannot convert []int directly to []float64.
- Structs with Different Fields: You cannot convert between two structurally different structs.

3. Accept an interface and do some work

```go
type GasEngine struct{
    KilometerPerLitre float64
    FuelCapacityInLitres float64
}
type ElectricEngine struct{
    KilometerPerKwh float64
    BatteryCapacityInKwh float64
}


func  (e GasEngine) Mileage() float64 {
    return e.KilometerPerLitre * e.FuelCapacityInLitres
}

func  (e ElectricEngine) Mileage() float64 {
    return e.KilometerPerKwh * e.BatteryCapacityInKwh
}

type Engine interface{
    Mileage() float64 // any type with this exact method signature satisfies Engine
}

func GiveMileageInfo(e Engine) float64{
    return e.Mileage()
}

func main(){
    gas := GasEngine{KilometerPerLitre: 15, FuelCapacityInLitres: 40}

    electric := ElectricEngine{KilometerPerKwh: 6, BatteryCapacityInKwh: 75}

    fmt.Println(GiveMileageInfo(electric))
    fmt.Println(GiveMileageInfo(gas))
}
```

4. Polymorphism

```go
type Shape interface{
    Area() float64
}

type Rectangle struct{
    Length float64
    Width float64
}

func (r Rectangle) Area() float64{
    return r.Length * r.Width
}

type Square struct{
    Side float64
}

func (s Square) Area() float64{
    return s.Side * s.Side
}

type Circle struct{
    Radius float64
}

const pi float64 = 3.14159

func (c Circle) Area() float64{
    return pi * c.Radius * c.Radius
}

func main(){
    shapes := []Shape{
        Circle{Radius: 2.34},
        Square{Side: 21},
        Rectangle{Length: 13, Width: 22}
    }
}
```

Q. How does the storage of interfaces make sense?

- It is actually a 16-byte space in memory with 2 parts:
  1. type pointer (8 bytes)
  2. value pointer (8 bytes)

- An interface value is a uniform 16-byte handle, regardless of where it lives. The concrete data sits elsewhere; the interface header carries two pointers — one to "what type is this?" and one to "where's the data?". This uniformity is what lets interfaces be stored in slices, maps, channels, struct fields, parameters, returns, anywhere.

#### All the places interface values get stored

1. As a plain variable
   ```go
   var s Shape = Circle{Radius: 2.34}
   ```
   `s` itself is 16 bytes on the stack (or heap, depending on escape analysis). Those 16 bytes contain:

```mermaid
┌──────────────────────┐
│ type pointer │ ← 1 word (8 bytes on 64-bit)
├──────────────────────┤
│ value pointer/data │ ← 1 word (8 bytes on 64-bit)
└──────────────────────┘
Total: 16 bytes
```

The Circle data lives somewhere else (often the heap because the interface needed to "box" it).

2. As a function parameter

```go
func GiveMileageInfo(e Engine) float64 {
    return e.Mileage()
}
```

When you call `GiveMileageInfo(gas)`, Go creates a 16-byte interface value e on the stack (or wherever the call frame lives), with its two pointers set up. Same shape, different location.

3. As a function return value

```go
func MakeShape() Shape {
    return Circle{Radius: 5}
}
```

The returned interface value is 16 bytes containing the type and data pointers. The caller receives that uniform header.

4. As a struct field

```go
type Drawing struct {
    Name string
    Shape Shape // interface field
}

d := Drawing{Name: "doodle", Shape: Circle{Radius: 1}}
```

The Shape field inside Drawing is 16 bytes. The Drawing struct itself includes that 16-byte slot.

5. As a map value (or key)

```go
shapesByName := map[string]Shape{
    "circle": Circle{Radius: 1},
    "rectangle": Rectangle{Length: 2, Width: 3},
}
```

Each map value slot holds an interface value (two pointers). Same uniform shape lets the map hold mixed concrete types.

6. As a channel element

```go
    ch := make(chan Shape, 10)
    ch <- Circle{Radius: 5}
    ch <- Square{Side: 3}
```

Channels store interface values too. Send a Circle, then a Square — each lands as a 16-byte header in the channel buffer.

7. Inside another interface

```go
var x any = Shape(Circle{Radius: 1})
```

You can wrap interfaces in interfaces. Same two-word layout.

Q. So when does the concrete type live "inline"?

- Only when you use the concrete type directly, without an interface wrapper:

```go
var c Circle = Circle{Radius: 5} // c is just a float64 — 8 bytes, inline
var s Shape = Circle{Radius: 5} // s is 16 bytes of interface header
// + Circle data stored elsewhere
```

| Variable                 | Storage                                |
| ------------------------ | -------------------------------------- |
| Circle{}                 | Inline, exactly the size of its fields |
| Shape holding a Circle{} | 16-byte header + Circle data elsewhere |

- Note: The "value pointer" half of an interface isn't always a pointer. For very small values that fit into a single word (≤ 8 bytes on 64-bit), the runtime can store the value directly in that slot — no separate allocation needed.

```mermaid
┌──────────────────────┐
│ type pointer ────────┼──→ static type descriptor (one per type, lives forever). It's not heap-allocated per use; it's static data baked into the program.
├──────────────────────┤
│ value pointer ───────┼──→ usually heap, sometimes stack
└──────────────────────┘
```

| Component                                      | Where does it live?                                                            |
| ---------------------------------------------- | ------------------------------------------------------------------------------ |
| Type pointer                                   | Static, shared globally per type. Not GC-managed.                              |
| Value pointer's target (the concrete data)     | Heap if escape analysis says so (usually). Stack in some optimized cases.      |
| The interface header itself (the two pointers) | Wherever the interface variable lives — stack, heap, slice backing array, etc. |
