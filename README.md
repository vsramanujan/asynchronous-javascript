## Agenda

### Async Patterns
- Parallel vs Async
- Callback
- Thunks
- Promises
- Generators / Coroutines
- Event Reactive (observables)
- CSP (Communicating Sequential Processes) - channel oriented concurrency

---

## Parallel vs Async


- Parallel:
	
    - Mostly denoted by threads
    - Doing multiple things at the exact same moment
    - Most computers have multiple cores, but threads:core is not 1:1
    - Since we don't have infinite number of cores the OS has what we would call `Virtual Threads` and the scheduler takes care of scheduling those virtual threads across cores
    - Its about optimization

- Async: 
	
    - JS is always running only on 1 thread
    - Web workers are still a completely new JS instance so they don't make JS multi threaded
    - Closely associated with concurrency

- Concurrency - 
	
    - Two higher level tasks thats happening within the same timeframe
    - So don't think of the time period as an instant but rather a timeframe
    - So think of user kicking of an AJAX request and then trying to scroll the page
    	- If we were to do them sequentially, aka finish the ajax request first and then do the UI scroll, it would be a nightmare user experience
        - Instead what happens is that the AJAX request is broken down in to micro tasks and so is the UI scroll, and these micro tasks are interleaved to give the perception of concurrency

##### Things that might be happening / waiting to happen on a web page at any point in time:
1. Response callback to async requests
2. Garbage colelctor
3. Style CSS repainter
4. Layout engine

So these tasks are just queued in the event loop -> they are done so in a sort of FCFS manner


#### _So, asynchronous programming, at the end of the day is basically figuring ways to manage concurrency in your program!_

---

## Callbacks

