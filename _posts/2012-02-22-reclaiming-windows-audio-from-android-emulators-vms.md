---
layout: post
permalink: /reclaiming-windows-audio-from-android-emulators
alias: [/archives/939]

title: 'Reclaiming Windows Audio from Android Emulators &#038; VMs'
author: Stephen Nicholas
excerpt: "Recently, every time I run the Android Emulator, I lose all sound output on my laptop (running Windows 7) and have to restart to get it back. It's very annoying. But luckily the solution is fantastically simple..."
---
Recently, every time I run the Android Emulator, I lose all sound output on my laptop (running Windows 7) and have to restart to get it back. It&#8217;s very annoying. Particularly as I like to have music when coding.

I&#8217;ve been putting up with this for a while, assuming it was some complicated driver issue, but a chance Twitter encounter today led me to a colleague who&#8217;s been having the same kind of problem (although his was caused mainly by running Virtual Machines).

He&#8217;d managed to figure out that it was just the Windows Audio service that was messed up and all you needed to do was to restart it to get sound back. Simple!

He was even kind enough to share a simple bat script that would do it for me (Run as Administrator):

{% highlight bat %}  
@echo off  
ECHO =====================================================  
ECHO Restarting Windows Audio  
ECHO =====================================================  
net stop "Windows Audio";  
net start "Windows Audio";  
{% endhighlight %}
