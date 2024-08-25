+++
title = "GSoC 2024 @ HPX - Final Report"
slug = "final-report-hpx-2024"
date = "2024-08-20"
description = "My final work product submission for the GSoC 2024 at HPX."
+++

# Adapting HPX's Parallel Algorithms for Usage with Senders and Receivers

This final report aims to give an overview over the background and objectives of
my project conducted at the [**Ste∥ar Group**](https://stellar-group.org/) as 
part of the [**Google Summer of Code**](https://summerofcode.withgoogle.com/) 
**2024**, as well as to document the work that was completed and all that's
still left to do in the future.

**Contents**

1. [S/R 101 - Intro to Senders and Receivers](#sr-101---intro-to-senders-and-receivers)
2. [The Blueprint of HPX's Parallel Algorithms](#the-blueprint-of-hpxs-parallel-algorithms)
3. [What's This Project About?](#whats-this-project-about)
4. [Completed Work](#completed-work)
5. [What's Left to Do ...](#whats-left-to-do-)
6. [Acknowledgements](#acknowledgements)


## S/R 101 - Intro to Senders and Receivers

Since this project revolves around senders and receivers (S/R), let's start with
a short introduction to what they are. The term might be somewhat misleading at
first, as it has nothing to do with networking or radio communication. Instead, 
S/R are central concepts in the standard model for *asynchronous 
execution on arbitrary execution resources* that is proposed in the 
[P2300](wg21.link/p2300) paper (`std::execution`) and might be adopted in the 
upcoming *C++26*.

Put very simply, S/R are a generalized form of callbacks, providing the 
programmer with means of specifying complex chains of asynchronous tasks 
together with full control over when it's execution is started and on which
execution resources the members of the chain run. The latter can be controlled
on per-task basis, i.e., it is possible to use difference execution resources 
for different tasks within the same chain, for instance first performing some 
setup on a CPU, but then moving to a GPU for computationally more intensive 
operations.

But since an example is certainly worth more than 1000 words, let's give a
simple one:

```c++
static_thread_pool thread_pool{};
auto scheduler = thread_pool.get_scheduler();  // 1.

auto triple = [](std::size_t i, std::vector<int>& nums){
    nums[i] *= 3;
};

auto task_chain =   // 5.
    stdexec::just(std::vector{2, 0, 2, 4})  // 2.
    | stdexec::continue_on(scheduler)       // 3.
    | stdexec::bulk(4, triple);             // 4.

auto [tripled_nums] = stdexec::sync_wait(task_chain).value(); // 6.

print(tripled_nums); // Output: 6 0 6 12
```

First of all, to prevent any confusion: The example uses an [experimental 
reference implementation](https://github.com/NVIDIA/stdexec) of P2300, which is 
why the namespace is `stdexec` instead of `std::execution`. Next, let's take a
closer look at the important parts in the example code:

1. In the S/R model, execution resources are abstracted into so-called
   *schedulers*. The method of obtaining such a scheduler depends on the
   execution resource, but ideally the schedulers can then be used uniformly
   within the S/R model.
2. The `just` algorithm is a *sender factory*: It returns a new *sender*, which
   *sends* the arguments that were passed to `just`. Phew, that's quite a lot of
   new terms in one sentence! But don't despair, coming back to task chain 
   analogy this just means that `just` represents a task that will pass on 
   (or *send*) its arguments to the task next in the chain. And *factory* simply
   signifies that `just` represents the beginning of a new task chain.
3. The piping operator (`operator|`) is similar to attaching a continuation
   to a task. Algorithms like `continue_on` take the previous sender, 
   wrap it and return a sender representing it's own action on top of all the
   operations embodied in the predecessor sender. That's why they are referred
   to as *sender adaptors*. The action that `continue_on` stands for 
   specifically is switching to another execution resource, meaning that the
   tasks succeeding it in the chain -- that is at least until the execution
   resource is switched again -- will run on the execution resource represented
   by the scheduler that was passed as an argument.
4. `bulk` is another sender adaptor taking an iteration shape and an invocable
   that will be called with every index in that shape and with any values sent 
   by the predecessor sender, depending on the underlying scheduler these calls
   might be parallelized. When all the function calls are completed it will
   simply send the values passed in by the predecessor sender.
5. `task_chain` is a sender representing the complete task chain composed by
   applying the two sender adaptors to the initial sender returned by `just`. It
   is important to note that at this point execution has not started yet, 
   since senders generally only describe the work that has to be done without
   eagerly launching it asynchronously. So senders are not quite like
   `std::future`s, since these rather represent the result of work instead of 
   the work itself and because obtaining a `std::future` through a call to 
   `std::async` can already asynchronously initiate the computation of that 
   result.
6. If the reader is impatiently thinking, 'But please start to execute something
   already!' at this point, they should know that this is exactly where 
   `sync_wait` comes in. `sync_wait` is a *sender consumer*, that is a function 
   taking a sender as an argument but not returning one. In particular
   `sync_wait` kicks off the execution of the work represented by the given 
   sender, then waits until the results are available and returns them. (In
   reality the return type is a bit more intricate, in this example for instance
   a `std::optional<std::tuple>`, because senders can also send multiple values,
   cancellation signals or errors, but this would be out of scope for this
   short summary.)

And there you have it — that’s the gist of how to use senders! If you are
curious about why no receivers appeared, it’s because they’re mostly hidden
within the implementation side of S/R, acting as the glue between the senders
created by sender adaptors and their predecessors.

The full, compilable example can be found 
[here on Compiler Explorer](https://godbolt.org/z/P536hM458).
Additionally, for further details on S/R see [P2300](wg21.link/p2300), which 
includes a plethora of additional examples using senders, or this 
[blog post](https://ericniebler.com/2024/02/04/what-are-senders-good-for-anyway/) 
by one of the principal authors of that proposal.

## The Blueprint of HPX's Parallel Algorithms

HPX implements the complete C++ standard algorithms library and more importantly
parallel as well as asynchronous versions for each of the included algorithms,
that can be selected through different *execution policies*. Additionally, HPX's
algorithms are *customizable*, i.e. it is possible to specialize them for 
specific parameter types. 

To support this kind of flexibility without additionally burdening the
implementers of the algorithms, HPX provides several internal wrapper and
utility classes that take care of many of the arising concerns. Most parallel
algorithms are therefore structured similar to an onion, where the outer layers
add functionality to actual algorithm implementation in it's core, as it is
depicted in the diagram below.

{{<figure src="/images/hpx_algorithm_blueprint.svg" width="100%">}}

Let's quickly dissect this onion:
* **`tag_parallel_algorithm` wrapper**: This class mainly takes care of turning the 
  algorithm into a *customization point object*, that is the implementer
  provides the standard conformant overloads, which will be used by default when
  the algorithm is called, but there is a way to provide customized overloads 
  that will take precedence. (In detail this is accomplished by using a 
  *tag_invoke* mechanism.)
* **`algorithm` wrapper**: This class dispatches calls to an algorithm to either its
  sequential or parallel implementation based on the requested execution policy.
* **parallel implementation**: The parallel implementation of the algorithm is the 
  function that the `algorithm` class dispatches to if the execution policy is 
  one representing parallel execution, passing along the concrete execution 
  policy and all other of the algorithm's arguments. The parallel
  implementation is also responsible for ensuring the correct return type, e.g.
  for asynchronous execution policies the algorithm's usual return type must be
  wrapped in a `hpx::future`. This is where two handy utilities come in:
   * **`algorithm_result`** takes the execution policy type and the usual return
     type as an argument and computes the correct overall return type
     accordingly. Additionally, it provides a static `get` method that
     automatically converts values passed to it into an object that can be 
     returned directly, for example by wrapping them into a ready future. Both
     features allow the implementer to stop worrying about the different return
     value requirement of different execution policies.
   * **Partitioners** are a family of utility classes which can be used as basis
     for different forms of parallelization. Put very simply, the implementer
     specifies functions for processing entire chunks of the input, then the
     partitioners take care of splitting the input up and of parallelizing the
     processing of the obtained chunks taking the execution policy into account.
     Conveniently, the results they return are also automatically compatible
     with the given execution policy.

For example, the snippet below is the simplified parallel implementation of the
[`none_of` algorithm](https://en.cppreference.com/w/cpp/algorithm/all_any_none_of).
(The full implementation can be found [here](https://github.com/STEllAR-GROUP/hpx/blob/d9b5f6cdb6d9e0a166ba17adb748039efdd5faa6/libs/core/algorithms/include/hpx/parallel/algorithms/all_any_none.hpp#L376).)

```c++
template <typename ExPolicy, typename FwdIter, typename Sentinel,
    typename Predicate>
static algorithm_result_t<ExPolicy, bool> parallel(
    ExPolicy&& ex_policy, FwdIter first, Sentinel last, Predicate&& pred)
{
    if (first == last)
    {
        return algorithm_result<ExPolicy, bool>::get(true);
    }

    util::cancellation_token<> tok;

    auto process = [tok, pred = HPX_FORWARD(Predicate, pred)](
                       FwdIter chunk_begin,
                       std::size_t chunk_size) mutable -> bool {
        sequential_find_if(
            chunk_begin, chunk_size, tok, HPX_FORWARD(Predicate, pred));

        return !tok.was_cancelled();
    };

    auto reduce = [](auto&& results) {
        return sequential_find_if_not(std::begin(results), std::end(results)) ==
            std::end(results);
    };

    // The plain partitioner applies the `process` function on every chunk
    // and then calls the the `reduce` function on the collected results.
    return util::partitioner<ExPolicy, bool>::call(
        HPX_FORWARD(ExPolicy, policy), first, std::distance(first, last),
        HPX_MOVE(process), HPX_MOVE(reduce));
}
```

Isn't that great? There is no need of explicitly handling the different 
execution policies, since `algorithm_result` and `partitioner` already take care
of this! 

Unfortunately however, if S/R are introduced into the equation there will be
some additional friction that cannot be handled by these mechanism alone
anymore, which is where my project comes in, as is described in the next 
section.

## What's This Project About?

In accordance with one of HPX's missions, namely staying on top of the latests 
developments in the C++ standards, the parallel algorithms should also be
seamlessly usable with senders and receivers.

The groundwork for this is already laid in HPX and the parallel algorithms.
First of all, the `tag_parallel_algorithm` wrapper automatically enables using 
HPX's parallel algorithms as pipeable sender adaptors with certain execution 
policies, for example like this:

```c++
namespace ex = hpx::execution::experimental;
namespace tt = hpx::this_thread::experimental;

// create an execution policy representing S/R execution
ex::thread_pool_scheduler scheduler{};
ex::explicit_scheduler_executor executor{scheduler};
auto ex_policy = hpx::execution::par.on(executor);

std::vector numbers{1, 2, 3, 4, 5};
auto is_equal = [](int i) { return i % 2 == 0; };

auto none_of_sender =
    ex::just(std::begin(numbers), std::end(numbers), is_equal) 
    | hpx::none_of(ex_policy);

auto [result] = tt::sync_wait(none_of_sender).value();
```

On the implementation side, `tag_parallel_algorithm` takes care of receiving
the values sent by the predecessor sender, performs the usual dispatching and
simply forwards the received values as arguments in the call to the underlying
algorithm implementation. Therefore the implementation does not have to add
a special overload for handling the case where it's input come from a sender.
However, there is an import caveat: If the execution policy indicates S/R 
execution, `tag_parallel_algorithm` expects the algorithm implementation to
return a sender that will ultimately send the return value of the algorithm
instead of directly returning the result. (Which is perfectly sensible, because
this way the algorithm implementations can easily leverage S/R internally.)

Luckily (some of) the partitioners support S/R execution policies and will return senders
accordingly, so call to them ideally do not need to be adapted in any way. 
However, early exit conditions (like the empty range check in `none_of` above)
pose a problem that cannot be solved by using the `algorithm_result` utility
and is caused by the nature of senders itself. Too see this, it is sensible to 
realize that two senders which eventually will send values of the same type are
not necessarily of the same type itself. For example suppose we have to senders
effectively doing the same thing:

```c++
auto senderA = stdexec::just(1);
auto senderB = stdexec::just(1) | stdexec::then(std::identity{});

auto [ resultA ] = stdexec::sync_wait(senderA).value();
auto [ resultB ] = stdexec::sync_wait(senderB).value();

// Both senders return the send an `int`.
static_assert(std::is_same_v<decltype(resultA), decltype(resultB)>);

// But the types of the senders are different!
static_assert(not std::is_same_v<decltype(senderA), decltype(senderB)>);
```

These static assertions compile! (See for yourself 
[here](https://godbolt.org/z/KsW3Y193b).) And since the types of the senders
return by the partitioners are not fixed, on top of that it would not really
be reasonable to try to reproduces these sender types outside of the
partitioners,`algorithm_result` simply can neither compute the return type
of an algorithm implementation nor compatibly wrap values in calls to its `get`
method. 

(Strictly speaking it would be possible to make `algorithm_result` work 
perfectly with S/R, but this would involve a technique called *type-erased
senders* that unfortunately comes at at performance cost disqualifying it for
usage in a performance-oriented library like HPX.)

So instead manual adjustments are required to make the algorithm implementations
work with S/R execution policies, for example in a simple case like `none_of`
its enough to disable the early exit condition at compile time and letting the
return type being deduced automatically:

```c++
template <typename ExPolicy, typename FwdIter, typename Sentinel,
    typename Predicate>
static decltype(auto) parallel(
    ExPolicy&& ex_policy, FwdIter first, Sentinel last, Predicate&& pred)
{
    constexpr bool has_scheduler_executor =
      hpx::execution_policy_has_scheduler_executor_v<ExPolicy>;

    if constexpr (!has_scheduler_executor)
    {
      if (first == last)
      {
          return algorithm_result<ExPolicy, bool>::get(true);
      }
    }

    util::cancellation_token<> tok;

    auto process = [tok, pred = HPX_FORWARD(Predicate, pred)](
                       FwdIter chunk_begin,
                       std::size_t chunk_size) mutable -> bool {
        sequential_find_if(
            chunk_begin, chunk_size, tok, HPX_FORWARD(Predicate, pred));

        return !tok.was_cancelled();
    };

    auto reduce = [](auto&& results) {
        return sequential_find_if_not(std::begin(results), std::end(results)) ==
            std::end(results);
    };

    return util::partitioner<ExPolicy, bool>::call(
        HPX_FORWARD(ExPolicy, policy), first, std::distance(first, last),
        HPX_MOVE(process), HPX_MOVE(reduce));
}
```

The effective goal of this project was therefore to make such manual changes to
as many parallel algorithms as possible, so that more of the parallel algorithms
can be used with S/R. Additionally, it was of course necessary to add unit test
case checking the correctness and success of the fixes introduced for the
algorithms.

## Completed Work

A comprehensive list of algorithms for which S/R unit tests were added or which 
were fixed as part of this project can be found in 
[this table](https://docs.google.com/spreadsheets/d/1gLoMLBLMWpbe_i2eEhdCIYz4VHmZyX27vnjV7tKhRqc/pubhtml?gid=0&single=true). 
Around 50 of HPX's parallel algorithms were adapted to work with S/R, including 
all those relying on either the `partitioner` or the `for_each_partitioner`. 
Additionally, unit test cases were added for the S/R version of the algorithms
accordingly, as well as further checks with a focus on edge cases that could
potentially be broken through the changes made.

Since some issues surfaced when using HPX's current experimental implementation
of the S/R mode and considering the likely deprecation of this implementation 
in favor of NVIDIA's reference implementation when HPX transitions from 
C++17 to C++20, all the fixes were developed on top of
[PR #6431](https://github.com/STEllAR-GROUP/hpx/pull/6431), which integrates 
the latter into HPX. Therefore, it is important to note that the S/R versions 
of the parallel algorithms are only available when opting in to use HPX with 
C++20.

The main products of this project can be found in the following two PRs, both 
of which have yet to be merged:
* [PR #6494](https://github.com/STEllAR-GROUP/hpx/pull/6494): This PR
  encompasses almost all the newly added unit tests and S/R fixes, as wells as
  a few incidental bug fixes.
* [PR #6529](https://github.com/STEllAR-GROUP/hpx/pull/6529): This smaller PR
  is focussed on adapting HPX's experimental for loop algorithms for S/R. It was
  separated from the other PR because some race conditions surface and the 
  implemented solution  might lead to a performance hit, making it necessary
  to review and merge this code independently.

Please note that S/R unit tests for algorithms not yet fixed were removed to 
avoid cluttering the tests with pointless, certainly failing test cases. These 
tests were removed in commit 
[a20f1b4 in PR #6494](https://github.com/STEllAR-GROUP/hpx/pull/6494/commits/a20f1b4da8e49de5d4380dd23a5c928525106768), 
making it easy to restore them by reverting this commit if needed.

## What's Left to Do ...

Since there is still a bunch of unadapted algorithms left, getting HPX's 
algorithms to work with S/R, is still a work in progress. To track this 
extensive task [Issue #6535](https://github.com/STEllAR-GROUP/hpx/issues/6535) has been opened on HPX's GitHub.

Additionally, as the standardization of the senders and receivers model for C++
is still ongoing, it's important to keep an eye on the evolution of P2300
and to respond to any changes that could impact usage of S/R in HPX.

However, staying up to date on the standardization efforts should not be limited 
to S/R and P2300 only, but other proposals that focus specifically
on the integration of C++'s algorithms library with S/R, such as 
[P2500](https://wg21.link/p2500) *"C++ parallel algorithms and P2300"*
or [P3300](https://wg21.link/p3300) *"C++ Asynchronous Parallel Algorithms"*,
should be watched as well.

Finally, while working on adapting the algorithm, some problems that are not
directly related to this project and should be tackled in the future, arose.
They are documented in the following GitHub issues:
* [Issue #6534](https://github.com/STEllAR-GROUP/hpx/issues/6534) *Addressing remaining Stdexec issues*
* [Issue #6536](https://github.com/STEllAR-GROUP/hpx/issues/6536) *Non-Standard Conforming Parallel Algorithms*


## Acknowledgements

Many thanks to [**Hartmut Kaiser**](https://github.com/hkaiser), 
[**Isidoros Tsaousis-Seiras**](https://github.com/isidorostsa) and 
[**Panos Syskakis**](https://github.com/Pansysk75) for their valuable support 
throughout this project, and for making our weekly meetings both fun and 
productive.

I’d also like to thank Google and the GSoC program admins for making this 
summer's great experience possible.
