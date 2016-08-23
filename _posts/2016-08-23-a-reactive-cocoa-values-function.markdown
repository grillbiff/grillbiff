---
title: "Splitting arrays the Reactive Cocoa way"
layout: post
date: 2016-08-23 08:22
image: '/assets/images/'
description: 
tag:
- "Reactive Cocoa"
- "RAC4"
- "Swift"
- "SignalProducer"
- "SequenceType"
blog: true
jemoji:
hidden: false
author: grillbiff
---
If you, like me, find yourself working with Rest API:s there is also a big chance you work alot with arrays of data. Usually you also want to do some work on each item in the array. The traditional way would be to loop over the data. We want to split it up and stream it down a pipe as individual elements, the reactive way. 

As it turns out, Reactive Cocoa has support for splitting up arrays and send each item down the pipe. Enter `SignalProducer(values: _)`. It will instantiate a SignalProducer which will send all items in the array and then complete. Just what we are looking for.

Let´s have a look at a basic example. We have some input in the form of an array. The array is split up, some work is performed on each element. Finally the array is assembled again.

{% highlight swift %}

SignalProducer<[String], NoError>{ sink, _ in
    // incoming data
    sink.sendNext(["ein", "zwei", "dry"])
    sink.sendCompleted()
    }
    .flatMap(.Latest) { (values: [String]) -> SignalProducer<String, NoError> in
        // split array into single values
        SignalProducer<String, NoError>(values: values)
    }
    .map { (value: String) -> String in
        // do some meaningful work on the individual values
        value + "!"
    }
    .collect() // assemble the array again (if you want to)
    .startWithNext { (values: [String]) in
        print(values) // prints "ein!", "zwei!", "dry!"
}

{% endhighlight %}

Ok, so that is good enough. It does exactly what it is supposed to. 

I found myself reimplementing this snippet over and over. Maybe we can DRY it up a bit? 

Let´s implement `SignalProducer(values: _)` as a function on the SignalProducer.

{% highlight swift %}

extension SignalProducer where Value: SequenceType {
    typealias T = Value.Generator.Element

    func values() -> SignalProducer<T, Error> {
        return self.flatMap(.Latest) { (values) -> SignalProducer<T, Error> in
            return SignalProducer<T, Error>(values: values)
        }
    }
}

{% endhighlight %}

We add a function which will work on SignalProducers which produces SequenceTypes (i.e. Arrays). The function will return a SignalProducer which produces items of the same type as found in the SequenceType. In our case it means we are operating on a `SignalProducer<[String], Error>`, which in turn will produce a `SignalProducer<String, Error>`.

Finally the original code, but with the new `SignalProducer.values()` function instead. 

{% highlight swift %}

SignalProducer<[String], NoError>{ sink, _ in
    // incoming data
    sink.sendNext(["ein", "zwei", "dry"])
    sink.sendCompleted()
    }
    .values() // split array into single values!
    .map{ value in
        // do some meaningful work on the individual values
        value + "!"
    }
    .collect() // assemble the array again (if you want to)
    .startWithNext { values in
        print(values)
}

{% endhighlight %}

Hope you enjoyed this article and as always, if you have any feedback I would be glad to hear it.