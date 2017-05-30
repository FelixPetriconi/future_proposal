
Document number:  Nnnnn=yy-nnnn

Date:  yyyy-mm-dd

Project:  Programming Language C++, Library Working Group

Reply-to:  Sean Parent sean.parent@gmail.com, David Sankel camior@gmail.com, Felix Petriconi felix@petriconi.net


# I. Table of Contents

# II. Introduction

From our point the design of std::future and its planned extension with the C++17 Concurrency TS can be improved in respect to performance, scalarbility and composability of its components.


# III. Motivation and Scope

## Copyable future
A common use case in execution graphs is that the result of an asynchronous calculation is needed as an argument to more than one further asynchronous operation. The current design of std::future is limited to one continuous operation, to one .then() continuation, because it accepts only a std::future as argument.

So it is necessary that futures become copyable and the following example of multiple continuatons into different directions would be possible. 

~~~C++
   std::future<> a;
   a.then([]{ /* do something */ });
   a.then([]{ /* also do something else. */ }
~~~



## Cancellation of futures
  destructing a future should cause any unexpected tasks to become no-ops. If this feature isn't build in it needs to composable without creating a new future type.

## Simplified interface

simplified interface and error handling (call the continuation with T not with future<T>!).


## Scalarbility 

scales to single threaded environments - either get rid of get() and wait() or define when and how tasks can be promoted to immediate execution and support get() and wait() in such circumstances without blocking. Note that get() and wait() as they are currently defined are potential deadlocks in any system without unlimited concurrency (i.e., in any real system).

##  Executors

5) ability to specify the scheduler/executor for continuations and override and any point in the chain

## Joins 

joins (when_all() & when_any())

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

