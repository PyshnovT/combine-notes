## combine-notes

My notes I wrote down while playing with Combine.

### Basics

Ok, as I understand, Combine attempts to structure async code on a library level. Instead of setting up delegate, calling different functions all over the class, simply ''subscribe''
to some value, shape it as you wish and save it straight in a model/UI. All in one chain of code.

#### What Publisher really is?

ok, Publisher is simply a generic protocol. I guess, anything can be a publisher, that's powerful.

    protocol Publisher<Output, Failure> {
        associatedtype Output
        associatedtype Failure: Error

        func receive<S>(subscriber: S) where S: Subscriber...
    }

There are a ton of extensions on this protocol, but I will leave it for now.

### Thoughts

- What I like very much is to stop a subscription you have to free a link to subscription (AnyCancellable) and it simply cancels on deinit. Wow. 
This way when a model deinits and frees all of it stores properties, all subscriptions cancel automatically, you don't do anything.
Side effect, if your subscription doesn't work, it probably because you did not store it.

- It is unexpected, but when I update a @Published property via `assign`, `didSet` {} doesn't get called. I suppose, it updates it internally via some pointer.

- Use assign(to:on:) when you want to update a property of an object. Use assign(to:) when the property is @Published and you don't want to make a reference cycle.

### What's a Subscription

Subscription is also a protocol. They didn't joke calling Swift a [Protocol-Oriented Language](https://developer.apple.com/videos/play/wwdc2015/408/)

    protocol Subscription: Cancellable, CustomCombineIdentifierConvertible {
        func request(_ demand: Subscribers.Demand)
    }

Subscription is what connects publishers to subscribers. You can cancel it and request a demand (?). Don't know what it means yet, 
because on subscription you get `AnyCancellable` and don't call `request()` explicitly.

### Who's a Subscriber?

Subscriber is, you guess it, a protocol. 

    protocol Subscriber<Input, Failure>: CustomCombineIdentifierConvertible {
        associatedtype Input
        associatedtype Failure: Error

        /// Publisher calls it to give a subscription 
        func receive(subscription: Subscription)

        /// Publisher calls it when it emits a value
        func receive(_ input: Self.Input) -> Subscribers.Demand

        /// Publisher calls it when it completes
        func receive(completion: Subscribers.Completion<Self.Failure>)
    }

#### Back to Subscription

