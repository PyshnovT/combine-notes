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
