
# Prepping for a talk. Please DO NOT share...
# Immutability: Perks and Quirks
If you have a function `createThumbnail` that takes a single argument `path`, one may ask, “what does this function do?”

```Swift 
func createThumbnail(path: UIBezierPath) -> UIImage {
    // ...
}
```

In a world with shared reference types, during the execution of this function, another function, say `apply(scale: CGFloat,to path: UIBezierPath)` with access to the same path, in another part of the program, possibly on another thread can change the value of `path`,  making it nearly impossible to know what function `createThumbnail` does without having intimate knowledge of function `apply(scale: CGFloat`.

```Swift
func apply(scale: CGFloat to path: UIBezierPath) {
  /// Is this really your path?
  path.apply(CGAffineTransform(scaleX: 0.25, y: 0.25))
}
```

If we changed the function signature so that it takes in a `CGPath`, an immutable object, reasoning about this function is trivial since the outcome only depends on the operations in function `createThumbnail` and its single argument `path`.

```Swift
func createThumbnail(path: CGPath) -> UIImage {
    // ...
}
```

Immutability lends itself to simplicity and clarity since implicit in an object’s creation is a guarantee that it will never change. 

```Swift
struct RGBA {
    let red: Int
    let green: Int
    let blue: Int
    let alpha: CGFloat
}
let orange = RGBA(red: 240, green: 83, blue: 5, alpha: 0.91)
orange.red = 150 /// Not allowed
```

Share mutable objects make it possible for objects to be modified by their references, making it difficult to understand,  maintain, or make changes to your code.

```Swift
class Person {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

let jazbo = Person(name: "Jazbo", age: 70)
let jonTait = jazbo // A reference to jazbo 
```

However, mutable objects or references are a critical part of OOP and are suited for situations where two parts of a program need to communicate conveniently by sharing a common mutable data structure. 

Let’s dig little a deeper and and I will share a demo about how choosing a value type lead to the most perplexing bug I have ever encountered.

## Immutability in Swift
Swift espouses safety which is reflected in the fact that over 90% of the types in its standard library are value types.

1. Collections
2. Structs
3. Enums

To achieve immutability in Swift, we lean on value types. Initializing, assigning, or passing a value type leads to the creation of an  independent copy,  making things significantly easier to reason about. 

### Pure Value Types

```Swift
enum TransmissionType {
    case standard
    case automatic
}

struct Transmission {
    let type: TransmissionType
}

struct Car {
    let color: RGBA
    let transmission: Transmission
    
    // ....
}

let orange = RGBA(red: 240, green: 83, blue: 5, alpha: 0.91)

let red = RGBA(red: 194, green: 0, blue: 0, alpha: 1)
let car = Car(color: orange, engine: Engine(type: .standard))

car.color = red // Not allowed
car.engine.type = .automatic // Not allowed
```

As shown here, not only are you forbidden from re-assigning an object’s fields, you are also forbidden from mutating them.

### Mixing Types
Mixing value types with reference types can lead to some unexpected behavior. You are not allowed to re-assign an object’s fields, but you can mutate them.

``` Swift
struct SvgPathElement {
    let path: UIBezierPath
}

let group = SvgPathElement(path: UIBezierPath())
let groupTwo = group

group.path = UIBezierPath() // Not allowed
let timbuk: CGFloat = 2.0
let tu: CGFloat = 4.0 
group.path.addLine(to: CGPoint(x: timbuk, y: tu)) // Allowed

groupTwo.path
// show new path
```

## Immutability: Perks
There are many reasons immutable ojects are preferrable. Here we will explore two:
1. No Referencing
2. Safer and easier to understand.

### Referencing: The Root of All Evil
Backing OOP is the idea of encapsulating abstract data in logical containers referred to as objects. They have state and behavior and an interface.  But single objects are not interesting. For an object to be useful, they have to be part of a system of objects that interact with each other’s interface. A system of objects is not necessarily encapsulated. In Swift, with reference types, we allow multiple pointers to point the same memory location or object. Multiple paths exist by which they can be accessed:

