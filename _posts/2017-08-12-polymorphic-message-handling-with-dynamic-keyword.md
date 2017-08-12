---
title: Polymorphic message handling with dynamic keyword in C#
permalink: /2017/08/12/polymorphic-message-handling-with-dynamic-keyword
---
![teaser]({{ site.url }}/images/csharp/c-sharp-dynamic.png)
# Polymorphic message handling with dynamic keyword in C##

Imagine that you have a hierarchy of messages, something like these:

```
class MessageBase
{
	// more fields and properties
}

class MessageA : MessageBase
{
	// more fields and properties
}

class MessageB : MessageBase
{
	// more fields and properties
}
```

, and those messages keep arriving e.g. from some message queue. Now you need to implement handlers for every specific message type. How would you do that with minimum efforts?

The naïve approach would be to construct a huge **switch(…)** statement for choosing a correct handler for every message type.
Messaging libraries, like [NServiceBus](https://particular.net/nservicebus), achieve the same target by doing some sophisticated reflection magic.
But today we can do the same thing in a much more elegant way by utilizing the brand new **dynamic** keyword.

You just write your handlers as methods:

```
static void Handle(MessageA message)
{
	Console.WriteLine("We got MessageA here");
}

static void Handle(MessageB message)
{
	Console.WriteLine("We got MessageB here");
}
```

And then pass your message to those handlers as a **dynamic** variable.

```
dynamic msg;

msg = new MessageA();
Handle(msg); // We got MessageA here

msg = new MessageB();
Handle(msg); // We got MessageB here
```

.Net does the rest for you. It picks up the correct handler at runtime, or throws an exception, if there isn’t any. 
If you don’t want an exception to be thrown, you can always define a “default” handler:

```
static void Handle(object message)
{
	Console.WriteLine("We got an object here");
}
```

Now what if you need to have a truly polymorphic handling, that is, some generic handler that is being executed for all messages that are of type **MessageBase** and also some specific handlers to be executed for **MessageA** and  **MessageB** (this is actually a typical scenario for handling events, like **GenericEvent**, **OrderReceivedEvent** and **WholesaleOrderReceivedEvent**)? 

You can do that by constructing a hierarchy of message handlers (one per every level in message hierarchy), like these:

```
class MessageHandlerBase
{
	public void Handle(object message)
	{
		Console.WriteLine("We got an object here");
	}

	public void Handle(MessageBase message)
	{
		this.Handle((object)message);
		Console.WriteLine("We got MessageBase here");
	}
}

class SpecificMessageHandler: MessageHandlerBase
{
	public void Handle(MessageA message)
	{
		base.Handle(message);
		Console.WriteLine("We got MessageA here");
	}

	public void Handle(MessageB message)
	{
		base.Handle(message);
		Console.WriteLine("We got MessageB here");
	}
}
```

And now:

```
static IEnumerable<object> FetchMessages()
{
	// some sample messages
	yield return 123;
	yield return new MessageBase();
	yield return new MessageA();
	yield return new MessageB();
}

static void Main(string[] args)
{
	var handler = new SpecificMessageHandler();

	foreach (dynamic message in FetchMessages())
	{
		Console.WriteLine("About to process message of type " + message.GetType().Name);
		handler.Handle(message);
		Console.WriteLine("------------");
	}
}
```

will output the following:

```
About to process message of type Int32
We got an object here
------------
About to process message of type MessageBase
We got an object here
We got MessageBase here
------------
About to process message of type MessageA
We got an object here
We got MessageBase here
We got MessageA here
------------
About to process message of type MessageB
We got an object here
We got MessageBase here
We got MessageB here
```

So that all the required handlers are executed for each specific message type.

That's exactly how I like it: smart and simple :)