---
title: 'Power Profiling: HTTPS Long Polling vs. MQTT with SSL, on Android'
author: Stephen Nicholas
excerpt: "A follow on from my previous post: 'Power profiling: MQTT on Android'. This time I decided to look at the power usage of SSL enabled MQTT in comparison to HTTPS Long Polling."
layout: post
permalink: /archives/1217
dsq_thread_id:
  - 4458084740
categories:
  - Android
  - Tech
tags:
  - android
  - comet
  - https
  - mqtt
  - power
  - profiling
---
# Introduction

<div class="linebreak">
</div>

A little while ago I performed some <a href="http://stephendnicholas.com/archives/219" target="_blank">power profiling of MQTT on Android</a> to try and put a figure on just how efficient this technology is on mobile devices. I think I got some pretty good results (enough to give an idea anyway), but the next question people always ask is: &#8216;So how does that compare to the alternatives?&#8217;.  
<a href="http://www.flickr.com/photos/21883298@N08/6843978633/" title="Apples Vs Oranges by Craig in TO, on Flickr" target="_blank"><img src="http://farm8.staticflickr.com/7208/6843978633_4c33e7ba6f_m.jpg" width="240" height="174" alt="Apples Vs Oranges" class="alignright" /></a>  
Well, first of all, it is really Apples vs. Oranges :D

<a name="applesvsoranges"></a><a name="whymqtt"></a>Of my three main choices for Android mobile push notifications: MQTT, HTTP or C2DM, each is designed for a different purpose; with different features, bad points and good points. I&#8217;m not going to do a full comparison here, but to summarise:

<ul style="padding-bottom:0em">
  <li style="margin-bottom:1em">
    <b>MQTT</b> &#8211; designed to provide low latency, assured messaging over fragile networks and efficient distribution to one or many receivers. Protocol focuses on minimising the amount of bytes flowing over the wire and low power usage. Maximum message size of 256MB, but not really designed for sending large amounts of data; better at a high volume of low size messages. Provides a two-way communication channel.
  </li>
  <li style="margin-bottom:1em">
    <b>HTTP</b> &#8211; designed as a request-response protocol for client-server computing. Best known as the foundation of data communication for the World Wide Web (well, duh). It can be &#8216;abused&#8217; to provide push like capabilities, for example by using Comet style approaches, but it really isn&#8217;t designed with this in mind. Provides two-way communication capabilities.
  </li>
  <li>
    <b>C2DM</b> &#8211; designed as a one-way &#8216;shoulder-tap&#8217; notification system. On the plus side, Google takes care of getting the messages to the device and to your application; waking it if necessary. However there are a number of restrictions. The main one is, as the <a href="https://developers.google.com/android/c2dm/" target="_blank">Google Docs</a> themselves say, that <i>&#8216;C2DM makes no guarantees about delivery or the order of messages&#8217;</i>. Therefore, the recommendation is that a C2DM message should never contain the data itself but, rather, be used to provide a notification that there is data available; and applications should then contact their own server to get the data. Other restrictions include: <ul style="margin-top:1em;">
      <li>
        Only available on devices running > Android 2.2.
      </li>
      <li>
        Requires a registered Google account on the device.
      </li>
      <li>
        Message size is limited to 1024 bytes.
      </li>
      <li>
        Google limits the number of messages a sender sends in aggregate.
      </li>
      <li>
        And the number of messages a sender sends to a specific device.
      </li>
    </ul>
  </li>
</ul>

However, fruit rivalry aside, it&#8217;s still an interesting question and my points above don&#8217;t tend to stop people asking, so I thought I&#8217;d try a comparison against the most equivalent & open approach, in my opinion, (and also the easiest one for me to test): HTTP. Also, what with security on mobile being such a big issue, I decided I&#8217;d test the secure versions: HTTPS and SSL enabled MQTT.

### What was I testing?