1. `self`
2. A global variable accessible from a function.
3. A parameter passed in to a function.
4. A parameter returned from a function.
5. A local variable in a function bound to anyone of the above. 

It can be very difficult to fully reason about programs with references since since we would need to survey the whole system at run-time to understand the implications of state changes.

#### Reference Types and Roles
If you have objects that are referenced, it means that they can play different roles. A problem occurs when those roles conflict. Matrices are used in a variety of APIs including `CGAffineTransform`. A matrix looks like this. 

_(Add diagram for matrix)_

Below, we have a class that represents a matrix. Matrices can be represented using nested arrays, each of which represents a row.

```Swift 
class Matrix<T>  {
    var backing: [[T]]
    
    init(backing: [[T]]) {
        self.backing = backing
    }
}
```

Here is a function that multiplies a matrix.

```Swift 
func multiply(a: Matrix<Int>, b: Matrix<Int>, into c: Matrix<Int>) {
    ///...
}
```

Multiply as defined above expects `a` to be a source and `c` to be a destination but `a` is playing both source and destination. This is clearly dangerous because as the operation goes on, we are modifying `a` with the results of `a * b`.

```Swift
let a = Matrix<Int>(backing: [[1, 8], [4, 5]])
let b = Matrix<Int>(backing: [[3, 9], [2, 6]])
multiply(a: a, b: b, into: a)
```

Some clever engineer might point out that we should change `multiply` to be `multiply(a: Matrix<Int>, b: Matrix<Int>) -> Matrix<Int>`. That is, create a an instance of `Matrix` inside the function and return that. This treating a symptom and not the problem because `a` and `b` are liable to be modified before or during the `multiply` operation. The problem is the function signature doesn't make it clear that the parameters can be mutated, which can lead to the caller and the function having different notions of the world.

_(Show a being populated with the results of `a*b`)_

#### Reference Types and Subclass Polymorphism
Sub-classing in OOP is common, if not simple. But this commonplace operation can make problems from referencing even more difficult to reason about and notice.

```Swift
class Person {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

class Tutor: Person {
    override init(name: String, age: Int) {
        super.init(name: name, age: age)
    }
}

class Department {
    func setTutorFor(person: Person, tutor: Tutor) {
       /// ...
    }
}
```

Here we have `Person` as a class and `Tutor` which is sub-class of `Person`. We also have department as a class that can set up a tutor for a person attending this institution.

```Swift
let t = Tutor(name: "Tee", age: 45)
department.setTutorFor(person: t, tutor: t)
```

Calling the function above is totally fine and we will have a tutor, tutoring themselves. No greater construct is required to make this happen than reference types. This is impossible with value types since they don't allow subclassing.

## Safer and easier to understand and change

Knowledge that an object’s properties can only be set by initializing makes these programs safer,  easier to document, reason about about, or change. In order for you to fully understand how an object affects your programs, you would have to trace through the entire program to find its points of reference, since every point of reference is potentially a point of mutation.

### Passing Reference Types

```Swift
class Stack<T: Comparable> {
    var list = [T]()
    
    var count: Int {
        return list.count
    }

    /// Add a new object to the stack.
    ///
    /// - Parameter value: Object to be added.
    func push(value: T) {
        list.append(value)
    }
    
    /// Remove and return last object added. Nil will be returned if
    /// if the list is empty.
    ///
    /// - Returns: Last objected added if there is one.
    func pop() -> T? {
        return list.popLast()
    }
}
```

A stack is a first in, last out data structure, something you could get from stacking books. Push adds a book to the top of the stack and pop takes the top book off of the stack.

```Swift
let homePricesStack = Stack<Int>()
homePricesStack.push(value: 760000)
homePricesStack.push(value: 850000)
homePricesStack.push(value: 575000)
```

Here we have a stack tracking prices for homes sold in Seattle, 3 so far. We needed to be able to find the median price of the homes sold so far so we created `findMedian(_:)` to do just that. 

