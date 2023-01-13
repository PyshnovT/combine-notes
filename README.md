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

