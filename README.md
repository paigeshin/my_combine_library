# my_combine_library

# V1

# Functional Programming

### Not Functional Programming

```swift
var age = 30 
age = 45 

func doSomething() {
	age = 67 
}
```

### Mutable State Problems

- Concurrency
- Race Conditions
- Dead Locks

### Immutable State

- First-Class and Higher-Order Functions ⇒ Function that takes and return another function
    - Filter
    - Map
    - Reduce

### Pure Functions

- The function **always** produce the same output when given the same input
- The function creates **zero** side effects outside of it

# What is Combine?

> **Customize handling of asynchronous events by combining event-processing operators.**
> 

The combine framework can be compared to frameworks like RxSwift and ReactiveSwift (formally known as ReactiveCocoa). It allows you to write functional reactive code by providing a declarative Swift API. **Functional Reactive Programming (FRP) languages allow you to process values over time.** Examples of these kinds of values include network responses, user interface events, and other types of asynchronous data. 

# Imperative Programming

```swift
import Foundation

let notification = Notification.Name("MyNotification")

let observer = NotificationCenter.default.addObserver(forName: notification, object: nil, queue: nil) { _ in
    print("Notification Received")
}

NotificationCenter.default.post(name: notification, object: nil)
NotificationCenter.default.removeObserver(observer)
```

# Reactive Programming

```swift
import Foundation
import Combine

let notification = Notification.Name("MyNotification")

let publisher = NotificationCenter.default.publisher(for: notification, object: nil)

var anyCancellable: AnyCancellable?

anyCancellable = publisher.sink { _ in
    print("Notifcation Received")
}

NotificationCenter.default.post(name: notification, object: nil)

anyCancellable?.cancel()
anyCancellable = nil
```

# Subscriber

```swift
import Foundation
import Combine

class StringSubscriber: Subscriber {
    
    typealias Input = String
    typealias Failure = Error 
    
    // Tells the subscriber that it has successfully subscribed to the publisher and may request items. Required.
    func receive(subscription: Subscription) {
        subscription.request(.max(3)) // BackPressing
    }
    
    // Tells the subscriber that the publisher has produced an element.
    func receive(_ input: String) -> Subscribers.Demand { // Subscribers.Demand =>  A requested number of items, sent to a publisher from a subscriber through the subscription.
        print(input)
        return .none // demand more 
    }
    
    
    // Tells the subscriber that the publisher has completed publishing, either normally or with an error. Required.
    func receive(completion: Subscribers.Completion<Error>) {
        print("Completed")
    }
    
}
    
    
let publisher = ["A", "B", "C", "D", "E", "F"].publisher

let subscriber = StringSubscriber()

publisher
    .subscribe(subscriber)
// A,B,C 
```

# Subjects

- It can be publisher and subscriber

```swift
let subject = PassthroughSubject<String, Error>()

var anyCancallable: AnyCancellable?
anyCancallable = subject.sink { _ in
    print("Received completion from sink")
} receiveValue: { string in
    print("Received value from sink => \(string)")
}

subject.send("A")
subject.send("B") // A, B
subject.send("C")
subject.send("D")
```

# TypeEraser

```swift
let publisher: AnyPublisher<Int, Never> = PassthroughSubject<Int, Never>().eraseToAnyPublisher()
```

# Transformation Operators

- scan
- map
- map keypath
- collect
- replaceEmpty
- flatMap
- replaceNil

### collect()

```swift
import Foundation
import Combine

["A", "B", "C", "D", "E"].publisher.collect(2).sink {
    print($0)
    // ["A", "B"]
    // ["C", "D"]
    // ["E"]
}
```

### map()

```swift
import Foundation
import Combine

let formatter = NumberFormatter()
formatter.numberStyle = .spellOut

[123, 45, 67].publisher.map {
    formatter.string(from: NSNumber(integerLiteral: $0)) ?? ""
}.sink {
    print($0)
    /*
     one hundred twenty-three
     forty-five
     sixty-seven
     */
}
```

### mapKeypath

- Access Object Properties

```swift
import Foundation
import Combine

struct Point {
    let x: Int
    let y: Int
}

let publisher = PassthroughSubject<Point, Never>()

publisher
    .map(\.x, \.y)
    .sink { x, y in
        print("x is \(x) and y is \(y)")
				// x is 2 and y is 10
    }

publisher.send(Point(x: 2, y: 10))
```