On the MQTT side, it was very similar to what I tested <a href="http://stephendnicholas.com/archives/219" target="_blank">previously</a>: a simple application using a custom wrapper around the standard Java MQTT client offered by IBM; but this time using an SSL connection against a SSL enabled instance of <a href="http://www-10.lotus.com/ldd/lewiki.nsf/dx/12022008074527PMBJA2VV.htm" target="_blank">Micro Broker</a> and performing mutual certificate based authentication between the client and server.

For the HTTPS side, I decided to use Comet style long polling. If you&#8217;re not familiar with this, the client-server interaction looks something like this:

[<img src="http://stephendnicholas.com/wp-content/uploads/2012/05/longpolling.png" alt="" title="longpolling" width="600" height="284" class="aligncenter size-full wp-image-1218" srcset="http://stephendnicholas.com/wp-content/uploads/2012/05/longpolling-300x142.png 300w, http://stephendnicholas.com/wp-content/uploads/2012/05/longpolling.png 600w" sizes="(max-width: 600px) 100vw, 600px" />][1]

Basically, the client makes a HTTPS request to the server, which is kept open until the server has new data to send to the client. When this data arrives, the server sends it to the client and closes the request. The client then initiates a new long polling request in order to obtain subsequent events.

On the server side, I used [a simple SSL enabled Comet style Pub/Sub server][2] I wrote recently in Node.js. And for the client side, I wrote a simple Android app that connects to this as needed using a standard <a href="http://developer.android.com/reference/javax/net/ssl/HttpsURLConnection.html" target="_blank">HttpsURLConnection</a>. Again performing mutual certificate based authentication between the two.

### How did I test them?

Again, it was very similar to what I did <a href="http://stephendnicholas.com/archives/219" target="_blank">previously</a>; using the Android applications mentioned above, along with simple desktop apps to drive them where required. The tests were kicked off manually, however I&#8217;ve improved the overall automation of them. The results from here should be reliably comparable with the previous ones.

### Caveats / Specifics

  * The power profiling was performed on my HTC Desire, now running Android 2.2.2 &#8211; Build 2.33.161.6 CL345089.
  * &#8216;% Battery / Hour&#8217; refers to the % of the fully charged capacity of my phone&#8217;s battery that is used per hour. My phone has a standard Li-Ion battery that is rated at: 1400mAh & 3.7V.
  * The tool I used to capture the power usage data ([PowerTutor][3]) uses real data, combined with a power usage model for some aspects. This model has been tailored against the type of device I used for this testing and is reported to be accurate to within 0.8% on average, with a 2.5% margin of error. If you&#8217;re interested, you can find out more [here][4].
  * I&#8217;ve made a number of decisions on my implementation of the two approaches. I&#8217;ve done this to try and make MQTT and HTTPS more equal and make things fairer; not to bias things one way or the other. I&#8217;ve tried to do this sensibly and I&#8217;ll explain myself as I go along; however some people may have decided to do things slightly differently.
  * I&#8217;ve tried my best to produce correct, consistent and usable results; however I am human and so there&#8217;s a chance I have made mistakes. As such, these figures shouldn&#8217;t be treated as gospel, but I would expect them to be representative of what you could expect to see and they should provide an accurate comparison between the two approaches.

<div class="linebreak">
</div>

# Results

<div class="linebreak">
</div>

## <a name="connecting"></a>Establishing & Maintaining a Connection

I decided to start out by looking at the cost of establishing and then maintaining an open connection for each approach. And I pretty much ran straight into one of the main challenges of this testing: how to make it a fair fight? How do I structure things so I&#8217;m not being un-necessarily biased to one technology or the other?

Straight out of the box, MQTT is much more feature rich than HTTPS and the main feature involved here is the Keep Alive. Basically this is a way for the client to detect in a timely manner when the server connection has been lost (and vice-versa) without having to wait for the often long TCP/IP timeout. To do this, the MQTT client and server exchange keep-alive messages every so often. This also serves to maintain the raw TCP/IP connection; as in some circumstances (e.g. on some 3G networks) long-running connections that have no data flowing my be purged.

