---
layout: post
title: Circular Increment
date: 2011-03-17
---

While some of the larger articles are still in the oven, I'll drop a little programming tip that most experienced programmers should already know, but I happen to find quite useful in a lot of scenarios. Here's the problem: you have an array of size *n* and an index *i* into that array. You'd like to increment *i* so that it points to the next position in the array, but have it return to the first position once the end of the array is reached. The most obvious way to write this would be something like:

~~~ c#
// The conventional way
++i;
if (i == n)
{
    i = 0;
}
~~~

But for some situations it's useful to know that you can also write this operation as a single statement, either by using the modulus operator or the ternary operator:

~~~ c#
// Using the modulus operator
i = (i + 1) % n;

// Using the ternary operator
i = (i == n-1) ? 0 : i+1;
~~~

I have personally grown to prefer the modulus operator version, because it is shorter, and because I've come to associate the modulus operator with range limiting whenever I see it.

## Usage Example

> Before continuing I should note that some readers have pointed out that the modulus operator is expensive, so it shouldn't be placed inside of any large loops. Therefore if you're worried about a performance hit, do not use this technique inside any loop (like I'll be doing next), and save it for one-shot scenarios. Having said that, I still find it useful when prototyping since it lets me get things done in fewer lines of code. And of course, it's perfectly fine to use it for things that won't be running every frame. For instance, if your character has a bunch of different weapons, and you'd like to cycle through them when the user presses a key, that's a nice place to use it!
{: .note}

Here's a XNA example where I find this trick useful. Let's say we have a list of vertices defining a polygon, and we'd like to draw the polygon's outline by using a **DrawLine(from, to)** function such as the one described [here](http://www.david-amador.com/2010/01/drawing-lines-in-xna/). Before I discovered this trick, I'd loop through each vertex in the list and draw a line from that vertex to the next one, and then handle the final line connecting the last and the first vertices separatedly:

~~~ c#
// Draw a line from each vertex to the next
for (int i = 0; i < vertices.Count - 1; ++i)
{
    DrawLine(vertices[i], vertices[i + 1]);
}
// Close the loop by connecting the last vertex to the first
DrawLine(vertices[vertices.Count - 1], vertices[0]);
~~~

But using one of the tricks outlined earlier, you can easily handle the final case inside the for loop with minor modifications:

~~~ c#
// Using the modulus operator
for(int i = 0; i < vertices.Count; ++i)
{
    DrawLine(vertices[i], vertices[(i+1) % vertices.Count]);
}

// Using the ternary operator
for (int i = 0; i < vertices.Count; ++i)
{
    DrawLine(vertices[i], vertices[(i == vertices.Count - 1) ? 0 : i + 1]);
}
~~~

Like I said, this trick does come at a performance cost since the modulus operator is likely more expensive than a simple branch. Despite that, I can't say I've ever noticed any sort of slowdown due to this, probably because I have only been developing for Windows, and all of the loops I use it in are rather small. Either way, it's a nice trick to have in the bag, if only so that you won't get alienated the next time you see that % operator used this way (e.g. I've seen it used today in one of the XNA education samples).
