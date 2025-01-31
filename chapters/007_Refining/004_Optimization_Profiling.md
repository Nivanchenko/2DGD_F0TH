{{pagebreak}}

Profiling and Optimization
==========================

:::::: {.epigraph author="Donald Knuth - Computer Programming as an Art"}
The real problem is that programmers have spent far too much time worrying about efficiency in the wrong places and at the wrong times; premature optimization is the root of all evil (or at least most of it) in programming.
::::::

Profiling your game
-------------------

### Does your application really need profiling?

In this section we will have a small check list that will let us know if our videogame really needs to be profiled. Sometimes the FPS counter is trying to tell us a different story than the one we have in our heads.

#### Does your FPS counter roam around a certain "special" value?

There are cases where the FPS counter shows a low counter, but it stays around a certain value. This means that the FPS value is artificially limited somewhere, either by VSync or something else.

Some special values you may see are:

- 25 FPS: PAL refresh rate
- 29.970 FPS: NTSC Refresh Rate
- 30 FPS: Used in some games
- 50 FPS: Used in some games
- 60 FPS: Used in most games
- 75 FPS or 80 FPS: Used in some LCD Monitors
- 90 FPS: Used mostly in VR games
- 144 FPS: Used in more modern, high-refresh rate monitors
- 240 FPS: Used in the most recent high-end games and monitors

#### Is the animation of your game stuttering but the FPS counter is fine?

