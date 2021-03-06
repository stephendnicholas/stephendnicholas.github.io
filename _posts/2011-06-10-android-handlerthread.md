---
layout: post
permalink: /posts/android-handlerthread
redirect_from: /archives/42/

title: How to use the Android HandlerThread
author: Stephen Nicholas
excerpt: Today I'm going to talk about something I ran into recently that turned out to be very simple, despite initially seeming a little confusing, the Android <a href="http://developer.android.com/reference/android/os/HandlerThread.html"><code>HandlerThread</code></a> class.

dsq_thread_id:
  - 4458084548
---
Today I'm going to talk about something I ran into recently that turned out to be very simple, despite initially seeming a little confusing, the Android [`HandlerThread`][1] class.

So, first of all, why did I run into this? Well, basically I've been doing some work to improve the multi-threading of one of my applications, by splitting off some of the background work into a number of different threads; and to allow those threads to be easily interacted with, I wanted each of them to have their own [`Handler`][2]. 

Now I'd read how to turn a thread into a [`Handler`][2] thread by calling `Looper.prepare()`, setting up my [`Handler`][2] & then calling `Looper.loop()`; however this left my thread unable to perform its own background activities and so I ended up creating sub-threads / runnables and things started getting rather messy.

Assuming there must be a better way to do this, I eventually came across [`HandlerThread`][1], which is described in the Android documentation as:

> Handy class for starting a new thread that has a looper. The looper can then be used to create handler classes. Note that start() must still be called. 

Now, in retrospect that actually seems quite clear- so I'm hoping that by writing this post, I'm not just exposing my stupidity- but initially I was a little unsure what it meant and how to use it.

Anyway, long story short, after some searching and some experimentation, I came across the correct way to use it. Which is, unsurprisingly, very similar to how it's described above:

First of all you create and start a [`HandlerThread`][1], which creates a thread with a [`Looper`][3]:  
{% highlight java %}
//Create and start the handler thread & give it a custom name  
HandlerThread handlerThread = new HandlerThread("MyHandlerThread");  
handlerThread.start();  
{% endhighlight %}
You then get the [`Looper`][3] from the [`HandlerThread`][1] as follows:  
{% highlight java %}  
//Get the looper from the handlerThread  
Looper looper = handlerThread.getLooper();  
{% endhighlight %}  
Note that the above method call can return null, in the unlikely event that the [`HandlerThread`][1] not been started or if for any reason `isAlive()` returns false. Practically I think this is unlikely to happen and I'm not sure if there's anything sensible to do if it does, so I wouldn't worry about it too much.

You can then create your [`Handler`][2], passing in the [`Looper`][3] for it to use  
{% highlight java %}  
//Create a new handler- passing in the looper for it to use  
Handler handler = new Handler(looper);  
{% endhighlight %}  
And that's all there is to it. You've now you got your own [`Handler`][2] running in it's own thread. Simple, right?

And finally, when your thread is finished, you can shut down the [`HandlerThread`][1] as follows:  
{% highlight java %}  
//Shut down the HandlerThread  
handlerThread.quit();  
{% endhighlight %}  
Just for completeness, below is an example of a complete class that extends thread and uses a [`HandlerThread`][1] to create a [`Handler`][2] for inter-thread communication. As a side note, it also demonstrates the use of the [`Handler.Callback`][4] interface to save you having to subclass [`Handler`][2] for performing your custom message handling.  
{% highlight java %}  
package com.stephendnicholas;

import android.os.Handler;  
import android.os.Handler.Callback;  
import android.os.HandlerThread;  
import android.os.Looper;  
import android.os.Message;

public class ExampleThread extends Thread implements Callback {

  public static final int CUSTOM_MESSAGE = 1;

  private boolean running = true;

  private final Object mutex = new Object();

  public ExampleThread() {  
  }

  @Override  
  public void run() {  
    // Create and start the HandlerThread- it requires a custom name  
    HandlerThread handlerThread = new HandlerThread("MyHandlerThread");  
    handlerThread.start();

    // Get the looper from the handlerThread  
    // Note: this may return null  
    Looper looper = handlerThread.getLooper();

    // Create a new handler- passing in the looper to use and this class as  
    // the message handler  
    Handler handler = new Handler(looper, this);

    // While this thread is running  
    while (running) {

      // TODO- custom thread logic

      // Wait on mutex  
      synchronized (mutex) {  
        try {  
          mutex.wait();  
        } catch (InterruptedException e) {  
          // Don't care  
        }  
      }  
    }

    // Tell the handler thread to quit  
    handlerThread.quit();  
  }

  public void shutdown() {  
    // Set running to false  
    running = false;

    // Wake anyone waiting on mutex  
    synchronized (mutex) {  
      mutex.notifyAll();  
    }  
  }

  @Override  
  public boolean handleMessage(Message msg) {

    switch (msg.what) {  
      case CUSTOM_MESSAGE:  
      // TODO- custom logic  
      break;

      default:  
      // Return false- as we have not handled the message  
      return false;  
    }

    // Return true- as we have handled the message  
    return true;  
  }

}  
{% endhighlight %}

 [1]: http://developer.android.com/reference/android/os/HandlerThread.html
 [2]: http://developer.android.com/reference/android/os/Handler.html
 [3]: http://developer.android.com/reference/android/os/Looper.html
 [4]: http://developer.android.com/reference/android/os/Handler.Callback.html