### flatMap ⭐️⭐️⭐️

- flatMap operator can be used to flatten the multiple upstream publishers into a single downstream publisher.

```swift
import Foundation
import Combine

struct School {
    let name: String
    let noOfStudents: CurrentValueSubject<Int, Never>
    
    init(name: String, noOfStudents: Int) {
        self.name = name
        self.noOfStudents = CurrentValueSubject(noOfStudents)
    }
    
}

let citySchool = School(name: "Fountain Head School", noOfStudents: 100)

let school = CurrentValueSubject<School, Never>(citySchool)

school
    .sink { school in
        print(school)
    }

citySchool.noOfStudents.value += 1 // Never Fired
```

```swift

import Foundation
import Combine

struct Point {
    let x: Int
    let y: Int
}

/// We can use `flatMap` over here to make sure that the internal publisher events can be emitted.
let citySchool = School(name: "Fountain Head School", noOfStudents: 100)

let school = CurrentValueSubject<School, Never>(citySchool)

school
    .flatMap { $0.noOfStudents } // Flattening Publisher
    .sink { school in
        print(school)
    }

citySchool.noOfStudents.value += 1 // Fired => Capture Internal Events

/*
 flatMap() will take in the streams of data, which is coming from the internal publisher events and it will allow you to access those events.
 
 */
```

⇒  `flatMap()` will take in the streams of data, which is coming from the internal publisher events and it will allow you to access those events.

### replaceNil

```swift
import Foundation
import Combine

["A", "B", nil, "C"].publisher.replaceNil(with: "*")
    .sink { print($0!) } // A B * C
```

### replaceEmpty

```swift
import Foundation
import Combine

// Combine Operator, it won't emit anything.
// It only publishes completion event
// We use `Empty` Publisher when you don't want publish value
let empty = Empty<Int, Never>()

empty
    .sink { completion in
        print(completion) // finished
    } receiveValue: { value in
        // Never Fired
        print(value)
    }

empty
    .replaceEmpty(with: 1)
    .sink { completion in
        print(completion) // finished
    } receiveValue: { value in
        print(value) // 1 
    }
```

### scan

```swift
import Foundation
import Combine

let publisher = (1...10).publisher

publisher.scan([]) { numbers, nextValue -> [Int] in
//    print("numbers: \(numbers)")
    /*
     []
     [1]
     [1, 2]
     [1, 2, 3]
     [1, 2, 3, 4]
     [1, 2, 3, 4, 5]
     [1, 2, 3, 4, 5, 6]
     [1, 2, 3, 4, 5, 6, 7]
     [1, 2, 3, 4, 5, 6, 7, 8]
     [1, 2, 3, 4, 5, 6, 7, 8, 9]
     */
    print(nextValue)
    /*
     1
     2
     3
     4
     5
     6
     7
     8
     9
     10
     */
    return numbers + [nextValue]
}
.sink {
    print($0)
    /*
     [1]
     [1, 2]
     [1, 2, 3]
     [1, 2, 3, 4]
     [1, 2, 3, 4, 5]
     [1, 2, 3, 4, 5, 6]
     [1, 2, 3, 4, 5, 6, 7]
     [1, 2, 3, 4, 5, 6, 7, 8]
     [1, 2, 3, 4, 5, 6, 7, 8, 9]
     [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
     */
}

var items: [Int] = [1,2,3]

var newItems = items + [4]
print(newItems) // [1,2,3,4]
```

# Filtering Operators

### filter

```swift
import Foundation
import Combine

let numbers = (1...20).publisher

numbers.filter { $0 % 2 == 0 }
    .sink {
        print($0)
    }
```

### removeDuplicates

```swift
import Foundation
import Combine

let words = "apple apple apple fruit apple apple mango watermelon apple".components(separatedBy: " ").publisher

words
    .removeDuplicates()
    .sink {
        print($0)
        /*
         apple
         fruit
         apple
         mango
         watermelon
         apple
         */
    }
```

### compactMap

```swift
import Foundation
import Combine

let strings = ["a", "1.24", "b", "3.45", "6.7"].publisher
    .compactMap { Float($0) }
    .sink {
        print($0)
        /*
         1.24
         3.45
         6.7
         */
    }
```

### ignoreOutput

- `ignoreOutput` will only emit the completion event.

