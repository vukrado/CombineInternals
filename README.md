# Combine Internals

By Vuk Radosavljevic


# Table of Contents

{{TOC}}

# Chapter 1: What is Combine?

Combine is a new framework announced at [WWDC 2019](https://developer.apple.com/wwdc19/722) written in Swift.

It is a unified declarative framework for processing values over time.

So what does that mean?

[From Apple's documentation](https://developer.apple.com/documentation/combine)

> The Combine framework provides a declarative Swift API for processing values over time. These values can represent many kinds of asynchronous events. Combine declares publishers to expose values that can change over time, and subscribers to receive those values from the publishers
> 
> ...
> 
> By adopting Combine, you’ll make your code easier to read and maintain, by centralizing your event-processing code and eliminating troublesome techniques like nested closures and convention-based callbacks.

## TODO - Add More 

# Chapter 2: Overview of Publishers & Subscribers

There are two key types in Combine:

* [Publishers](#Publisher)
* [Subscribers](#Subscriber)

*There are also `Operators` which play an important role in Combine but we won't worry about them for now as they will be covered in a later chapter. Operators are also `Publishers`.*
<div id="Publisher"></div>
## Publisher

`Publisher` is a protocol that declares a type that can deliver 
a sequence of values over time.

- A `Publisher` must declare the output type of elements it produces and the type of failure (if any) that may terminate publishing.

It is generic over two types and has one required method as shown below.

```swift
protocol Publisher {
    associatedtype Output
    associatedtype Failure: Error
    
    func receive<S: Subscriber>(subscriber: S)
}
```

`Publisher` also provides a convenience method `subscribe<S>(_ subscriber: S)` which calls through to the `receive(subscriber:)`.

```swift
extension Publisher {
    func subscribe<S: Subscriber> where Self.Failure == S.Failure, Self.Output == S.Input {
        receive(subscriber: subscriber)
    }
}
```

As you see above, there are two generic constraints to ensure that the `Publishers` `Output`& `Failure` types match the `Subscribers` `Input` & `Failure`.
<div id="Subscriber"></div>
## Subscriber

`Subscriber` is a protocol that declares a type that can receive input from a publisher. It is generic over two types: `Input` and `Failure` respectively.

```swift
protocol Subscriber: CustomCombineIdentifierConvertible {
    associatedtype Input
    associatedtype Failure: Error
}
```

So how do these work together?

![Combine Pattern Image](https://github.com/vukrado/CombineInternals/blob/master/images/Combine-Pattern.png)

1. The `Subscriber` subscribes to the `Publisher`. 
2. The `Publisher` creates a `Subscription` and sends it to the `Subscriber`.
3. The `Subscriber` requests *N* values.
4. The `Publisher` sends *N* values.
5. The `Publisher` eventually sends a completion event if it is finite.

One thing mentioned above that we have yet to cover is the concept of a `Subscription`. So what are `Subscriptions`?

## Subscription

`Subscription` is a protocol that represents the connection of a `Subscriber` to a `Publisher`.
> Cancelling a subscription must be thread-safe and you can only cancel a `Subscription` once.

A subscription is how the `Subscriber` controls the flow of data from a `Publisher` to a `Subscriber`.

```swift
protocol Subscription: Cancellable {
    func request(_ demand: Subscribers.Demand)
}
```

When using Combine you will not interact with `Subscriptions` directly, if you search the documentation the only publicly available implementation of a `Subscription` is `Subscriptions.empty` which returns a `Subscription` that ignores requests and cancellation.

Despite this, `Subscriptions` play an important role in Combine. We will see how they are used when we build our first publisher from the ground up.

# Chapter 3: Just Publisher From Scratch

Now that we understand the basic interfaces Combine provides us, let's start by building out our first `Publisher`.

## Just
Just is a simple convenience publisher that emits an output to each subscriber just once, and then finishes. It can not fail with an error and will always produce a value. Let's take a look at the public interface. 

Since it's generic over `Output` we don't need a typealias for `Publishers.Output` and since `Just` can not fail with an error it's `Failure` type is set `Never`.

```swift
struct Just<Output>: Publisher {
    public typealias Failure = Never
    
    public let output: Output
    
    public init(_ output: Output) {
        self.output = output
    }
    
    public func receive<S>(subscriber: S: Subscriber) where Output == S.Input, Self.Failure == S.Failure {
    
    }
```

We need to fill out the receive method, if we remember step 2. from the graph earlier, the `Publisher` needs to create a `Subscription` and send it to the `Subscriber`. Let's disassemble the Combine executable in Hopper and see what we can find.

## TODO: Instructions for installing Hopper & GitHub link to Combine executable

If we do a search on Just.receive we will see the generated assembly for the `receive(subscriber:)` method.

![Just Receive Method Assembly](https://github.com/vukrado/CombineInternals/blob/master/images/Just-receive-asm.png)

Inside this method we see that there is a hidden type `Just.Inner`. Its initializer takes a generic downstream and the generic `Output` as a value so let's set that up.

> If you are interested on how Hopper was able to convert this `$s7Combine4JustV5Inner33_A1E16B5C396F72DD7005B0D825898FD5LLC10downstream5valueAFyx_qd__Gqd___xtcfcTf4gnn_`
> 
> into this
> 
> `Combine.Just.(Inner in _A1E16B5C396F72DD7005B0D825898FD5).init(downstream: A1, value: A) -> Combine.Just<A>.(Inner in _A1E16B5C396F72DD7005B0D825898FD5)<A1>`
> 
> Take a look at the [Swift Name Mangling](https://github.com/apple/swift/blob/master/docs/ABI/Mangling.rst) document in the swift repo.
> 
> You can also open up your terminal and run
> `swift demangle $s7Combine4JustV5Inner33_A1E16B5C396F72DD7005B0D825898FD5LLC10downstream5valueAFyx_qd__Gqd___xtcfcTf4gnn_`
> and you should get the same output.

```swift
extension Just {
    private final class Inner<Downstream> {
        
        private let downstream: Downstream
        private let value: Output
        
        init(downstream: Downstream, value: Output) {
            self.downstream = downstream
            self.value = value
        }
    }
}
```

If we do another search but now on `Just.Inner` we can get some more information about this hidden type. We see 6 protocol conformance descriptors for `Just.Inner`

1. `Subscription`
2. `CustomStringConvertible`
3. `CustomReflectable`
4. `CustomPlaygroundDisplayConvertible`
5. `Cancellable`
6. `CustomCombineIdentifierConvertible`

Since `Subscription` inherits both the `Cancellable` and `CustomCombineIdentifierConvertible` protocol we only need to conform to `Subscription`. I will be ignoring `CustomStringConvertible`, `CustomReflectable` & `CustomPlaygroundDisplayConvertible` but feel free to implement them if you'd like. Let's conform to `Subscription` and add the required methods.

```swift
extension Just {
    private final class Inner<Downstream> {
        
        private let downstream: Downstream
        private let value: Output
        
        init(downstream: Downstream, value: Output) {
            self.downstream = downstream
            self.value = value
        }
        
        // Mark: Subscription
        func request(_ demand: Subscribers.Demand) { }
        
        func cancel() { }
    }
}
```

If you are wondering what `Subscribers.Demand` is we will cover that in a later chapter, for this chapter I have provided an implementation that will work for what we need.

## TODO - Add GitHub link for Subscribers.Demand

Now let's take a look at the `request(_ demand:)` method in Hopper.

Taking a closer look we can see that the Generic constraint on `Just.Inner` is optional. This will make sense as to why in just a second but let's change `private let downstream: Downstream` to `private var downstream: Downstream?`.

Going back to the graph of the overall flow, we see that after the Subscriber requests some demand, the Publisher should tell the subscriber that the publisher has produced an element by calling `receive(_ input:)` and since this only produces one value we will then want to tell the subscriber that the publisher has completed publishing by calling receive(completion:). Since we have completed, we will want to set the downstream to nil in case another request comes in, the Publisher will not obey it.

Putting all this together we end up with:

```swift
extension Just {
    private final class Inner<Downstream: Subscriber> {
        
        private let downstream: Downstream
        private let value: Output
        
        init(downstream: Downstream, value: Output) {
            self.downstream = downstream
            self.value = value
        }
        
        // Mark: Subscription
        func request(_ demand: Subscribers.Demand) {
            _ = downstream?.receive(value) // 1.
            downstream?.receive(completion: .finished) // 2.
            self.downstream = nil // 3.
        }
        
        func cancel() { }
    }
}
```
If you inspect the assembly a little more work goes on inside the `request(_ demand:)`. It looks like it does some unwrapping of the downstream as well as checking to ensure the initial demand was greater than 0.

The `cancel()` method is much simpler as all we need to do is set the downstream to nil to ensure that no more work will be done.

## Recap
Just is a `Publisher` that only emits one value and can never fail. It is shadowed by a class that conforms to `Subscription` `Just.Inner` which handles receiving the values, processing them, and passing them down the line.

We now have created our first publisher, but without a `Subscriber` we can't do anything with it. 


# Chapter 4. Sink from Scratch
## Sink
`Sink` is a `Subscriber` that requests an unlimited number of values upon subscription. Let's take a look at it's public interface.

```swift
final class Sink<Input, Failure: Error> : Subscriber, Cancellable {

    public let receiveCompletion: (Subscribers.Completion<Failure>) -> Void
    
    public let receiveValue: (Input) -> Void
    
    public init(receiveCompletion: @escaping ((Subscribers.Completion<Failure>) -> Void),
                receiveValue: @escaping ((Input) -> Void)) {
            self.receiveCompletion = receiveCompletion
            self.receiveValue = receiveValue
    }
    
    // Subscriber
    func receive(subscription: Subscription) { }
    
    func receive(_ input: Input) -> Subscribers.Demand { }
    
    func receive(completion: Subscribers.Completion<Failure>) { }
    
    // Cancellable
    func cancel() { }

} 
```

> `Sink` also conforms to `CustomStringConvertible`, `CustomReflectable`, & `CustomPlaygroundDisplayConvertible`. Which we will be ignore in this implementation.

`Sink` takes two escaping closures in its initializer that both return `Void`. In the `init()` we can just set the respective properties.

Next we see that it implements the 3 required methods for the `Subscriber` protocol as well as the `cancel()` to conform to `Cancellable`.

To start we can begin by implementing `receive(completion:)` since it is rather straightforward. We will want to pass in the Completion event to the `receiveCompletion` closure.

```swift
func receive(completion: Subscribers.Completion<Failure>) {
    receiveCompletion(completion)
} 
```

`receive(_ input:)` is similar to `receive(completion:)`. When we receive any `Input` we will simply want to invoke the closure, passing in the `Input`. It also needs to return some `Demand`. Since we know `Sink` requests an unlimited number of values upon subscription, we can just `return Demand.None` which will have no effect. 
> We can technically return any `Demand`. The reason being, `Demand` is additive and once a `Publisher` has unlimited demand, there can be no further negotiation of demand. Adding any `Demand` to `Demand.unlimited` results in `.unlimited`. This will be covered more in depth in the Back Pressure chapter.

```swift
public func receive(_ input: Input) -> Subscribers.Demand {
    receiveValue(input)
    return .none
}
```



Now let's take a look at `receive(subscription:)`. As stated earlier, we know that upon subscription `Sink` will request unlimited demand. So let's add that in.

```swift
public func receive(subscription: Subscription) {
    subscription.request(.unlimited)
} 
```


## Putting It Together
> Not complete

```swift
let just = Just("Hello World")
let sink = Subscribers.Sink()  

// Attaches the specified subscriber to this publisher.
just.subscribe(sink)

// Just will create a Subscription Just.Inner
just.receive(subscriber: sink) {
    let inner = Just.Inner()
    // Tells the subscriber that it has 
    // successfully subscribed to the publisher and may request items.
    sink.receive(subscription: inner)
}
```

# Chapter 5: Operators
# Chapter 6: Subjects
# Chapter 7: Back Pressure
# Resources
[Cocoa with Love](https://www.cocoawithlove.com/blog/twenty-two-short-tests-of-combine-part-1.html)

[Apple Doc On Back Pressure](https://developer.apple.com/documentation/combine/processing_published_elements_with_subscribers)

# Questions
`Subscription` protocol says its class constrained but it is not? 
