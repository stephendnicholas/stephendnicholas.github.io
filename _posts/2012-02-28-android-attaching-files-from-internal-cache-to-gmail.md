---
layout: post
permalink: /android-attaching-files-from-internal-cache-to-gmail
alias: [/archives/974]

title: 'Android: Attaching files from internal cache to Gmail'
author: Stephen Nicholas
excerpt: "How to use a custom ContentProvider to attach files from an application's internal cache to a Gmail message (as the standard way to reference files as attachments does not work)."

dsq_thread_id:
  - 4458084681
---

*Note: the method described in the post works well for Gmail, but apparently has some issues with other ACTION_SEND handlers (e.g. the MMS composer). A workaround is described in the comments.*


I've just spent the day modifying one of my Android applications so that it no longer requires the use of the SD card; making it use the internal cache for what little storage is required. Everything went pretty smoothly, until I got to a part of the app that tries to share some data by sending an email with an attachment via Gmail (using an <a href="http://developer.android.com/reference/android/content/Intent.html#ACTION_SEND" target="_blank"><code>ACTION_SEND</code></a> intent). Then things started behaving very oddly.

What I saw was that the Compose activity would launch correctly and my file would be shown as attached (as an example, I've attached a text file called Test.txt here); however when I sent the message, the attachment was not on the received email:

<div class="inline_imgs">
<img src="{{ site.baseurl }}/assets/img/gmail_attach_1.png" style="width:45%" /><img src="{{ site.baseurl }}/assets/img/gmail_attach_2.png" style="width:45%" />
</div>

Or even shown in my Gmail sent folder.

A quick browse through the log showed the reason:

<code>02-28 21:01:28.434: E/Gmail(19673): file:// attachment paths must point to file:///mnt/sdcard. Ignoring attachment file:///data/data/com.stephendnicholas.gmailattach/cache/Test.txt</code>

So, it looks like this is something Gmail has explicitly ruled out; though I'm not sure why. At this point I could easily get all moany and ranty, but today was a glass half full day and so it was time for a workaround.

After a few false starts and a lot of searching, what I actually ended up doing was using a <a href="http://developer.android.com/reference/android/content/ContentProvider.html" target="_blank"><code>ContentProvider</code></a> to provide access to the file from my application's internal cache; as apparently Gmail can happily resolve attachments this way.

Unfortunately there wasn't much example code to help me on my way, which is why I've decided to include a simple example here. There's four main parts to this:

1. The <a href="http://developer.android.com/reference/android/content/ContentProvider.html" target="_blank"><code>ContentProvider</code></a> that provides access to the files from the application's internal cache. Complete code, with comments, is shown below:

{% highlight java %}
package com.stephendnicholas.gmailattach;

import java.io.File;
import java.io.FileNotFoundException;
import android.content.ContentProvider;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.database.Cursor;
import android.net.Uri;
import android.os.ParcelFileDescriptor;
import android.util.Log;

public class CachedFileProvider extends ContentProvider {

  private static final String CLASS_NAME = "CachedFileProvider";

  // The authority is the symbolic name for the provider class
  public static final String AUTHORITY = "com.stephendnicholas.gmailattach.provider";

  // UriMatcher used to match against incoming requests<br /> private UriMatcher uriMatcher;
  @Override
  public boolean onCreate() {
    uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    // Add a URI to the matcher which will match against the form
    // 'content://com.stephendnicholas.gmailattach.provider/*'
    // and return 1 in the case that the incoming Uri matches this pattern
    uriMatcher.addURI(AUTHORITY, "*", 1);

    return true; 
  }

  
  @Override
  public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {

    String LOG_TAG = CLASS_NAME + " - openFile";
    Log.v(LOG_TAG, "Called with uri: '" + uri + "'." + uri.getLastPathSegment());

  // Check incoming Uri against the matcher
  switch (uriMatcher.match(uri)) {
    // If it returns 1 - then it matches the Uri defined in onCreate
    case 1:
      // The desired file name is specified by the last segment of the path
      // E.g. 'content://com.stephendnicholas.gmailattach.provider/Test.txt'
      // Take this and build the path to the file
      String fileLocation = getContext().getCacheDir() + File.separator + uri.getLastPathSegment();

      // Create & return a ParcelFileDescriptor pointing to the file
      // Note: I don't care what mode they ask for - they're only getting
      // read only
      ParcelFileDescriptor pfd = ParcelFileDescriptor.open(new File(fileLocation),  ParcelFileDescriptor.MODE_READ_ONLY);
      return pfd;

    // Otherwise unrecognised Uri
    default:
      Log.v(LOG_TAG, "Unsupported uri: '" + uri + "'.");
      throw new FileNotFoundException("Unsupported uri: " + uri.toString());
    }
  }

  // //////////////////////////////////////////////////////////////
  // Not supported / used / required for this example
  // //////////////////////////////////////////////////////////////

  @Override
  public int update(Uri uri, ContentValues contentvalues, String s, String[] as) { return 0; }

  @Override
  public int delete(Uri uri, String s, String[] as) { return 0; }

  @Override
  public Uri insert(Uri uri, ContentValues contentvalues) { return null; }

  @Override
  public String getType(Uri uri) { return null; }

  @Override
  public Cursor query(Uri uri, String[] projection, String s, String[] as1, String s1) { return null; }
}
{% endhighlight %}
  
As you can see, all you really need to do is to overwrite the <code>openFile(...)</code> method. Although you can also override the <code>query(...)</code> method to provide more information to the application that calls you (it would make this example unnecessarily complicated, but I'm happy to provide code on request).

To make this provider available to use, you need to add a line to your <code>AndroidManifest.xml</code> defining the class and the authority (symbolic name) used to reference it:

{% highlight xml %}
<provider android:name="CachedFileProvider" android:authorities="com.stephendnicholas.gmailattach.provider"></provider>
{% endhighlight %}

<b>Note:</b> this needs to go inside the <code>&lt;application&gt;...&lt;/application&gt;</code> definition in your <code>AndroidManifest.xml</code>.
  
<ol start="2">
  <li>
    The utility method that creates a file in the internal cache:
  </li>
</ol>

{% highlight java %}
public static void createCachedFile(Context context, String fileName, String content) throws IOException {

  File cacheFile = new File(context.getCacheDir() + File.separator + fileName);
  cacheFile.createNewFile();

  FileOutputStream fos = new FileOutputStream(cacheFile);
  OutputStreamWriter osw = new OutputStreamWriter(fos, "UTF8");
  PrintWriter pw = new PrintWriter(osw);
  
  pw.println(content);
  
  pw.flush();
  pw.close();
}
{% endhighlight %}
  
<ol start="3">
  <li>
    The utility method that creates the intent to send the content via Gmail (explicitly using Gmail, rather than using a chooser):
  </li>
</ol>
  
{% highlight java %}
public static Intent getSendEmailIntent(Context context, String email, String subject, String body, String fileName) {

  final Intent emailIntent = new Intent(android.content.Intent.ACTION_SEND);

  //Explicitly only use Gmail to send
  emailIntent.setClassName("com.google.android.gm","com.google.android.gm.ComposeActivityGmail");
  
  emailIntent.setType("plain/text");

  //Add the recipients
  emailIntent.putExtra(android.content.Intent.EXTRA_EMAIL, new String[] { email });

  emailIntent.putExtra(android.content.Intent.EXTRA_SUBJECT, subject);

  emailIntent.putExtra(android.content.Intent.EXTRA_TEXT, body);

  //Add the attachment by specifying a reference to our custom ContentProvider
  //and the specific file of interest
  emailIntent.putExtra(Intent.EXTRA_STREAM, Uri.parse("content://" + CachedFileProvider.AUTHORITY + "/" + fileName));

  return emailIntent;
}
{% endhighlight %}

<ol start="4">
  <li>
    The code that calls 2 & 3 to do something on a button press. Triggered in my dummy app by a button click:
  </li>
</ol>
  
{% highlight java %}
Button button = (Button) findViewById(R.id.dostuff);

button.setOnClickListener(new OnClickListener() {
  @Override
  public void onClick(View v) {
    try {
      Utils.createCachedFile(GmailAttacherActivity.this, "Test.txt", "This is a test");

      startActivity(Utils.getSendEmailIntent( GmailAttacherActivity.this, "<YOUR_EMAIL_HERE>@<YOUR_DOMAIN>.com", "Test", "See attached", "Test.txt"));
    } catch (IOException e) {
      e.printStackTrace();
    } catch (ActivityNotFoundException e) {
      Toast.makeText(GmailAttacherActivity.this, "Gmail is not available on this device.", Toast.LENGTH_SHORT).show();
    }
  }
});
{% endhighlight %}

Hopefully that's all pretty straight forward, however feel free to ask any questions in the comments and I'll help where I can.

The full source is now available as a complete (albeit very simple) Android app on github: <a href="https://github.com/stephendnicholas/Android-Apps/tree/master/Gmail%20Attacher">Gmail Attacher</a>.

