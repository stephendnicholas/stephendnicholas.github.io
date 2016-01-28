---
layout: post
permalink: /a-simple-ssl-enabled-comet-server-in-node
alias: [/archives/1131]

title: A simple SSL enabled Comet style Pub/Sub server with Node.js
author: Stephen Nicholas
excerpt: 'My first attempt at programming with Node.js: a simple SSL enabled Comet style pub/sub server, in just over 100 lines of code.'
---

I've been meaning to have a play with Node.js for a while, but up until now I haven't had a good reason. Running through a 'Hello World' example is all well and good, but I'm the kind of person who actually needs to do something when learning a new technology; otherwise I'll just pick it up for an afternoon and then put it down and forget about it. 

So I decided to try writing a simple SSL enabled Comet style pub/sub server (for reasons that may become apparent in future posts) and it turned out to be really easy; taking just over 100 lines of code (including comments and liberal spacing).

The complete code's at the bottom of this post, so if that's what your after, feel free to skip ahead.

Otherwise, I'm just going to quickly run through some of the steps it took me to get there:

### 1. Learning the basics of Node.js

For this I turned to the <a href="http://www.nodebeginner.org/" target="_blank">The Node Beginner Book</a>, which I'd really recommend. It's a nice advanced introduction for developers familiar with at least one object-orientated programming language and completely new to Node.js. It skips over the simple stuff, like data types and basic programming flows, and takes you through developing '*a complete web application which allows the users of this application to view web pages and upload files*'; introducing a lot of the important Node.js concepts along the way.

### 2. Figuring out how to implement SSL with Node.js

With a basic understanding of Node.js under my belt, I then decided to approach what I thought would be the most tricky part: adding SSL to the server. I've had issues with this before in different languages and so was expecting some pain; however it was pretty simple.

Here's an quick example of how to create a SSL enabled 'hello world' server, taken from the <a href="http://nodejs.org/docs/v0.3.7/api/https.html" target="_blank">Node.js documentation for https</a>:

{% highlight js %}
//Include the https & file system modules  
var https = require('https');  
var fs = require('fs');

//Create the server options object, specifying the SSL key & cert  
var options = {  
  key: fs.readFileSync('privatekey.pem'),  
  cert: fs.readFileSync('certificate.pem')  
};

//Create the HTTPS enabled server - listening on port 8000  
https.createServer(options, function (req, res) {  
  res.writeHead(200);  
  res.end("hello world\n");  
}).listen(8000);  
{% endhighlight %}

It turned out the most tricky part was generating the certificates and this was still pretty easy. All I needed to do was install <a href="http://www.openssl.org/" target="_blank">openssl</a> and then issue the following command line commands:

{% highlight console %}
openssl genrsa -out privatekey.pem 1024
openssl req -new -key privatekey.pem -out certrequest.csr
openssl x509 -req -in certrequest.csr -signkey privatekey.pem -out certificate.pem
{% endhighlight %}