```swift
import Foundation
import Combine

let numbers = (1...5000).publisher

numbers
    .ignoreOutput()
    .sink {
        print($0) // finished
    } receiveValue: {
        print($0) // nothing is printed
    }
```

### first

- when condition is met, it stops publishing values

```swift
import Foundation
import Combine

let numbers = (1...9).publisher

numbers
    .first {
				// going through forward 
        $0 % 2 == 0
    }
    .sink {
        print($0) // 2
    }
```

### last

```swift
import Foundation
import Combine

let numbers = (1...9).publisher

// going through from backward 
numbers
    .last { value in
        print(value)
        /*
         9
         8
         */
        return value % 2 == 0
    }
    .sink {
        print($0) // 8
    }
```

### dropFirst

- The drop first operator allows you to drop certain items, certain values from your sequence

```swift
import Foundation
import Combine

let numbers = (1...10).publisher

numbers
    .dropFirst(5)
    .sink {
        print($0)
    }
/*
 6
 7
 8
 9
 10
 */
```

### drop(while: )

- `drop(while:)` operator will be based on a particular condition and it’s going to drop the values **while the condition will be true**

```swift
import Foundation
import Combine

let numbers = (1...10).publisher

numbers
    .drop(while: { $0 % 3 != 0 })
    .sink {
        print($0)
        /*
         3
         4
         5
         6
         7
         8
         9
         10
         */
    }
```

### drop(untilOutputFrom:)

> Wait until another publisher sends an event
> 
- `drop(untilOutputFrom:)` publisher will drop the events or the values from a sequence until it gets an output from another publisher
- For example, every time you will tap, all of those things will be ignored until it gets an output from a particular publisher

```swift
import Foundation
import Combine

let isReady = PassthroughSubject<Void, Never>()
let taps = PassthroughSubject<Int, Never>()

taps
    .drop(untilOutputFrom: isReady)
    .sink { print($0) }

taps.send(0) // ignored
taps.send(1) // ignored
taps.send(2) // ignored

isReady.send()

taps.send(0) // 0
taps.send(1) // 1
taps.send(2) // 2
```

### prefix

- Go ahead and select `n` items

```swift
import Foundation
import Combine

let numbers = (1...10).publisher

numbers
    .prefix(2)
    .sink {
        print($0)
        /*
         1
         2
         */
    }

// 1, 2
numbers
    .prefix(while: { $0 < 3})
    .sink {
        print($0)
    }
```

# Combining Operators

### prepend

```swift
import Foundation
import Combine

let numbers = (1...5).publisher

numbers
    .prepend(100, 101)
    .prepend([45, 67])
    .sink {
        print($0)
        /*
         45
         67
         100
         101
         1
         2
         3
         4
         5
         */
    }

import Foundation
import Combine

let numbers = (1...5).publisher
let publisher2 = (500...510).publisher

numbers
    .prepend(100, 101)
    .prepend([45, 67])
    .prepend(publisher2)
    .sink {
        print($0)
        /*
         500
         501
         502
         503
         504
         505
         506
         507
         508
         509
         510
         45
         67
         100
         101
         1
         2
         3
         4
         5
         */
    }
```

### append

```swift
import Foundation
import Combine

let numbers = (1...10).publisher
let publisher = (100...110).publisher

numbers
    .append(11, 12)
    .append(13, 14)
    .append(publisher)
    .sink {
        print($0)
        /*
         1
         2
         3
         4
         5
         6
         7
         8
         9
         10
         11
         12
         13
         14
         100
         101
         102
         103
         104
         105
         106
         107
         108
         109
         110
         */
    }
```

### switchToLastest

- `switchToLastest` allows you to switch to different publisher

```swift
import Foundation
import Combine

let publisher1 = PassthroughSubject<String, Never>()
let publisher2 = PassthroughSubject<String, Never>()

let publishers = PassthroughSubject<PassthroughSubject<String, Never>, Never>()

publishers
    .switchToLatest()
    .sink { string in
        print(string)
    }

publishers.send(publisher1) // publisher1 is hooked up to publishers
publisher1.send("Publisher 1 - Value 1") // OUTPUT: Publisher 1 - Value 1
publisher1.send("Publisher 1 - Value 2") // OUTPUT: Publisher 1 - Value 2

publisher2.send("Publisher 2 - Value 1") // OUTPUT: Ignored
publishers.send(publisher2) // publisher2 is hooked up to publishers, Switching to Publisher 2
publisher2.send("Publisher 2 - Value 1") // Publisher 2 - Value 1

publisher1.send("Publisher 1 - Value 3") // OUTPUT: Ignored
```

