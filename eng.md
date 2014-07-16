<section class="byline"></section>
## Introduction {#toc-introduction}

So you get an email saying how your web-game / web-app is performing badly
after a certain amount of time, you dig through your code, don’t see anything 
that stands out, until you open up Chrome’s memory performance tools, and see 
this:

![I wonder what all those saw-tooths are?][1]
*One of your co-workers chuckles, because they realize that you’ve got a
memory-related performance problem.*

In the memory graph view, this saw-tooth pattern is very telling about a
potentially critical performance problem. As your memory usage grows, you’ll see
the chart area also grow in the timeline capture. When the chart dips suddenly, 
it’s an instance when the Garbage Collector has run, and cleaned up your 
referenced memory objects.

![Look at all those GC Events!][2]
In a graph like this, you can see that there’s lots of Garbage Collection
events occurring, which can be harmful to your web-apps’ performance. This 
article will talk about how to take control of your memory usage, reducing the 
impact on your performance.

## Garbage collection and performance costs {#toc-gcperf}

JavaScript’s [memory model][3] is built on a technology known as a 
[Garbage Collector][4]. In many languages, the programmer is directly
responsible for allocating and freeing memory from the system’s[Memory Heap][5]

In computer science, [Garbage Collector][4] (GC) is a form of automatic memory
management. The garbage collector attempts to reclaim garbage, or memory 
occupied by objects that are no longer in use by the program.

Garbage collection is often portrayed as the opposite of manual memory
management, which requires the programmer to specify which objects to deallocate
and return to the memory system

The process in which a GC reclaims memory is not free, it usually cuts into
your available performance by taking a block of time to do its work; alongside 
that, the system itself makes the decision when to run. You have no control over
this action, a GC pulse can occur at any time during code execution, which will 
block code execution until it’s completed. The duration of this pulse is 
generally unknown to you; will take some amount of time to run, depending on how
your program is utilizing memory at any given point.

[High performance][6] applications rely on consistent performance boundaries to
ensure a smooth experience for users. Garbage collector systems can short 
circuit this goal, as they can run at random times for random durations, eating 
into the available time that the application needs to meet its performance goals.

### Reduce Memory Churn, Reduce Garbage Collection taxes {#toc-churn-taxes}

As noted, a GC pulse will occur once a set of heuristics determines that there
are enough inactive objects that a pulse would be beneficial. As such, the key 
to reducing the amount of time that the Garbage Collector takes from your 
application lies in eliminating as many cases of excessive object creation and 
release as you can. This process of creating/freeing object frequently is called
“memory churn”. If you can reduce memory churn during the lifetime of your 
application, you also reduce the amount of time GC takes from your execution. 
This means you need to remove / reduce the number of created and destroyed 
objects, effectively, you have to stop allocating memory. This process will move
your memory graph from this:

![I wonder what all those saw-tooths are?][1]
to this:

