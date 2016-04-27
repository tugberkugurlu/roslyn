# C# designe rationale and alternatives: arbitrary async returns

*This document explores the design space for the feature proposal [arbitrary async returns](feature - arbitrary async returns.md).


## Discuss: connection between tasklike and builder

**Question.** How does the compiler know which builder type to use for a given tasklike?
```csharp
// Option1: via attribute on the tasklike type itself
[Tasklike(typeof(BuilderType))] class MyTasklike { ... }

// Option2: via an attribute on the async method
[Tasklike(typeof(BuilderType))] async Task TestAsync() { ... }

// Option3: via a dummy call to "var builder = default(Tasklike).GetBuilder();"
public static class Extensions
{
	public static void GetBuilder(this MyTasklike dummy) => new BuilderType();
}
```
Option2 has the slight benefit of being able to specify a builder even when you're returning the existing `Task`. But it's worse for the typical `ValueTask` usecase because it requires you to type out the attribute every single time you want to return `ValueTask`. It also doesn't work with lambdas, which don't have a place to hang that attribute.

Option3 is ugly. We could live with that ugliness if it was useful to extend third-party tasklike types, but experience is that the implementation of the builder and the tasklike are closely related, so closely that it's not feasible to build someone else's tasklike. So the ugliness isn't merited.

Option3 has the slight advantage of not requiring `TasklikeAttribute` to be defined somewhere.


## Discuss: genericity of tasklike and builder

**Question.** Why do you need a non-generic `MyBuilder` to build a non-generic `MyTask`? And why do you need an arity-1 `MyBuilder<T>` to build an arity-1 `MyTask<T>`? Why can't we be more flexible about arities?

**Question.** Why can't we write `[Tasklike(typeof(MyBuilder<object>))] MyTask` and use an instantiated `MyBuilder<object>` as the builder-type for building a non-generic tasklike `MyTask`?

*These two things might be possible for top-level methods, but they don't work with lambdas and type inference. Let's spell out the related question about type inference:*

**Question.** When the compiler does generic type inference with the argument `async()=>{return 3;}` being passed to a method `void f(Func<MyTask<T>> lambda)`, how does it go from `3` to `int` to `T = int` ?


**Current behavior:** Let's start with the traditional behavior for `Task`:
```csharp
void f(Func<Task<T>> lambda);
var xf = f(async () => {return 3;});
```
* This infers `T = int`


**Proposal:** Under the proposal, it will work like this:
```csharp
void g(Func<MyTasklike<T>> lambda);
var xg = g(async () => {return 3;});
```
* Under the proposal, we required a generic builder `MyBuilder<T>`
* and we first inferred the *result type* `int` from the lambda. (Result type is the type of the return operand)
* and figured out that `T = int`
* and we pick the concrete type `var builder = MyBuilder<int>.Create()`
* and we happily called the `builder.SetResult(3)` method


**More general attempt 1:** But can we make it more general? Like this?
```csharp
void h(Func<MyPerverse<T>> lambda);
var xh = g(async () => {return 3;});
// Imagine if we tried to do something more general...
// class MyPerverseBuilder<U> {
//   public void SetResult(U value) { }
//   public MyPerverse<IEnumerable<U>> Task { get; }
// }
```
* Start from the inferred result type of the lambda `int`
* to see that *the builder* has a SetResult method that takes `U`,
* and therefore `U = int`,
* and therefore the builder was a `MyPerverseBuilder<int>`
* and therefore, by looking at its `Task` property, we get `T = IEnumerable<int>`


**More general attempt 2:** And how about this generalization?
```csharp
void k(Func<MyWeird<T>> lambda);
var xk = k(async () => {return 3;});
// As before, can we do something more general?
// class MyWeirdBuilder {
//   public void SetResult<U>(U value) { }
//   public MyWeird<string> Task { get; }
// }
```
* Start from the inferred result type of the lambda `int`
* to see that *the builder* has a SetResult method that takes `U`
* and therefore `U = int`
* but that doesn't inform us about the builder; instead the builder is just `MyWeirdBuilder`
* and therefore, by looking at its `Task` property, we get `T = string` ?

**Impossible:** The two general attempts aren't possible: they both fall down in the second step, when they attempt to look for `SetResult` methods on the builder type, in order to infer the builder's type. But this is circular since it presupposes knowing the builder type!



