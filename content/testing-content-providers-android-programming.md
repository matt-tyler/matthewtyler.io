+++
date = "2012-09-06"
title = "Testing Content Providers: Android Programming"
+++
<p>
Content Providers are a means of encapsulating and providing data to applications through a simple interface. Content Providers are therefore one of the main building blocks in android applications along with services and activities. </p>
<p>
Now, you can quite easily embed raw sql queries and data access methods directly into your activity, but it is cleaner and better practice to have some separate interface that you can use isolate and test data access methods. The Content Provider is also required if you want to expose data in your application to third-party applications. Imagine you wanted to implemented a widget that uses the data from your application; it would require you to define a Content Provider to do so.
</p>
<p>
In order to define a Content Provider, you must implement the following methods;
</p>
<p>
<li>onCreate() which is called to initialize the provider</li>
<li>query(Uri, String[], String, String[], String) which returns data to the caller</li>
<li>insert(Uri, ContentValues) which inserts new data into the content provider</li>
<li>update(Uri, ContentValues, String, String[]) which updates existing data in the content provider</li>
<li>delete(Uri, String, String[]) which deletes data from the content provider</li>
<li>getType(Uri) which returns the MIME type of data in the content provider</li>
</p></br>
<p>
It is important to note, that the primary mechanism by which we are making queries is through the Uri. Every Content Provider defines an authority (usually the package name with that of Content Provider appended to it ie: com.example.myapp.MyContentProvider) which is declared in your applications manifest file, within the application tag, like thus;</p>

<pre><code>&lt;manifest xmlns:android=&quot;http://schemas.android.com/apk/res/android&quot;
    package=&quot;com.example.myapp&quot;
    android:versionCode=&quot;1&quot;
    android:versionName=&quot;1.0&quot; &gt;

    &lt;uses-sdk
        android:minSdkVersion=&quot;11&quot;
        android:targetSdkVersion=&quot;15&quot; /&gt;
    &lt;uses-permission android:name=&quot;android.permission.INTERNET&quot; /&gt;&lt;uses-permission android:name=&quot;android.permission.INTERNET&quot; /&gt;
    
    &lt;application
        android:icon=&quot;@drawable/ic_launcher&quot;
        android:label=&quot;@string/app_name&quot;
        android:theme=&quot;@style/AppTheme&quot; &gt;
        &lt;activity
            android:name=&quot;.Workspace&quot;
            android:label=&quot;@string/title_activity_workspace&quot; &gt;
            &lt;intent-filter&gt;
                &lt;action android:name=&quot;android.intent.action.MAIN&quot; /&gt;

                &lt;category android:name=&quot;android.intent.category.LAUNCHER&quot; /&gt;
            &lt;/intent-filter&gt;
        &lt;/activity&gt;
        &lt;provider     android:name=&quot;com.flashics.sutra.AsanaProvider&quot; 
                android:authorities=&quot;com.example.myapp.MyContentProvider&quot;/&gt;  
    &lt;/application&gt;
&lt;/manifest&gt;
</code></pre>
<p>
The next step in building a Content Provider is to identify what data needs to be queried. For the sake of brevity, we will implement a content provider that will can query the asana api to return a list of workspaces, and a list of projects in a particular workspace. This link contains the asana api documentation <a href="http://developer.asana.com/documentation/" title="Asana API">http://developer.asana.com/documentation/</a>.
</p>
<p>
From this documentation, we can see that the two http requests that need to be made are;
</p>
<p>
https://app.asana.com/api/1.0/workspaces
</p>
<p>
and
</p>
<p>
https://app.asana.com/api/1.0/workspaces/x/projects where x is the id of the workspace
</p>
<p>
For actually making the requests, I will use the aquery library for simplicity, which can be found here - <a href="http://code.google.com/p/android-query/" title="AQuery">http://code.google.com/p/android-query/</a>
 - though you should be able to follow along without knowledge of this library.