HTTPS doesn&#8217;t have this built in, but it seems like useful functionality for an application to have and so I decided to add this into my HTTPS client by using a <a href="http://developer.android.com/reference/java/net/URLConnection.html#setReadTimeout(int)" target="_blank">read timeout</a> of the same duration as the MQTT keep alive interval. This means that if the server doesn&#8217;t respond with any data within *x* seconds then an Exception is thrown and the connection torn down. The client will then establish a new one. If it can&#8217;t, then the client knows the server is unreachable.

Hopefully that sounds like a fair thing to do, but don&#8217;t worry, it also flows the other way. As, in my implementation, the HTTPS client also &#8216;subscribes&#8217; as part of it&#8217;s connection (the topic of interest being part of the URL), I decided to consider the act of MQTT connecting to include both the connection and a subscription step.

Anyway, enough waffling, onto the results. First of all, the amount of power taken to establish the initial connection to the server:

<table style="text-align:center;margin-left:auto;margin-right:auto;margin-bottom:1.5em;">
  <tr>
    <th colspan="4" style="background-color:#E6E6FA;">
      % Battery Used
    </th>
  </tr>
  
  <tr>
    <th colspan="2" style="background-color:#F8E7B2;">
      3G
    </th>
    
    <th colspan="2" style="background-color:#F8E7B2;">
      Wifi
    </th>
  </tr>
  
  <tr>
    <th style="width:6em;background-color:#f0fae6;">
      HTTPS
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      MQTT
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      HTTPS
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      MQTT
    </th>
  </tr>
  
  <tr>
    <td>
      <b>0.02972</b>
    </td>
    
    <td>
      0.04563
    </td>
    
    <td>
      <b>0.00228</b>
    </td>
    
    <td>
      0.00276
    </td>
  </tr>
</table>

As you can see, HTTPS wins this one hands down; and by quite a significant amount (~30% for 3G). However this isn&#8217;t really that much of a surprise:

With the HTTPS approach, all we&#8217;re doing is opening a connection to the appropriate URL and exchanging certificates. Whereas, for MQTT, we establish the raw connection, perform the certificate exchange and then flow additional information; including the unique Client Id. We then wait to receive a confirmation from the server and then subsequently send additional messages to subscribe to the test topic (which shouldn&#8217;t be required for subsequent reconnections; as the server can remember our subscriptions for us).

*Note: just to emphasise, this is an artificial limitation that I&#8217;ve placed on MQTT. If you don&#8217;t need the subscribe step, then the actual cost of connection is very comparable. However, this is something that I <u>decided</u> to do to make the comparison more fair.*

<a name="openconnection"></a><a name="keepalive"></a>Now onto the cost of &#8216;maintaining&#8217; that connection (in % Battery **/ Hour**):

