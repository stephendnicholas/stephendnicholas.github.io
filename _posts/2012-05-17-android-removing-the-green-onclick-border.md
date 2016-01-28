---
layout: post
permalink: /android-removing-the-green-onclick-border
alias: [/archives/1428]

title: 'Android: Removing the green onclick border'
author: Stephen Nicholas
excerpt: For a while now, I've been finding the green highlight / box that the Android <a href="http://developer.android.com/reference/android/webkit/WebView.html" target="_blank"><code>WebView</code></a> displays when you click something particularly annoying. Here's how to get rid of it.

dsq_thread_id:
  - 4458280814
---
For a while now, I&#8217;ve been finding the green highlight / box that the Android <a href="http://developer.android.com/reference/android/webkit/WebView.html" target="_blank"><code>WebView</code></a> displays when you click something particularly annoying.

<img src="{{ site.baseurl }}/assets/img/annoying_green_highlight_thumb.png" style="width:50%" />

I understand why it&#8217;s there and UI feedback is definitely an important part of mobile design, but a lot of the time I like to take care of that myself and I find the green boxes can really spoil the user experience. Particularly when using buttons & form elements and anything involving Canvas.

It&#8217;s been a mild thorn in my side for a while, but today I finally decided to look into it and it turns out the solution is remarkably simple. 

First of all, all credit goes to <a href="http://stackoverflow.com/questions/2728566/android-browser-green-border-on-click" target="_blank">Stack Overflow</a>. I&#8217;m not trying to steal credit, please <a href="http://stackoverflow.com/questions/2728566/android-browser-green-border-on-click" target="_blank">go there</a> and vote the solution up (I already have); but did it take some Google-Fu to find it and so I&#8217;m hoping by posting it here I might make it easier to find.

Anyway, onto the solution, which is just to add the following to your pages stylesheet:

{% highlight css %}
`* {-webkit-tap-highlight-color: rgba(0, 0, 0, 0);}`
{% endhighlight %}

For example, so that:

{% highlight html %}
<!DOCTYPE html>  
<html>  
  <head>  
  </head>  
  <body>  
    <a href="" style="font-size:4em;">Annoying!</a>  
  </body>  
</html>  
{% endhighlight %}

Becomes:

{% highlight html %}
<!DOCTYPE html>  
<html>  
  <head>  
    <style type="text/css">  
      * {-webkit-tap-highlight-color: rgba(0, 0, 0, 0);}  
    </style>  
  </head>  
  <body>  
    <a href="" style="font-size:4em;">Annoying!</a>  
  </body>  
</html>  
{% endhighlight %}

Fantastically simple. And that stops the green boxes dead.

It does seem to be on a page-by-page / per stylesheet basis though and also it is all or nothing (you can&#8217;t override the selection colour by changing the rgba, you can only seem to disable it as above), but that&#8217;s all I want, so I&#8217;m happy :)
