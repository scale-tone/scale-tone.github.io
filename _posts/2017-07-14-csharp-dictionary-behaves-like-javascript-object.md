---
title: C#: Dictionary<> + operator++ = JavaScript object???
permalink: /2017/07/14/csharp-dictionary-behaves-like-javascript-object
---
![teaser]({{ site.url }}/images/csharp/teaser1.png)
# C#: Dictionary<> + operator++ = JavaScript object???

Consider this JavaScript code:

```
	var d = {"k": 0};
	d["k"]++;
	console.log(d["k"]); // 1
```

As expected, the output is **1**. In JavaScript **d["k"]** is equivalent to **d.k**, so the **++** operator is actually applied to the object's field, it's value changes and **1** is printed. 

Now look at this C++ code:

```
	std::map<std::string, int> d;
	d["k"] = 0;
	d["k"]++;
	std::cout << d["k"]; // 1
```

As expected, the output is **1**. The **std::map**'s **operator[]** actually returns a reference, which is then passed to **operator++**. So, the line

```
	d["k"]++;
```

technically equals to

```
	int& v = d["k"];
	v++;
```
	
, and therefore **1** is printed.

Now let's do the same in C#:

```
	var d = new Dictionary<string, int> { {"k", 0} };
	d["k"]++;
	Console.WriteLine(d["k"]); // 1  But why???..
```

The output is **1** again! But you wouldn't expect that, would you? As we all know, int is a value type, so the Dictionary’s indexer method is supposed to return a copy of the stored value. Which (the copy) would then be sent to **operator++**, and the original stored value (**0**) to be left intact. But this does not happen. Why?!...

The trick is being done by the compiler. For this specific line of code it "silently" adds a call to **Dictionary<>.set_Item()** method, which updates the stored value:

```
    // [15 13 - 15 22]
    IL_0014: ldloc.0      // d
    IL_0015: dup          
    IL_0016: ldstr        "k"
    IL_001b: callvirt     instance !1/*int32*/ class [mscorlib]System.Collections.Generic.Dictionary`2<string, int32>::get_Item(!0/*string*/)
    IL_0020: stloc.1      // V_1
    IL_0021: ldstr        "k"
    IL_0026: ldloc.1      // V_1
    IL_0027: ldc.i4.1     
    IL_0028: add          
    IL_0029: callvirt     instance void class [mscorlib]System.Collections.Generic.Dictionary`2<string, int32>::set_Item(!0/*string*/, !1/*int32*/)
    IL_002e: nop 
```
	
I believe, the creators of C# languate just didn't want to disappoint developers with C++ and JavaScript background and decided to make this little "hack" at the compiler level.

The only tricky thing to remember about that small line of code is that it won't be atomic a.k.a thread-safe. Even if you use **ConcurrentDictionary<>** instead of **Dictionary<>**.

After so many years C# still surprises me sometimes :)