<table style="text-align:center;margin-left:auto;margin-right:auto;margin-bottom:1.5em;">
  <tr>
    <th style="border:0;">
    </th>
    
    <th colspan="4" style="background-color:#E6E6FA;">
      % Battery / Hour
    </th>
  </tr>
  
  <tr>
    <th style="border:0;">
    </th>
    
    <th colspan="2" style="background-color:#F8E7B2;">
      3G
    </th>
    
    <th colspan="2" style="background-color:#F8E7B2;">
      Wifi
    </th>
  </tr>
  
  <tr>
    <th style="width:7em;background-color:#E6E6FA;">
      Keep Alive<br />(Seconds)
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      HTTPS
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      MQTT
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      HTTPS
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      MQTT
    </th>
  </tr>
  
  <tr>
    <td>
      60
    </td>
    
    <td>
      1.11553
    </td>
    
    <td>
      <b>0.72465</b>
    </td>
    
    <td>
      0.15839
    </td>
    
    <td>
      <b>0.01055</b>
    </td>
  </tr>
  
  <tr>
    <td>
      120
    </td>
    
    <td>
      0.48697
    </td>
    
    <td>
      <b>0.32041</b>
    </td>
    
    <td>
      0.08774
    </td>
    
    <td>
      <b>0.00478</b>
    </td>
  </tr>
  
  <tr>
    <td>
      240
    </td>
    
    <td>
      0.33277
    </td>
    
    <td>
      <b>0.16027</b>
    </td>
    
    <td>
      0.02897
    </td>
    
    <td>
      <b>0.00230</b>
    </td>
  </tr>
  
  <tr>
    <td>
      480
    </td>
    
    <td>
      0.08263
    </td>
    
    <td>
      <b>0.07991</b>
    </td>
    
    <td>
      0.00824
    </td>
    
    <td>
      <b>0.00112</b>
    </td>
  </tr>
</table>

As you can see, this is where MQTT gains back ground. In all cases it uses less power and in most a fair bit less. So the longer the connection is established, the &#8216;cheaper&#8217; MQTT is to use.

If we consider the 3G case (the most relevant to the mobile story) and a keep alive / read timeout interval of 240 seconds (my personal favourite) then by my calculations we make up for the difference in the cost of connecting after ~5 &frac12; minutes of being connected:

<p style="text-align:center;">
  <i>Note: You can click & drag on an area of the graph to zoom to it.<br />Single click anywhere to zoom back out.</i>
</p>

<h4 style="text-align: center;">
  <b>3G &#8211; 240s Keep Alive &#8211; % Battery Used Creating and Maintaining a Connection</b>
</h4>

<div id="noCellBorder">
  <table style="padding:0.5em;padding-right:3em;">
    <tr>
      <td style="-webkit-transform: rotate(-90deg); font-weight:bold;">
        mW
      </td>
      
      <td>
        <div id="_C_240" style="width:550px;height:300px;">
        </div>
      </td>
    </tr>
    
    <tr>
      <td>
      </td>
      
      <td style="text-align:center; font-weight:bold;">
        1 second period
      </td>
    </tr>
  </table>
</div>

After that, everything else is gravy, and based on this I reckon you&#8217;d save ~4.1% battery per day just by using MQTT over HTTPS to maintain an open stable connection.

The reason for this is simple. While it costs MQTT more to create the initial connection, this is essentially a one off and the cost of the following keep alives is comparatively small. Whereas for HTTPS it needs to perform the &#8216;expensive&#8217; connection stage every time it has to reconnect (up to once each keep alive interval, in my implementation).

Interestingly, this does seem to suggest that if the phone is constantly cycling / dropping connections **and** you&#8217;ve decided not to let the server remember your connections (so that MQTT has to perform the additional subscribe step on every reconnect &#8211; cleanstart=true for those in the know) then HTTPS may be a better choice for that particular scenario; and in that case MQTT would be better in slightly more stable situations. The obvious question then is: how often is the connection likely to drop in typical mobile usage?

I&#8217;ve tried searching, but not really found any sensible statistics so far. The mobile network should handle routing TCP/IP connections from tower to tower, so that *shouldn&#8217;t* be an issue; however you might go out of range or hit a tower with no spare capacity. Although I guess that&#8217;s not likely to happen very often (?). It&#8217;s something that I&#8217;d be interested in finding out, but it sounds like it would need a large field trial and that&#8217;s a little outside the scope of this blog post :)

## <a name="receiving"></a>Receiving

The next thing I looked at was receiving messages on the phone. To do this, I sent messages to the phone in two different ways:

### 1. Sporadically

To try and emulate a more realistic style of notification sending, I decided to send 6 messages to the phone, with an average of 1 message per 10 minute interval, but with the message being sent &#8216;randomly&#8217; during that time. This would enable me to test the long-term performance of each approach, but also to add in some unpredictability. For MQTT, the messages were delivered at QoS 1. The results are shown below:

<table style="text-align:center;margin-left:auto;margin-right:auto;margin-bottom:1.5em;">
  <tr>
    <th colspan="4" style="background-color:#E6E6FA;">
      % Battery Used
    </th>
  </tr>
  
  <tr>
    <th colspan="2" style="background-color:#F8E7B2;">
      3G
    </th>
    
    <th colspan="2" style="background-color:#F8E7B2;">
      Wifi
    </th>
  </tr>
  
  <tr>
    <th style="width:6em;background-color:#f0fae6;">
      HTTPS
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      MQTT
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      HTTPS
    </th>
    
    <th style="width:6em;background-color:#f0fae6;">
      MQTT
    </th>
  </tr>
  
  <tr>
    <td>
      0.34645
    </td>
    
    <td>
      <b>0.27239</b>
    </td>
    
    <td>
      0.04817
    </td>
    
    <td>
      <b>0.00411</b>
    </td>
  </tr>
</table>

<p style="text-align:center;">
  <i>Note: the timing of the messages was calculated beforehand, so both MQTT and HTTPS experienced exactly the same delays, etc.</i>
</p>

And here&#8217;s some graphs:

<h4 style="text-align: center;">
  <b>3G &#8211; Receive 6 x 1 byte messages over 60 minutes – Total mW</b>
</h4>

<div id="noCellBorder">
  <table style="padding:0.5em;padding-right:3em;">
    <tr>
      <td style="-webkit-transform: rotate(-90deg); font-weight:bold;">
        mW
      </td>
      
      <td>
        <div id="_3G_Sporadic" style="width:550px;height:300px;">
        </div>
      </td>
    </tr>
    
    <tr>
      <td>
      </td>
      
      <td style="text-align:center; font-weight:bold;">
        1 second period
      </td>
    </tr>
  </table>
</div>

<h4 style="text-align: center;">
  <b>Wifi &#8211; Receive 6 x 1 byte messages over 60 minutes – Total mW</b>
</h4>

<div id="noCellBorder">
  <table style="padding:0.5em;padding-right:3em;">
    <tr>
      <td style="-webkit-transform: rotate(-90deg); font-weight:bold;">
        mW
      </td>
      
      <td>
        <div id="_W_Sporadic" style="width:550px;height:300px;">
        </div>
      </td>
    </tr>
    
    <tr>
      <td>
      </td>
      
      <td style="text-align:center; font-weight:bold;">
        1 second period
      </td>
    </tr>
  </table>
</div>

As you can see, in both cases MQTT wins; by ~30% for the 3G case and an order of magnitude for Wifi.

Also, I actually could&#8217;ve made the MQTT implementation more efficient: Currently I blindly send the keep alive message every *x* seconds, regardless of when the last message was exchanged between client and server. Whereas I&#8217;d actually only need to send the keep alive message *x* seconds after any message was last sent / received. Interestingly, the HTTPS implementation already exhibits this behaviour and so has a slight advantage.

One final thing of note is that the Wifi graph above nicely shows the difference in the keep alive cost between the two approaches. This is not as obvious in the 3G graph, as the CPU & data transmission costs are overwhelmed by the cost of having the 3G active.

### <a name="highvolume"></a>2. As Fast As Possible

To be consistent with the results I produced previously, I also decided to test sending 1024 messages, of 1 byte a piece, to the phone, as quickly as possible. The results are shown below:

