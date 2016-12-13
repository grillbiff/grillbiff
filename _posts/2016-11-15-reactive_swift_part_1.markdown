---
title: "Reactive Swift Tutorial Part 1"
layout: post
date: 2016-11-15 21:58
image: '/assets/images/'
description:
tag:
- "ReactiveSwift"
- "RAC4"
- "Swift"
- "SignalProducer"
blog: true
jemoji:
author:
---

# Reactive Swift Tutorial Part 1

## Introduction
Reactive programming is a great way to manage the complexity of user interfaces. It is also a natural fit for the asynchronous nature of mobile applications. Reactive Swift has been around for a couple of years, though it was previously named ReactiveCocoa and targeted ObjC. The latest version of ReactiveSwift targets Swift3 and is available for iOS, macOS, tvOS and watchOS. ReactiveCocoa is still around but now only handles integration with Cocoa.   

## Mental model
A metaphor often used in reactive programming is that of events flowing through pipes. Pipes can connect to eachother, one pipe can split into multiple pipes and multiple pipes can merge into a single pipe. Events always flow in one direction through the pipes, from one end to the other. A network of pipes is not a closed system so somewhere events has to enter, and they also have to exit somewhere.

## Hello world
Ok, so how much reactive fun can we cram out "hello world". Let's start with a minimal example.

```
import ReactiveSwift
import Result

let sp = SignalProducer<String, NoError>{ (sink: Observer, disposable: Disposable) in
	sink.send("Hello world")
	sink.sendCompleted()
}

sp.startWithValues { (input: String) in
	print(input) // "Hello World"
}
```

First we create a `SignalProducer` which is just as the name implies, something that creates signals. You can think of it as a factory for signals. A `SignalProducer` is initialized with a closure which receives the building blocks for a signal; an observer and a disposable. Just as the kitchen sink is the interface for the piping system in your house, the observer is the entry point to your reactive piping system. So it only makes sense to call it the `sink`. The `disposable` is a construct for the outside world to cancel a signal, i.e aborting a network call.

Nothing is actually computed until we start pulling values from our producer. Each time we call `startWithValues` a new `signal` is created and a new instance of the string "Hello world" is sent down the pipes.

The first example showed you a very short pipe, there was a entry point and an exit point but not much in between. Let's add a "pipe" that will uppercase "Hello World".

```
import ReactiveSwift
import Result

let sp = SignalProducer<String, NoError>{ (sink: Observer, disposable: Disposable) in
	sink.send("Hello world")
	sink.sendCompleted()
}
.map{ (s: String) in
	return s.upperCase()
}

sp.startWithValues { (input: String) in
	print(input) // "HELLO WORLD"
}
```

We added a map operator which, you´ve guessed it, maps one value to another. In our case a String is converted to its uppercase equivalent. There are of course [more operators available](https://github.com/ReactiveCocoa/ReactiveSwift/blob/master/Documentation/BasicOperators.md#mapping) so feel free to experiment with them.

That is it for this time. As always, feedback is much appriciated.
