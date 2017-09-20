---
title: Memoization tricks in C#
permalink: /2017/09/18/memoization-tricks
---
![teaser]({{ site.url }}/images/csharp/c-sharp-memoization.png)
# Memoization tricks in C\#

For sure, you’re all aware of the [memoization](https://en.wikipedia.org/wiki/Memoization) concept: when having some rather expensive method, which is called rather often with the same set of parameters, you might want to cache it’s result – and so you memoize it. And you probably have seen a famous and quite elegant implementation of that pattern in C#:

```
/// <summary>
/// Memoizes a function with one parameter
/// </summary>
public static Func<T1, TResult> Memoize<T1, TResult>(this Func<T1, TResult> func)
{
	var cache = new ConcurrentDictionary<T1, TResult>();
	return key => cache.GetOrAdd(key, func);
}
```

_(Explanation: Memoize() takes a [functor](https://en.wikipedia.org/wiki/Function_object) and returns another functor, which either gets the existing result from cache, or calls the original functor, stores it’s result in cache and returns that result. Cache is [captured](https://blogs.msdn.microsoft.com/matt/2008/03/01/understanding-variable-capturing-in-c/) and therefore exists as long as the resulting functor exists. **Functor** is a shorter name for a typed [delegate](https://msdn.microsoft.com/en-us/library/system.delegate(v=vs.110).aspx) instance)._

Which you then normally use like this:

```
static int CalculateTheAnswer(string theQuestion)
{
	// emulate the Deep Thought
	Thread.Sleep(TimeSpan.FromSeconds(10));
	return 42;
}

static void Main(string[] args)
{
	var memoizedCalculationMethod = ((Func<string, int>)CalculateTheAnswer).Memoize();

	var answer1 = memoizedCalculationMethod("The Ultimate Question of Life, The Universe, and Everything"); // returns 42 after 10 seconds
	var answer2 = memoizedCalculationMethod("The Ultimate Question of Life, The Universe, and Everything"); // returns 42 immediately
}
```

If your calculation method takes e.g. two parameters, you can extend the approach with similar technics:

```
/// <summary>
/// Memoizes a function with two parameters
/// </summary>
public static Func<T1, T2, TResult> Memoize<T1, T2, TResult>(this Func<T1, T2, TResult> func)
{
	var memoizedFunc = new Func<Tuple<T1, T2>, TResult>(tuple => func(tuple.Item1, tuple.Item2)).Memoize();
	return (t1, t2) => memoizedFunc(new Tuple<T1, T2>(t1, t2));
}
```

And so on and so forth for more parameters.

Still, this approach has a drawback, which makes it quite hazardous in the hands of… less experienced developers :)
It only works well for methods, that take [value-typed](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/value-types) or string parameters. Or e.g. for methods with parameters of type [**Type**](https://msdn.microsoft.com/en-us/library/system.type(v=vs.110).aspx). Or e.g...

While trying to use it with a method, that takes some object (reference-typed) parameters, e.g.:

```
static double CalculateArea(Region region)
{
	Thread.Sleep(TimeSpan.FromSeconds(10));
	return 42;
}
```

is a straightforward way to [shoot your own foot](https://english.stackexchange.com/questions/137187/what-does-it-mean-to-shoot-oneself-in-the-foot). Not only it doesn’t memoize anything anymore, but, which is even worse, the memory is leaking (until it reaches the end).

So, is there a way to protect ourselves from such unexpected behavior? Or at least to provide a clear warning whenever someone tries to misuse our memoization method?

To figure that out let's take a closer look at the requirements for those parameter types. In other words, let's determine the required and sufficient set of conditions for a type, that can be safely used for parameters of that memoizable method. And it's actually pretty simple: we store parameters and cached results in a **Dictionary\<TKey, TValue\>** (more precisely, in a **ConcurrentDictionary\<TKey, TValue\>**), therefore the parameter type should be able to propely act as a **Dictionary** key. And to be a proper **Dictionary** key, a type should properly implement [Equals()](https://msdn.microsoft.com/en-us/library/bsc2ak47(v=vs.110).aspx) and [GetHashCode()](https://msdn.microsoft.com/en-us/library/system.object.gethashcode(v=vs.110).aspx) methods. Value types do implement those methods properly (btw. did you know, that `123.GetHashCode() == 123` ?), as well as e.g. **string** type. But custom reference types, they, of course, by default inherit the implementation from **object**. Which is no good for us because, as you know, two different object instances are never equal to each other. So, in order to be valid parameters for a memoizable method, custom reference types need to implement those two methods in some more meaningful way. How do we check that they do?

Well, of course, it would be hard to automatically check, that an arbitrary class implements [Equals()](https://msdn.microsoft.com/en-us/library/bsc2ak47(v=vs.110).aspx) and  [GetHashCode()](https://msdn.microsoft.com/en-us/library/system.object.gethashcode(v=vs.110).aspx) _properly_. But what we can do - is to check, that the class _overrides_ those methods, by comparing [MethodInfo](https://msdn.microsoft.com/en-us/library/system.reflection.methodinfo(v=vs.110).aspx)'s:

```
/// <summary>
/// Checks, that an object overrides GetHashCode() and Equals(), which is crucial for memoization to work
/// </summary>
private static bool ObjectOverridesGetHashCodeAndEquals(object obj)
{
	return
	(
		(((Func<int>)obj.GetHashCode).GetMethodInfo() != ObjectGetHashCode)
		&&
		(((Predicate<object>)obj.Equals).GetMethodInfo() != ObjectEquals)
	);
}
private static readonly MethodInfo ObjectGetHashCode = ((Func<int>)new object().GetHashCode).GetMethodInfo();
private static readonly MethodInfo ObjectEquals = ((Predicate<object>)new object().Equals).GetMethodInfo();
```

And now, with that in place, we just need to pepper our **Memoize()** methods with this check:

```
public static Func<T1, TResult> Memoize<T1, TResult>(this Func<T1, TResult> func)
{
	var cache = new ConcurrentDictionary<T1, TResult>();
	return key =>
	{
		// For the sake of performance we only do this check in Debug mode
		Debug.Assert(ObjectOverridesGetHashCodeAndEquals(key));
		return cache.GetOrAdd(key, func);
	};
}
```

Yes, this approach does not give you a 100% guarantee, that your Memoize() won't ever be misused. But it does protect you from most common stupid bugs, and that seems reasonable enough.