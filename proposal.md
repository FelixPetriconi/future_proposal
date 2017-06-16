---
Towards a Good Future
---

Document number: P0676R0

Date: 2017-06-18

Project: Programming Language C++, Library Working Group

Reply-to:  Felix Petriconi felix{at}petriconi[dotnet], David Sankel camior{at}gmail[dotcom], Sean Parent sean.parent{at}gmail[dotcom]

<!--
TODO: Insert Tony-table here. Tony-tables are before/after comparisons
-->

# I. Table of Contents

# II. Introduction

The standard library needs a high-quality vocabulary type to represent asynchronous values. Having many variants of this concept in the wild is a pain point, but, even worse, the judicious use of "callback soup" makes asynchronous code difficult to develop and maintain. Unfortunately, based on the experience of the authors in real-world, production applications, neither the crippled `std::future`, nor the extensions proposed in the Concurrency TS are going to remedy the situation.

In the negative, we recommend against adoption of the `std::future` related extensions in the Concurrency TS. In the positive, we provide recommendations for an alternative that we feel will meet the demands of real world applications and, more importantly, gain widespread adoption.

# III. Motivation and Scope

## Copyable future

A common use case in graphs of execution is that the result of an asynchronous calculation is needed as an argument for more than one further asynchronous operation. The current design of std::future is limited to one continuous operation, to one `.then()` continuation, because it accepts only an r-value `std::future` as argument. So the `std::future` must be moved into the continuation and after that it cannot be used as an argument for an other continuation.

Because of an other reason it is necessary to add the possibilities of splits to the interface of futures. Without it would not be symmetrical.

So it is necessary that futures become copyable and the following example of multiple continuations into different directions would be possible. 