- Think about callbacks as wrapping up some code that you want to execute 'later' (we don't know when that later will be)
- So there exists a notion of __"now"__ and __"later"__
- So _callbacks = continuations_!

#### Callback hell

- It is nothing to do with indentation or nesting!
- Much deeper conceptual deficiency 
- You could rewrite your indented code using continuation passing style which will remove all indentation -> but it doesn't get rid of callback hell

``` js
function one(cb) {
    	console.log("one");
    	setTimeout(cb,1000);
}

function two(cb) {
    	console.log("two");
    	setTimeout(cb,1000);
}

function three(cb) {
    	console.log("three");
    	setTimeout(cb,1000);
}

one(function () {
    two(three);
});
```

- The above code is also an example of callback hell but since it is written in continuation passing style -> you just don't see the indentation


#### EX 1 -  3 AJAX CALLS -> 3 CALLBACKS -> FILES 1, 2, 3, AND RETURNED -> YOU ONLY RENDER THEM IN ORDER BUT THEY CAN COME BACK IN ANY ORDER -> SOLVE USING ONLY FUNCTIONS + CLOSURES

##### SOLUTION
You cannot solve this without having some global state between the 3 callbacks so that you can coordinate!


#### TWO PROBLEMS WITH CALLBACKS

1. Inversion Of Control -
	- There is a part of my code that I am in control of executing and then there is an other portion that I am not in control of executing
    
    - So whenever there's a callback, there is a trust that the caller of the callback
    	
        - doesn't call it too many or too few times. 
        - doesn't call it too few times
        - not too late
        - not too soon
        - no lost context
        - no swallowed errors
        
2. They are not reasonable 

	- The only way to express temporal dependecies (one starts only after something finishes) using callbacks is by nesting
    
    - So what happens is that we are forced to keep taking these jumps in our code to actually figure out the path of execution
    
    
#### Some unsuccessful attempts at fixing callbacks

1. Split callbacks -
	- Have a separate callback for err and a separate one for success
    - Just made the problem worse since now you had 2 callbacks for which you had to maintain the above things for
    - What if they called both callbacks?
    
2. Callback with 2 parameters - 
	- Have 1 callback with 2 params (err, res) 
    - This is how Node callbacks are designed
    - Well, the same problems remain, and this doesn't really take us anywhere
---    
    
## Thunks

#### Synchronous thunk
- From a sync perspective, a thunk is a function that has everything already that it needs to do to give you some value
- So you just call it and it will give you a value back
- So a thunk is just a function with some closured state keep track of some values, and when called it will give the result over that state

```js

var thunk = function() {
    return add(10,15);
}

```

- So the thunk itself has become a container around that particular collection of state (aka 10,15 here)
- This container can now passed around anywhere in my program and to extract the value I just need to call it
- So I don't need to pass the state itself around, but only the container needs to be passed around

- __This is the fundamental underpinning for what a promise is - a wrapper around a value__

- Here the wrapper is a function, in a promise its a much more sophisticated thing


#### Async thunk

- A function that doesn't need any arguments passed to it to do its job, except for a callback so that you can get the value out

```js

var thunk = function(cb) {
	addAsync(10,15,cb);
}

```


##### Why is this a powerful pattern?

- When modeled as an async thunk, from the outside world, we do not know, nor do we have to care, whether that value is available immediately or whether it is going to take a while to get us that

- So by wrapping this function around the state and allowing it to be async in nature, we have essentially normalized/factored time out of the equation

- Hence this wrapper is now __*time independent!*__ 

- Time is the most difficult piece of state any of us has ever had to manage!


``` js
// makeThunk general utility
function makeThunk(fn) {
    var args = [].slice.call(arguments,1);
    return function(cb) {
        args.push(cb);
        fn.apply(null,args);
    }
}

var thunk =  makeThunk(addAsync, 10, 15);

thunk(function(sum) {
    	console.log(sum);
})
```

- This makeThunk is actually similar to a promise constructor

- Important to understand that thunks do not solve the problem of inversion of control we went over before -> promises do that

- Thunks are basically the conceptual underpinning for promises

- Now the version that we are seeing is what is called as __*lazyThunk*__. It doesn't calculate the value until you invoke it

- You could come up with another type of thunk called as __*activeThunk*__, which did the work and just held on to its response


- So essentially, if you want to do things in parallel, you're going to have to have activeThunks! If you want to do them in sequence, you can use lazyThunks


Lets rewrite EX1, to appreciate what thunks can actually do for us - remember that we want all async actions to happen parallely : 

```js
// Setup
function readFile(fileName) {
    fakeAjax(fileName, function() {
    	// To be written
    });
    return function(cb) {
        // Thunk function to be written
    }
}
```

- Now, assume you have an async function readFile that does the call for you. It accepts a callback, obviously.

- We've wrapped the function with our own function `readFile` which is going to be our thunk

- Why would our `makeThunk` not work here? Because, our makeThunk is a utility that only produces `lazyThunks` - and as we noted before - `lazyThunks` can only help us in modelling sequential invocations of actions

- Lets look at the usage first:

```js
function readFile(fileName) {
    fakeAjax(fileName, function() {
    	// To be written
    });
    return function(cb) {
        // Thunk function to be written
    }
}

var file1Thunk = readFile("file1");
var file2Thunk = readFile("file2");
var file3Thunk = readFile("file3");

file1Thunk(function(file1Contents) {
	console.log(file1Contents);
    file2Thunk(function(file2Contents) {
        console.log(file2Contents);
    	file3Thunk(function(file3Contents) {
            console.log(file3Contents);
            console.log("Completed");
        }));
    }));
}));
```

- Now, since we are now operating on the the `thunk` and not the async function itself, not only are we guaranteed the order here (without having to maintain complicated global state) - we are also executing all 3 async operations in parallel!

- We no longer need a global shared state between the 3 of these file reads that we would have needed in case we were not to use thunks!

- Last piece is to write the actual thunk: 

``` js
function readFile(fileName) {
    var contents, fn;
    
    fakeAjax(fileName, function(fileContent) {
    	if(fn){
            fn(fileContent);
        } else {
        	contents = fileContent;   
        }
    });
    
    return function(cb) {
        if(contents) {
            cb(contents);
        } else {
         fn = cb;   
        }
    }
}

var file1Thunk = readFile("file1");
var file2Thunk = readFile("file2");
var file3Thunk = readFile("file3");

file1Thunk(function(file1Contents) {
	console.log(file1Contents);
    file2Thunk(function(file2Contents) {
        console.log(file2Contents);
    	file3Thunk(function(file3Contents) {
            console.log(file3Contents);
            console.log("Completed");
        }));
    }));
}));
```


- Woah! Mind blown! So here, fakeAjax is basically kinda like a makeThunk for our activeThunks!

- To summarize: thunks aren't really useful in solving the callback hell problem or make our async code look more `sequential`. They are slightly better to reason about though. But they are the conceptual basis on top of which promises are built on - and they help us massively by removing time as a piece of state!

- So thunk -> codification of the pattern of managing state internally inside of a wrapper in a time independent fashion!

---

## PROMISES

- Promises are a codification of the idea that we need a placeholder to eliminate time as the concern wrapped around a value. We need a container around it. 

- This placeholder in programming is called a `Future`.

- Tidbits: 
	
    - Promises come to Javascript from a language called *__E__*
	
    - Promises, through their mechanism (not much so because of their API), is bascially a *__monand__* (functional programming nerds, calm down)

- Promises also uninvert the control (actually not invert it in the first place) during the handling of asynchronous code. Instead of you giving a callback to the async code and asking it to invoke it when done (the inversion of control we saw before), you expect the called async code to give you back something - namely, an Event Listener. We can then subscribe to the *__completion event__* of that Event Listener

- In fact, in addition to an *__completion event__*, we could also listen to an *__error event__* or even other events that the API might want to notify the caller about


- So we could do something like this now,

```js
var listener = trackCheckout(purchaseInfo); // async call

listener.on("completion", finish);
listener.on("error", error);

function finish() {
    chargeCreditCard(purchaseInfo);
    showThankYouPage();
}

function error(err) {
    logStatsError(err);
    finish();
}
```


- Here, since the event listener is in our control, we can do what we want with it. We could also do things like unsubscribe to the event the first time it happens for eg, thus alleviating any problems of multiple calls etc

- Hence we've uninvert the inversion of control problem here!

- That is important to understand about promises -> they uninvert the inversion of control problem

- But we don't listen to the "completed" event in a promise exactly using the way mentioned above - but its an important conceptual base for understanding what a promise is like - a promise is **like** an event listener

- And instead of calling it the "completed" event, we call it the then event


``` js 
function trackCheckout(purchaseInfo) {
    return new Promise(
        function(resolve,reject) {
            
            // attempt to track checkout here
            // if success, call resolve
            // if failure, call reject
        }
        );
}


var promise = trackCheckout(purchaseInfo);

promise.then( finish, error); // This is sort of like registering for the then event (metaphorically)

function finish() {
    chargeCreditCard(purchaseInfo);
    showThankYouPage();
}

function error(err) {
    logStatsError(err);
    finish();
}
```

- Tidbit: Inversion Of Control is one of those key features that differentiates a framework from a library. It is a proper design choice in a lot of frameworks. But as far as asynchrocity is concerned, its a terrible idea


- Important to see here: we still have a lot of callbacks! In fact, we may be having more callbacks than before! But those were not the problem that promises were created to solve!

- Promises instill trust:
	
    1. Only resolved once
    2. Either success OR error
    3. Messages passed/kept
    4. Exceptions become errors - nothing is swallowed
    5. Immutable once resolved
    
    
- We can actually think about promises as simply "callback managers". It is a pattern of managing our callbacks in a trustable fashion

- The immutability part of it (points 3 and 5) are what make promises very appealing. You can confidently pass around your value anywhere and be confident that nobody can change it without you knowing it. It avoids what is known in software engineering as __*action at a distance*__.


#### Flow control

- Flow control is basically the concept of saying 'do task 1, then 2 , then 4 and 3 parallely and finally 5'


- API design decision to allow this : __chaining promises__


Consider the following chain:

``` 
doFirstThing
	then doSecondThing
    then doThirdThing
    then doFourthThing
or error
```

- Essentially if anywhere in the chain an error happens, come out and execute the __**error chain**__

- How does chaining happen? **You return the second promise from the success handler of the first promise**

Eg: 
```js
doFirstThing()
.then(function() {
    return doSecondThing();
})
.then(function() {
    return doThirdThing();
})
.then(
    complete,
    error
);
```


EX3 - WRITING THE FILE THINGY WITH PROMISES

``` js

function getFile(file) {
 	return new Promise(function(resolve,reject) 		{
    	fakeAjaxCall(file,resolve);    
   	})
    
}

```

- This process of wrapping a callback only async function with a promisification wrapper is called `lifting`

- Most promise libraries already provide this utility out of the box since this is a pattern that is reused in a lot of places


``` js
function getFile(file) {
 	return new Promise(function(resolve,reject) 		{
    	fakeAjaxCall(file,resolve);    
   	})
    
}

var p1 = getFile("file1");
var p2 = getFile("file2");
var p3 = getFile("file3");

p1
.then(output)
.then(function() {
    return p2;
})
.then(output)
.then(function() {
    return p3;
})
.then(output)
.then(function() {
    output("Complete");
});
```

- We could have had the `output` to be the first line of the function that returns p2 as well. But as a general programming practice, keep functions as single purpose as possible. This also shows that you can `.then` a purely sync function

- When you `.then` a purely sync function, it immdiately goes to the next block in the chain, but when you do return a promise from a `.then` block, it waits for the promise to resolve. This is the crux of promise chaining

- Also, you could have multiple `.then` handlers to the same promise. For eg:

```js
p1
.then(function(res) {
    console.log(res); // Called when P1 is resolved
}

p1
.then(output) // Also called when P1 is resolved
.then 
...
```

- The only assurance that the promise API gives you is that the resolved value of a promise is immutable

- Under the covers, `.then` actually returns its own promise. This promise is by default implemented to resolve immediately, **__UNLESS__** it itself returns a promise 


- When you have multiple `.thens` registered to the same promise, the execution of those handlers is register first order. But it is really bad programming practice to depend on such things.

- If any of the promises `rejects`, then each of the subsequent `promise.thens` which does not result to the reject, they automatically have an implied reject handler that simply propogates it to the next step. And hence the `rejection` would be propogated all the way through the chain

- It does not get swallowed by any of the `.then`. It is still present at the end of the last `.then`, it is just that we are not passing any rejection handlers to observe it

- The `default rejection handler` in each `.then` just takes the rejection and sort of 're-rejects' it


- So if you pass a second function to a `.then`, that behaves as the rejection handler for the chain. It won't even go to the `.catch` handler after it, if any.

- It is important to understand that any errors that occur anywhere inside the promise chain, will get converted into a promise rejection and is propogated down the chain

- Also, a `.catch` is basically the same thing as a `.then` where we don't pass anything to the success handler

```js
.then(null,() => console.log)
// is the same as
.catch(() => console.log)
```

- `.then` and `.catch` behave exactly like `try-catch`. If you have a catch in the middle of your chain, all errors before that will be caught there, and the rest of the chain will proceed as though nothings happened.

- So, if you want all errors in your chain to be fatal, have only 1 catch in the end

- So if you have an intermediate `.catch`, its return value will then be the input to the next `.then`

```js
// We could have easily written the above solution as 
p1.then(function(res1) {
    output(res1);
    p2.then(function(res2){
         output(res2);
    p3.then(function(res3){
        output(res3);
		output("Complete");
    	})
	})
})
```

- This is a very valid way of writing promises as well, but this is **BAD**

- The elegance of promises is that you can return a promise to return to the main chain so that you don't have to get into these `promise hell`


#### Limitation with the manual chaining pattern

- A fundamental limitation in chaining promises like how we've done for flow control is that you need to know how many promises you have ahead of time.

EX4: Same excercise, but arbitrary number of urls, we need to chain them

``` js
//getFile from last time
function getFile(file) {
 	return new Promise(function(resolve,reject){
    	fakeAjaxCall(file,resolve);    
   	})
    
}
const urls = ["file1", "file2", "file3"];

urls
.map(getFile)
// What we use as initial value for reduce is an 
// immediately resolved promise
.reduce(function combine(){/* We will write this below */},
        new Promise(function(resolve){ resolve()}));

// Turns out that there is an easier way to have
// an immediately resolved promise
urls
.map(getFile)
.reduce(function combine(acc,promise){
	return acc.then(function() {
        return pr;
    }).then(output);
},Promise.resolve())// basically creates and resolves a promise
.then(function() {
    output("Complete");
}); 

```

- We could write the reduce that we've written above as a utility that we use again and again


#### Abstractions 

- Promise. all
	
    - Takes an array of promises and waits till all of them finish
    - The `.then` of `promise.all` will take the result of all promises in the order give in the initial function, irrespective of when they finish
    - So, every step in a promise chain could essentially be a promise.all as well
    - The old school terminology for this is called a __**gate**__
    	- Gate is basically when you have multiple things that need to happen paralelly and you need to wait till all of them finish before moving on, that step is called a gate
   - If any of the pormises in a Promise .all rejects, then the final promise is rejected as well. So promise.all necessitates successful completion
   
   
   
- Promise.race
	- Also takes an array of promises, and returns the first promise that finishes (either success or failure) and ignores all other promises
``` js
var p = trySomeAsyncThing();

Promise.race([
    p,
    new Promise(function(_,reject) {
        	setTimeout(function() {
                reject("Timeout!");
            }, 3000);
    })
])
.then( success, error);

```
    
- So the above is how you would basically make sure that the promise just doesn't go on indefinitely and we set a timeout for it

- Promise.race will throw away all its references once completed so that if there are failures, the other promise can be garbage collected

- Most promise libraries have an higher order abstraction for setting out timeouts for promises

- We can think of a lot more abstractions here:
	
    1. First one to succeed
    2. Last one to finish
    3. See whether all calls reject
    4. Everything that finish before a time period
    
    
#### Sequence

- One abstraction that is particularly useful is to think about things in terms of a sequence of events

- Defn of sequence: 
	A list of automatically chained promises
    
- So basically, in our code, we might be creating promises a lot of times to chain off the rest (and wait for the current step to finish - since it would make sense to create promises in `.then` only when you have a callback there instead)


- Also, a thing to note is that `.then` function is polymorphic. It changes its behaviour based on what is passed in. Generally, understanding polymorphic functions could be confusing


- Sequences are promise replacements in a library called `asynquence` which is one of the many promise libraries out there

----

## Generators

- If promises were about the solving of the inversion of control problem in callback hell, generators are used to address the non-local, non-sequential problem in callbacks

- A fundamental and powerful property of JS is that they have this property of `run-to-completion`

- The real idea behind generators: Gives the ability to create a syntactic form of a declaring a state machine

- Used to implement `cooperative concurrency` in JS

- When you call a generator -> it yields an `iterator`. They're a programmatic way to step through a list (generally)

- But the iterator of a generator is used to step through the control of our generator instead of some data

``` js
function* gen() {
	console.log("Hello");
    yield;
    console.log("World");
}

const it = gen(); // returns an iterator
it.next(); // "Hello"
it.next(); // "World"
```

- Gonna assume you know what generators are -> plentry of documentation out there to learn them. Please stop here -> go learn them and come back and continue. Learn value passing into a generator


- Okay, all done? Lets continue


- Think deeply about a `yield` keyword from the context of within the generator. When the generator hits a `yield` keyword, what does it say? It actually says `Oh, I need a value here but I dont have it right now. I'm going to wait around until I get that!`

- Now if you simply yield, it'll wait around till the iterator next method is called with a value. But what if you yielded something like an AJAX call? Yep, its going to wait around until that call finishes right?

- Another thing is -> you never have to complete your generators. Putting a while true in there is actually fine! It'll be garbage collected whenever its references are null


``` js
function getData(d){
    	setTimeout(function() {
            	run(d);
        }, 1000);
}

// Coroutine basically takes in a generator as 
// argument, executes it to get the iterator 
// and returns the iterator
var run = coroutine(function*() {
   	var x = 1 + (yield getData(10));
    var y = 1 + (yield getData(30));
    var answer = (yield getData(
        			"Meaning of life: " + (x+y);
        		));
    console.log(answer);
});

run();
```


- What have we done above? WE HAVE ACHIEVED GREATNESS. We have `synchronous looking async code` inside the generator!

- In fact, the yields in the above lines does not care about whether the data it yields are synchronous or asynchronous! So in effect, we have factored out asynchocity itself out of our code

- Even our error handling becomes synchronous looking again! If the getData function threw an error, it will get caught only within the try catch that we'll have to add inside the generator since thats the caller


- Lets dig a little deeper, when you do :
``` js
	var x = 1 + (yield getData(10));
```

- What you're actually doing is yielding undefined out, since getData doesn't return anything. But since getData sets of a setTimeout that later calls run, the generator continues.This means -> never call run below the first `run() ` invocation in the last line! You dont want your generator being controlled by multiple functions!

- But now, what happens is that we've run into the inversion of control/ callback hell issue again right? Anybody can call my iterator and Ima be screwed!

- To solve this: combine generators and promises

#### Promises + Generators

- yield promise!!!

- Basically we combine the best of both worlds!


- What happens when you yield a promise? The running driver code gets it. Think of it from the driver code's perspective - whenever you call `it.next()`, you get a promise back! What you do with the promise? You `.then` it to and call `it.next()`again.

- So we basically keep doing this loop of `it.next()` and `yield promise` until the generator has run out of promises to give.


- Moreover, higher order promise libraries will allow you to map these generator runs as a step in the sequence

- Now, `async await` basically does what these libraries do for you -> they took it from C#

- Async functions returns a promise that gets resolved when the async function finishes


EX 7 - SAME STUFF AS ALWAYS BUT WITH GENERATORS AND PROMISES

```js
// Assume runner() is the plumbing for generators with promises which keeps called it.next() to get a promise from a generator

runner(function *main() {
    	var p1 = getFile("file1");
    	var p2 = getFile("file2");
    	var p3 = getFile("file3");
    
    // Same as : var text1 =yield p1; output(text1);
    	output(yield p1); 
    	output(yield p2);
    	output(yield p3);
    	output("Complete");
```

---

## Observables


- So we started off talking about concurrency right? Whats concurrency - essentially managing flow control!

- We know that promises works well if there is a single request and a single response -> but what if the source of our information is actually a repeated stream of information thats coming in?

- Can we write promises into an event stream?

- Because think about it for a second - most of the async stuff happening in our programs are event oriented - all of our UI is event oriented

``` js
// Problem with promises and events

var p1 = new Promise(function(resolve,reject) {
    $("#button").click(function(evt) {
        var classname = evt.target.classname;
        if(/foobar/.test(classname)) {
            	resolve(classname);
        } else {
            reject();
        }
    });
});

p1.then(function(classname) {
    console.log(classname);
});
```

- What we're trying to do her is to listen to the click event of a button and print it only if its classname matches `foobar`. But there is a problem here...

- PROMISES ARE RESOLVED ONLY ONCE! So, this would only work for 1 click of the button. Uh-oh!

- One way that we could try addressing this is to create the event handler and then use the promise inside it - but then... what sthe point of it? You'll be resolving/rejecting a promise and immediately doing a `.then`. Moreover, your promise only exists within the event handler -> you cannot set up the event handler promise in one place and handle the promise elsewhere!


- So.. enter observables!


- They aren't in JS yet, and they might come in to be natively supported sometime in the future, but currently RxJS is the best library that gives you this -> Rx: React Extensions


- An observable is like a chain of calculated fields in a spreadsheet. We might have 1 cell like C1 to have the data, and C2,D4,K8 might be cells that depend on C1 to calculate their values. So these calculation chain is essentially a data flow chain!

- Thats how reactive programming is done : The source of my data is some data source, and once I get that data its going to go through a series of steps, some of those steps might be just direct and responsive (sync) steps, and some of them might be asynchronous steps.

- So it is essentially a data flow mechanism!

- Technial defn: An observable is an adapter, hooked onto an event source that produces a promise everytime an event comes through

- But it does so in a separate way that I can set up my event source, and in an entirely different part of the program I can declaratively say what the data flow is. So you'd have an observable that represents your data source and you can subscribe to that observable in one or more locations in an entirely separate way

- So, in the Excel sheet, you can think of each of the different fields as a different step in a chain of responses to an observable 


``` js
var obs = Rx.Observable.fromEvent(btn,"click");

obsv.map(function mapper(evt) {
    	return evt.target.className;
	})
	.filter(function filterer(className) {
    	return /foobar/.test(className);
	})
	.distinctUntilChanged()
	.subscribe(function(data) {
    	var className = data[1];
    	console.log(className);
	});
```

- `fromEvent` takes a DOM element and an event name, and everytime that DOM element fires that event, it pumps another piece of data through the chain

- So the `map` and `filter` working here can be thought of as the same functions that are working on an array of never ending stream of events

- `distinctUntilChanged` means - first time piece of data comes through, let it pass. But if another one of the same data comes next, don't let it pass. So, essentially [1,1,3] -> [1,3]. But importantly, the `UntilChanged` bit signifies: [1,2,1] -> [1,2,1]

- `subscribe` is basically how you signal the end of the chain and it typically returns a synchronous response that you want the entire chain to finally give out

- Checkout [RxMarbles](www.rxmarbles.com) to learn about observables

- Rx.of() allows you to create observables out of constants or even null.

- Now if you think of observables as streams, you can also define higher level operations like stream operations. So, we can have operations to compose these `streams` together! 

Eg:
``` js
var obs1 = Rx.Observable.fromEvent(btn,"click");
var obs2 = Rx.Observable.fromEvent(input,"keypress");

// fire when both have occured - look at rxmarbles for this
var obs3 = Rx.zip(obs1,obs2);

// fire when either occurs
var obs4 = Rx.merge(obs1,obs2);

obs3.subscribe(function(event1,event2) {
    	console.log(event1.target.className);
    	console.log(event2.target.className);
});

obs4.subscribe(function(event) {
    	console.log(event.target.className);
});

```



EX 8 - LISTEN TO CLICKS BUT ONLY HONOUR 1 CLICK FOR EVERY 1000MS AND ADD THEM TO LIST

```js
// Approach 1 -> treat the timer and the button 
// clicks as separate streams
$(document).ready(function() {
    var $btn = $("#btn");
    var $list = $("#list");

    clicks = Rx.of(),
        timer = Rx.of() 
    
    $btn.click(function(evt){
        clicks.push(evt); // might be called next in Rx world - but lets call it 'push' conceptually    
    });
    
    setInterval(function() {
        time.push();
    },1000);

// So now I have 2 streams, one with clicks and 
// other time stream that happens every 1000ms
    
// So if I fire an event every time both of them 
// occur, then thats our required stream!
    
    var requiredStream = Rx.zip(clicks,timer);
    
    requiredStream.subscribe(function(click) {
        $list.append($("<div>Clicked</div>"));
    }                     
});
```


- But there is a problem with the above approach! What is it?

- What zip does is that it will keep zipping events from stream 1 and 2, without ever discarding any events from either stream! So whats going to happen in this approach is that, if you click a lot of times, even after you stop, there are going to be click events in the click stream pending which will keep getting appended to the dom whenever the timer emits an event (aka every 1 second)

- This is NOT what we want

```js
// Approach 2
$(document).ready(function() {
    var $btn = $("#btn");
    var $list = $("#list");

    var clicks = Rx.of(),
        msgs = Rx.of(),
        latest;
    
    $btn.click(function(evt){
        clicks.push(evt); 
    });
    
// -------------------------------------------- 
    // Separate part of application
    
    setInterval(function() {
        if(latest) {
        	msgs.push("Clicked");
            latest = null;
        }
    },1000);
    
    clicks.val(function(evt) {
       latest = evt;
    });
    
    msgs.subscribe(function(msg) {
        $list.append($("<div>" + msg + :"</div>"));
    }                     
});
```

-----

## CSP

- CSP is all about modelling concurrency with `channels`

- A `channel` is kind of like a `stream/pipe` but with 1 key difference: 
	
    - A `pipe/stream` has no buffer size and hence has this automatic notion of backpressure built into it
    	- Backpressure:
        	
        
        Imagine a `stream/pipe`, which has two ends, one `producing` end and one `consuming` end. The `producer` and the `consumer` have no way of communication. So when the consumer stops consuming, backpressure is a way of signalling to the producer to stop since the pipe would become full
        
        
    - A `channel` is a pipe that takes only one message at a time and it automatically has backpressure. 
    	- What that means: `You can't send me something until I'm ready to consume and I can't consume something unless you are ready to send it`. 
    	- There is only 1 message that transfers through the channel
    	- So it is this notion of blocking channels
        
        
        
- CSP -> Communicating Sequential Processes

- Invented in 60s by Hoare 

- CSP is similar to Actor model but with 1 difference: 
	
    - In the Actor model, the messages are asynchronous
    - In CSP, the messages are synchronous
    
- CSP is all about modelling your application, with lots of tiny independent pieces that we call `processes`

- In a multithreaded environment, every independent piece would be on its own thread and would be running truly independent of everything else. And when at a point when it needs to talk to somebody else, it shoots off a message, and blocks itself until some condition is met -> and then is unblocked and continues executing independently

- But alas, JS is not multithreaded. But we have seen a concept that models our application as individual processes that can stop and start - GENERATORS!

- So, if you model your application as a bunch of generators, which can pass messages to each other when required and unblock themselves, then you've done CSP!

``` js
var ch = chan();

function *process1() {
    yield put(ch,"Hello");
    var msg = yield take(ch);
    console.log(msg);
}

function *process2() {
    var greeting = yield take(ch);
    yield put(ch,greeting + "World");
    console.log("done");
}

// Hello world!
// done!

```


- There are libraries that allow you to do csp in JS, and one such library is called.... csp

```js
csp.go(function*() {
    while(true) {
        yield csp.put(ch,Math.random());
    }
});

csp.go(function*() {
    while(true) {
        yield csp.take( csp.timeout(500) );
        var num = yield csp.take(ch);
        console.log(num);
    }
});
        
```


- In the above, the first generator is generating random numbers as quickly as it can. There is no setTimeout here to throttle it. But this is allowed in csp, because of backpressure. Even though the first generator produces random numbers as quickly as it can, its ability to dump stuff through the channel is restricted by the speed at which the channel is consumed.

- The consumer is taking something out of the stream every 500ms

- Importantly, the consumer has no idea where the data is coming from and the producer has no idea who is going to consume it

- So these are independent sequential processes who are able to communicate using blocking semantics

- Here `csp.timeout(500)` returns a channel that doesn't send any data until 500ms has passed. So this is how you block stuff. So the actual consumption from the channel is happening in the `var num = yield csp.take(ch);`

- Drawback of this particular library is that the producer and consumer need to have the same reference to the channel and hence we are forced to have this global state which is the channel. 

- But there are some other libraries that do not have this drawback.

- There are also other primitives in CSP other than `take` and `put`

``` js
csp.go(function*() {
    while(true) {
        var msg = yield csp.alts(ch1,ch2,ch3);
        console.log(msg);
    }
});
```

- `alts` takes in multiple channels as arguments and its job is to do an action when the first of these channel lets it do something (it looks for in order specified).

- And its not just take from the above channels, thats the default, you can also put to channels


```js
csp.go(function*() {
    var table= csp.chan();
    
    csp.go(player, ["ping",table]);
    csp.go(player, ["pong",table]);
    
    yield csp.put(table, {hits: 0});
    yield(csp.timeout(1000));
    table.close();

});

function* player(name,table) {
    
    while(true){
        var ball = yield csp.take(table);
        if(ball === csp.CLOSED) {
            console.log(name + ": table is gone");
            return;
        }
        ball.hits += 1;
        console.log(name + " " + ball.hits);
        yield csp.timeout(100);
        yield csp.put(table, ball);
    }
}
    
}
```

- How cool is that!!


- Also, it is not that we'll always have to be within a generator to produce data into the channel

``` js
function fromEvent(el,eventType) {
    var ch = csp.chan();
    $(el).bind(eventType, function(evt) {
        csp.putAsync(ch,evt); // Allows you to put data into channel from a normal function
    });
    return ch;
}

csp.go(function*() {
    var ch = fromEvent(el,"mousemove");
    while (true) {
        var evt = yield csp.take(ch);
        console.log(evt.clientX + "," + evt.clientY);
    }
});
```

- `putAsync`actually returns a promise denoting whether or not the put succeeded -> but we've ignored checking for that here

- So this feels very similar to pushing events into a stream and reading them off the stream using observables and subscribers

- The major difference between what we are modelling here and what we did with observables is `backpressure`
	
    - I dont want you to send me anything until I am ready to take it -> this comes for free
    
    
- Also, important to notice that `csp.putAsync` does not throw any events away. If the channel hasn't taken a message yet, putAsync just stores all the events as promises and puts them in whenever possible


- So making the producer throw away things or something like that are not as straightforward with channels as they were with observables but it is possible by using this like `alts`

- Also, channels can have a buffer size of greater than 1. This puts the channel in `buffering` mode. Which would mean that I can take events in batches of 10 before I start blocking and things like that


- Some libraries solve the "having to maintain a global variable for the channel reference" by creating a default channel that all go routines automatically subscribe to and if you need specific go routines to communicate over different channels, you can send the channel itself through the original channel


EX 9 - SAME AS 8 BUT WITH CSP

``` js
$(document).ready(function() {
    var $btn = $("#btn"),
    	$list = $("#list"),
    	clicks= csp.chan(),
        msgs = csp.chan(),
        queuedClick;
    
    $btn.click(listenToClicks);
    
    // run go-routines
    csp.go(sampleClicks);
    csp.go(logClick);
    
    // If this were a generator, we would need need the storing and listening to 
    // a promise as we would have yielded on the 'put'
    function listenToClicks(evt) {
     	if(!evt) {
            // Backpressure simulated by only putting event in channel if the 
            // promise hasn't resolved yet
            // Which means previous putAsync hasn't been taken
            // If this wasn't done, events would just stack up as seen before
            queuedClick = csp.putAsync(clicks,evt); 
            queuedClick.then(function() {
                queuedClick = null;
            });
    }
        
        // sample clicks channel
    	function *sampleclicks() {
            while(true) {
                yield csp.take(csp.timeout(1000));
                yield csp.take(clicks); // There can only be 1 click if any because of the above backpressure simulation
                yield csp.put(msgs,'clicked');
            }
        }
        
        //subscribe to sampled message channel
        function *logClick() {
            while(true) {
                var msg = yield csp.take(msgs);
                $list.append($("<div>" + msg + "</div>"));
            }
        }
});


```

- Important disticntion between this solution and the rx solution with observables is that: Here we are sotring the first click event that comes, and in the rx solution, we were storing the last



-----

#### Conclusion

- Callbacks
- Thunks
- Promises
- Generators
- Observables
- CSP

None of these are going to replace the other. Instead these are concepts that build on top of the other and that is exactly why learning about the conceptual underpiunnings of each of them is really crucial.

You will have places in your program where you would need to use any of the above, and it is a decision that you'll have to take at that point as to which one is most suitable for the situation. 