<table style="text-align:center;margin-left:auto;margin-right:auto;">
  <tr>
    <th style="border:0;">
      <th colspan="2" style="background-color:#F8E7B2;">
        3G
      </th>
      
      <th colspan="2" style="background-color:#F8E7B2;">
        Wifi
      </th></tr> 
      
      <tr>
        <th style="border:0;">
        </th>
        
        <th style="width:6em;background-color:#f0fae6;">
          HTTPS
        </th>
        
        <th style="width:6em;background-color:#f0fae6;">
          MQTT
        </th>
        
        <th style="width:6em;background-color:#f0fae6;">
          HTTPS
        </th>
        
        <th style="width:6em;background-color:#f0fae6;">
          MQTT
        </th>
      </tr>
      
      <tr>
        <th style="width:6em;background-color:#E6E6FA;">
          % Battery / Hour
        </th>
        
        <td>
          18.43%
        </td>
        
        <td>
          <b>16.13%</b>
        </td>
        
        <td>
          <b>3.45%</b>
        </td>
        
        <td>
          4.23%
        </td>
      </tr>
      
      <tr>
        <th style="width:6em;background-color:#E6E6FA;">
          Messages / Hour
        </th></td> 
        
        <td>
          1708
        </td>
        
        <td>
          <b>160278</b>
        </td>
        
        <td>
          3628
        </td>
        
        <td>
          <b>263314</b>
        </td>
      </tr>
      
      <tr>
        <th style="width:6em;background-color:#E6E6FA;">
          % Battery / Message <a href="#BatteryPerMessageDisclaimer" style="background:none;">*</a>
        </th>
        
        <td>
          0.01709
        </td>
        
        <td>
          <b>0.00010</b>
        </td>
        
        <td>
          0.00095
        </td>
        
        <td>
          <b>0.00002</b>
        </td>
      </tr>
      
      <tr>
        <th style="width:6em;background-color:#E6E6FA;">
          Messages Received
        </th>
        
        <td>
          240 / 1024
        </td>
        
        <td>
          <b>1024 / 1024</b>
        </td>
        
        <td>
          524 / 1024
        </td>
        
        <td>
          <b>1024 / 1024</b>
        </td>
      </tr></table> 
      
      <p>
        <a name="BatteryPerMessageDisclaimer" style="background:none;">&nbsp;</a><br /> * &#8211; % Battery / Message is a bit of a silly metric. There is a fixed cost in having the Wifi or 3G active and so the actual cost of just sending / receiving a single message would be higher. However these figures do serve to indicate the difference between the two approaches and hopefully give you an indication of the battery usage involved.
      </p>
      
      <p>
        And here&#8217;s some more graphs:
      </p>
      
      <h4 style="text-align: center;">
        <b>3G &#8211; Receive 1024 x 1 byte messages – Total mW</b>
      </h4>
      
      <div id="noCellBorder">
        <table style="padding:0.5em;padding-right:3em;">
          <tr>
            <td style="-webkit-transform: rotate(-90deg); font-weight:bold;">
              mW
            </td>
            
            <td>
              <div id="_3G_R" style="width:550px;height:300px;">
              </div>
            </td>
          </tr>
          
          <tr>
            <td>
            </td>
            
            <td style="text-align:center; font-weight:bold;">
              1 second period
            </td>
          </tr>
        </table>
      </div>
      
      <h4 style="text-align: center;">
        <b>Wifi &#8211; Receive 1024 x 1 byte messages – Total mW</b>
      </h4>
      
      <div id="noCellBorder">
        <table style="padding:0.5em;padding-right:3em;">
          <tr>
            <td style="-webkit-transform: rotate(-90deg); font-weight:bold;">
              mW
            </td>
            
            <td>
              <div id="_W_R" style="width:550px;height:300px;">
              </div>
            </td>
          </tr>
          
          <tr>
            <td>
            </td>
            
            <td style="text-align:center; font-weight:bold;">
              1 second period
            </td>
          </tr>
        </table>
      </div>
      
      <p style="text-align:center;">
        <i>Note: the data in the graphs for each approach stops 5 seconds after all messages have been received.</i>
      </p>
      
      <p>
        Again, MQTT wins this one. Despite it being a bit of a silly metric, the % Battery / Message really is the interesting one to look at, and in the 3G case MQTT is over two orders of magnitude better than HTTPS. This highlights both the low power usage of MQTT and also the speed with which the messages were received (averaging 160278 messages per hour for MQTT versus only 1708 for HTTPS).
      </p>
      
      <p>
        <a name="reliable"></a>Another important thing to highlight is the reliability (or perhaps, more accurately, success rate) of delivery. For 3G, every MQTT message got through, whereas HTTPS only managed 240 of the 1024, ~24%. This is because a lot of messages were missed in the interval between when the connection closed with the previous message and was subsequently re-established to receive the next. This is somewhat mitigated on Wifi, as the connection can be re-established more quickly. I could&#8217;ve decided to implement a queueing mechanism on the server to help with this, but as MQTT didn&#8217;t need it, I thought it was fair not to.
      </p>
      
      <h2>
        <a name="sending"></a><a name="twoway"></a>Sending
      </h2>
      
      <p>
        Finally, I looked at sending messages from the phone. I found it somewhat difficult to get sensible & consistent figures for sending a single message and so I decided to scale up to sending 1024 messages, of 1 byte a piece, as quickly as possible. The results are shown below:
      </p>
      
      <table style="text-align:center;margin-left:auto;margin-right:auto;">
        <tr>
          <th style="border:0;">
            <th colspan="2" style="background-color:#F8E7B2;">
              3G
            </th>
            
            <th colspan="2" style="background-color:#F8E7B2;">
              Wifi
            </th></tr> 
            
            <tr>
              <th style="border:0;">
              </th>
              
              <th style="width:6em;background-color:#f0fae6;">
                HTTPS
              </th>
              
              <th style="width:6em;background-color:#f0fae6;">
                MQTT
              </th>
              
              <th style="width:6em;background-color:#f0fae6;">
                HTTPS
              </th>
              
              <th style="width:6em;background-color:#f0fae6;">
                MQTT
              </th>
            </tr>
            
            <tr>
              <th style="width:6em;background-color:#E6E6FA;">
                % Battery / Hour
              </th>
              
              <td>
                18.79%
              </td>
              
              <td>
                <b>17.80%</b>
              </td>
              
              <td>
                5.44%
              </td>
              
              <td>
                <b>3.66%</b>
              </td>
            </tr>
            
            <tr>
              <th style="width:6em;background-color:#E6E6FA;">
                Messages / Hour
              </th></td> 
              
              <td>
                1926
              </td>
              
              <td>
                <b>21685</b>
              </td>
              
              <td>
                5229
              </td>
              
              <td>
                <b>23184</b>
              </td>
            </tr>
            
            <tr>
              <th style="width:6em;background-color:#E6E6FA;">
                % Battery / Message <a href="#BatteryPerMessageDisclaimer" style="background:none;">*</a>
              </th>
              
              <td>
                0.00975
              </td>
              
              <td>
                <b>0.00082</b>
              </td>
              
              <td>
                0.00104
              </td>
              
              <td>
                <b>0.00016<br /> </b>
              </td>
            </tr></table> 
            
            <p style="margin-top:1.5em;">
              And here&#8217;s, yes you guessed it, yet more graphs:
            </p>
            
            <h4 style="text-align: center;">
              <b>3G &#8211; Send 1024 x 1 byte messages – Total mW</b>
            </h4>
            
            <div id="noCellBorder">
              <table style="padding:0.5em;padding-right:3em;">
                <tr>
                  <td style="-webkit-transform: rotate(-90deg); font-weight:bold;">
                    mW
                  </td>
                  
                  <td>
                    <div id="_3G_S" style="width:550px;height:300px;">
                    </div>
                  </td>
                </tr>
                
                <tr>
                  <td>
                  </td>
                  
                  <td style="text-align:center; font-weight:bold;">
                    1 second period
                  </td>
                </tr>
              </table>
            </div>
            
            <h4 style="text-align: center;">
              <b>Wifi &#8211; Send 1024 x 1 byte messages – Total mW</b>
            </h4>
            
            <div id="noCellBorder">
              <table style="padding:0.5em;padding-right:3em;">
                <tr>
                  <td style="-webkit-transform: rotate(-90deg); font-weight:bold;">
                    mW
                  </td>
                  
                  <td>
                    <div id="_W_S" style="width:550px;height:300px;">
                    </div>
                  </td>
                </tr>
                
                <tr>
                  <td>
                  </td>
                  
                  <td style="text-align:center; font-weight:bold;">
                    1 second period
                  </td>
                </tr>
              </table>
            </div>
            
            <p>
              And again, MQTT wins. It uses significantly less power (see % Battery / Message) and is a lot quicker. I don&#8217;t think there&#8217;s anything else I need to say about this one.
            </p>
            
            <div class="linebreak">
            </div>
            
            <h1>
              <a name="conclusions"></a>Conclusions
            </h1>
            
            <div class="linebreak">
            </div>
            
            <p>
              Well, I have to admit I&#8217;m somewhat relieved. I&#8217;ve been using MQTT on mobile for a while now and I&#8217;ve been confident that the power consumption was lower than that of HTTPS, but it&#8217;s good to see that the figures support that. I&#8217;m definitely an MQTT fan, but I&#8217;ve been keen to make this testing fair and unbiased and I would&#8217;ve reported the results whichever came out on top (although I know a few people who would be none too happy about that). So definitely relieved :)
            </p>
            
            <p>
              As this testing has shown, MQTT uses less power to maintain an open connection, to receive messages and to send them. It also does these last two more quickly and reliably. The only place where it loses out is in establishing the initial connection (with cleanstart=true) and that&#8217;s mitigated after ~5 &frac12; minutes of being connected.
            </p>
            
            <p>
              There&#8217;s a number of other benefits of MQTT over HTTPS as well, which I&#8217;ve not really included in this testing, which include:
            </p>
            
            <ul>
              <li>
                Assured delivery
              </li>
              <li>
                Retained messages
              </li>
              <li>
                Last will & testament
              </li>
              <li>
                Multiple subscriptions &#8216;multiplexed&#8217; over one connection
              </li>
            </ul>
            
            <p>
              These could theoretically be built into a HTTPS approach, but they&#8217;re not there by default.
            </p>
            
            <p>
              So I think it&#8217;s definitely fair to say that MQTT wins overall and is my technology of choice for providing true push capabilities on Android.
            </p>
            
            <h2>
              What next?
            </h2>
            
            <p>
              I&#8217;ve addressed a few of the additional use cases I identified in my <a href="http://stephendnicholas.com/archives/219" target="_blank">last bout of MQTT power profiling</a>, but there&#8217;s still a couple of things I&#8217;d like to look in to:
            </p>
            
            <ul>
              <li>
                Testing the effect of message size on sending & receiving.
              </li>
              <li>
                Testing power usage when running on fragile networks that keep dropping & being re-established.
              </li>
              <li>
                Performing some longer term live field testing.
              </li>
            </ul>
            
            <div class="linebreak">
            </div>
            
            <p>
              So, congratulations on reading this far. That&#8217;s a bit of an epic post, my longest to date, but again hopefully it was interesting and informative. And I welcome any comments.
            </p>
            
            <link href="/blog_misc/jquery/jquery-ui-1.8.14.custom.css" rel="stylesheet" type="text/css" />
            
            <br />

 [1]: http://stephendnicholas.com/wp-content/uploads/2012/05/longpolling.png
 [2]: http://stephendnicholas.com/archives/1131
 [3]: http://ziyang.eecs.umich.edu/projects/powertutor/index.htm;
 [4]: http://ziyang.eecs.umich.edu/projects/powertutor/camera-ready.pdf