~~~C++
   std::future<int> a;
   a.then([](int x){ /* do something */ });
   a.then([](int x){ /* also do something else. */ }
~~~


## Cancellation of futures

Because of different reasons it might be that the result of an asynchronous operation and its continuation(s) is not needed any more; e.g. the user has canceled an operation. The current design of `std::future` and the TS does not support any kind of cancellation. So it is required to wait for its fulfillment even when the result is not be needed any more. On systems with limited resources, e.g. mobile devices, this is a waste of resources.
Even it is possible to implement cancellation on top of the existing design, it would be preferable, if the futures would have this capability by themselves. 

So we think that it is necessary that a future can be destructed without the need to execute its associated task, when this has not started. In case that it has started, it should finish, the result would be dropped and the attached continuations should not be executed.

Tasks, indicated as circles and futures, indicated as squares, are building a graph of execution in the following image.

![](images/FutureChain01.png)   

In case that one is not interested any more in future F3, it gets destructed. Then the graph would become:

![](images/FutureChain02.png) 

Now there is no need to execute task T3 and so it will automatically be dropped as well and the graph would change to:

![](images/FutureChain03.png)

Now the futures F2a and F4a are obsolete and they go away as well.

![](images/FutureChain04.png)


## Simplified interface

The associated operation of a continuation shall be invoked with a `std::future<T>` according with the C++17 TS. 

~~~C++
  std::future<int> getTheAnswer = std::async([]{ return 42; };
  auto next = getTheAnswer.then([](std::future<int> x) { std::cout << x.get(); };
~~~

The associated operation must be called with a `future<tuple<future<Args>...>>` in case of a `when_all()` continuation. In the following example a possible parameter declaration of "auto x" is written explicitly for clarification.)

~~~C++
  std::future<int> an = std::async([]{ return 40; });
  std::future<int> swer = std::async([]{ return 2; });
  
  auto answer = std::when_all(std::move(an), std::move(swer)).then( 
    [](std::future<std::tuple<std::future<int>, std::future<int>>> x) {
      auto t = x.get();
      std::cout << get<0>(t).get() + get<1>(t).get();
    });
~~~

This makes the code much more difficult to reason about, because as a first step the tuple must be extracted from the future and then all values for the actual calculation of the continuation must be accessed through the tuple and then through the futures.
So either the interface of the callable operation gets "infected" by the interface of `std::future` or an additional layer of extraction is necessary to invoke the operation. 
Both choices are not optimal from our point of view and they can be avoided by instead allowing to call the continuations by value.

So the code could then be written like this:

~~~C++
  future<inr> an = async([]{ return 40; });
  future<int> swer = async([]{ return 2; });
  
  auto answer = when_all(an, swer).then( 
    [](int x, int y) {
      std::cout << x + y;
    });
~~~

One argument for passing a future into the continuation is, that the future encapsulates either the real value or an occurred exception. But this implies that everyone has to use the more complicated interface by passing futures, even there might be use cases where never an exception might occur. From our point of view this is against the general principle within C++, that one only should have to pay for what one really needs.

For cases that error handling is necessary, a new `.recover()` method would serve the same purpose.

~~~C++
  auto getTheAnswer = [] {
    throw std::exception("Bad thing happened: Vogons appeared");
    return 42;
  };

  auto handleTheAnswer = [](int v) { 
    if (v == 0) 
      std::cout << "No answer\n"; 
    else
      std::cout << "The answer is " << v '\n'; 
  };

  auto f = async(default_executor, getTheAnswer)
    .recover([](future<int> result) {
      if (result.error()) {
        std::cout << "Listen to Vogon poetry!\n";
        return 0;
      }
      return result.get_try().value();
  }).then(handleTheAnswer);
~~~


## Scalability 

It is necessary that futures scale in the same way from single threaded to multi threaded environments. So either it it necessary to get rid of `.get()` and `.wait()` or define when and how tasks can be promoted to immediate execution and support `.get()` and `.wait()` in such circumstances without blocking. 

Note that `.get()` and `.wait()` as they are currently defined are potential deadlocks in any system without unlimited concurrency (i.e., in any real system).

##  Executors

Many of the todays used UI libraries allow changes of the UI elements only from within the main-event-loop or main-thread. But with the design of std::async and the continuations of C++11 and the C++17 TS it is not easily possible to perform changes in the UI, because it is not possible to define in which thread a future or a continuation shall be executed.

So we propose that it should be possible to specify an executor while using `async` or `package` to create a new future or pass it as additional argument when calling a continuation.

Example:

~~~C++
  future<int> calculateTheAnswer = async(std::default_executor, []{ return 42; } );
  
  future<void> displayTheAnswer = 
    calculateTheAnswer.then( QtMainLoopExecutor{}, [this](int a) { _theAnswerDisplayField.setValue(a); } ); 
~~~

Here the task associated with the first future shall be executed on the default executor, which we think should be based on the system's thread pool. And then the continuation shall run on an executor that schedules all tasks to be executed within the Qt main loop.


## Joins 

As it is specified in the C++17 TS, there should be joins as `when_all` and `when_any`. But as already pointed out above, the attached function object should take its arguments per value and not by `future<T>`.

Now it would be possible with the support of cancellation of futures that in case of a single failing future for a `when_all`, all not started futures are canceled, because the overall `when_all` cannot be fulfilled any more. The same is valid for a `when_any`. So as soon as one future is fulfilled, all other non started futures could be canceled. 


7) Ideally this is all paired with a rich standard tasking system, and forms the basis for a rich channel system for use with co-routines.

I like the idea of getting down to a single type like JS promise, however, I don't know quite how to make that work with cancellation. I think not treating the processor as RAII - when it is arguably the most important resource in the machine, is nuts and yet cancellation is left out of nearly every model. Usually with a hand wave about it "being something you can add in", when in any non-trivial system there is no good way to do so.


## Implementation

With the concurrency library [https://github.com/stlab/libraries](https://github.com/stlab/libraries) an implementation is available that fulfills all the above stated requirements. This library should not be seen as a 1:1 blue print for a new standard, but serve as an illustration what could be possible.

# IV. Impact On the Standard

Only the existing future library is influence by this proposal.

# V. Design Decisions

Why did you choose the specific design that you did? What alternatives did you consider, and what are the trade-offs? What are the consequences of your choice, for users and implementers? What decisions are left up to implementers? If there are any similar libraries in use, how do their design decisions compare to yours?

# VI. Technical Specifications

The technical documentation of the existing library with several code examples is available on [http://www.stlab.cc/libraries/concurrency/index.html](http://www.stlab.cc/libraries/concurrency/index.html) to clarify the details.

# VII. Acknowledgments

We thank Gor Nishanov for encouraging us in pursuing with this proposal and for sharing his thought.

# VIII. References