</p>
<p>
Content Providers function by being passed a Uri which determines what data it should query. You then need to determine a sensible set of Uri's for determining what data to fetch. 
</p>
<p>
Usually, the uri should be of the format 'content://<authority>/tablename' so in our example, we will define our two uri's as
</p>
<p>
content://com.example.myapp/workspaces
</p>
<p>
and
</p>
<p>
content://com.example/myapp/projects/workspaces/x where x is the id of the workspace
</p>
<p>
Note that I've rearranged the uri a little as compared with the http request. The URI object libraries in the android/java api have helper functions that make it easy to strip the last path segment.
</p>
<p>
Now onto the actual content provider code!
</p>
<p>
We first define a Uri matcher class within the ContentProvider Class ie;
</p>
<pre><code>public class MyContentProvider extends ContentProvider {
    
    private static final String TAG = AsanaProvider.class.getName(); // Helpful for debugging
    
    private static final String AUTHORITY = &quot;com.example.myapp.MyContentProvider&quot;; //Our Authority
    
    private static final UriMatcher sURIMatcher = new UriMatcher(UriMatcher.NO_MATCH);
      static {          
        sURIMatcher.addURI(AUTHORITY, &quot;workspaces&quot;, 1); 
        sURIMatcher.addURI(AUTHORITY, &quot;workspaces/projects/#&quot;,2);
      }

}
</code></pre>
<p>
The UriMatcher is a handy class for constructing switch statements, like we will do in the following code for query().
</p>
<p>
Note the "YOUR API KEY KEY GOES HERE"; if you want to compile and test the code in the android app, you'll need to make an account at Asana and then copy and paste your api key in.
</p>
<pre><code>    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
            String[] selectionArgs, String sortOrder) {
        
        //Log.i(TAG,uri.toString());
        int match = sURIMatcher.match(uri);
        
        String[] columns = {&quot;data&quot;};
        MatrixCursor cursor = new MatrixCursor(columns);
        
        String url = &quot;https://app.asana.com/api/1.0/workspaces&quot;;
        String encoding = Base64.encodeToString(&quot;YOUR_API_KEY_GOES_HERE:&quot;.getBytes(),2);
        
        
        StringBuilder baseurl= new StringBuilder(&quot;https://app.asana.com/api/1.0/&quot;);
        switch(match) {
        case    1:
            baseurl.append(&quot;workspaces&quot;);
            break;
        case    2:
            baseurl.append(&quot;workspaces/&quot;).append(uri.getLastPathSegment()).append(&quot;/projects&quot;);
            break;
        }
        
        AjaxCallback&lt;String&gt; cb = new AjaxCallback&lt;String&gt;();
        cb.url(baseurl.toString()).type(String.class).weakHandler(this,&quot;stringCb&quot;);
        cb.header(&quot;Authorization&quot;, &quot;Basic &quot; + encoding);
        
        AQuery aq = new AQuery(getContext());
        
        aq.sync(cb);
        
        String jo = cb.getResult();
        AjaxStatus status = cb.getStatus();
        
        //Log.i(TAG, jo);
        //Log.i(TAG,status.toString());
        
        try {
            JSONObject workspaceWrapper = (JSONObject) new JSONTokener(jo).nextValue();
            JSONArray  workspaces       = workspaceWrapper.getJSONArray(&quot;data&quot;);
            
            for (int i = 0; i &lt; workspaces.length(); i++) {
                String[] workspace = {workspaces.getJSONObject(i).toString()};
                cursor.addRow(workspace);
            }
        }
        catch (JSONException e) {
             Log.e(TAG, &quot;Failed to parse JSON.&quot;, e);
        }
        
        return cursor;
    }
</code></pre>
<p>
The switch statement will check what uri we have received, and then use it to build the corresponding uri request. The rest of the code is just making the http request and parsing the JSON response into a matrixcursor. Note that you must return a cursor; The easiest way to do this is by using MatrixCursor, which you construct by passing a String array which contains a list of column names to the constructor.
</p>
<p>
That's basically it. 
</p>
<p>
To access the data in your activity, service etc can be done in a few different manners
</p>
<p>
<li>
Use a loader - which will involved creating a CursorLoader which can query the Uri directly, load everything in another thread, and keep everything all nice and organised (this is probably the best way)</li></p><p>
<li>
Calling getContentResolver() in your activity and using the ContentResolver to directly query the ContentProvider - note that if your content provider is synchronous it will block the main thread, so make sure the call is asynchronous if you are going to do this. In my example, I've made a synchronous call.
</li></p>
<p>
Android Design Patterns recommend using loaders.
</p>
<p>
Now lets say we want to make some tests for content provider, to check that we've done everything properly. Androids test framework (based on junit) allows you to do this with the ProviderTestCase2 class, which will set up the MockContentResolver object that will allow us to test our Content Provider.
</p>
<p>
Using ProviderTestCase2 we request a MockContentResolver object and use this to make queries to the content provider. The setup() and tearDown() methods execute before each test and are used to perform initialisation. You can then define as many test cases as you like and use assertions to check that you are retrieving the correct data from the Content Provider.
</p>
<p>
The below is an example of how to set this class up;
</p>
<pre><code>package com.example.myapp.test;

import org.json.JSONException;
import org.json.JSONObject;

import android.content.ContentValues;
import android.database.Cursor;
import android.net.Uri;
import android.test.ProviderTestCase2;
import android.test.mock.MockContentResolver;
import android.util.Log;

import com.example.myapp.MyContentProvider.*;

public class CPTester extends ProviderTestCase2&lt;MyContentProvider&gt; { // Extend your base class and replace the generic with your content provider
    
    private static final String TAG = CPTester.class.getName();
    
    private static MockContentResolver resolve; // in the test case scenario, we use the MockContentResolver to make queries
    
    public CPTester(Class&lt;AsanaProvider&gt; providerClass,String providerAuthority) {
        super(providerClass, providerAuthority);
        // TODO Auto-generated constructor stub
    }

    public CPTester() {
        //this.CPTester(&quot;com.example.myapp.MyContentProvider&quot;,AsanaProvider.class);
        super(MyContentProvider.class,&quot;com.example.myapp.MyContentProvider&quot;);
    }
        

    @Override
    public void setUp() {
        try {
            Log.i(TAG, &quot;Entered Setup&quot;);
            super.setUp();
            resolve = this.getMockContentResolver();
        }
        catch(Exception e) {
            
            
        }
    }
    
    @Override
    public void tearDown() {
        try{
            super.tearDown();
        }
        catch(Exception e) {
            
            
        }
    }
    
    public void testCase() {
        Log.i(&quot;TAG&quot;,&quot;Basic Insert Test&quot;);
    }
    
    public void testPreconditions() {
        // using this test to check data already inside my asana profile

        Log.i(&quot;TAG&quot;,&quot;Test Preconstructed Database&quot;);
        String[] projection = {&quot;workspace_id&quot;,&quot;name&quot;};
        String selection = null;
        String[] selectionArgs = null;
        String sortOrder = null;
        Cursor result = resolve.query(Uri.parse(&quot;content://com.example.myapp.MyContentProvider/workspace&quot;), projection, selection, selectionArgs, sortOrder);
        
        assertEquals(result.getCount(), 3); //check number of returned rows
        assertEquals(result.getColumnCount(), 2); //check number of returned columns
        
        result.moveToNext();
        
        for(int i = 0; i &lt; result.getCount(); i++) {
            String id = result.getString(0);
            String name = result.getString(1);
            Log.i(&quot;TAG&quot;,id + &quot; : &quot; + name);
            result.moveToNext();
        }
    }
    
}
</code></pre>
<p>
Hopefully, this gives you a basic introduction to Content Providers on Android and how to test them.
</p>