## Discuss: async method interacts with the builder instance

In the "async pattern", cancellation and progress are done with parameters:
```csharp
void f(int param, CancellationToken cancel, IProgress<string> progress)
```

But for some tasklikes, cancellation and progress are instead faculties of the tasklike (equivalently, of its builder)...
```csharp
// Windows async tasklikes:
async IAsyncActionWithProgress<string> TestAsync() { ... }
var a = TestAsync();
a.Progress += (s) => Console.WriteLine($"progress {s}");
a.Start();
a.Cancel();
```

```csharp
// IObservable tasklike:
async IObservable<string> TestAsync() { ... }
var s = TestAsync().Subscribe(s => Console.WriteLine(s));
s.Dispose(); // maybe this should cancel the work
```

It's possible, in the proposal as outlined, for the async method body to communicate with its builder. The way is a bit hacky: the async method has to await a custom awaiter, and the builder's `AwaitOnCompleted` method is responsible for recognizing that custom awaiter and doing the right thing:
```csharp
async IAsyncActionWithProgress<string> TestAsync()
{
   var (cancel, progress) = await new MyCustomAwaiter();
   ...
}
```

It would be possible to augment the language further, so that within the body of an async method it can use a contextual keyword like `async` to refer in a strongly-typed way its current builder, e.g. `var c = async.CancellationToken` or `async.Progress?.Invoke(i)`. That's an interesting idea that I want to explore more fully in the area of `IAsyncEnumerable`.

I also wonder whether the *caller* could construct and manipulate the builder in code before the async method started, to give it some context. But I don't see any good way to write this.



## Discuss: overload resolution with async lambdas

There's a thorny issue around overload resolution. The proposal has one solution for it. I want to outline the problem and discuss alternatives. We have to build up to the problem with some examples...

**Example1:** This is allowed and infers `T = int`. Effectively, type inference can "dig through" `Task<T>`. This is a really common pattern.
```csharp
// Example 1
void f<T>(Func<Task<T>> lambda)
f(async () => 3); // infers T = int
```

