---
title: On the matter of applying the Monte Carlo Method to investigating bugs in Software Development.
permalink: /2019/12/06/on-the-matter-of-monte-carlo-method-in-sw-dev
---
![teaser]({{ site.url }}/images/monte-carlo/teaser1.png)
# On the matter of applying the Monte Carlo Method to investigating bugs in Software Development.

Many people ask me: how to be successful in investigating complex problems, finding the root cause of sophisticated failures and analyzing most nasty [heisenbugs](https://en.wikipedia.org/wiki/Heisenbug)? Well, not so many. Well, in fact, it was just me asking myself :) But still, sometimes it seems like some people do have a gift for that, while others use to give up too quickly or tend to just sit and stare...

Here I could probably point to some cool scientific disciplines like [Abductive Reasoning](https://en.wikipedia.org/wiki/Abductive_reasoning) or [Design of Experiments](https://en.wikipedia.org/wiki/Design_of_experiments). But each of those is a complete rocket science to me, so I'd better tell you about the [Monte Carlo Method](https://en.wikipedia.org/wiki/Monte_Carlo_method).

If you're a mathematician struggling to integrate an unintegratable (or hardly integratable) function, or if you're a physicist measuring the amount of [arsenic](https://en.wikipedia.org/wiki/Arsenic) in your piece of [silicon](https://en.wikipedia.org/wiki/Silicon) with your [electron microscope](https://en.wikipedia.org/wiki/Electron_microscope), or if you you're a financial analyst estimating whether it is worth investing into some new technology - then Monte Carlo Method is your friend. The idea is very simple. Let's say, you need to calculate the area below some function graph (see the picture above), but you can't do that precisely, because calculations would take ages. But good news is that for each particular point you can instantly say whether it is below or above that graph. So what you do is just picking up random points and check them. The more points you check - the more precise your estimation is.

It is known that the first "modern" version of Monte Carlo Method was applied in the late 1940s at the Los Alamos National Laboratory for analyzing the ways neutrons transport through materials (they even made a "hardware" implementation of it - [FERMIAC](https://en.wikipedia.org/wiki/FERMIAC)). And the method took its name from [Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo) - a well-known gambling place in Europe. While the idea of picking up input parameters randomly for estimating something was widely used long before - e.g. for [calculating the value of PI](https://en.wikipedia.org/wiki/Buffon%27s_needle_problem).

I never had a chance to study or apply the above method in practice, so it is a complete rocket science to me either. But what I do know for sure is that **Monte Сarlo Method doesn't work when it comes to investigating bugs and issues in Software Development!**

Randomly picking up things to try and hoping for the best is never a good idea. A much more efficient way would be to utilize what I call a "Scientific Try" approach (you could also call it "Scientific Thinking" or just "Common Sense"), which means that you need to:
1. Obviously, collect as much information about the issue as possible (error messages, logs, results of previous experiments etc.).
2. Organize all the things to try (that you have in mind so far) according to some order. Then try them **one-by-one** and **write down** the results. Making experiments one-by-one is important to ensure that they do not influence each other. And writing down the results is important because otherwise you'll quickly forget the sequence of conclusions that brought you to the current state.
3. If the issue still not resolved, repeat the process from step1.

Yes, sometimes this approach takes too many iterations. Well, you need to be patient. And yes, sometimes it might seem that you've run out of things to try at step2. Well, what typically helps me in that situation is to put the problem aside for a while and think it over later. 

What gives me strength to continue looping through those steps even when there seems to be no hope left? And that is going to be another fundamental statement here, which is: **There're no unresolvable issues in Software Development**. This is actually a theorem, which I do believe someone has already bothered themselves to find a formal proof for. But that proof becomes sort of obvious, if you remember that at least so far all computers in the world are based on some sort of **logic** (boolean, trinary, whatever). So it's just a matter of time for you to understand that logic and figure out what's wrong with it :)

Bad news though is that human life is still finite...