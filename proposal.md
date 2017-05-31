
---
This document is work in progress!
---

Document number:  Nnnnn=yy-nnnn

Date:  yyyy-mm-dd

Project:  Programming Language C++, Library Working Group

Reply-to:  Sean Parent sean.parent{at}gmail[dotcom], David Sankel camior{at}gmail[dotcom], Felix Petriconi felix{at}petriconi[dotnet]


# I. Table of Contents

# II. Introduction

From our point the design of std::future and its planned extension, which is currently described in the C++17 Concurrency TS, can be improved in respect to performance, scalarbility and composability of its components.


# III. Motivation and Scope

## Motivation for the use of futures


## Copyable future

A common use case in graphs of execution is that the result of an asynchronous calculation is needed as an argument for more than one further asynchronous operation. The current design of std::future is limited to one continuous operation, to one .then() continuation, because it accepts only a std::future as argument.

So it is necessary that futures become copyable and the following example of multiple continuatons into different directions would be possible. 

~~~C++
   std::future<> a;
   a.then([]{ /* do something */ });
   a.then([]{ /* also do something else. */ }
~~~


## Cancellation of futures

Because of different reasons it might be that the result of an asynchronous operation and its continuation is not needed any more; e.g. the user has cancelled an operation in the user interface. So it is neecessary that a future can be destructed without the execution of its associated task. In this case all attached continuations should not be executed as well. The std::future from C++11 is required to wait for its fulfillment even the result might not be needed any more. From our point of view this is a waste of resources.

(destructing a future should cause any unexpected tasks to become no-ops. If this feature isn't build in it needs to composable without creating a new future type.)

## Simplified interface

With the C++17 TS interface it is necessary that the associated operation of a continuation is invoked with a future<T>. 

~~~C++
  std::future<int> getTheAnswer = std::async([]{ return 42; };
  auto next = getTheAnswer.then([](std::future<int> x) { std::cout << x.get(); };
~~~

In case of a when_all() continuation the associated operation must be called with a future<tuple<future<Args>...>>. (In the following example a possible argument declaration of "auto x" is for illustrational purpose written explicitly.)

~~~C++
  auto a = std::async([]{ return 40; });
  auto b = std::async([]{ return 2; });
  
  auto answer = std::when_all(std::move(a), std::move(b)).then( 
    [](std::future<std::tuple<std::future<int>, std::future<int>>> x) {
      auto t = x.get();
      std::cout << get<0>(t).get() + get<1>(t).get();
    });
~~~

This makes the code much more difficult to reason about, because as a first step the tuple must be extracted from the future and then all values for the actual calculation of the continuation must be accessed through the tuple and then through futures.
So either the interface of the callable operation gets "infected" by the interface of std::future or an additional layer of extraction is necessary to invoke the operation. 
Both choices are sub-optimal from our point of view and they can be avoided by instead allowing to call the continuations by value

// TODO Erroro handling
simplified interface and error handling (call the continuation with T not with future<T>!).


## Scalarbility 

scales to single threaded environments - either get rid of get() and wait() or define when and how tasks can be promoted to immediate execution and support get() and wait() in such circumstances without blocking. Note that get() and wait() as they are currently defined are potential deadlocks in any system without unlimited concurrency (i.e., in any real system).

##  Executors

Many of the todays used UI libraries allow changes of the UI elements only from within the main-event-loop or main-thread. But with the approach of C++11 and the C++17 TS it is not easily possible to perform changes in the UI, because it is not possible to define in which thread a future and a continuation shall be executed.

So we propose that it should be possible to specify an executor while using std::async to create a new future or pass it as additional argument when definining a continuation.

Example:

~~~C++
  auto calculateTheAnswer = std::async(std::default_executor, []{ return 42; } );
  
  stlab::future<void> displayTheAnswer = 
    calculateTheAnswer.then( QtMainLoopExecutor{}, [this](int a) { _theAnswerDisplayField.setValue(a); } ); 
~~~

Here the first future shall be executed on the default executor, which we think should be the systems thread pool. And then the continuation shall run on an executor that schedules all tasks to be executed within the Qt main loop.

## Joins 

As it is specified in the C++17 TS, there should be joins like when_all and when_any. But as already pointed out above the attached function object should take its arguments per value. 

Now it would be possible with the support of cancellation of futures that in case of a single failing future for a when_all, all not started futures are cancelled, because the when_all cannot be fullfilled any more. The same is valid for a when_any. So as soon as one future is fullfilled, all other non started futures could be cancelled. 


7) Ideally this is all paired with a rich standard tasking system, and forms the basis for a rich channel system for use with co-routines.

I like the idea of getting down to a single type like JS promise, however, I don't know quite how to make that work with cancellation. I think not treating the processor as RAII - when it is arguably the most important resource in the machine, is nuts and yet cancellation is left out of nearly every model. Usually with a handwave about it "being something you can add in", when in any non-trivial system there is no good way to do so.


Why is this important? What kinds of problems does it address? What is the intended user community? What level of programmers (novice, experienced, expert) is it intended to support? What existing practice is it based on? How widespread is its use? How long has it been in use? Is there a reference implementation and test suite available for inspection?

# IV. Impact On the Standard

What other library components does does it depend on, and what depends on it? Is it a pure extension, or does it require changes to standard components? Can it be implemented using C++11 compilers and libraries, or does it require language or library features that are not part of C++11?

# V. Design Decisions

Why did you choose the specific design that you did? What alternatives did you consider, and what are the tradeoffs? What are the consequences of your choice, for users and implementers? What decisions are left up to implementers? If there are any similar libraries in use, how do their design decisions compare to yours?

# VI. Technical Specifications

The committee needs technical specifications to be able to fully evaluate your proposal. Eventually these technical specifications will have to be in the form of full text for the standard or technical report, often known as "Standardese", but for an initial proposal there are several possibilities:

Provide some limited technical documentation. This might be OK for a very simple proposal such as a single function, but for anything beyond that the committee will likely ask for more detail. 
 
Provide technical documentation that is complete enough to fully evaluate your proposal. This documentation can be in the proposal itself or you can provide a link to documentation available on the web. If the committee likes your proposal, they will ask for a revised proposal with formal standardese wording. The committee recognizes that writing the formal ISO specification for a library component can be daunting and will make additional information and help available to get you started.
 
Provide full "Standardese." A standard is a contract between implementers and users, to make it possible for users to write portable code with specified semantics. It says what implementers are permitted to do, what they are required to do, and what users can and can't count on. The "standardese" should match the general style of exposition of the standard, and the specific rules set out in 17.5, Method of description (Informative) [description], but it does not have to match the exact margins or fonts or section numbering; those things will all be changed anyway.

# VII. Acknowledgements

# VIII. References

