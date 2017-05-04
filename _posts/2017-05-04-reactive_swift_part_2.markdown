---
title: "Reactive Swift Tutorial Part 2"
layout: post
date: 2017-05-04 16:50
image: '/assets/images/'
description:
tag:
- "ReactiveSwift"
- "RAC4"
- "Swift"
- "SignalProducer"
blog: true
jemoji:
author: grillbiff
---

# Reactive Swift Tutorial Part 2

In [part 1](/2016-11-15-reactive_swift_part_1.markdown) you got a simple hello world up and running. In this part, we will take a look at the core concepts of `Signals` and `SignalProducers`.

## A brief history of Signals and SignalProducers

In the previous part we created a `SignalProducer`, which is exactly what it says it is, a producer of `Signals`. What exactly is the difference between using a signal producer by a `SignalProducer` and using `Signal` straight up? 

To understand the difference, let's have a look at some reactive concepts. A signal in ReactiveSwift can also be called a stream (**streams of values over time** if you go by the ReactiveCocoa definition) and there are basically two types of streams, hot and cold streams.

Cold streams are *passive* and start producing values only on when you ask for it. Think http requests, it will only trigger when you explicitly ask for it. 

Hot streams are *active* and will happily keep producing values irregardless if someone is listening or not. Think of a clock, it will keep ticking and producing values even if you are not really listening.

So what has hot and cold streams to do with `Signals` and `SignalProducers`?. In most other reactive frameworks, hot and cold streams are implemented using the same type. The creators of ReactiveSwift decided that it was not clear enough when you where working on a hot signal or a cold signal and thus introduced `Signals` and `SignalProducers`. The `Signal` represents a hot stream, while the `SignalProducer` represents a cold stream. Why is it significant to know if you are working on a hot or cold signal? While observing a hot stream will not trigger anything that was not already in progress, starting a cold stream will trigger a sequence of events that was not going to happen otherwise. 
 
## Signal    
Let's implement a `Signal` that relenlessly keeps producing values a regular intervals, like maybe, a clock!

```
let clock = Signal<Date, NoError> { (sink: Observer<Date, NoError>) -> Disposable? in
	Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true, block: { (t: Timer) in
 		print("tick")
 		sink.send(value: Date())
	})
            
 	return SimpleDisposable()
}
```
The `Signal` is initialized with an observer, which we by convention call the sink. We also have the option to return something called a disposable from the init closure but more on that later. We start a `Timer` which will fire once each second. For each time the `Timer` fires we send the current `Date` down the sink. We also print a tick just to proove to you that it is actually running even though you are not listening. If you run the program now it should print a neverending stream of "ticks".

Ok, so now that we have our clock ticking away, how do we actually listen to it? Easy

```
clockSignal.observeValues { (d: Date) in
	print("Received tick at \(d)")
}
```

So what about the `Disposable`? A `Disposable` is a protcol for cancelling the observation of a signal by calling `.dispose()`. 

```
var i = 0;
var disposable: Disposable?
disposable = clockSignal.observeValues { (d: Date) in
	print("Received value \(d)")
    i += 1
    if i == 10 {
    	disposable!.dispose()
    }
}
```

Observing the `clockSignal` will return a `Disposable`. We use this disposable to cancel our observer after 10 ticks. Even though our observer will stop receiving values, the timer `clockSignal` will keep happily producing values. 

## SignalProducer
   
A `SignalProducer` represents something that can be started and stopped, so keeping with the time examples, let's implement a timer.

```
let sp = SignalProducer<Date, NoError> { sink, disposable in
	Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true, block: { (timer: Timer) in
    	print("tick")
    	sink.send(value: Date())
	})
}
```

If we run the program, nothing happens. This is because the `SignalProducer` will only start producing values when we request them.

```
sp.startWithValues { (time: Date) in
	print("time: \(time)")
}
```

When you run the program now, it will start producing values. But we should be able to stop a timer as well so let's see how we can use the `Disposable` for that. 

```
let sp = SignalProducer<Date, NoError> { sink, disposable in
    Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true, block: { (timer: Timer) in
		// check if the disposable has been disposed
       if disposable.isDisposed {
           timer.invalidate()
       }
       print("tick")
       sink.send(value: Date())
    })
}
        
var i = 0
var disposable: Disposable?
disposable = sp.startWithValues { (time: Date) in
    print("time: \(time)")
    i += 1
    if i == 10 {
        disposable!.dispose()
    }
}
```

The `Disposable` for this `SignalProducer` is first sent as a parameter to the creation closure. We use it the check if it has been disposed each time a timer event fires. If it has been disposed, we invalidate the timer. Calling `dispose()` on a `Disposable` will send an `interrupted` event down the sink which will close the `Signal`. When a `Signal` is closed, no further values will be sent on it.

A more realistic implementation of a `SignalProducer` and a `Disposable` would be a http request. Each time some new data arrives, the `SignalProducer` checks if it has been disposed, maybe by an impatient user, and can then cancel the request.    

Now that you have grasped the basic concept of `Signals` and `SignalProducers` it is time put them to work. 

In the next tutorial we will look at the most common operators.