```swift
let taps = PassthroughSubject<Void, Never>()

func getImage() -> AnyPublisher<UIImage?, Never> {
    return Future<UIImage?, Never> { promise in
        //simulate delay for download
        DispatchQueue.global().asyncAfter(deadline: .now() + 3.0) {
            promise(.success(UIImage(named: images[index])))
        }
    }.print().map { $0 }
    .receive(on: RunLoop.main)
    .eraseToAnyPublisher()
}

let subscription = taps.map { _ in getImage() }
    .switchToLatest()
		.sink {
        print($0)
    }

taps().send() 
```

### merge

- Merge two publishers and emit the same output.
- Publishers Output and Failure Type must be the same.

```swift
import Foundation
import Combine

let publisher1 = PassthroughSubject<Int, Never>()
let publisher2 = PassthroughSubject<Int, Never>()

publisher1
    .merge(with: publisher2)
    .sink { item in
        print(item)
        /*
         1
         2
         */
    }

publisher1.send(1)
publisher2.send(2)
```

### combineLatest

- Firstly, it should takes one event from one publisher and another event from anther publisher to emit the value.
- It emits tuples.
- Publishers Output and Failure Type needn’t be the same.

```swift
import Foundation
import Combine

let publisher1 = PassthroughSubject<Int, Never>()
let publisher2 = PassthroughSubject<String, Never>()

publisher1
    .combineLatest(publisher2)
    .sink {
        print("Publisher1: \($0), Publisher2: \($1)")
    }

publisher1.send(1) // OUTPUT: VOID
publisher2.send("hello") // OUTPUT: "Publisher1: 1, Publisher2: hello"
publisher1.send(2) // OUTPUT: "Publisher1: , Publisher2: hello"
```

### zip

- `zip` only emits when all provided publishers emit the events.
- It emits tuples.
- Publishers Output and Failure Type needn’t be the same.

```swift
import Foundation
import Combine

let publisher1 = PassthroughSubject<Int, Never>()
let publisher2 = PassthroughSubject<String, Never>()

publisher1
    .zip(publisher2)
    .sink {
        print("Publisher1: \($0), Publisher2: \($1)")
    }

publisher1.send(1) // OUTPUT: VOID
publisher2.send("hello") // OUTPUT: "Publisher1: 1, Publisher2: hello"
publisher1.send(2) // OUTPUT: Void
publisher2.send("world") // OUTPUT: "Publisher1: 2, Publisher2: world"
```

# Sequence Operators

### min and max

```swift
import Foundation
import Combine

let publisher = [1, -45, 3, 45, 100].publisher

publisher
    .min()
    .sink {
        print($0) //-45
    }

publisher
    .max()
    .sink {
        print($0) //100
    }
```

### first and last

- first ⇒ iterate forward
- last ⇒ iterate from backward

```swift
import Foundation
import Combine

let publisher = ["A", "B", "C", "D"].publisher

publisher
    .first()
    .sink {
        print($0) // "A"
    }

publisher
    .first(where: { $0 == "B" })
    .sink {
        print($0) // "B"
    }

publisher
    .first(where: { $0 == "F" })
    .sink {
        print($0) // VOID
    }

publisher
    .last()
    .sink {
        print($0) // "D"
    }
```

### output

```swift
import Foundation
import Combine

let publisher = ["A", "B", "C", "D"].publisher

// takes index and output
publisher.output(at: 2)
    .sink {
        print($0) // "C"
    }

// takes ranges and output
publisher.output(in: (0...2))
    .sink {
        print($0)
        /*
         A
         B
         C
         */
    }
```

### contains

```swift
import Foundation
import Combine

let publisher = ["A", "B", "C", "D"].publisher

publisher
    .count()
    .sink {
        print($0) // 4
    }

let publisher1 = PassthroughSubject<Int, Never>()

publisher1.count()
    .sink {
        print($0)
    }

// Nothing's gonna be printed out, becuase publisher is not finished
publisher1.send(10) //OUTPUT: VOID

publisher1.send(completion: .finished) // 1
```

### contains

```swift
import Foundation
import Combine

let publisher = ["A", "B", "C", "D"].publisher

publisher
    .contains("C")
    .sink {
        print($0) // true 
    }
```

### allSatisfy