**Example2:** This is also allowed and infers `T = Task<int>`. Effectively, type inference can *synthesize* the type `Task<>` based solely on the `async` modifier. It's a weird behavior, one that doesn't happen elsewhere in the language, and is typically the wrong thing to do (because it's rare that you can write a correct `f` which handles both async Task-returning lambdas and non-async lambdas).
```csharp
// Example 2
void f<T>(Func<T> lambda)
f(async () => 3); // infers T = Task<int>
```


**Example3:** When the two examples above are overloaded, the compiler has rules to pick the winner. In particular, if two overload candidates have identical parameter types (after inferred types have been substituted in) then it uses the *more specific candidate* **tie-breaker rule** ([Better function member](https://github.com/ljw1004/csharpspec/blob/gh-pages/expressions.md#better-function-member)). This *tie-breaker* ends up being used a lot for async lambdas, particularly for things like `Task.Run`.
```csharp
// Example 3
void f<T>(Func<Task<T>> lambda)   // infers T = int and is applicable
void f<T>(Func<T> lambda)         // infers T = Task<int> and is applicable
f(async () => 3);   // picks more specific first overload with T = int
```

**Example4:** We want type inference to be able to dig through other Tasklikes as well, not just `Task`. But we can never change the Example3 rule which gives privileged inference to `Task<T>`:
```csharp
// Example 4
void f<T>(Func<MyTask<T>> lambda)
f(async () => 3);   // we want this to work and infer T = int

void f<T>(Func<T> lambda)
f(async () => 3);   // for back-compat, this will always infer T = Task<int>
```

**Example5:** So what do we want to happen when the two cases of Example4 are overloaded?
```csharp
// Example 5
void f<T>(Func<MyTask<T>> lambda)  // infers T = int and is applicable
void f<T>(Func<T> lambda)          // infers T = Task<int> and is applicable
f(async () => 3);   // what should this do?
```
If we do indeed change type inference to dig through Tasklikes, but we don't change the rules for overload resolution, then it will give an **ambiguous overload** error. That's because it looks at the two candidates `f(Func<MyTask<int>> lambda)` and `f(Func<Task<int>> lambda)`...
* The two don't have identical parameter types, so it won't go through the *tie-breaker* rules about more specific
* Instead it looks at the rules for **better function member**
* This requires to judge whether the conversion from expression `async()=>3` to type `Func<MyTask<int>>` or to type `Func<Task<int>>` is a [better conversion from expression](https://github.com/ljw1004/csharpspec/blob/gh-pages/expressions.md#better-conversion-from-expression)
* This requires to judge whether the type `Func<MyTask<int>>` or `Func<Task<int>>` is a [better conversion target](https://github.com/ljw1004/csharpspec/blob/gh-pages/expressions.md#better-conversion-target)
* This requires to judge whether there is an implicit conversion from `MyTask<int>` to `Task<int>` or vice versa.

Folks might decide to have user-defined implicit conversions between their tasklikes and `Task`. But that's beside the point. The only way we'll get a good overload resolution disambiguation between the candidates is if we go down the *tie-breaker* path to pick the more specific candidate. Once we instead start down the *better function member* route, it's already too late, and forever will it dominate our destiny.

**Problem statement: Somehow, Example5 should pick the first candidate on grounds that it's more specific.**

### Overload resolution approach 0

**Approach0:** We could just leave Example5 to give an ambiguous overload error, i.e. not do anything.

This would be a shame. It would disallow some common patterns like `Task.Run`. Also, as discussed in Example2, you can't really write a single method `f<T>(Func<T> lambda)` which works right for both normal and async lambdas, so you really do have to provide the second overload, so Example5 is a common case.

I think that folks should be able to come up with their own parallel world of `MyTask` that looks and feels like `Task`, with the same level of compiler support.

### Overload resolution approach 1

**Approach1:** We could use two new pseudo-types `InferredTask` and `InferredTask<T>` as follows:

1. We modify type inference so that, whenever it's asked to infer the *return* type of an async lambda, then in the non-generic case it always gives `InferredTask` and in the generic case it always gives `InferredTask<T>` where `T` is the inferred *result* type of the lambda.
2. These new pseudo-types are merely placeholders for the fact that type inferences wanted to infer a tasklike but is flexible as to which one.
3. When overload resolution attempts to judge whether two parameter sequences `{P1...Pn}` and `{Q1...Qn}` are identical, it treats these pseudo-types as identical to themselves and all tasklikes of the same arity. This would allow it to go down the *more specific* tie-breaker route.
4. If overload resolution picks a winning candidate with one of the pseudo-types, only then does the pseudo-type get collapsed down to the concrete type `Task` or `Task<T>`. 

I struggled to make sense of this approach. Consider what is the principle behind this type inference? One possible principle is this: *when type inference succeeds and mentions one of the pseudo-types, it is a statement that **all tasklikes would be equally applicable in its place**. * But that's not really how type inference works in C#. Consider:
```csharp
void f<T>(Func<T> lambda1, Func<T,T> lambda2)
f( ()=>3, (x)=>x.ToString());
```
This will happily infer `T = int` and simply not bother checking whether this candidate for `T` makes sense for the second argument. In general, C# type inference doesn't guarantee that successful type inference will produce parameter types that are applicable to arguments. This is a weak foundation upon which to build "all tasklikes would be applicable".

The opposite possible principle behind type inference is that *it should aggressively prefer to give type inference results that include the pseudo-types* rather than any particular concrete tasklikes. The rationale here is that the whole point of the pseudo-types is to encourage candidates to be treated equivalent up to tasklikeness that arises from async lambdas. But this principle doesn't feel very principled... (see Approach 2 below).

Next we'd have to decide how the chosen principle informs how the pseudo-types work with [Fixing](https://github.com/ljw1004/csharpspec/blob/gh-pages/expressions.md#fixing). For instance, if a generic type parameter has lower bounds `InferredTask<int>` and `Task<int>` then what should it fix as? What if it has lower bounds `IEnumerable<InferredTask<int>>` and `IEnumerable<Task<int>>` and `IEnumerable<MyTask<int>>`?

We'd also have to decide how the pseudo-types work with [lower bound inference](https://github.com/ljw1004/csharpspec/blob/gh-pages/expressions.md#lower-bound-inferences) and upper bound inference and exact bound inference. For instance, if doing a lower bound inference from `InferredTask<int>` to `Task<T>`, we can certainly dig in to do a lower bound inference from `int` to `T`, but should we also do an exact inference from `Task` to this `InferredTask`? How about lower bound inference from `Task<int>` to `InferredTask<T>`? How about an upper bound inference?

We'd also have to decide how the pseudo-types work with [applicable function member](https://github.com/ljw1004/csharpspec/blob/gh-pages/expressions.md#applicable-function-member). Presumably any async lambda has an implicit conversion to a delegate type with return type `InferredTask<T>`. But what about all the other implicit conversions?

In the end, there are too many difficult questions to address here, and we can't give good answers because we don't have good principles to start from.

The next approach takes the second principle, and turns it into something simpler and more intellectually defensible.

### Overload resolution approach 2

**Approach2:** The whole point is that we want tasklikes to go down the "tie-breaker" path. So let's achieve that directly: if no candidate is better, then *when judging whether `{P1...Pn}` and `{Q1...Qn}` are identical, do so **up to tasklikes**. * This way we don't need to mess with pseudo-types.

This is the approach put forwards in the the feature proposal.

I'd initially put it forwards only as a joke. It seems shocking! My first instinct is that we should only allow tasklike equivalence where it relates directly to an async lambda (as in approach 1), not everywhere.

But the thing is, this change in rule only applies if none of the applicable candidates was a better function member. It only ever turns ambiugity-errors into success. So the way to judge the merits of this approach are whether it allows any winners in cases where we think it should really be an ambiguity failure.

Honestly? There aren't any, not that I can think of.

```csharp
// EXAMPLE 1
void f(Task<int> t) => 0;
void f(MyTask<int t) => 1;
Task<int> arg; f(arg); // prefers "0" because it is a better function member (identity)

// EXAMPLE 2
void f<T>(Task<T> t) => 0;
void f<T>(MyTask<T> t) => 1;
Task<int> arg; f(arg); // prefers "0" because the other candidate fails type inference

// EXAMPLE 3
void f<T>(Task<T> t) => 0;
void f(MyTaks<int> t) => 1;
Task<int> arg; f(arg); // prefers "0" because it's a better function member (identity)
```


## Discuss: IObservable

It would be great if this feature -- along with async iterators -- could offer native support for producing `IObservable`.

> The question of *consuming* `IObservable` is different. To write imperative (pull-based) code that consumes `IObservable` (push-based) you need a buffer of some sort. That's orthogonal and I don't want to get derailed.

I don't know what kind of semantics you'd expect from an `IObservable`-returning async method. Here are two options:

```csharp
async IObservable<string> Option1()
{
    // This behaves a lot like Task<string>.ToObservable() -- the async method starts the moment
    // you execute Option1, and if you subscribe while it's in-flight then you'll get an OnNext+OnCompleted
    // as soon as the return statement executes, and if you subscribe later then you'll get OnNext+OnCompleted
    // immediately the moment you subscribe (with a saved value).
    // Presumably there's no way for Subscription.Dispose() to cancel the async method...
    await Task.Delay(100);
    return "hello";
}

async IObservable<string> Option2()
{
    // This behaves a lot like Observable.Create() -- a fresh instance of the async method starts
    // up for each subscriber at the moment of subscription, and you'll get an OnNext the moment
    // each yield executes, and an OnCompleted when the method ends.
    await Task.Delay(100);
    yield return "hello";
    await Task.Delay(200);
    yield return "world";
    await Task.Delay(300);
}
```

I don't know which of these options feels best. Probably Option2, since Option1 is akin (in the words of Reed Copsey) to an iterator method with only a single yield. Pretty unusual.

I also don't know how cancellation should work. The scenario I imagine is that an `async IObservable` method has issued a web request, but then the subscriber disposes of its subscription. Presumably you'd want to cancel the web request immediately? Is disposal the primary (sole?) way for `IObservable` folks to cancel their aysnc sequences? Or do they prefer to have a separate stream which receives cancellation requests?

Also, when you call `Dispose` on an iterator it's a bit different. By definition the iterator isn't currently executing. So all it does is resume iterator execution by going straight to the finally blocks. This might be good enough for async - e.g. the finally block could signal a cancellation that was being used throughout the method. (It could even await until the outstanding requests had indeed finished their cancellation, although this would be cumbersome to code.) But it's uneasy, and I wonder if a different form of cancellation is needed?

We need guidance from `IObservable` subject matter experts.