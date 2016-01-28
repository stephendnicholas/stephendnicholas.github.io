---
layout: post
permalink: /eclipse-quick-line-count
alias: [/archives/109]

title: Eclipse quick line count
author: Stephen Nicholas
excerpt: Today I needed to find out how many lines of code were in one of my projects, so I loaded up Eclipse, loaded the project and then... was stumped.

---
Today I needed to find out how many lines of code were in one of my projects, so I loaded up Eclipse, loaded the project and then&#8230; was stumped. I couldn&#8217;t think what to do next. Surely, I thought, there must be a simple menu option for that, but no I couldn&#8217;t find anything and it appears that Eclipse doesn&#8217;t have this functionality built-in. Colour me boggled.

Anyway, a quick search pointed me to a number of plug-ins I could install, but also to a really simple &#8216;hack&#8217; (which I found  [here][1]). I thought it was so nice, I&#8217;d share it here in its entirety for everyone to see.

The steps are as follows:

  1. Open the Eclipse search dialog. &#8216;Search -> File&#8230;&#8217; from the main menu.
  2. Set the &#8216;Containing text:&#8217; to &#8216;\n&#8217;.
  3. Check the &#8216;Regular expression&#8217; checkbox.
  4. Set the &#8216;File name patterns:&#8217; to &#8216;*.java&#8217;.
  5. Select a scope that is appropriate. The whole workspace, the current project or a working set.

<img title="Eclipse search dialog" src="{{ site.baseurl }}/assets/img/search.png" alt="Eclipse search dialog"  />

Then when you run the search, it will come back with a search result for every line of code in your application. You can see the total number of matches at the top, and it&#8217;s also broken down by class; exactly what I wanted. Here&#8217;s some example results from another project:

<img title="Eclipse search results" src="{{ site.baseurl }}/assets/img/search_results.png" alt="Eclipse search results" style="width:75%"  />

Now, there does appear to be one issue with this approach, as it can sometimes not count the final line in a class (if it doesn&#8217;t have a line break after it), but in my case that only left me about 30 lines out and that was good enough for me. Cool eh?

**Additional**

Here&#8217;s a couple of other regexs that I found useful for reducing the above results to just lines of code:

`//.*(\n{1})?` &#8211; search for comment lines.

`^\s*\n` &#8211; search for blank lines.

 [1]: http://www.binaryfrost.com/index.php?/archives/207-Easy-way-to-count-Lines-of-Code-in-Eclipse.html