If your animation stutters or its speed varies according to the load of your platform but your FPS counter is still stuck at the maximum allowed framerate, you may have forgotten to tie the animation to the delta-time in your game loop. Check the [timing your game loop](#timingloops) section for more information.

{{placeholder}}
<!-- TODO: Continue with more profiling suggestions -->

### First investigations

First of all, we need to understand what is the bottleneck of your game: check your task manager and see how your game is performing.

#### Is your game using 100% of the CPU?

Is your game using 100% of the CPU (if you're on Linux, you may see percentages over 100%, that just means your game is using more than one CPU core)?

First of all, you should check if you're using the frame limiting approaches offered by your framework or game engine: if they're not active, your game will run "as fast as possible", which means it will occupy all the CPU time it can. This can result in high FPS count (in the thousands) but high energy consumption and slowdowns in other tasks.

If you have taken all the frame limiting approaches as stated above, that may mean that the game is doing a lot of CPU work and you may need to make the game perform less work for each frame. In this case profiling tools are precious to find the spots where the CPU spends most of its time: Valgrind or GProf are great profiling tools.

![Using Valgrind's Callgrind tool and Kcachegrind we can see what is bogging down our game](./images/profiling_optimization/callgrind.png){width=60%}

#### Is your game overloading your GPU?

If instead your game is not using all of the CPU computing power, you may have a problem on the GPU: your game may be calling the drawing routines too often. The less a game has to communicate with the hardware, the higher the performance. In that case using Sprite Atlases and other "batching techniques" that allow to draw many objects with only one call will help your game perform better.

#### Is your game eating up more and more RAM as it's running?

Your game starts well enough, but after just a few minutes it starts slowing down and becomes choppy. Your may have a memory problem at hand.

If your game supports windowed mode, keep your task manager (or "top"/"htop"/"bpytop" if you're on Linux) open and look at your game's process: does the memory used by your game increase as you're playing it?

If so, you may be having a so-called *memory leak*~[g]~: somewhere during its running cycle, your game forgets to clean up something, which stays resident in memory until your game closes. The result, after creating and deleting a lot of entities and leaving a lot of *garbage* behind is that the total memory used increases.

This is especially common in languages like C++, where there is no automatic "garbage collecting" and having cases of so-called *unreachable memory*[g] can really mess up your memory usage.

::: pitfall :::
Some people call unreachable memory cases "*dangling pointers*~[g]~", but technically they are two different (and opposite) things.

Check the glossary for more information.
:::::::::::::::

If you suspect a memory leak, you may want to take a look at these sections:

- [Entity Cleanup and Memory Leaks](#cleanup_leaks)
- [Using memory analyzers to detect leaks](#mem_analyzers)
- [Resource Pools](#res_pools)

{{placeholder}}

<!-- TODO: Continue with more investigation suggestions -->

Optimizing your game
--------------------

After accurate profiling, you need to intervene and try to get more out of your code. In this section we'll talk about some guidelines and tips on how to optimize your game.

### Working with references vs. returning values

Depending on the programming language you're using, and the amount of internal optimization its compiler/interpreter has, you may have the possibility to choose between two main ways of working, when it comes to functions:

- Returning a value from a function;
- Passing a reference to variables into the function and use that reference in your function (for instance in C++).

"Value Copying" can be a real resource hog when your functions work with heavy data. Every time you return a value, instead of working on a reference, you are creating a new copy of the data you're working on, that will be later assigned.

This can happen also when passing parameters to a function (in this case you say the "parameter is passed by value"): a new copy of the parameter is created locally to the function, using up memory. "Value Copying" can help when you don't want to modify the data outside your function, but is a waste when instead you **want** to modify such values.

Using things like "references", "constant references" and "pointers" can be really precious in making your game leaner memory-wise, as well as saving you all the CPU cycles wasted in memory copying.

### Optimizing Drawing

This heavily depends on the type of framework and engine you are using, but a good rule of thumb is using the lowest amount of calls to the draw routines as possible: drawing something entails a great amount of context switching and algorithms, so you should do it only when necessary.

If your engine/framework supports it, you should use sprite atlases/batches, as well as other interesting structures like Vertex Arrays (used in SFML), which can draw many elements on the screen with only one draw call.

Another way to optimize your drawing routine is avoiding to change textures often: changing textures can result in a lot of context changes (like copying the new texture from the RAM to the GPU memory), so you should use only one oversized texture (in the form of a [Sprite Sheet](#SpriteSheets)) and draw only a part of it, changing the coordinates of the rectangle that gets drawn. This way you'll save the PC a lot of work.

{{placeholder}}

<!-- TODO: Finish talking about optimizing drawing -->

### Reduce the calls to the Engine Routines

Some engines have routines that introduce sanity checks, logic optimizations and more, and calling such routines more than necessary can burden your game's performance, even worse when you're calling them per-frame.

If you want to move a character diagonally both up and right, don't do this:

```{src='optimization/double_move_call' caption='Double Engine Movement Call'}
```

As all the sanity checks in the `Move` function will be executed twice per frame (since we're in the "Update" function). Instead you should get the resulting movement vector first, and then use the `Move` function only once:

```{src='optimization/single_move_call' caption='Single Engine Movement Call'}
```

This way instead we're doing sanity checks and related operations only once, moving the character in its final position without wasting resources.

### Entity Cleanup and Memory leaks {#cleanup_leaks}

One of the biggest scourges in software development (and an even bigger one in game development) are memory leaks: the program allocates memory but doesn't release it properly.

Memory management (as well as any kind of "resource management") can be summarized in 3 phases:

- Acquisition;
- Usage;
- Release.

This is especially annoying when languages that don't have automatic garbage collection (like C++) are involved, but it can affect any programming language. Memory management is hard, and we should always release any resource that we acquire as soon as we're done using it, but that's not always easy: for instance when loading and unloading levels is involved.

As mentioned before, this problem affects all languages, since some resources may be acquired by some "active code" that is actually never running, thus preventing the garbage collector from working as it should.

Besides "being careful" with your resource management, you can check for memory leaks by using specific tools.

### Using analyzers to detect Memory Leaks {#mem_analyzers}

When developing a game, there are a lot of tools that allow you to inspect your game and find possible memory leaks. Some are "static scanners" while other (usually called "dynamic testing tools") require the game to be running.

#### Static Scanners

These tools analyze the code without running it, checking the style and common bugs that can be inserted by mistake. An example of these static tools are "linters" (or linting tools).

Most of these tools are included in IDEs but some (like LLVM's scan-build) are standalone.

![An example screen from LLVM's scan-build](./images/profiling_optimization/scan_build.png){width=50%}

#### Dynamic testing tools

Some tools require the game to be running, some general-purpose ones are used to find memory leaks (like Valgrind), while others have more specific purposes and are usually integrated into the engine.

![A screenshot from Valgrind, looks like we have a memory leak here](./images/profiling_optimization/valgrind.png){width=50%}

These more specific tools can track the FPS, memory as well as the calls done to each function, allowing you to track down what is bogging down your game.

![A screenshot from Godot's profiler](./images/profiling_optimization/godot_profiler.png){width=50%}

### Resource Pools {#res_pools}

{{placeholder}}
<!-- TODO: resource pools of reusable items are great -->

### Lookup Tables

Inside older games, where CPU cycles were at a premium, a widely used trick to gain performance were "lookup tables".

These tables would store the result values for certain expensive functions, given certain inputs, thus replacing the expensive operation with a lookup inside a certain data structure (which is usually really fast).

This has a tradeoff: you're trading CPU time for Memory space, since the lookup tables are meant to stay into RAM.

In modern games instances of lookup tables are as rare as hens' teeth, but it's an interesting historical view over some older forms of optimization.

### Memoization {#memoization}

Memoization (sometimes known as "tabling") is an optimization technique that consists in saving the result of an expensive function, as well as the function's arguments for later calls: this way when the same arguments are passed to the function, you can return the stored value instead of performing the calculation again.

This is due to the fact that functions are deterministic, so if you have the same inputs you will always receive the same outputs: this allows to minimize expensive computations at the expense of memory.

Obviously this technique can't really be applied to functions that make use of pseudo-random numbers and connected functionalities, because memoization would completely void such randomization.

Memoization is usually implemented via decorators that check if the arguments passed are inside a defined data structure (usually a hash table): if there is a hit, the result is returned immediately, if not the original (expensive) function is run and its result is memorized in said structure.

Memoization should be used only on functions that are:

- Expensive
- Called often with the same arguments

If we start using this technique on all functions, we may end up with a software that occupies a lot of memory without any significant speedup.

### Approximations

Many times when developing games we don't need to have a value that is precise to the 10th decimal digit, that's where approximation comes into hand.

A prime example of approximation was used in Quake III Arena, via the algorithm known as "Fast Inverse Square Root". Back in 1999 calculating the inverse square root of a number was an expensive calculation for the CPU, so the developers decided to create an algorithm that would calculate an approximation quickly.

This was done by playing around with the floating point low-level structure and using a "magic constant" (`0x5f3759df`) to create a good "first guess", after that a single iteration of the [Newton-Raphson Method](#newtonmethod) is applied to refine the guess.

This proved to be faster than directly calculating a normalized vector (which uses a square root and a division, expensive at the time) and also faster than using a lookup table. The algorithm proved to be slower (and less precise) than the dedicated SSE instruction in the newer x86 CPUs.

### Eager vs. Lazy Evaluation

Lazy objects are yet another possibility when it comes to optimization, with some drawbacks: you create an object but the calculations related to its state are performed when the object is first used, instead of when it is constructed.

This can be really useful when you have a great quantity of items that you are iterating through, one at a time, but don't need the whole collection at hand at once. When it comes to collections, lazy objects help saving memory at the cost of more CPU cycles while the game is running.

In some languages, this concept is abstracted in a language feature (like "generator expressions" in Python), while in others you'll have to work a little bit harder to get them.

Let's take an example, we have a custom object that contains a reference to a list of numbers: when we iterate through this object, we want it to return the numbers saved, halved.

::: note :::
What follows is just a didactic example, but should be simple enough to understand the difference between "eager" and "lazy" objects.
::::::::::::

#### Eager approach

The eager approach is to take the list of numbers, create a second list inside our object with the numbers halved: this will make sure that the values are always ready and readily available, but will consume more memory. Here's the example:

```{src='optimization/eager_mode' caption='An eager object'}
```

#### Lazy approach

If we know that we are working with millions of values, and we are going through them kind of rarely, saving all the halved values in RAM may not be a good idea. This is where lazy evaluation comes into play: instead of memorizing the value in RAM, we calculate it on-demand. Here's the example:

```{src='optimization/lazy_mode' caption='A lazy object'}
```

Tips and tricks
---------------

### Be mindful of your "updates"

It is a common mistake among new game developers of putting the whole game logic inside the engine's `update()` method: this will eventually bog down the game and create inconsistencies when the framerate varies.

Input should be handled in your engine's event-based input system (very rarely you will need to check the keyboard status inside the `update()` method), also you should absolutely take advantage of your engine's facilities when it comes to managing how the game updates.

For instance, Unity offers 3 update functions:

- `FixedUpdate()`
- `Update()`
- `LateUpdate()`

`FixedUpdate()` is executed with the Physics engine, so here is where you should apply forces, torques and any other physics-related function. Being run with the physics engine, this function may be called zero, one or more times per frame.

`Update()` is your run of the mill update function, it is always executed once per frame, without fail. This is used for other kinds of updates, if you do physics operations here the results may be inconsistent (since it doesn't run in sync with the physics engine). You can still move objects that are not tied to physics.

`LateUpdate()` is a utility function that is run once per frame, after the `Update()` function. This is useful for all kinds of operations that would require the `update()` calculations to be completed.

### Dirty Bit

Not all entities in your game need to have their state updated all the time. Continuously updating all entities' internal state can be really costly in terms of game performance.

A quick way to make your game lighter on resources (and thus more performing) can be putting a boolean check at the beginning of the update function, checking if the object really needs to have its internal state updated.

A possible example could be the following:

```{src='optimization/dirty_bit' caption='Example on how to optimize entities with a dirty bit'}
```

If your code is well-done, you won't have issues like animations freezing, because those will be separated from the "update routine", since the animator will chug along its frames when requested by the `draw` function.

### Far-Away entities (Dirty Rectangles)

Another way to optimize your game performance is not updating entities way off screen: this is also a technique used in the game Minecraft, where entities are frozen when you are far away from them, to save on resources.

A possible idea would be having an "updatable rectangle" (sometimes called "Dirty Rectangle"), bigger than the screen, and only the entities inside such rectangle will be updated.

This could create some issues when it comes to games that have their challenge deriving from entities updating in sync with each other, thus if we implement this "updatable rectangle" one or more entities would fall "out of sync", possibly making beating the level impossible.

In that case we may just put out an exception (where certain entities are updated no matter what) or divide our level into smaller "rooms" that are instead entirely updated all the time.

### Tweening is better than animating

Animators and animation frames are performance-hungry and should absolutely not be used in all those situations where you can instead use inbetweening techniques.

This means that frame-by-frame animations should not be used when taking care of moving UI parts: if you want to slide a piece of UI (take for instance a drop-down terminal from a "computer hacking" game) from the top, you can just tween its position and save a lot of memory.

Remember that Tweening doesn't apply only to positions, you can tween any property of a game object.

So a quick way you can optimize your game, is removing all the unnecessary animations and replace them with tweening, your game will surely benefit from that.

### Remove dead code

There are many definitions for "dead code", some use the "unreachable code" definition (for instance code placed after a "return statement") some use a more extensive definition.

I like to think of dead code as "wasted code", which is:

- Anything that happens to be written after a "return statement" in a function: return statements are used to give control of the program back to the caller of a function, so this code will never be executed;
- Unused variables: variables are allocated in memory, require calculations and CPU cycles, if not used that's just a waste;
- Unused code: complete functions that are never called are a waste of memory (because they may be loaded in RAM) and of disk space, making the executables bigger;
- Debug code: sometimes we need to write code to debug other code, this code may end up being part of a "release version" and weigh it down.

You should be careful when optimizing out dead code, even more when you are dealing with functions which result is not used: those functions may change some global state (or change stuff by usage of *side effects*~[g]~).

Non-Optimizations
-----------------

In this small section we take a look at some alleged "optimizations" that actually do nothing (or close to nothing) for our game's performance.

### "Switches are faster than IFs"

Some people allege that using "switch" statements instead of "if" statements is bound to optimize the game. This an overstatement, and we can prove it with a simple test.

Let's create two C++ listings, like follows:

```{.cpp caption="IFs vs Switch - IF Statements"}
#include <iostream>
using namespace std;
int main(){
  for (int i = 0; i < 10000000; i++){
    int x = rand() % 5;
    if (x==1){
      cout << "One" << endl;
    }else if (x==2){
      cout << "Two" << endl;
    }else if(x==3){
      cout << "Three" << endl;
    }else if(x==4){
      cout << "Four" << endl;
    }else if(x==5){
      cout << "Five" << endl;
    }
  }
  return 0;
}
```

```{.cpp caption="IFs vs Switch - Switch Statements"}
#include <iostream>
using namespace std;
int main(){
  for (int i = 0; i < 10000000; i++){
    int x = rand() % 5;
    switch(x){
      case 1:
        cout << "One" << endl;
        break;
      case 2:
        cout << "Two" << endl;
        break;
      case 3:
        cout << "Three" << endl;
        break;
      case 4:
        cout << "Four" << endl;
        break;
      case 5:
        cout << "Five" << endl;
        break;
    }
  }
  return 0;
}
```

These pieces of code will be compiled without any optimization, using G++, using the following command:

```{.sh}
g++ -Wall -Wextra -Werror -O0 filename.cpp -o filename.bin
```

Where "filename" is replaced by the source name, then each file will be executed using the "time" linux command, like follows:

```{.sh}
time ./filename.bin
```

Below we can see the results for both the codes:

![Time taken by the IF code](./images/profiling_optimization/if_time.png){width=60%}

![Time taken by the Switch code](./images/profiling_optimization/switch_time.png){width=60%}

We can see a difference of just around 0.25 seconds, over 10 Million iterations. If you changed an equivalent IF statement for a Switch statement, you would earn a quarter of a second every 46 hours of gameplay at 60fps.

The right choice is the simply choose the structure that lets you have the most readable code: the more your code is readable, the easier it is to understand; the easier to understand, the lower the probability that there is a bug in there (or a performance hog of some sort).

### Blindly Applying Optimizations

There rarely is something more wrong you can do when optimizing than blindly applying optimizations without considering your application context.

Using resource pooling in an environment with limited memory (but plenty of CPU power) can prove a disaster: it's better to instantiate and destroy objects in such case.

Sometimes animators can be faster than LERPing/Tweening, mostly when you have to tween objects with multiple children: tweening would create a lot of CPU-bound calculations for the new position and size that will make the whole thing bog down and get choppy.

The only thing you can do is think first and try later: this book can give you some suggestions, but nothing should be taken at face value. Remember the context your game is working in and **do not treat all platforms like they're the same**: WebGL is different than Console which is different than Mobile.
