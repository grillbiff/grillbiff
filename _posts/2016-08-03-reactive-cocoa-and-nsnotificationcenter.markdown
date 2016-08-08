---
title: "Reactive Cocoa and NSNotificationCenter"
layout: post
date: 2016-08-03 07:49
image: '/assets/images/'
description:
- "Reactive Cocoa"
- "RAC4"
- "Swift"
- "SignalProducer"
blog: true
jemoji:
hidden: false
author: grillbiff
---

## Reactive Cocoa and NSNotificationCenter

Wouldn´t it be nice to receive notifications from NSNotificationCenter on a signal so that it plays nice with the rest of your Reactive Cocoa? Of course it would!

If you are not already familiar with NSNotificationCenter it is a hub for broadcasting information within an app. You can listen in on messages beeing sent (mainly by the system) and you can send your own custom messages. To start receiving notfications you add an observer (essentially subscribe) to the NSNotificationCenter for a specific notification name (i.e `UIKeyboardWillShowNotification`). When you are done listening for notifications you have to unsubscribe your observer from the NSNotificationCenter.

Since we have a start/stop behaviour when we subscribe/unsubscribe from the NSNotificationCenter we will be using a `SignalProducer`.

Let's define a function on the NSNotificationCenter which returns a SignalProducer that will emit NSNotifications

{% highlight swift %}

	extension NSNotificationCenter {
		        func observerSignalForName(name: String, object: AnyObject? = nil, queue: NSOperationQueue? = nil) -> SignalProducer<NSNotification, NoError> {
		        ...
		}
	}

{% endhighlight %}

Ok, great. Now the returned SignalProducer needs subscribe to NSNotificationCenter and start emitting NSNotifications.

{% highlight swift %}

	extension NSNotificationCenter {
		        func observerSignalForName(name: String, object: AnyObject? = nil, queue: NSOperationQueue? = nil) -> SignalProducer<NSNotification, NoError> {
		        return SignalProducer<NSNotification, NoError>{ sink, _ in 
		        	self.addObserverForName(name, object: object, queue: queue) { (notif: NSNotification) in 
		        	sink.sendNext(notif)
		        	}
		        }
		}
	}

{% endhighlight %}

That's pretty straightforward. When we start the SignalProducer it subscribes to the NSNotificationCenter and starts sending NSNotifications down the pipe. 

Now for the tricky part, we need some way of unsubscribing from the NSNotificationCenter when we are done with the SignalProducer. Somehow we need to listen for events sent by the SignalProducer and find out if the signal has completed, failed or been interrupted and then unsubscribe. To accomplish that we wrap the subscribing SignalProducer (inner) in another SignalProducer (outer). 

{% highlight swift %}

	extension NSNotificationCenter {
        
        func observerSignalForName(name: String, object: AnyObject? = nil, queue: NSOperationQueue? = nil) -> SignalProducer<NSNotification, NoError> {
            
            // The outer SignalProducer
            return SignalProducer<NSNotification, NoError>{ sink, disposable in
                var o: NSObjectProtocol!
                
                // Start a new inner SignalProducer which adds an observer to NSNotificationCenter
                SignalProducer<NSNotification, NoError>{ innerSink, _ in
                    o = self.addObserverForName(name, object: object, queue: nil) { (notif: NSNotification) in
                        innerSink.sendNext(notif)
                    }

                    }.startWithSignal({ (signal: Signal<NSNotification, NoError>, signalDisposable: Disposable) in
                        // Add disposable to outer disposable so that we are notified when the outer SignalProducer is disposed.
                        disposable.addDisposable(signalDisposable)
                        
                        // Observe any events sent by the inner SignalProducer and forward them to the outer SignalProducer
                        signal.observe({ (event: Event<NSNotification, NoError>) in
                            
                            sink.action(event)
                            
                            // Check if the SignalProducer is beeing disposed
                            if event.isTerminating {
                                self.removeObserver(o)
                            }
                        })
                    })
            }
        }
    }

{% endhighlight %}

The inner SignalProducer subscribes to NSNotificationCenter and is started with `startWithSignal`. All events emitted by the inner SignalProducer will now be sent on the signal given as input to `startWithSignal`. We observe the signal and forward all events to the outer SignalProducer but we also inspect the event to see if the signal is terminating with `event.isTerminating`. This way we can now if the signal is beeing closed and unsubscribe from the NSNotificationCenter. 

In general, this is a good method if you need to sneak peek at events beeing emitted from a SignalProducer before the events are sent further down the pipe where it may no longer be under your control.

  