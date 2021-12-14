I am super excited for Swift concurrency.

In this year’s WWDC, Apple took that to the next level and introduced Swift concurrency, a built-in language support that promises asynchronous code that is simple to write, easy to understand, and the best of all, free from race conditions.

# Async/await
By only using async/await does not make your code run concurrently, they are just keywords introduced in **Swift 5.5** that tells the compiler that a certain block of code should run asynchronously.
```swift
func performTask() async {
    
    // Run some heavy tasks here...
}
```

An asynchronous function is a special type of function that can be suspended while it is partway through execution. However, just like a normal function, an asynchronous function can also return a value and throw an error.
```swift
func performThrowingTask() async throws -> String {
    
    // ...tasks here...
    
    return ""
}
```
If a function is marked as async, then we must call it using the await keyword like so:
```
await performTask()
```
The await keyword indicates that the performTask() function might be suspended due to its asynchronous nature. If we try to call the performTask() function like a normal (synchronous) function, we will get a compilation error saying that – ‘async’ call in a function that does not support concurrency.

Why we are getting this error is because we are trying to call an asynchronous function in the synchronous context. In order to bridge between the synchronous and asynchronous world, we must create a Task.

Task is introduced in **Swift 5.5**. According to Apple, Task is a unit of asynchronous work. Within the context of a task, code can be suspended and run asynchronously. 

```swift
func doSomething() {
    Task {
        await performTask()
    }
}
```
The code above will give us behavior similar to using a global dispatch queue.

But, they work behind the scene are actually quite different.
```swift
func doSomethingElse() {
    
    DispatchQueue.global().async {
        self.performAnotherTask()
    }
    
}
```
```swift
// this function is not marked as async
func performAnotherTask() {
    // ...task here...
}
```

# Async/await vs. Dispatch Queue
When we create a task, the task will run on an arbitrary thread. When the thread reaches a suspension point (await), the system will suspend the code and unblock the thread so that the thread can proceed with some other work while waiting for performTask() to finish. Once performTask() is finished, the task will gain back control of the thread and the code will resume.
Just like a task, the global dispatch queue is also running on an arbitrary thread. However, the thread is blocked while waiting for performAnotherTask() to finish. Therefore, the blocked thread will not be able to do anything else until performAnotherTask() returns. This makes it less efficient compared to the async/await approach.

### Simulate Long-running Task
```swift
func performTask() async {
    await Task.sleep(30 * 1_000_000_000) // Wait for 30 seconds
}
```
You do not need to create a task when calling the Task.sleep(_:) method because the performTask() is marked as async, meaning it will be running in an asynchronous context, thus creating a task is not required.

# Structured Concurrency
Let’s say we have 2 async functions that returns an integer value as shown below:

```swift
func performTaskA() async -> Int {
    await Task.sleep(1 * 1_000_000_000) // Wait for 1 seconds
    return 1
}
```

```swift
func performTaskB() async -> Int {
    await Task.sleep(2 * 1_000_000_000) // Wait for 2 seconds
    return 2
}
```

If we would to get the sum of values returned by both of these functions, we can do it like this:

```swift
func doSomething() {
    
    Task {
        let a = await performTaskA()
        let b = await performTaskB()
        let sum = a + b
        print(sum) // Output: 3
    }
}
```
The above code will take 3 seconds to complete because performTaskA() and performTaskB() runs in serial order, performTaskA() must finish first before performTaskB() can kicks in.
As you might have noticed by now, the code above is not optimum. Since performTaskA() and performTaskB() are independent from each other, we can improve the execution time by concurrently running both performTaskA() and performTaskB(), making it only takes 2 seconds to complete. This is where structured concurrency comes in.
How structured concurrency works is that we will create 2 child tasks that execute performTaskA() and performTaskB() concurrently. 

In **Swift 5.5**, there are 2 main ways to create a child task:
1. Using async-let binding
1. Using task group

# Async-let Binding
```swift
func doSomething() {
    
    Task {
        async let a = performTaskA()
        async let b = performTaskB()
        let sum = await (a + b)
        print(sum) // Output: 3
    }
}
```
In the above code, notice how we combine the async and let keyword to create an async-let binding on both performTaskA() and performTaskB() functions. Doing so will create 2 child tasks that execute both of these functions concurrently.
Since both performTaskA() and performTaskB() are marked as async, we will need to wait for both of these functions to complete in order to get the value of a and b. Therefore, when getting the value of a and b, we must use the await keyword to indicate that the code might suspend while waiting for performTaskA() and performTaskB() to complete.


# Actor
When working on asynchronous and concurrent code, the most common problems that we might encounter are data races and deadlock. These kinds of problems are very difficult to debug and extremely hard to fix. With the inclusion of actors in **Swift 5.5**, we can now rely on the compiler to flag any potential race conditions in our code.

Actors are reference types and work similarly to classes. However, unlike classes, actors will ensure that only 1 task can mutate the actors’ state at a time, thus eliminating the root cause of a race condition — multiple tasks accessing/changing the same object state at the same time.
In order to create an actor, we need to annotate it using the actor keyword. Here’s a sample Counter actor that has a count mutable state which can be mutated using the addCount() method:
```swift
actor Counter {
    private let name: String
    private var count = 0
    
    init(name: String) { self.name = name}
    
    func addCount() { count += 1 }
    
    func getName() -> String { name }
}
```

We can instantiate an actor just like instantiating a class:
```swift
let counter = Counter(name: "My Counter")
```
If we try to call the addCount() method outside of the Counter actor, we will get a compiler error saying that – Actor-isolated instance method ‘addCount()’ can not be referenced from a non-isolated context.


The reason we are getting this error is that the compiler is trying to protect the state of the Counter actor. If let’s say there are multiple threads calling addCount() at the same time, then a race condition will occur. Therefore, we cannot simply call an actor’s method just like calling a normal instance method.
To go about this restriction, we must mark the call site with the await keyword, indicating that the addCount() method might suspend when called. This actually makes a lot of sense because in order to maintain mutual exclusion on the count variable, the call site of addCount() might need to suspend so that it can wait for other tasks to finish before proceeding.
With that in mind, we can apply what we learn in the async/await section and call addCount() like so:
```swift
let counter = Counter(name: "My Counter")
Task {
    await counter.addCount()
}
```

### The nonisolated Keyword
I would like to draw your attention to the getName() method of the Counter actor. Just like the addCount() method, calling the getName() method will require the await annotation as well.
However, if you look closely, the getName() method is only accessing the Counter‘s name constant, it does not matate the state of the Counter, therefore it is impossible to create a race condition.
In this kind of situation, we can exclude the getName() method from the protection of the actor by marking it as nonisolated.
```swift
nonisolated func getName() -> String { name }
```
We can now call the getName() method like a normal instance method:
```swift
let counter = Counter(name: "My Counter")
let x = counter.getName()
```

# MainActor
MainActor is a special kind of actor that always runs on the main thread. In **Swift 5.5**, all the UIKit and SwiftUI components are marked as MainActor. Since all the components related to the UI are main actors, we no longer need to worry about forgetting to dispatch to the main thread when we want to update the UI after a background operation is completed.
If you have a class that should always be running on the main thread, you can annotate it using the @MainActor keyword:
```swift
@MainActor
class MyClass {
    
}
```
However, if you only want a specific function in your class to always run on the main thread, you can annotate the function using the @MainActor keyword:
```swift
class MyClass {
    @MainActor
    func doSomething() {
        // on main thread
    }
}
```