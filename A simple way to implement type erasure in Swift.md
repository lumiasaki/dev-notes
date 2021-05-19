## Why Swift needs type erasure

Swift supports very powerful generic types that can help us write more extensible code and also make it concise.

If you want to use a generic type in a protocol, you can use `associatedtype` to declare a generic type that can later be used in a protocol, for example in a Redux framework, its `Component` type might be declared as follow:

```swift

protocol Component {

       // ...

       associatedtype StateType
       associatedtype PropType

       func mapStateToProps(_ state: StateType) -> PropType

       // ...
}

```

Where `StateType` and `PropType` are the generic types. If you want to make a type conform to the `Component` protocol, you need to specify exactly what the corresponding type is, because of the presence of `associatedtype`.

```swift

struct State { }
struct Prop { }

class ComponentA: Component {

       typealias StateType = State
       typealias PropType = Prop

       func mapStateToProps(_ state: State) -> Prop {
              // ...
       }

       // ...
}

```

As you can see, after specifying exactly what `StateType` and `PropType` are, the compiler automatically derives `State` and `Prop` for us when declaring `mapStateToProps(_:)`, which is very helpful for expanding the use of the protocol.

But a significant side effect is that when our protocol contains `associatedtype` or `Self`, the protocol will no longer be able to be used as a type declaration, for example the following code will give an error, while in a protocol without `associatedtype` or `Self`, such it is legal to use:

```swift

// error
var components: [Component]

// error
var components: [Component<State, Prop>]

```

The reason for this is that when a protocol becomes a protocol with some generic types, the compiler cannot deduce exactly what type it is without a clue, so it has to give an error.

> Strictly speaking, protocols cannot be used as a concrete type in Swift, they can only be used to constrain generic parameters. When we use a protocol as a concrete type, the compiler creates a wrapper type for the protocol, called existential, and in Swift 5, existential is only for protocols that do not have associated types and Self constraints.

## How to implement type erasure for custom types

The `Foundation` framework provides some type erasure types to help solving the problem we encountered above, such as `AnyHashable`, `AnySequence`, etc. These types hide the type that adheres to the corresponding protocol so that it can be declared using that type.

We can also simply implement a type erasure type to wrap our custom types.

### Step 1

The first step in implementing our own type erasure type is to declare these `associatedtypes' as generic types in the class declaration.

```swift

protocol Component {

       associatedtype StateType
       associatedtype PropType

       func mapStateToProps(_ state: StateType) -> PropType
}

```

Both `StateType` and `PropType` in the above code are `associatedtype`, and we need to extract them when we implement our own type erasure class and adhere to that hidden protocol:

```swift

class AnyComponent<StateType, PropType>: Component {

       // ...
}

```

### Step 2

After the first step, we are halfway there, the next step is to make `AnyComponent` conform to the `Component` protocol, and this requires implementing the methods declared in the protocol:

```swift

class AnyComponent<StateType, PropType>: Component {

       func mapStateToProps(_ state: StateType) -> PropType {
              // ...
       }
}

```

Since `AnyComponent` is a wrapper class for hiding specific type details, we need to pass the method call back to the corresponding instance, which can be done by initializing `AnyComponent` with codes as following:

```swift

class AnyComponent<StateType, PropType>: Component {

       private let mapStateToPropsBlock: (_ state: StateType) -> PropType

       init<C: Component>(_ wrapped: C) where C.StateType == StateType {
              self.mapStateToPropsBlock = wrapped.mapStateToProps(_:)
       }

       func mapStateToProps(_ state: StateType) -> PropType {
              // ...
       }
}

```

Since `closure` is a first-class citizen in Swift, it is feasible to assign the method of the `wrapped` instance to `mapStateToPropsBlock` in our initialization method.

Note that in the `init` method, we declare `C`, which conforms to the protocol of `Component`, and we use `where` to qualify `C.StateType == StateType`, which ensures that `wrapped` is of the same type as that expressed in `AnyComponent`.

### Step 3

Soon we come to the last step, where we just need to conduct the methods of the `Component` implementation of `AnyComponent` back to the implementation of the hidden object `wrapped`.

```swift

class AnyComponent<StateType, PropType>: Component {

       private let mapStateToPropsBlock: (_ state: StateType) -> PropType

       init<C: Component>(_ wrapped: C) where C.StateType == StateType {
              self.mapStateToPropsBlock = wrapped.mapStateToProps(_:)
       }

       func mapStateToProps(_ state: StateType) -> PropType {
              mapStateToPropsBlock(state)
       }
}

```

## Summary

A simple implementation of Type Erasure typically requires, in general, three steps.

1. Extract the type declared by the `associatedtype` in the protocol, create the `AnyXXX` type class, declare the extracted type as a generic types in the declaration of `AnyXXX`, and adhere to the corresponding protocol
2. Internally store the `closure`s references of the methods of the corresponding protocol, and initialize these `closure`s references with the hidden instance `wrapped` in the initialization method.
3. Implement the methods of the corresponding protocol in the `AnyXXX` type and call the methods of the `closure`s held by itself inside the methods (these methods come from the `wrapped` instance).