One slight gotcha was that, as a Windows user, the second command (openssl req&#8230;) kept failing for me with the error message: <b>"Unable to load config info from /usr/local/ssl/openssl.cnf"</b>. A <a href="http://irwinj.blogspot.com/2008/11/unable-to-load-config-info-from.html" target="_blank">quick tip</a> pointed out that all you need to do to solve this is specify the config path yourself, by using -config. E.g.

{% highlight console %}
openssl req -new -key privatekey.pem -out certrequest.csr -config C:\OpenSSL-Win64\bin\openssl.cfg
{% endhighlight %}

### 3. Figuring out how to do Comet with Node.js

Again it turns out this is really simple. Simple (at least for the moment) seems to be a word I'm associating a lot with Node.js. To implement a long polling Comet approach, the two key elements appear to be:

  1. Set it so that the incoming connection doesn't expire - by calling <code>req.connection.setTimeout(0);</code>
  2. Store the response objects for the incoming connections somewhere and then use them to send the data later on.

Here's a relatively simple example (based on one I found <a href="http://zenmachine.wordpress.com/2010/01/31/node-js-and-comet/" target="_blank">here</a>) that causes each request to '/comet' to wait up to 5 seconds before it gets a reply:

{% highlight js %}
var sys = require("sys");  
var http = require("http");

//Array of responses awaiting replies  
var waitingResponses=[];

//Function that will send a message to each waiting response  
function sendToWaitingResponses() {

  //If there are some waiting responses  
  if (waitingResponses.length) {

    //For each one - respond with 'Hello World - <current timestamp>'  
    for (var i = 0; i < waitingResponses.length; i++) {
      var res = waitingResponses[i]; 
      res.writeHead(200, {'Content-type':'text/plain'}); 
      res.write('Hello World - ' + (new Date().getTime())); 
      res.end();
    }
  } 
    
  //Schedule this method to be called again in 5 seconds 
  setTimeout(sendToWaitingResponses, 5000);
} 
    
//Schedule the first call of sendToWatitingResponses() for 5 seconds from now 
setTimeout(sendToWaitingResponses, 5000); 
    
//Start the server - listening on port 8000 
http.createServer(function(req, res) { 
  
  //Only handle requests to /comet 
  if (req.url == '/comet') { 
    //Set it so the connection doesn't time out 
    req.connection.setTimeout(0); 
    
    //Add the response object to the array of waiting responses 
    //To be replied to at some point by the sendToWaitingResponses() method 
    waitingResponses.push(res);
  } else {
    res.writeHead(404, {'Content-type':'text/plain'}); 
    res.write('not found'); 
    res.end();
  }
}).listen(8000);
{% endhighlight %} 

Here I've used a repeating method (called every 5 seconds using <code>setTimeout(...)</code>) to trigger sending the data; however in a real implementation this would be triggered by some kind of event (e.g. a publish being received).

### 4. Putting it all together

The final step was pulling all this together to create my pub/sub server; allowing applications to:

  * **Subscribe** - by sending a GET request to <code>'/s?t=&lt;topic&gt;'</code>. The connection will then remain open until a publish for that topic comes in, at which point the message is sent and the connection closed.
  * **Publish** - by sending a POST request to <code>'/p?t=&lt;topic&gt;'</code> with the content of the message as the body of the post, encoded in utf-8.

Here's the full code, including copious comments, so hopefully you can understand what I've done:

{% highlight js %}
var https = require('https');  
var fs = require('fs');  
var url = require('url');

//Really undefined  
var undefined;

//Setup server options with SSL key & cert  
var options = {  
  key: fs.readFileSync('privatekey.pem'),  
  cert: fs.readFileSync('certificate.pem')  
};

//Multidimensional array of  
//[topic][subscription response objects waiting for data]  
//I.e. each topic has an array of response objects waiting for the next publish  
var subscriptions=[];

//Create the server - listenting on port 8000  
https.createServer(options, function (req, res) {

  console.log('Incoming request for:', req.url);

  //Parse out the pathname  
  var pathname = url.parse(req.url).pathname;

  //Parse out the topic query parameter  
  //Note: true tells parse(...) to parse the query string for us too (neat)  
  var topic = url.parse(req.url, true).query["t"];

  //If subscribe request  
  if (pathname == '/s') { 

    //If topic not specified - then error  
    if(topic === undefined) {  
      console.log(' Error: Topic (t=...) not specified.');  
      res.writeHead(400, {'Content-type':'text/plain'});  
      res.end();  
    } else {  
      console.log(' Topic:', topic);

      //Set timeout to 0 - so the request doesn't timeout - making it comet-y  
      req.connection.setTimeout(0);

      //Initialise the topic specific subscription array if it doesn't already exist  
      if(subscriptions[topic] === undefined) {  
        subscriptions[topic] = [];  
      }

      //Add the response object to array of 'subscriptions' to be served for that topic  
      subscriptions[topic].push(res);  
    }  
  }  
  //Else if publish request  
  else if (pathname == '/p') {

    //If topic not specified - then error  
    if(topic === undefined) {  
      console.log(' Error: Topic (t=...) not specified.');  
      res.writeHead(400, {'Content-type':'text/plain'});  
      res.end();  
    } else {  
      console.log(' Topic:', topic);

      //Parse in all the post data - the published 'message'  
      //Each chunk of post data is simply appended to the var postData  
      var postData = "";  
      req.setEncoding("utf8");  
      req.addListener("data", function(postDataChunk) {  
        postData += postDataChunk;  
      });

      //Once we've got all the post data  
      req.addListener("end", function() {  
        console.log(' All post data received. Post data:', postData);

        //For each subscription FOR THIS TOPIC - send them the post data and close with 200 response  
        if (subscriptions[topic] !== undefined && subscriptions[topic].length) {  
          for (var i = 0; i < subscriptions[topic].length; i++) {
            var s = subscriptions[topic][i]; 
            s.writeHead(200, {'Content-type':'text/plain'}); 
            s.write(postData); 
            s.end(); 
          } 
          
          //Clear all 'subscriptions' for the topic 
          subscriptions[topic] = []; 
        }
        
        //Send response to the publish request 
        res.writeHead(200, {'Content-type':'text/plain'}); 
        res.end(); 
      }); 
    }
  } 
  //Else not supported path 
  else { 
    console.log(' Not Found'); 
    res.writeHead(404, {'Content-type':'text/plain'});
    res.end();
  }
    
  console.log('Finished request.'); 

}).listen(8000);
{% endhighlight %} 

Now, this is my first piece of Node programming, so please be gentle; however I don't think I've done anything too stupid. First of all, it works and does what I want. Secondly, I think it should be relatively scalable (though I've not tested this). And thirdly, due to the single threaded nature of Node.js (Yay!) there <i>shouldn't</i> be any real concurrency issues.

The one thing I am missing, is some kind of logic to purge dead connections, but I guess you can't have everything :)