```swift
import Foundation
import Combine

let publisher = [1, 2, 3, 4, 5, 6, 7].publisher

publisher
    .allSatisfy {
        // evaluate values
        $0 % 2 == 0
    }
    .sink {
        print($0) // false
    }
```

### reduce

```swift
import Foundation
import Combine

let publisher = [1, 2, 3, 4, 5, 6, 7].publisher

publisher.reduce(0) { accumulator, value in
    return accumulator + value
}
.sink {
    print($0) // 28 
}
```

# Combine Network

### URLSession.dataTaskPublisher

```swift
func getPosts() -> AnyPublisher<Data, URLError> {
    guard let url = URL(string: "https://jsonplaceholder.typicode.com/posts") else {
        fatalError("Invalud URL")
    }
    
    return URLSession.shared.dataTaskPublisher(for: url)
        .map { $0.data }
        .eraseToAnyPublisher()
    
}

var anyCancellable: AnyCancellable? = getPosts()
    .sink { _ in
        
    } receiveValue: { print($0) }
```

### JSON Decoder

```swift
import Foundation
import Combine

struct Post: Codable {
    let title: String
    let body: String
}

func getPosts() -> AnyPublisher<[Post], Error> {
    guard let url = URL(string: "https://jsonplaceholder.typicode.com/posts") else {
        fatalError("Invalud URL")
    }
    
    return URLSession.shared.dataTaskPublisher(for: url)
        .map { $0.data }
        .decode(type: [Post].self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
    
}

var anyCancellable: AnyCancellable? = getPosts()
    .sink { _ in
        
    } receiveValue: { print($0) }
```

# Debugging Combine

### Printing Events

```swift
import Foundation
import Combine

let publisher = (1...20).publisher

publisher
    .print("Debugging")
    .sink {
        print($0)
    }

publisher
    .print() // prints all the events happening 
    .sink {
        print($0) 
    }
```

### Performing side effects with `handleEvents`

```swift
import Foundation
import Combine

let url = URL(string: "https://jsonplaceholder.typicode.com/posts")!

let request = URLSession.shared.dataTaskPublisher(for: url)

request
    .handleEvents(receiveSubscription: { _  in },
                  receiveOutput: { _, _ in},
                  receiveCompletion: { _ in },
                  receiveCancel: {},
                  receiveRequest: { _ in})
    .sink {
    print($0)
} receiveValue: { (data, response) in
    print(data)
}
```

### Using Debugger with `breakPoint`

```swift
import Foundation
import Combine

let publisher = (1...10).publisher

publisher
    .breakpoint(receiveOutput: { return $0 > 9 }) // break 
//    .breakpoint(receiveSubscription: <#T##((Subscription) -> Bool)?##((Subscription) -> Bool)?##(Subscription) -> Bool#>,
//                receiveOutput: <#T##(((data: Data, response: URLResponse)) -> Bool)?##(((data: Data, response: URLResponse)) -> Bool)?##((data: Data, response: URLResponse)) -> Bool#>,
//                receiveCompletion: <#T##((Subscribers.Completion<URLError>) -> Bool)?##((Subscribers.Completion<URLError>) -> Bool)?##(Subscribers.Completion<URLError>) -> Bool#>)
```

# Combine Timers

### RunLoop Timer

```swift
import Foundation
import Combine

let runLoop = RunLoop.main // UIThread

let subscription = runLoop.schedule(
    after: runLoop.now,
    interval: .seconds(2),
    tolerance: .milliseconds(100), // timer is late, don't worry about that => related with energy consumption
    options: .none) {
        // action fired
        print("Timer fired")
    }

runLoop.schedule(after: .init(Date(timeIntervalSinceNow: 3.0))) {
    print("canceled")
    subscription.cancel()
}
```

### Timer Publisher