Ok, I found out what Subscribers.Demand is. This is a struct containing an integer that is sent by the Subscriber indicating how many values it can handle. 
This is a solution to [Backpressure](https://medium.com/@jayphelps/backpressure-explained-the-flow-of-data-through-software-2350b3e77ce7): when you get more data 
than you could possibly process at the moment. In case of Combine, Publisher could potentially flood a Subscriber with values and that is not desirable and could freeze the UI. 
The way it is solved in Combine is that a Subscriber requests a demand (see! request(_ demand)...) for new values, and Publisher can't send more than this amount. More than that,
on every `receive(_ input:)` a Subscriber returns a new Demand, thus adjusting the limit after getting each value. Beautiful. But be careful, this value is *added* 
to your initial Demand. You can't send negative and decrease it.
Now, back to reading.

#### Custom Subscriber

There is an example in Kodeco's tutorial

    let publisher = (1...6).publisher
    
    final class IntSubscriber: Subscriber {
        typealias Input = Int
        typealias Failure = Never
        
        func receive(subscription: Subscription) {
            subscription.request(.max(3))
        }
        
        func receive(_ input: Int) -> Subscribers.Demand {
            print("Received value", input)
            return .none
        }
        
        func receive(completion: Subscribers.Completion<Never>) {
            print("Received completion", completion)
        }
    }
    
    let subscriber = IntSubscriber()
    publisher.subscribe(subscriber)

We see the following printed:

    Received value 1
    Received value 2
    Received value 3

We requested a maximum of 3 elements when we got a subscription and didn't increase the demand, so we've received only 3 values. If we change the returned value to
`.unlimited` or `max(1)` we will get all 6 values and even a completion.

### Promise and Future

Now, moving further and we meet Promises. This thing exists in JavaScript and it's interesting how it got implemented in Swift with Combine. 
Pre-Combine people wrote [libraries](https://github.com/khanlou/Promise) to support it.

The idea is to asynchroniously get the returned value in the *future*. This is exactly like completion handlers. And, actually, 
`Future` is simply a class which conforms to Publisher and contains a completion handler AKA Promise. Let's peek inside:

    class Future<Output, Failure> : Publisher where Failure : Error {

        /// A type that represents a closure to invoke in the future, when an element or error is available.
        public typealias Promise = (Result<Output, Failure>) -> Void

        /// Creates a publisher that invokes a promise closure when the publisher emits an element.
        public init(_ attemptToFulfill: @escaping (@escaping Future<Output, Failure>.Promise) -> Void)

        /// Attaches the specified subscriber to this publisher.
        final public func receive<S>(subscriber: S) where Output == S.Input, Failure == S.Failure, S : Subscriber
    }

You also can get a value from a Future

    var value: Output { get async }

Interesting that it also uses Swift Concurrency.

#### Thoughts

- If you sink to a future again, it will not execute a promise again, it will immediately return a value. Futures emit value only once.
- Future executes before the subscription. It is greedy and not like regular publishers that are lazy.

### Subject

#### PassthroughSubject

Tutorial says we use `PassthroughSubject` to send values to Subscribers from non-Combine imperative code. I have to look at an example to understand.

Let's take a look inside a documentaion:

    /// A subject that broadcasts elements to downstream subscribers.
    ///
    /// As a concrete implementation of ``Subject``, the ``PassthroughSubject`` provides a convenient way to adapt existing imperative code to the Combine model.
    final public class PassthroughSubject<Output, Failure> : Subject where Failure : Error {

        public init()

        /// Sends a subscription to the subscriber.
        ///
        /// This call provides the ``Subject`` an opportunity to establish demand for any new upstream subscriptions.
        final public func send(subscription: Subscription)

        /// Attaches the specified subscriber to this publisher.
        final public func receive<S>(subscriber: S) where Output == S.Input, Failure == S.Failure, S : Subscriber

        /// Sends a value to the subscriber.
        final public func send(_ input: Output)

        /// Sends a completion signal to the subscriber.
        final public func send(completion: Subscribers.Completion<Failure>)
    }

It looks like you have to send values to the subject and they will be published to all subscribers.
Which means this is sort of a very lightweight Publisher that simply redirects values, heh.

Down the rabbit hole:

    /// A publisher that exposes a method for outside callers to publish elements.
    ///
    /// A subject is a publisher that you can use to ”inject” values into a stream, by calling its ``Subject/send(_:)`` method. This can be useful for adapting existing imperative code to the Combine model.
    @available(macOS 10.15, iOS 13.0, tvOS 13.0, watchOS 6.0, *)
    public protocol Subject<Output, Failure> : AnyObject, Publisher {

        /// Sends a value to the subscriber.
        func send(_ value: Self.Output)

        /// Sends a completion signal to the subscriber.
        func send(completion: Subscribers.Completion<Self.Failure>)

        /// Sends a subscription to the subscriber.
        func send(subscription: Subscription)
    }

So, a Subject is a Protocol which also conforms to a Publisher. Now I think I got it.

Side note: (I tried to nil a subscription and it has the same effect as calling .cancel() on it)

Since PassthroughSubject is simply a wrapper, we can't get the current value! Imagine that. For this we need to look at `CurrentValueSubject`.
I guess it is the same thing but it stores last published value for us.

    /// A subject that wraps a single value and publishes a new element whenever the value changes.
    ///
    /// Unlike ``PassthroughSubject``, ``CurrentValueSubject`` maintains a buffer of the most recently published element.
    final public class CurrentValueSubject<Output, Failure> : Subject where Failure : Error {

        /// The value wrapped by this subject, published as a new element whenever it changes.
        final public var value: Output

        /// Creates a current value subject with the given initial value.
        ///
        /// - Parameter value: The initial value to publish.
        public init(_ value: Output)

        /// ... evereything else is same as in `PassthroughSubject` ...
    }

### eraseToAnyPublisher()

I saw this method almost on every chain and always wondered why they use it that much.
Let's dissect it:

Tutorial says you erase type to hide concrete publishers from subscribers.

`func eraseToAnyPublisher()` lives in an extension on Publisher protocol, which means any publisher can be erased to AnyPublisher (no pun intended!)

    public struct AnyPublisher<Output, Failure> : CustomStringConvertible, CustomPlaygroundDisplayConvertible where Failure : Error {

        public var description: String { get }

        public var playgroundDescription: Any { get }

        /// Creates a type-erasing publisher to wrap the provided publisher.
        ///
        /// - Parameter publisher: A publisher to wrap with a type-eraser.
        @inlinable public init<P>(_ publisher: P) where Output == P.Output, Failure == P.Failure, P : Publisher
    }

So it is simply a struct which holds a pointer to the actual Publisher.

I believe its a wrapper-struct and not simply a typealias, because otherwise you could just (publisher as? CurrentValueSubject).send..

Oh, the tutorial confirmed this thought.

"""
The eraseToAnyPublisher() operator wraps the provided publisher in an instance of AnyPublisher, 
hiding the fact that the publisher is actually a PassthroughSubject. 
This is also necessary because you cannot specialize the Publisher protocol, 
e.g., you cannot define the type as Publisher<UIImage, Never>.
"""

### Swift Concurrency + Combine

No way, you can simply for-loop on asynchronously emitted values!

    let subject = CurrentValueSubject<Int, Never>(0)
    
    Task {
        for await element in subject.values {
            print("Element: \(element)")
        }
        
        print("Completed.")
    }
    
    subject.send(1)
    subject.send(2)
    subject.send(3)
    
    subject.send(completion: .finished)

Output:

    Element: 0
    Element: 1
    Element: 2
    Element: 3
    Completed.

Isn't it just amazing? You don't even have to subscribe, you can just use for-await-in loop! It's possible because subject.values is 
an `AsyncPublisher` which conforms to `AsyncSequence`. And anything that's `AsyncSequence` can be used with for-await-in.


