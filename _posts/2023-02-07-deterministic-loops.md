---
title: Deterministic and reliable execution of loop functions in C/C++
author: mp1994
date: 2023-02-07
category: linux
tags: linux, embedded, loop function, timer, deadline timer, engineering
layout: post
published: true
---

This post illustrates how to execute in a reliable and *deterministic* way a function at a (fixed) frequency, i.e. after a user-defined time interval that we'll call *cycle time*. In the following, we will implement a basic loop function with timer re-arming and self-benchmarking features, and we will wrap it to *hide* this functionalities to the end user, allowing them to only change the user code that will be executed in the loop. For variable cycle durations, or to implement an adaptive frequency (e.g., switch between *idle* state and *active* state), we could also expose to the user timer re-arming. As always, the implementation and the code shown below are tested on Linux only and are not guaranteed to work under any other OS.

## SIGALRM 
Under Linux, POSIX provides the `SIGALRM` signal that fires when a timer expires. We can exploit the `alarm(int seconds)` function to set the `SIGALRM` after a specified time interval in seconds. This can execute any function we previously assigned to this signal.

{% include codeHeader.html %}
{% highlight cpp %}
// Loop function
void loop_function(int s) {
    
}

// Link our callback to the SIGALRM signal
signal(SIGALRM, loop_function);
// Set the timer to expire in 2 seconds
alarm(2);
{% endhighlight %}

The main limitation of this approach is that `alarm()` takes an integer (`unsigned int`) and thus we cannot achieve sub-second precision. One could use `setitimer()` instead, but we are not going to do that. Instead, we are going to take advantage of the [Boost](https://www.boost.org/) library.

## Boost ASIO

Asynchronous input/output (ASIO) allows concurrent, non-blocking I/O operations to run without affecting the main program. In my (limited) experience, [Boost.Asio](https://www.boost.org/doc/libs/1_65_1/doc/html/boost_asio/overview/rationale.html) is the way to go for C++ applications that require asynchronous I/O. The [documentation](https://www.boost.org/doc/libs/1_65_1/doc/html/boost_asio.html) also contains useful tutorials and examples. Please mind that I have been posting links to Boost version `1.65.1` since it is the version installed by `apt` on Ubuntu 18.04 LTS.

### deadline_timer

The `basic_deadline_timer` class template provides the ability to perform a blocking or asynchronous wait for a timer to expire. The `wait()` method can be used to wait for the timer to expire while blocking the execution of the current thread, while we can use `async_wait()` to avoid blocking thread's execution while waiting for the deadline timer to expire. 

To create a deadline timer, we need few lines of code:

{% include codeHeader.html %}
{% highlight cpp %}
// IO service that handles the deadline_timer
boost::asio::io_service timer_io;

// Create the timer object
boost::asio::deadline_timer t(timer_io);
// Set the expire time relative to the current time
t.expires_from_now(boost::posix_time::seconds(5));

// Non-blocking wait
t.async_wait(&loop_function);

// Start the timer
// The IO service run() is blocking, so the main thread
// will be blocked by this function call.
timer_io.run();
{% endhighlight %}

You can find the full code of the example above and run it [here](https://wandbox.org/permlink/AJPcQAiT5pOdZhWs). 
There are some things that are worth mentioning. Every `deadline_timer` needs an `io_service` to run. One or more timers can be assigned to the same IO service when they are created. Although the call to the method `async_wait()` is non-blocking, if we are going to run the `run()` method of the IO service, the main thread will be blocked. This means that we need to wrap the call to the IO service in another thread to fully exploit the deadline timer to generate a non-blocking loop.

The example code above sets a deadline timer to fire after 5 seconds and calls the handler function `loop_function` after waiting. Hence, the timer only fires once, `timer_io.run()` returns and the program exits. To effectively run a loop, we need to re-arm the timer, and we can do this directly inside the `loop_function`.

{% include codeHeader.html %}
{% highlight cpp %}
void loop_function(const boost::system::error_code& /*e*/) {

    // Re-arm the timer
    t.expires_from_now(boost::posix_time::seconds(5));
    t.async_wait(&loop_function);

    /** User code **/
    std::cout << "deadline_timer expired\n";
    /** User code **/

}
{% endhighlight %}

In this way, the callback first resets the timer, and then it actually executes the user code. In this example, the timer object `t` must be a global variable, or we must use a global (shared) pointer. We may want to *hide* the code that handles the timer itself to the user and only provide a function that gets called at every cycle. If this is the case, we can wrap a user-defined loop function into the *actual* callback that gets called by the deadline timer.

{% include codeHeader.html %}
{% highlight cpp %}
static void w_loop_function(const boost::system::error_code& /*e*/) {

    // Re-arm the timer
    t.expires_from_now(boost::posix_time::seconds(5));
    t.async_wait(&w_loop_function);

    // Call the user-defined loop function
    loop_function();

    // Update the loop counter
    loop_counter++;

}

void loop_function(void) {

    /** User code **/
    std::cout << "deadline_timer expired\n";
    /** User code **/    

}
{% endhighlight %}

In this way, the `w_loop_function` can be hidden to the final user of our API, exposing only a template `loop_function` that can be defined as required by the application. In the case we may want to achieve a variable loop frequency, we could have a global variable to be used as the argument of the call `t.expires_from_now()`, or -- even better -- we could put everything into a class.

[This example](https://wandbox.org/permlink/J9SdvVP2DJVwUbks) shows the implementation of a custom deadline timer class with a wrapped loop function to handle timer re-arming and benchmarking the effective cycle time duration. Moreover, it sets real-time priority (`sched_priority = 99`) with FIFO scheduling to the dedicated thread that runs the IO service.

### A note on Boost and portability

I am a fan of the Boost library, which I consider one of the best open-source projects. Yet, preparing the example code for this post I was a bit disappointed. I was familiar with [wandbox](), an online C++ compiler that lets you play around with the code, build it and run it online. It is great for sharing code with people and for quick demos, as I intended to do here. Wandbox lets you choose the compiler, and I chose `gcc 7.5.0`, the same I have installed on my machine. I could not choose the version of Boost, which is `1.75.0`. Linking `boost_thread` and `boost_system` works fine and the first example code run as smoothly as on my laptop (the output is buffered so it won't update during the execution of the program...).

Things went differently when I copy-pasted the code of the second example. Turns out `boost::posix_time::milliseconds()` cannot take a non-integer argument, and so my code could not compile on Wandbox. So basically, I need to convert to integer the cycle time `dt` before converting it to milliseconds, effectively limiting the frequency of the loops to 1000 Hz. Fortunately, I could change my code and use instead `boost::posix_time::microseconds()`, that limits the maximum loop frequency to 1 MHz. Although modern CPUs clock faster than 1 GHz, on my laptop I could not achieve a rate higher than 10 kHz with reliable cycle times. 

I further investigated what I believed was a portability issue. Newer versions of Boost, like the `1.75.0`, provide mostly header-only libraries that do not require compilation, which is great for a service like Wandbox as it reduces the computation cost of the build process. 
It turns out that the difference between the two versions of Boost was indeed a bugfix that also fixed my code: `boost::posix_time::milliseconds()` (as well as `microseconds()` and `seconds()`) *could* take a non-integer argument, but then it could not convert it to the proper time, thus generating the wrong loop rate.