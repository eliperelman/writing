You can hardly write ten lines of JavaScript before you're confronted with asynchronous behavior. In this post we're going to look at 5 patterns that can help you tame the beast - or at least keep it at bay. It would be nearly impossible to cover each pattern in depth in this post alone, but my aim is to provide enough information to get you started, with an additional reading section at the end.

Let me emphasize here that none of these patterns are *only* asynchronous. In fact, all of them work well in synchronous contexts also. 

 
#What's 'Asynchronous'? A Quick Primer…
Your JavaScript code is executed in an event loop, on a single thread. The reality is that *all* JavaScript executes synchronously - it's the event loop that allows you to queue up an action that won't take place until the loop is available some time *after* the code that queued the action has finished executing. So code is said to execute asynchronoulsy when it is queued to run sometime after the event loop is available. Or, as Trevor Burnham said in his book [Asynchronous JavaScript](http://pragprog.com/book/tbajs/async-javascript):

> "Events can be queued while code is running, but they can't fire until the runtime is free."

Enough talk, code is proof:

```
function maybe() {
	console.log("…execute async, maybe?");
}

function proveIt() {
	setTimeout(maybe, 0);
	console.log("Hey, you just invoked me, and this is crazy…");
	console.log("But I'll queue you up");
	return "and you'll…"
};

proveIt();

```
Running this in Chrome's JavaScript console will give you the following:

![](./asyncproof1.jpg)

We used `setTimeout` in the above example to queue up the `maybe` function to execute 0 milliseconds after the event loop is available. The event loop won't be available until it completes its current execution stack - in this case, the rest of the `proveIt` function which includes two `console.log` statements & the return. As soon as the `proveIt` function returns, the runtime is free, and the queued up `maybe` function can execute. You can actually see where `proveIt` returns in the above output: where you see the string "and you'll…" printed on the console. You can tell from the console output that at this point, `proveIt` has completed, and our queued `maybe` function's output appears after that.

> Just in case `setTimeout` is new to you (or you are new to JavaScript), two async 'primitives' exist across all major runtimes:
>
> * `setTimeout(fn, x)` (which queues a function `f` to be invoked after `x` milliseconds, though the actual delay could be longer)
> * `setInterval(fn, x)` (which allows repeated invocations of function `fn` with a delay of `x` between each - though the delay could be longer).
>
>For more information, check out the MDN articles on [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/window.setTimeout) and [setInterval](https://developer.mozilla.org/en-US/docs/Web/API/window.setInterval). Some environments provide additional async primitives: node.js has [`process.nextTick`](http://nodejs.org/api/process.html#process_process_nexttick_callback) and many browsers support [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/window.requestAnimationFrame) - which also provides a means to queue a callback to be invoked before a repaint.

Even our contrived example above shows you that the JavaScript code you write will need to handle the reality of asynchronicity. Whether it's waiting for an AJAX response, a button click, a mouseover or any number of other possibilities, there's no escape. No need to worry - we have the tools to meet the challenge. The first one we'll cover is the foundation upon which all the others stand.

#1.) Callbacks
Callbacks are the currency…the fundamental building block…the *lynchpin* of asynchronicity in JavaScript. No matter where you go from here, the higher-level abstractions are just intelligent wrappers (for good reason, btw) around plain, vanilla callbacks. If you're new to JavaScript – and depending on what language you hail from – seeing functions passed as arguments might be a bit unfamiliar, but don't fret, it's easy and fun:

```
// plain, non-jquery version of hooking up an event handler
var clickity = document.getElementById('clickity');
clickity.addEventListener("click", function(e) {
	//console log, since it's like ALL real world scenarios, amirite?
	console.log("Alas, someone is pressing my buttons…");
});

// the obligatory jquery version
$("#clickity").on("click", function(e) {
	console.log("Alas, someone is pressing my buttons…");
});
```

Even if you're new to JavaScript, if you've spent *any* amount of time in the web, you've probably written something similar to the above code. Both `addEventListener` and `on` take a callback argument - the second argument being passed. This function will be invoked when the `click` event occurs. At the moment, we're just taking advantage of the fact our DOM element (presumably a button) has the underlying functionality to store the callback we pass to it and invoke it later when the event occurs (btw - this is an implementation of the [Observer Pattern](http://en.wikipedia.org/wiki/Observer_pattern), more on that later when we talk about Events). However - it's very easy to write your own objects to be capable of invoking callbacks:

```
var worker = {
	updateCustomer: function(customerInfo, cb) {
		$.ajax({
			url: "/some/api",
			type: "POST",
			data: customerInfo
		}).done(function(data){
			// let's pretend there's some DOM to update…
			$("#update-status").removeClass("editing").addClass("saved");
			cb(null, data);
		}).fail(function(data){
			// let's pretend there's some DOM to update…
			$("#update-status").removeClass("editing").addClass("failed");
			cb(data);
		});
	}
	// other methods, properties, etc
};

// and somewhere else, the calling code:
// (assuming we already have a currentCustomer instance, etc)
worker.updateCustomer(currentCustomer, function(err, data) {
	alert(err || data.msg); // *sigh*, who really uses alerts these days?!
});

```
>There are some problems with the above `worker` instance (it's tightly coupling AJAX & DOM updates, for one), but we'll overlook those *for now*.

Since functions in JavaScript are [first class citizens](http://en.wikipedia.org/wiki/First-class_function), we are able to pass them as arguments to other functions. In our example above, the `updateCustomer` method takes two arguments, the second of which – `cb` – is a callback function that is to be invoked once `updateCustomer` has completed executing. This is the essence of passing callbacks, and writing methods that take callback arguments - it doesn't get much lower-level than this. This is often referred to as "[continuation passing style](http://en.wikipedia.org/wiki/Continuation-passing_style)" (CPS). The callback is the "continuation", and the invoked function passes control to it as it invokes it, passing in the result(s).

I should mention that you will often run into situations where you want to specify the "context" of your callback function (i.e. - what "this" will be inside the function when invoked by the method to which you are passing it). You will need to bind your function (in most cases) prior to passing it as an argument. For example:

```
// assuming we have a notifier instance, with a "showAlert" method on it...

// ES5 bind function (or using a shim)
worker.updateCustomer(currentCustomer, function(err, data) {
	this.showAlert(err || data);
}.bind(notifier));

// using underscore/lodash
worker.updateCustomer(currentCustomer, _.bind(function(err, data) {
	this.showAlert(err || data);
}, notifier));


// using jquery
worker.updateCustomer(currentCustomer, $.proxy(function(err, data) {
	this.showAlert(err || data);
}, notifier));
```

In the examples above, we've bound the "this" context of our callback to be the `notifier` instance, so that `this.showAlert` inside our callback function will be the same as if the worker had access to call `notifier.showAlert`. But why is this useful?

The method we pass our callback into may belong to an instance that has no knowledge of (nor even a reference to) the object calling it - this is a good thing! It's called separation of concerns & encapsulation. However, if we're passing a callback that's actually a *method* on the calling object, we have to bind the context, or we're going to have a bad experience. Let's check a slightly modified example:

```
var updateForm = {
    submit: function() {
    	// get the data and store it in currentCustomer
    	worker.updateCustomer(currentCustomer, this.showAlert.bind(this));
    },
    showAlert: function(err, data) {
    	// I don't care how, just show an alert :-)
    }
}
```

We could spend an entire post on binding and context alone, but we'll stop there. If you're interested in reading more, I recommend checking out "[Getting Into Context Binds](http://freshbrewedcode.com/jimcowart/2013/02/12/getting-into-context-binds/)".

##Callback Conclusion
###Pros

* **Simplicity** - Callbacks are simple! (The caveat being that you are responsible for maintaining the correct context, if needed.) You pass a function, it gets invoked at some point in the future.
* **Lightweight** - No extra libs required. Functions-as-first-class-citizens is built into the language. No need for additional code to make it work.

###Cons

* **Can Be An Insufficient Abstraction** - Sometimes you need additional 'sugar'. This is where patterns 4-5 will come into play, each build upon the foundation of callbacks in different ways.
* **Complex When Nested** - Callbacks are difficult to read, debug and maintain when deeply nested. The goto-example these days to describe this problem is "the pyramid of doom". Let's take a look...

```
var allTheCustomerThings;
$("#getCustomer").click(function(cust) {
	var id = $("#cust-id").val();
	getCustomer(id, function(cust) {
		allTheCustomerThings = cust;
    	getContacts(id,function(contacts) {
    		allTheCustomerThings.contacts = contacts;
    		getOrders(id, function(orders){
    			allTheCustomerThings.orders = orders;
    			getAccountsRecv(id, function(ar){
    				allTheCustomerThings.ar = ar;
    				// OK - we got all the data, NOW WHAT?! :-)
    			});
    		});
    	});
    });
});
```

The deeper the nesting, the more you see the pyramid. This is both a *real problem* and a *straw man*. It's real problem in that I've seen it happen - but primarily by developers new to JavaScript and/or developers not spending the time to properly plan and architect their application. It's a straw man in that it's often blindly cited as *the knee-jerk reason* to use promises, without proper attention being paid to good design, even *if* promises are used to mitigate the nesting. (There are legitimate and compelling reasons to use promises, of course! But I can just as easily slap promises on a terribly written code base only to eliminate nesting.) Low level patterns don't fix higher level design problems - so be sure to think through how you are modeling a problem in code if you end up with deep callback nesting, and consider it both a code *and* design smell that should prompt you to address the systemic issue(s) with the overall design and not just the nesting strategy.

#2.) [Observer Pattern](http://en.wikipedia.org/wiki/Observer_pattern) (a.k.a. Events)
One of the cons of plain callbacks – "Can Be An Insufficient Abstraction" – is the perfect segue to where events can be useful. What if we don't want to keep passing a callback each time we invoke the target method? What if we wanted our own callback to be invoked if *another piece of calling code* also invoked the target method? The [Observer Pattern](http://en.wikipedia.org/wiki/Observer_pattern) fits well here.

The what?!

The Observer Pattern isn't hard at all. You have a *subject* that emits events (often referred to as an "event emitter"), and *observers*, which register callbacks with the *subject* to be notified as to when those events occur. In other words, let's assume [Doug Neiner](http://dougneiner.com/) and I are playing Halo 4. [Alex Robson](http://freshbrewedcode.com/alexrobson/) tells me that he'd like to know when Doug steals one of my kills. In this case, I am the subject and Alex is the observer. Unfortunately, Alex will want to debounce his callback, since he will receive *a lot* of event notifications.

There are a few key details to bear in mind with the Observer Pattern:

* Observers must have a direct reference to the subject.
* Subjects are responsible for maintaining the internal state of subscriber callbacks
* JavaScript implementations of this pattern usually involve subscriber callback method signatures of 0-n arguments (in other words, every event could have a different signature).

Let's demonstrate the Observer pattern in code, describing our Halo 4 example (you can view this in a fiddle [here](http://jsfiddle.net/ifandelse/gstrW/)):

```
// This observer object can be mixed into any object, giving it the basic
// API necessary to add and remove subscribers as well as emit events
var observer = {
	// 'subscribers' will keep track of subscribers by event name
	// each event name subscribed to will be a member name on
	// this object, w/ the value as an array of objects containing
	// the subscriber callback and optional function context
    subscribers: {},
    
    // the 'on' method is used by subscribers to add a callback
    // to be invoked when a specific event is emitted
    on: function (event, cb, context) {
        this.subscribers[event] = this.subscribers[event] || [];
        this.subscribers[event].push({
            callback: cb,
            context: context
        });
    },
    
    // 'off' allows subscribers to remove their callbacks
    off: function (event, cb, context) {
        var idx, subs, sub;
        if ((subs = this.subscribers[event])) {
            idx = subs.length - 1;
            while (idx >= 0) {
                sub = subs[event][idx];
                if (sub.callback === cb && (!context || sub.context === context)) {
                    subs[event].splice(idx, 1);
                    break;
                }
                idx--;
            }
        }
    },
    
    // iterates over the subscriber list for 
    // a given event and invokes the callbacks
    emit: function (event) {
        var subs, idx = 0,
            args = Array.prototype.slice.call(arguments, 1);
        if ((subs = this.subscribers[event])) {
            while (idx < subs.length) {
                sub = subs[idx];
                sub.callback.apply(sub.context || this, args);
                idx++;
            }
        }
    }
};
// We're using jQuery's extend function to copy the observer
// object's members over a new object, creating the "jim" instance
var jim = $.extend({
    dangItDoug: function (numberStolen) {
        this.emit("stolenkill", numberStolen);
    }
}, observer);

var alex = {
    tauntJim: function(numStolen, dt) {
        $('body').append("<div>Jim, your incompetence has cost you " + 
        	numStolen + (numStolen > 1 ? " kills" : " kill") + ".</div>");
    }
};
// the "jim" instance is a subject, and we're subscribing to it, to 
// have alex.tauntJim invoked any time the 'stolenkill' event occurs
jim.on("stolenkill", alex.tauntJim);
var i = 0;
jim.dangItDoug(++i);
jim.dangItDoug(++i);
jim.dangItDoug(++i);
jim.dangItDoug(++i);
```

Many popular libaries already provide built-in event-emitting behavior – backbone.js & jQuery, for example. There are several other stand-alone implementations worth checking out as well – here are a few:

* [EventEmitter](https://github.com/Wolfy87/EventEmitter)
* [EventEmitter2](https://github.com/hij1nx/EventEmitter2)
* [monologue.js](https://github.com/postaljs/monologue.js)

##Events Conclusion
###Pros

* **Encourages Good De-coupling** - Using the Observer pattern will help you tease apart behaviors in your application since it's forcing you to think through the interactions that can occur. This leads to well-encapsulated components, and typically results in a design more resistant to the traps of deeply nested callbacks.
* **Lends Itself to Testability** - Due to the bias towards de-coupling that comes with this approach, you end up with components that are testable in isolation and easy to mock in most cases.

###Cons

* **Direct Reference Required** - This is the Achilles' heel for event-emitters. If you have a subject that's *very* interesting to most of your application, you will end up passing a reference to it everywhere. This begins to undermine any 'loose coupling' gains you've racked up, creating a kind of tight coupling that can be harder to detect until you have to change the subject's API and break most of your app in the process.

##Elephant, Meet Room
So, let's tackle what I've avoided until now:

* Many popular libraries (like jQuery) provide what looks like an "Observer pattern" implementation, when in fact they are acting as a mediator between observer and subject yet still presenting the API as if it were being called directly on the subject. This isn't a problem, of course! It's just good to know how things work…
* Many of these implementations provide event-emitting behaviors, with the intent that the API is on the prototype, or mixed-in as instance members, etc. However, many developers take these event-emitting libraries and create a singleton instance and use this instance as a generalized "emitter", so subjects no longer have the on/off/etc. calls themselves. When this happens, we are, in my opinion, leaving the observer pattern behind and we've begun to wander into the territory of messaging...

#3.) Messaging
Well, it turns out that [Jonathan Creamer](http://freshbrewedcode.com/jonathancreamer/) found out Alex is watching some friends play Halo 4 and he's curious to hear how things are going. Only problem is, he has no idea how to get to my house and, in fact, doesn't want to leave his own house. So he calls Alex and asks to be notified whenever we win or lose, when we start a game, and when we earn a medal. Alex is now acting as a mediator, or broker. Jonathan isn't subscribing directly to me or Doug (in fact, he may not have any idea who's playing), instead he's told Alex what he's interested in, and Alex will relay any relevant information back to Jonathan, whether it comes from me, Doug or someone else. This is the essence of "messaging" in JavaScript.

This is much like the Observer pattern, in that we have observers interested in events occuring - but we've introduced a third party to handle managing subscriptions and matching events (which we'll now refer to as messages) to the correct subscribers. There's some debate about the best pattern name to apply here, but here are some descriptions that can be legitimately argued:

* [Mediator](http://en.wikipedia.org/wiki/Mediator_pattern) - "Communication between objects is encapsulated within a mediator. Objects no longer communicate directly with each other, but instead communicate through the mediator."
* [Event Aggregator](http://martinfowler.com/eaaDev/EventAggregator.html) - "Channel events from multiple objects into a single object to simplify registration for clients."
* [Client-side Message Broker](http://www.enterpriseintegrationpatterns.com/MessageBroker.html) - "Receives messages from multiple destinations, determines the correct destination & routes the message."

<div style="margin-left:auto; margin-right:auto;text-align:center;">
    <div style="font-size:24pt;line-height:26pt;">Regardless of the actual pattern used,<br/> people still call it "PubSub".</div>
    <img src="./3rdparty.jpg">
</div>

I personally lean towards "message broker", though I'm guilty of using all three pattern names to describe this approach. Regardless, it's important to realize that we aren't trying to re-create full blown server-side message brokers (i.e. - RabbitMQ or ZeroMQ). Instead, this approach accepts the weaknesses that can appear with the Observer pattern, especially in larger applications, and provides one common dependency (the message bus) for all subscribers and publishers to use, rather than requiring direct references to subjects be passed all over the application. Here are some common features of JavaScript message bus implementations:

* The message "payload" uses a structured envelope, rather than 0-n arguments passed into a callback. This envelope contains the data of the message as well as other metadata about the message (event name, channel name, timestamp, etc.). *Inlcuding behavior on a message is strongly discouraged. Ideally, the entire message envelope should be preserved if passed to `JSON.stringify`.*
* Subscriber callbacks have a consistent signature. Thanks to the above point, subscriber callbacks usually end up with a 1 or 2 argument method signature that's consistent for every message.
* Where the Observer pattern might have "event" names, messages in JavaScript usually have "topics" that can often be hierarchical (or namespaced). This opens the door for wildcard bindings - we'll see that in a moment.
* Some, not all, implementations allow topics to be segmented by 'channel' - a logical grouping of topics – for both performance and architectural reasons.

##Why Would I Choose This Over Events?
You wouldn't. In fact, some of the most elegant scenarios I've seen used the Observer and Message Broker patterns in tandem. As you look at your application's JavaScript, you'll often see several distinct "bounded contexts" (to steal a phrase from Mr. Evans). Within these contexts (for example, a Backbone view and model), the Observer pattern fits very well, since the observers already have a direct reference to the subject. However, it's often the case that separate "contexts" (often in separate modules) within the application are interested in the same data. Instead of coupling those separate modules just to share the data, we can have both modules subscribe to a message bus. So - using messaging in tandem with events can help you de-couple components at a different level of abstraction as you move from finer grained instance-to-instance communication (events) up to module-to-module communication (messaging).

Another important benefit arises if you follow the general rule of "Don't publish behavior on a message" (in other words, your message data shouldn't have functions, only data). You now have the ability to easily extend the reach of your messages across iFrames, Web Workers or even Websocket connections. [postal.js](https://github.com/postaljs/postal.js) – a library I've written – has a [federation](https://github.com/postaljs/postal.federation) add-on that allows this kind of bridging. These approaches will become more important as larger numbers of web applications are taking advantage of background processing in Web Workers and iFrames.

##Less Talk, More Code
This example is using [postal.js](https://github.com/postaljs/postal.js) – a JavaScript message bus implemention I've created:

```
var jonathan = (function() {
	// postal allows you to group topics by channel.
	// Here we get a channel for "xbox" related topics.
	// From it we can publish and subscribe.
    var xboxChannel = postal.channel("xbox");
    var reactions = {
        newgame: function(data, env) {
            return "I hope this goes better than the last one did!";
        },
        newmedal: function(data, env) {
            return "Nice, a new medal - about time!";
        },
        lostgame: function(data, env) {
            return "Ouch, you got pwned!";  
        },
        wongame: function(data, env) {
            return "Stroke of luck, perhaps?";
        }
    }
    var module = {
        handleMessage: function(data, env) {
        	// if we have a reaction for this kind of message
        	// we'll call makeComment….
            if(reactions[env.topic]) {
                this.makeComment(reactions[env.topic](data, env));    
            }
        },
        makeComment: function(msg) {
            $('body').append("<div>" + msg + "</div>");
        }
    };
    
    // we are subscribing using the "#" wildcard
    // which would match ANY topic of any length
    // (postal follows the AMQP binding approach)
    // notice we're using a 'fluent' configuration 
    // method to provide the function context we
    // want to be present (the value of "this")
    // in our subscriber callback
    xboxChannel
        .subscribe("#", module.handleMessage)
        .withContext(module);
    
    return module;
}());

var chan = postal.channel("xbox");
// we're briding a socket.io connection to our message bus
var socket = io.connect(location.origin);
_.each(['newgame', 'newmedal', 'lostgame', 'wongame'], function(evnt){
	socket.on(evnt, function(data){
		chan.publish(evnt, data);
	});
});
```
The above example is just a *taste* of messaging. We have a websocket connection, over which we're subscribing to `newgame`, `newmedal`, `lostgame` and `wongame` events. As we handle those websocket events (which will happen asynchronously, btw), we publish the received data to our local message bus and any subscriber will receive the message without having to know anything about the websocket connection, nor any other subscriber to the same data. This kind of loose coupling lends itself to good testability, portability (across boundaries such as iFrames and Web Workers) and composability. When messaging is the communications glue between your modules, the message is the contract. As long as you honor the contract – or even go a step further and version your message types – you can compose your app of any number of drop-in modules that subscribe to the topics they need and adhere to message contracts on what they publish.

##Messaging Conclusion
###Pros
* **Promotes clean separation of concerns** - you don't have to build a large app. Instead, build several smaller ones and use messaging for communications between them.
* **Very testable** - it's super easy to fake a message or test for message output.
* **Complements events nicely** - Observer pattern 'events' work nicely for local concerns. Messaging complements this when those events need to be promoted to app-level messages.

###Cons
* **Can be prone to "boilerplate proliferation"** - you can mitigate this by writing app-specific helpers
* **Can be confusing to devs who are new to the concept in general** - take the proper time to explain and teach.

#4.) Promises
Promises provide a very powerful way to express asynchronous code. Instead of "[continuation passing style](http://en.wikipedia.org/wiki/Continuation-passing_style)" (which we see with plain callbacks), our target methods return a promise - an "eventual value". The [Promises/A spec](http://wiki.commonjs.org/wiki/Promises/A) defines a promise as "an object that has a function as the value for the property then". The `then` property on a promise is a method that takes a success and (optional) error callback argument, and returns a new promise. Under the hood, a promise can be in 1 of 3 states: unfulfilled, fulfilled or failed. Once it has been fulfilled (resolved) or failed (rejected), the state should NOT change again. Success callbacks are invoked when the promise is fulfilled, error callbacks when it fails. If the success handler(s) are added after the promise has already been fulfilled, they will be invoked immediately (same goes for error – if the promise failed – and 'finally' callbacks).

> It's my understanding that the Promises/A+ spec states that success/error handlers added after a promise has fulfilled/failed will be queued to execute with a setTimeout of 0ms. This will potentially be a change from current Promises/A compliant libs. I'm happy to correct this if I'm wrong here.

You get an idea of the power a promise provides when you see it in action. Consider this nested-callback-laden login viewmodel:

```
// functional viewmodel for a mobile login screen
// the ugly nested callback version
var loginViewModel = (function () {
  var login = function () {
    var username = $('#loginUsername').val();
    var password = $('#loginPassword').val();
    el.Users.login(
      username,
      password,
      function () {
        usersModel.load(
          function () {
            mobileApp.navigate(
              'views/notesView.html',
              function () {
                // YAY! We made it!
              },
              function (err) {
                showError(err.message);
              });
          },
          function (err) {
            showError(err.message);
          });
      },
      function (err) {
        showError(err.message);
      });
  };
}());
```

What would this look like if we could use promises instead?

```
// Converted to use promises
var loginViewModel = (function () {
  var login = function () {
    var username = $('#loginUsername').val();
    var password = $('#loginPassword').val();
    el.Users.login(username, password)
      .then(function () {
        return usersModel.load();
      })
      .then(function () {
        mobileApp.navigate('views/notesView.html');
      })
      .then(
        null, // YAY! We made it!
        function (err) {
          showError(err.message);
        }
    );
  };
  return {
    login: login
  };
}());
```
Arguments over 'readability' are often a time-sucking-black-hole-of-subjectivity, but this is an instance where I will definitely argue that promises made this code *much* more readable. The important thing to realize is that this isn't just syntactic/API sugar. Promises enable developers to think and express asynchronous code in a more synchronous way - giving you back return values (promises) and enabling error handling in a convenient way. Note that in our snippet above, the very last `then` call handles *any* error that could occur while logging in, loading the users model, or navigating to the notesView.html view.
##Taking it Further
While the basline of the spec defines a promise as an object that has a function value for the `then` property, popular promise libraries provide additional utilities that you will quickly become addicted to! For example, the [Q](https://github.com/kriskowal/q) library provides a `spread` method that allows you to pass an array of promises, and the result of each promise will be passed, in order, as arguments to the fulfillment handler. Let's re-visit one of our callback examples and apply this:

```
// In this refactor, our getCustomer, getContacts, getOrders &
// getAccountsRecv methods now all return promises
$("#getCustomer").click(function(cust) {
    var id = $("#cust-id").val();
    Q.spread([
    		getCustomer(id),
    		getContacts(id),
    		getOrders(id),
    		getAccountsRecv(id)
    	],
    	function(cust, contacts, orders, ar){
    		cust.contacts = contacts;
    		cust.orders = orders;
    		cust.ar = ar; 
    		// Now we can do something with
    		// our fully-hydrated customer
    	}
    );
});
```

If you've used promises before, it's quite likely that your library of choice provides shorthand methods for adding only a success, or error or "finally" callback. jQuery provides `done`, `fail` and `always`. Q provides `fail` (or `catch`) and `fin`. Due to the overwhelming populariy of jQuery, I've encountered many developers that think promises are defined by `done`, `fail` and `always` being present, rather than `then`. This leads me to a complaint…*\*pulls pin, throws grenade\**…

##Conversation Grenades
My first exposure to promise implementations in JavaScript was through jQuery. When I compared it to the Promises/A spec, I was very frustrated! jQuery didn't return a new promise from `then` (until version 1.8) - so developers depending on jQuery were not only *not* getting the full power of promises, but they were, IMO, cranking out a lot of code that didn't play well with other promise implementations that *did* follow the spec. Using a properly implemented promises library should mean that you "[can write very extensive libraries that are entirely agnostic to the implementation of the promises they accept](http://domenic.me/2012/10/14/youre-missing-the-point-of-promises/)". I don't think anyone has explained this issue better than Domenic Denicola in his post "[You're Missing the Point of Promises](http://domenic.me/2012/10/14/youre-missing-the-point-of-promises/)" – I highly recommend reading it. If all the major promise lib authors would follow his advice, my complaints about having a non-standard promise implementation forced on me when I use libs that take it as a dependency would disappear entirely.

##Promises Conclusion
###Pros
* **Reduces Complexity of Nesting & Flow** - Flattening pyramids of doom is nearly always a win. Being able to express certain aspects of asynchronous code in a synchronous style (with return values, etc.) empowers developers at nearly any level to quickly understand what a section of code is doing.
* **Results Can be "Cached"** - If you need to keep a resolved promise around, any additional success handlers added to it are immediately invoked and passed the resulting value. This can save you some boilerplate as well as prevent unnecessary operations (like a duplicate AJAX request, for example).

###Cons
* **Results Can be "Cached"** - Wait - wasn't this a PRO?! Yes - but it's a double-edged sword. If you're keeping promise instances around, avoid doing so for values that frequently change (and, for example, might require another HTTP request to fetch the latest).
* **Opinionated on OSS APIs** - If you're an open source author writing libraries for general consumption, please use a spec-compliant library! When you don't, you force a highly opinionated dependency on consuming developers.
* **Many Non-Spec-Adhering Implementations Exist** - Choosing one of these can get you past that:
	* [Q](https://github.com/kriskowal/q)
	* [when.js](https://github.com/cujojs/when)
	* [RSVP.js](https://github.com/tildeio/rsvp.js)

##One More Thought
Promises are not the only alternative to plain callbacks. We've already discussed eventing and messaging, and we're about to discuss finite state machines (FSMs). In my experience I've found promises to be a powerful tool for concise units of behavior (handling a login, handling an HTTP request, etc.). However, I've seen promises used in longer-running workflows where an FSM is better suited to the task, and also in high level UI management between views where something more reactive (events, messaging) would have been a better choice.

#5.) Finite State Machines
Sometimes you need an abstraction that can react differently to the same input. No - I promise I didn't just lose my marbles. Consider a client-side router in a single page app: you may want any attempts to navigate while the user isn't authenticated to do *nothing* other than direct them to the login screen. However, once they are authenticated, the same attempts to navigate should actually take them to the desired view(s), etc. In [another post](http://www.icenium.com/blog/icenium-team-blog/2013/05/09/is-this-thing-on-(part-2), I described Finite State Machines (FSMs) this way:

> "A finite state machine is a "computational abstraction" – in other words, it's something we use in order to compute a yes or no answer to a question – that is capable of reacting differently to the same input, given a change in its internal state."

You're already using state machines, and may not even realize it. The most basic of FSMs can be produced by a simple switch statement:

```
var fsmLite = (function(){
	var _state = "notReady";
	var _result;
	// We're watching a subject somewhere to know
	// when we can move to ready state
	someSubject.once("ready", function() {
		_state = 'ready';
	});
	return {
		doSomething: function(callback) {
			switch(_state) {
				// if someone wants us to "doSomething"
				// but we aren't ready, we invoke their
				// callback and pass an error message.
				// Other strategies are applicable here
				// as well - such as queueing up any
				// callbacks to be invoked once we reach
				// the "done" state, etc.
				case 'notReady':
					callback("Not ready yet!", null);
				break;
				// we actually do the work
				case 'ready':
					worker.doWork(function(result) {
						// Do other work, etc. and then:
						_result = result;
						_state = 'done';
						callback(null, _result);
					});		
				break;
				// The work has already been done, we just
				// invoke the callback and pass the result
				case 'done':
					callback(null, _result);
				break;
			}
		}
	}
}());
```
The above example responds differently to the same input (`doSomething`), depending on the `_state` value. It also demonstrates a few other FSM characteristics:

* It has a finite number of states in which it can exist
* It can only exist in one state at a given time
* It accepts input
* It might produce output (in this case, the `_result` value passed to the `callback`)
* It can transition from one state to another

##Promises are Specialized State Machines
You might have already put these together, but a promise is an FSM. It can be in one of three states: unfulfilled, fulfilled or failed. It accepts input (at least) from callers using the `then` method. It transitions from unfulfilled to fulfilled or failed, and won't transition again. Let's take a look at a directed graph representation of a promise-FSM (this is a common way to represent FSMs visually):

![](./PromisesFSM.png)

Each circle represents a state in which the promise can exist. The arrows represent the input each state handles (`then`, `fulfill` and `fail`). If the arrow points back to the state, this means the input does not cause a transition to a new state. If the input *does* cause a transition, the arrow points to the new state. Both the `Fulfilled` and `Failed` states have "entry actions" - behavior performed as soon as we enter the state (listed next to the "E:" on the state bubble).

##When Would I Use This?
Once you grasp the nature of an FSM, it's hard to *not* see areas where it could serve well. A few of the areas I've encountered where FSMs have been a huge help are:

* **Online/Offline connectivity**. (In fact I have a [series on this topic already](http://www.icenium.com/blog/icenium-team-blog/2013/04/23/is-this-thing-on-(part-1).)
* **Managing Views**. For example, a Backbone view might be in an "accepting-input" or "not-accepting-input" state. The view still presents the same API to any calling code, but will respond differently based on the state. This is useful for submitting forms or any number of other scenarios.
* **Initialization/Bootstrapping**. Standing up a single page app, or a [RabbitMQ client](https://github.com/a2labs/amqp-bootstrapper) in node.js (just to name two examples) can involve a number of asynchronous calls that need to be performed in a certain order, and may involve a need to replay part or all of that workflow. FSMs fit this like a glove.
* **State-based Routing**. I mentioned this in the leading example. You might want your client side router to behave differently based on some state in the app.
* **Wizard/Step-Driven UI**. You can use an FSM to control a wizard-style workflow - and even have sibling/reactive FSMs that drive menu-ing, etc. in tandem with the wziard FSM.

##Can I Get Some Help Here?
Rolling your own FSM is easy - but you might quickly outgrow the limited capabilities. I created [machina.js](https://github.com/ifandelse/machina.js) because I wanted a low-level FSM utility library packed with some powerful features. Let's convert our switch-based FSM above to a machina FSM (be sure to read the comments!):

```
var fsm = new machina.Fsm({
	
	// This will be the FSM's starting state
	initialState: "notReady",
	
	// initialize is invoked as soon as the
	// constructor has completed executing
	initialize: function() {
		var self = this;
		someSubject.on("ready", function() {
			self.handle("someSubject.ready");
		});
	},
	
	// The states object lets you organize your
	// states and input handlers. Each top level
	// member name is a state. The state object 
	// for each state contains function handlers
	// OR a string value, indicating the only
	// action is to transition to the state name
	// specified in the string value.
	states: {
		notReady: {
			"do.something" : function() {
				// deferUntilTransition is one of my
				// favorite machina features. It queues
				// the input up to be replayed in the
				// specified state name. The argument
				// is optional, and if not provided
				// machina will try to replay this
				// input again on the next transition
				this.deferUntilTransition("done");
			},
			// machina gives you a shortcut by allowing
			// the input handler value to be string state
			// name if the only reaction to an input is
			// to transition to a new state
			"someSubject.ready": "ready"		
		},
		ready: {
			"do.something" : function() {
				worker.doWork(function(result) {
					// Do other work, etc. and then:
					this.result = result;
					// machina FSMs are event-emitters!
					this.emit("result.ready", this.result);
					this.transition("done");
				});		
			}
		},
		done: {
			"do.something" : function() {
				this.emit("result.ready", this.result);			}
		}
	},
	
	// this is a top level method that wraps the "handle"
	// call that machina provides.
	doSomething: function(callback) {
		// we could have easily required that callers
		// subscribe to events themselves, but we're
		// trying to be nice here
		this.once("result.ready", function(result){
			callback(result);
		});
		this.handle("do.something");
	}
});
```

So - machina provides features such as (but not limited to): 

* providing custom post-construction initialization behavior
* deferring input until a later time without requiring the caller to do anything
* the ability to provide "catch-all" handlers to match unexpected input (this isn't shown above - check the examples in the repo for more information)
* FSMs are event emitters, so other components can observe them as subjects
* \_onEnter and \_onExit handlers for any state (not shown above)

##Complementary Patterns 
You'll notice that our FSM above makes use of continuation-passing as well as events. In fact, our entire journey through these 5 patterns has progressively advanced to higher-level abstractions. Don't be surprised to see yourself mixing these patterns together to produce the abstraction you need. If anything, focusing on *just* the lower level pattern of passing callbacks and using promises to eliminate nesting has, in my opinion, done our community a disservice. In addition to grasping these patterns, we need to see the potential of higher-level abstractions which we can create from these patterns working *together*.

##FSM Conclusion
###Pros
* **Extremely Versatile** - FSMs model so many real-world problems well, it's hard to find a scenario that they don't work well for.
* **Wokflow** - FSMs handle long-running, asynchronous workflow well.

###Cons
* **Lesser known Pattern** - FSM awareness in the JavaScript developer community *appears* to be low. As a result, the pattern is often not known, feared or rejected as too complex.

#Wrapping Up
So - we survived our whirlwind tour together through these 5 patterns. It's worth emphasizing that these aren't the *only* five that can be useful in taming asynchronous code, but I do consider them "*must-know*". I'm interested in your feedback & war stories, as well - what have you found to be useful in this space?

#Additional Reading
##Callbacks/Continuation Passing Style
* ["Asynchronous programming and continuation-passing style in JavaScript"](http://www.2ality.com/2012/06/continuation-passing-style.html) by [Dr. Axel Rauschmayer](https://twitter.com/rauschma)
* [A beginner's primer on callbacks in JavaScript](http://recurial.com/programming/understanding-callback-functions-in-javascript/) 
* [By example: Continuation-passing style in JavaScript](http://matt.might.net/articles/by-example-continuation-passing-style/)

##Eventing & Messaging
* [Client-side Messaging Essentials](http://www.freshbrewedcodes.com/jimcowart/2013/02/07/client-side-messaging-essentials/)
* Client-side Messaging in JavaScript – [Part 1](http://www.freshbrewedcodes.com/jimcowart/2011/12/05/client-side-messaging-with-postal-js-part-1/), [Part 2](http://www.freshbrewedcodes.com/jimcowart/2012/02/02/client-side-messaging-in-javascript-part-2-postal-js/) and [Part 3 (anti-patterns)](http://www.freshbrewedcodes.com/jimcowart/2012/03/19/client-side-messaging-in-javascript-part-3-anti-patterns/)
* [Cross Frame Messaging with postal.xframe](http://www.freshbrewedcodes.com/jimcowart/2013/02/26/cross-frame-messaging-with-postal-xframe/)
* [Patterns For Large-Scale JavaScript Application Architecture](http://addyosmani.com/largescalejavascript/)

###Libraries
* [postal.js](https://github.com/postaljs/postal.js) (shameless plug on my part)
* [EventEmitter](https://github.com/Wolfy87/EventEmitter)
* [EventEmitter2](https://github.com/hij1nx/EventEmitter2)
* [monologue.js](https://github.com/postaljs/monologue.js)
* [js-signals](https://github.com/millermedeiros/js-signals)
* [msgs](https://github.com/cujojs/msgs)


##Promises
* [You're Missing the Point of Promises](http://domenic.me/2012/10/14/youre-missing-the-point-of-promises/)
* [What's the Point of Promises](http://www.kendoui.com/blogs/teamblog/posts/13-03-28/what-is-the-point-of-promises.aspx)
* [Promises/A+ spec](http://promises-aplus.github.io/promises-spec/)
* [Async JavaScript (book by Trevor Burnham)](http://pragprog.com/book/tbajs/async-javascript)

###Libraries
* [Q](https://github.com/kriskowal/q)
* [when.js](https://github.com/cujojs/when)
* [RSVP.js](https://github.com/tildeio/rsvp.js)


##Finite State Machines
* [State Machines – Basics of Computer Science](http://blog.markwshead.com/869/state-machines-computer-science/)* [http://machina-js.org/](http://machina-js.org/) (shameless plug!)* [Finite state machines in JavaScript, Part 1](http://www.ibm.com/developerworks/library/wa-finitemach1/) (great additional reading resources on this one)* [Learn You Some Erlang - Finite State Machines](http://learnyousomeerlang.com/finite-state-machines)
* [Harvey Mudd CS paper on FSMs](http://www.cs.hmc.edu/~keller/cs60book/12%2520Finite-State%2520Machines.pdf)
* [Finite State Machine - Wikipedia](http://en.wikipedia.org/wiki/Finite-state_machine)
* Is This Thing On? - [Part 1](http://www.icenium.com/blog/icenium-team-blog/2013/04/23/is-this-thing-on-(part-1), [Part 2](http://www.icenium.com/blog/icenium-team-blog/2013/05/09/is-this-thing-on-(part-2), [Part 3](http://www.icenium.com/blog/icenium-team-blog/2013/05/30/is-this-thing-on-(part-3) and [Part 4](http://www.icenium.com/blog/icenium-team-blog/2013/06/04/is-this-thing-on-(part-4)
* [Taking Control With machina.js (presentation by Doug Neiner)](http://code.dougneiner.com/presentations/machina/)

###Libraries
* [machina.js](https://github.com/ifandelse/machina.js)
* [state](https://github.com/nickfargo/state)
* [javascript-state-machine](https://github.com/jakesgordon/javascript-state-machine)