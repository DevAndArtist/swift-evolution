# Any-class and Class requirement for `any<...>`

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): [Adrian Zubarev](https://github.com/DevAndArtist), [Austin Zheng](https://github.com/austinzheng)
* Status: [Awaiting review](#rationale)
* Review manager: TBD

## Introduction

This is a follow up proposal to [SE-0095](https://github.com/apple/swift-evolution/blob/master/proposals/0095-any-as-existential.md) which enhances its capabilities and solves the [issue of a missing type](https://openradar.appspot.com/20990743). The functionality of this enhancement would allow developers to combine classes with protocols without the need of generics.

Swift-evolution thread: [\[Pitch\] Merge `Types` and `Protocols` back together with `type<Type, Protocol, ...>`](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160502/016523.html)

## Important note

This proposal should be considered for Swift 3 if the core team desides that it is *relatively easy* to implement due the short timeframe left. 

## Motivation

Types involving classes with additional requirements can be expressed in Objective-C like this:

```Objective-C
// Objective-C
// A UIView or subclass thereof, which conforms `ProtocolA` and `ProtocolB`
@property (nonatomic, strong) UIView<ProtocolA, ProtocolB> *someView;
```

However, this sort of type is impossible to express in Swift without generic scope. This proposal aims to solve this [problem](https://openradar.appspot.com/20990743) and creates a new `Type` as part of `any<...>` existential. Additionally the proposed solution will allow to express *any-class requirement* which could be used as a constraint in library design and replace `protocol A: class { ... }`.

## Proposed solution

* **Any-class requirement:** This must be the first requirement, if present. This requirement consists of the keyword `class`, and requires the existential to be of any class type.

* **Class requirement:** This must be the first requirement, if present. This requirement consists of the `name` of a class type, and requires the existential to be of a specific class or its subclasses. There can be only one class name constraint, and it is mutually exclusive with the any-class requirement.

* **Nested `any<...>`:** This requirement consists of another `any<...>` construct.

Example:

```swift
protocol SomeProtocol {}
extension SomeProtocol {
	func foo() { print("fooooo") }
}
extension UIButton: SomeProtocol {}

class A {
	
	// Class requirement:
	var view: any<UIView, SomeProtocol>
	
	// Class requirement:
	init(view: any<UIView, SomeProtocol>) {
		self.view = view
	}
	
	// `dynamicType` of `view` might be `UIButton` for example and we will
	//  still be able to acces `SomeProtocol` which only `UIButton` conforms.
	func doSomeWork() {
		self.view.removeFromSuperview() 
		self.view.foo()
	}
}

// Any-class requirement: can be any class type that conforms to `SomeProtocol`.
let button: any<class, SomeProtocol> = UIButton()

typealias AnyClass = any<class>

// With new `AnyClass` typealias in mind the example can be rewritten:
let button: any<AnyClass, SomeProtocol> = UIButton()

// Class requirement:
if let mergedValue = button as? any<UIView, SomeProtocol> {
	let a = A(view: mergedValue)
	a.doSomeWork()
}
```

## Detailed design

### Usage

An `any<...>` existential type can be trivially used in any of the following situations:

* Local variable type
* Property type
* Function parameter
* Function return value
* Dynamic casting (the object of `as?`, an `as` case, `as!`, etc)

Existentials cannot be used as part of the `superclass`/`protocol` conformance portion of a type declaration, except if `any<...>` is constructed of protocols only or combined with *any-class requirement*:

* Valid examples: 
	
	```swift
	// Any-class requirement:
	typealias AnyClass = any<class>
	
	protocol A: AnyClass {} /* equivalent today */ protocol A: class {}
	
	// We should be able to apply `any<...>` to a protocol directly.
	// Today it is only possible to apply a typealias of `protocol<X, Y>` to another protocol.
	protocol B: any<class, SomeProtocol> {}
	protocol B: any<AnyClass, SomeProtocol> {}
	
	class C: any<AnyClass, SomeProtocol> {}
	```
	
* Not valid examples:

	```swift
	// Redundant: one may use just `UIView` instead.
	protocol A: any<UIView> {}
	
	// Protocol `B` could only be applied to `UIView` or its subclasses which conform to `SomeProtocol`.
	// We could allow this someday, but not with this proposal!!!
	protocol B: any<UIView, SomeProtocol> {}
	
	// Equivalent version:
	protocol A {}
	extension A where Self == UIView {}
	
	protocol B {}
	extension B where Self == any<UIView, SomeProtocol> {}
	```
		
### Nested `any<...>`	

A nested `any<...>` may declare a class or any-class constraint, even if its parent `any<...>` contains a class or any-class constraint, or a sibling `any<...>` constraint declares a class or any-class constraint. However, one of the classes must be a subclass of every other class declared within such constraints, or all the classes must be the same. This class, if it exists, is chosen as the `any<...>` construct's constraint.

```swift
// Can be any type that is a UITableView conforming to ProtocolA.
// UITableView is the most specific class, and it is a subclass of the other
// two classes.
let a : any<UIScrollView, any<UITableView, any<UIView, ProtocolA>>>

// REDUNDANT: could be banned by another proposal.
// Identical to any<ProtocolA, ProtocolB>
let b : any<any<ProtocolA, ProtocolB>>

// NOT ALLOWED: no class is a subclass of all the other classes. This cannot be
// satisfied by any type.
let c : any<NSObject, any<UIView, any<NSData, ProtocolA>>>
```

Using typealiases, it is possible for `any<...>` existentials to be composed. This allows API requirements to be expressed in a more natural, modular way:

```swift
// Any custom UIViewController that serves as a delegate for a table view.
typealias CustomTableViewController = any<UIViewController, UITableViewDataSource, UITableViewDelegate>

// Pull in the previous typealias and add more refined constraints.
typealias PiedPiperResultsViewController = any<CustomTableViewController, PiedPiperListViewController, PiedPiperDecompressorDelegate>
```

Any `any<...>` containing nested `any<...>`s can be conceptually 'flattened' and written as an equivalent `any<...>` containing no nested `any<...>` requirements. The two representations are exactly interchangeable.

## Impact on existing code

These changes will break existing code. Projects using old style `protocol<...>` mechanism will need to migrate to the new `any<...>` mechanism. The code using old style `protocol<...>` won't compile until updated to the new conventions.

## Alternatives considered

* Defer this enhancement to a future version of Swift or find a better way to solve the mentioned problem.

## Future directions

* `any<...>` should reflect powerful generalized generic features to be able to constrain types even further (e.g. `where` clause).

* Adding constraints for other extendable types like `struct` and `enum` and generalize these. This will allow us to form more typealiases like:

	```swift
	typealias AnyStruct = any<struct>
	typealias AnyEnum = any<enum>
	```
			
	* Example:
	
	```swift
	protocol A { func boo() }

	// Structs that conforms to a specific protocol.
	func foo(value: any<struct, SomeProtocol>) 
	func foo(value: any<AnyStruct, SomeProtocol>) 
	
	// Enums that conforms to a specific protocol.
	func foo(value: any<enum, SomeProtocol>) 
	func foo(value: any<AnyEnum, SomeProtocol>) 
	
	// Lets say this array is filled with structs, classes and enums,
	// which all conforms to `SomeProtocol`.
	let array: [SomeProtocol] = // fill

	// Filter these types:
	var classArray: [any<class, SomeProtocol>] = array.flatMap { $0 as? any<class, SomeProtocol> }
	var structArray: [any<struct, SomeProtocol>] = array.flatMap { $0 as? any<struct, SomeProtocol> }
	var enumArray: [any<enum, SomeProtocol>] = array.flatMap { $0 as? any<enum, SomeProtocol> }
	```


Flattened operators or even `Type operators` for `any<...>`:

```swift
class B {
	var mainView: UIView & SomeProtocol
	
	init(view: UIView & SomeProtocol) {
		self. mainView = view
	}
}
```

> The `&` type operator would produce a “flattened" `any<>` with its operands.  It could be overloaded to accept either a concrete type or a protocol on the lhs and would produce `Type` for an lhs that is a type and `any<...>` when lhs is a protocol. `Type operators` would be evaluated during compile time and would produce a type that is used where the expression was present in the code.  This is a long-term idea, not something that needs to be considered right now. 
> 
> Written by: [Matthew Johnson](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160509/017499.html)

Adding `one<...>` which can reduce overloading (idea provided by: [Thorsten Seitz](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160509/017397.html)). `one<...>` will pick the first type match from within the angle brackets to the dynamic type at compile time and proceed. One would then need to handle the value by own desire.

```swift
func foo(value: one<String, Int>) {
	
	if let v = value as? String {
		// Do some work
	} else if let v = value as? Int {
		// Do some work
	}
}
	
// Flattened version:
func foo(value: String | Int)
```

Mix `any<...>` and `one<...>`:

```swift
// Accept only types constrained by 
// (`ClassA` AND `SomeProtocol`) OR (`ClassB` AND `SomeProtocol`)
func foo(value: any<one<ClassA, ClassB>, SomeProtocol>)

// Flattened version:
func foo(value: (ClassA | ClassB) & SomeProtocol)
```

Typealias `AnyValue` (for extendable types only):

```swift
// Isn't this like magic
typealias AnyValue = one<any<struct>, any<enum>> 
typealias AnyValue = one<AnyStruct, AnyEnum>
typealias AnyValue = any<struct> | any<enum>
typealias AnyValue = AnyStruct | AnyEnum

// Any value which conforms to `SomeProtocol`.
// Reference types are finally out the way.
func foo(value: any<AnyValue, SomeProtocol>) 

// Flattened version:
func foo(value: AnyValue & SomeProtocol)
```

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to (TBD) this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.