![Ahhhh, that's better.][7]
In this model, you can see that the graph no longer has a [sawtooth][8] like
pattern, but rather grows a great deal in the beginning, and then slowly 
increases over time. If you’re running into performance problems due to memory 
churn, this is the type of graph you’ll want to create.

## Moving towards static-memory JavaScript {#toc-moving-static}

*Static Memory JavaScript* is a technique which involves [pre-allocating][8],
at the start of your app, all the memory that will be needed for its lifetime, 
and managing that memory during execution as objects are no longer needed. We 
can approach this goal in a few simple steps:

1.  Instrument your application to determine what the maximum number of
    required live memory objects (per type) are for a range of usage scenarios
   
2.  Re-implement your code to pre-allocate that maximum amount, and then
    manually fetch/release them rather than going to main memory.
    In reality, accomplishing #1 requires us to do a bit of #2, so let’s
start there.

### Object Pools {#toc-objpool}

In simple terms, [object pooling][9] is the process of retaining a set of
unused objects which share a type. When you need a new object for your code, 
rather than allocating a new one from the system[Memory Heap][5], you instead
recycle one of the unused objects from the pool. Once the external code is done 
with the object, rather than releasing it to main memory, it is returned to the 
pool. Because the object is never[dereferenced][10] (aka deleted) from code it
won’t be garbage collected.**Utilizing object pools puts control of memory back
in the hands of the programmer, reducing the influence of the garbage collector 
on performance.**

Since there’s a heterogenous set of object types that an application
maintains, proper usage of object pools requires you to have one pool per type 
that experiences high-churn during your application’s runtime.

    var newEntity = gEntityObjectPool.allocate();
    newEntity.pos = {x: 215, y: 88};
    
    //..... do some stuff with the object that we need to do
    
    gEntityObjectPool.free(newEntity); //free the object when we’re done
    newEntity = null; //free this object reference
    

For the large majority of applications, you’ll eventually hit some level-off
in terms of needing to allocate new objects. Over multiple runs of your 
application, you should be able to get a great feel for what this upper limit is,
and can pre-allocate that number of objects at the start of your application.

### Pre-allocating objects

Implementing object pooling into your project will give you a theoretical
maximum for the number of objects required during the runtime of your 
application. Once running your site through various testing scenarios, you can 
get a good sense of the types of memory requirements that will be needed, and 
can catalog that data somewhere, and analyze it to understand what the upper 
limits of memory requirements are for your application.

Then, in the shipping version of your app, you can set the initialization phase
to pre-fill all the object pools to a specified amount. This act will push all 
the object initialization to the front of your app, and reduce the amount of 
allocations that occur dynamically during its execution.

    function init() {
      //preallocate all our pools. 
      //Note that we keep each pool homogeneous wrt object types
      gEntityObjectPool.preAllocate(256);
      gDomObjectPool.preAllocate(888);
    }
    

The amount you choose has a great deal to do with the behavior of your
application; sometimes the theoretical maximum isn’t the best option. For 
instance, choosing the average maximum may give you a smaller memory footprint 
for non power-users.

### Far from a silver bullet

There’s a whole classification of apps in which static memory growth patterns
can be a win. As fellow Chrome DevRel[Renato Mangini][11] points out however,
there are a few drawbacks.

## Conclusion

One of the reasons that JavaScript is ideal for the web relies on the fact it
’s a fast, fun and easy language to get started with. This is mainly due to its 
low barrier to syntax restrictions and its handling of the memory issues on your
behalf. You can code away and let it take care of the dirty work. However for 
high-performance web applications, like[HTML5 games][12], the GC can often eat
away at critically needed[frame rate][13], reducing the experience for the end
user. With some careful instrumentation and adoption of object pools, you can 
reduce this burden on your frame rate, and reclaim that time for more awesome 
things.

## Source Code

There are lots of implementations of object pools floating around on the web,
so I won’t bore you with yet another one. Instead, I’ll direct you to these, 
each of which has specific implementation nuances; which is important, 
considering that each application usage may have specific implementation needs.

## References {#toc-references}

 [1]: img/fig1.jpg "A snapshot from your memory timeline"
 [2]: img/fig2.jpg "What the Saw-Tooths mean"
 [3]: http://en.wikipedia.org/wiki/Memory_model_(programming)
 [4]: http://en.wikipedia.org/wiki/Garbage_collection_(computer_science)
 [5]: https://en.wikipedia.org/wiki/Memory_management
 [6]: https://en.wikipedia.org/wiki/Supercomputer
 [7]: img/fig3.jpg "Static Memory Javascript"
 [8]: http://en.wikipedia.org/wiki/Sawtooth_wave
 [9]: http://en.wikipedia.org/wiki/Object_pool_pattern
 [10]: http://en.wikipedia.org/wiki/Reference_(computer_science)
 [11]: https://plus.google.com/u/0/+RenatoMangini/posts
 [12]: https://www.udacity.com/course/cs255
 [13]: http://en.wikipedia.org/wiki/Frame_rate