```swift
import Foundation
import Combine

/*
MARK: ARUGMNETS EXPLAINED

 interval
 The time interval on which to publish events. For example, a value of 0.5 publishes an event approximately every half-second.

 tolerance
 The allowed timing variance when emitting events. Defaults to nil, which allows any variance.

 runLoop
 The run loop on which the timer runs.

 mode
 The run loop mode in which to run the timer.

 options
 Scheduler options passed to the timer. Defaults to nil.

 
 */

/*
 
 MARK: System Run Loop Modes
 
 static let common: RunLoop.Mode
 A pseudo-mode that includes one or more other run loop modes.
 
 static let `default`: RunLoop.Mode
 The mode set to handle input sources other than connection objects.
 
 static let eventTracking: RunLoop.Mode
 The mode set when tracking events modally, such as a mouse-dragging loop.
 
 static let modalPanel: RunLoop.Mode
 The mode set when waiting for input from a modal panel, such as a save or open panel.
 
 static let tracking: RunLoop.Mode
 The mode set while tracking in controls takes place.

 */

let cancallable = Timer.publish(every: 1.0,
              on: .main,
              in: .common) // common
.autoconnect()
.scan(0, { counter, _ in counter + 1 })
.sink(receiveValue: {
    print("Timer fired => \($0)")
    /*
     
     Timer fired => 1
     Timer fired => 2
     Timer fired => 3

     ...
     */
})
```

### Create Timer with DispatchQueue

```swift
import Foundation
//import Combine

let subscription = DispatchQueue.main.schedule(after: DispatchQueue.main.now,
                            interval: .seconds(1),
                            tolerance: .milliseconds(200),
                            options: .none, {
    print("fired")
})
```

### Timer with DispatchQueue

```swift
import Foundation
import Combine

let queue = DispatchQueue.main

let source = PassthroughSubject<Int, Never>()

var counter = 0

let anyCancellable = queue.schedule(after: queue.now,
               interval: .seconds(1),
               {
    source.send(counter)
    counter += 1
})

let subscription = source.sink {
    print($0) // Nothing gets printed
}
```

# Resources in Combine

### Understanding the problem

- In the code below, you can see that you are downloading twice the data from the remote server.

```swift
import Foundation
import Combine

let url = URL(string: "https://jsonplaceholder.typicode.com/posts")!

let request = URLSession.shared.dataTaskPublisher(for: url).map(\.data).print()

// subscription 1 with the same publisher
let subscription1 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print($0)
    })

// subscription 2 with the same publisher
let subscription2 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print($0)
    })
```

### share

- By sharing those resources, all the different subscribers can use those resources.
- If you have 50 different subscribers, this means that you will only fetch the information from your first subscriber and you share it.

```swift
import Foundation
import Combine

let url = URL(string: "https://jsonplaceholder.typicode.com/posts")!

let request = URLSession.shared.dataTaskPublisher(for: url)
	.map(\.data)
	.print()
	.share() // 🔥🔥🔥🔥🔥

// subscription 1 with the same publisher
let subscription1 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print("Subscription 1")
        print($0) // Fired
    })

// subscription 2 with the same publisher
let subscription2 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print("Subscription 2")
        print($0) // Fired
    })
```

### Another Problem

- How to receive shared resources when the publisher already finished the job?

```swift
import Foundation
import Combine

let url = URL(string: "https://jsonplaceholder.typicode.com/posts")!

let request = URLSession.shared.dataTaskPublisher(for: url).map(\.data).print().share()

// subscription 1 with the same publisher
let subscription1 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print("Subscription 1")
        print($0) // Fired
    })

// subscription 2 with the same publisher
let subscription2 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print("Subscription 2")
        print($0) // Fired
    })

var subscription3: AnyCancellable? = nil
DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
    // Already Too Late After three seconds, because the events are already finished
    // This is when `multicast` operator comes in to play 
    subscription3 = request.sink(receiveCompletion: { _ in }, receiveValue: {
        print("Subscription 3")
        print("Data Received from subscription3 => \($0)") // It does not work.
    })
}
```

### multicast

```swift
import Foundation
import Combine

let url = URL(string: "https://jsonplaceholder.typicode.com/posts")!

let subject = PassthroughSubject<Data, URLError>()

let request = URLSession.shared.dataTaskPublisher(for: url)
    .map(\.data)
    .print()
    .multicast(subject: subject) // 🔥🔥🔥

// subscription 1 with the same publisher
let subscription1 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print("Subscription 1")
        print($0) // Fired
    })

// subscription 2 with the same publisher
let subscription2 = request
    .sink(receiveCompletion: { _ in },
          receiveValue: {
        print("Subscription 2")
        print($0) // Fired
    })

let subscription3 = request.sink(receiveCompletion: { _ in }, receiveValue: {
    print("Subscription 3")
    print($0) // Fired
})

// MARK: 🔥🔥🔥 CONNECT ALL THE SUBSCRIPTIONS 🔥🔥🔥
request.connect()
subject.send(Data())
```
