---
title: My favourite catastrophic failures. №1. StackOverflowException.
permalink: /2018/01/20/my-favourite-catastrophic-failures-stackoverflowexception
---
![teaser]({{ site.url }}/images/csharp/catastrophic-failure.png)
# My favourite catastrophic failures. №1. StackOverflowException.

Let me start this with a small quiz. I'm sure you know that **finally** blocks in C# are *always* executed, no matter what happens underway. Even if you have a **return** statement inside your **try** block. But given the following simple piece of code:

```
static int Main()
{
    try
    {
        Console.WriteLine("Inside try block...");

        // Add something catastrophic here

        return 0;
    }
    finally
    {
        Console.WriteLine("Typically, some cleanup here...");
    }
}
```

, can you replace the comment inside **try** block with one line, that would lead the **finally** block to never be executed?
No, it shouldn't be an infinite loop :) And no, it shouldn't be **Environment.Exit(0)**.

Yes, exactly. That would be a recursive call to **Main()** itself:

```
static int Main()
{
    try
    {
        Console.WriteLine("Inside try block...");

        // Ooops...
        Main();

        return 0;
    }
    finally
    {
        Console.WriteLine("Typically, some cleanup here...");
    }
}
```

The result would be a devastating **StackOverflowException** followed by a process crash.

**StackOverflowException** is the one of so called [uncatchable](https://en.wikipedia.org/wiki/Russian_jokes#Cowboys) exceptions in .Net (others are **OutOfMemoryException** and **ThreadAbortException**, but the list might differ from one .Net version to another). As the name states, ~~nobody wants them~~ they cannot be caught with a **try-catch** block and therefore do not cause **finally{}** block to be executed as well. The idea was, that these types of exceptions typically indicate such a huge application state corruption, that it’s safer to shoot the process right away rather than trying to recover (also note, that just doing a *throw new StackOverflowException()* doesn’t do the trick, you'll need an infinite recursion for that).

The decision of .Net creators to include **StackOverflowException** into that list is often considered arguable (it isn't so in JavaScript and it isn't so in Java, AFAIK). Unlike other two, **StackOverflowException** is a lot more likely to happen in a common .Net application/service, just because it is so much easier to introduce a bug that causes it. And that’s scary.

Imagine that such an infinite recursion bug, by some unfortunate confluence of circumstances (barely reachable codepath, not caught by either unit or integration tests, etc.), reaches the production endpoint of your ASP.Net application. What goes next? AppPools on your server instances start to suddenly die one by one. Web server (typically, IIS, but it might be any other host you prefer) tries to restart them for a limited amount of times, but then gives up and just responds with **503 Service Unavailable**. Application logs are as clean as a teardrop, and it's only the huge RpS drop that might alert you of something bad going on. Then you go investigating, and for you it looks like one single particular request crashes the whole service (while you know for sure, that there can be no any unhandled exceptions in your code). And that's the most misleading thing, because it makes you think of possible hacker attacks, platform vulnerabilities etc., while in fact it's just a stupid bug in your own code. Then you finally got a quick fix (or an old build to rollback) - but what if you have hundreds of thousands of machines to deploy the fix to?.. Apocalypse.

More technically speaking, it's not the process but the [AppDomain](https://msdn.microsoft.com/en-us/library/windows/desktop/system.appdomain(v=vs.85).aspx), that is being killed upon **StackOverflowException**. Therefore e.g. Microsoft SQL Server developers (and any other developers who deal with separate **AppDomains**) they do have a way to cope with **StackOverflowExceptions** in some buggy third-party code. But running your own separate **AppDomains** is quite an advanced topic, which I'll maybe write on in a separate post in some future :)