```Swift
func findMedian(stack: Stack<Int>) -> Int {
    stack.list.sort()

    let index = (stack.count - 1) / 2
    if stack.count % 2 == 0 {
        return (stack.list[index] + stack.list[index + 1]) / 2
    }

    return stack.list[index]
}
```

We call median and we get the expected result. So far, so good. But now, we need to get the last value added to the stack. Is there a conflict here between what should `pop` return and what it will pop return? The answers might surprise you.

```swift
findMedian(stack: homePricesStack) // 760000
homePricesStack.pop() // 850000, should be 575000
```

Passing around mutable objects can lead to latent bugs (It is an existing error that has yet not caused a failure because the exact condition was never fulfilled).

Is it easy to understand what’s going on here?  Is this safe?

This is not the case with immutable objects.

### Returning Reference Types
Let’s say you are displaying elements from an svg file, something like this, a simple triangle.

```xml
<?xml version="1.0" encoding="utf-8"?>
<svg version="1.1">
	<path d="M5,5h260L135,230"/>
</svg>
```

The `d` attribute is a string that contains information about how a path should be drawn. So we wrote a function to generate a UIBezierPath from this element and a function to get the path. 

```Swift
/// Converting the dpath from an svg file into a bezier path.
///
/// - Parameter dpath: dpath from svg file.
/// - Returns: Bezier path.
func convertToBezierPath(dpath: String) -> UIBezierPath {
    let path = UIBezierPath()
    
    /// ....
    
    return path
}

func elementBezier() -> UIBezierPath {
    return convertToBezierPath(dpath: "M150 0 L75 200 L225 200 Z")
}
```

Then,  two critical things happened.

First, we determined that `convertToBezierPath` is an expensive operation, so we cached the result the first time the method is called.

```Swift 
/// Accessible to functions in class
var path = UIBezierPath()
func elementBezier() -> UIBezierPath {
    if path.isEmpty {
        path = convertToBezierPath(dpath: "M150 0 L75 200 L225 200 Z")
    }
    
    return path
}
```
This seems helpful and appears innocuous. 

Second, we decided to get a generate a thumbnail to show the all or elements in a collection view. The engineering implementing this decided that the path was too big for a thumbnail and scaled it down to their desired size.

```Swift 
func thumbnailGeneration() {
    let path = elementBezier()
    path.apply(CGAffineTransform(scaleX: 0.25, y: 0.25))
    
    /// ...
}
```

What's the result here? Who will find this bug first? Would it be `pathForThumbail` or someone who innocently calls `elementBezier`? Were we able to make changes without rewriting portions of the code?

Immutability facilitates code changes since once an object is created, it can’t be changed, so any code that depends on an immutable object can be _trusted_ and won’t need revision when a code change is made.

### Why do Roles Conflict?
In some cases, errors references occurs because of insufficient analysis leading to roles conflicting. But, I think they probably occur more because the design and implementation of programs fails to take the possibility of referencing into consideration, or you disallow aliasing without making it clear to the clients using your API. **Either way, objects are declared with variable names describing their role but are manipulated based on their identities**.

Therefore, in object oriented environments, referencing is always present and as long as we are using mutable objects, it is going to cause problems. If you remove the possibility of referencing, all of this goes away.

### So, should we default to value types?
No.

Here is why. 

## Demo
_(Add demo notes)_

Object oriented design allows us to be incredibly expressive. We can build incredibly powerful systems by allowing objects to interact freely. However we need to balance this expressiveness with control. Architectural constraints like immutability help to bring simplicity to our systems, making them safer, receptive to change, and welcoming to new engineers, and even our future selves. Value types can be mutated but independent copies are created when passing them around. If someones changes their copy, those changes won't be reflected in yours. 

When you are designing a system, you should think about the semantics that you necessary to make things work seamlessly. If a reference type is needed, don’t fight the system or the environment, but be careful to provide as much reference advertisement and control as possible.

