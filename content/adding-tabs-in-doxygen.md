+++
date = "2014-05-17"
title = "Adding Tabs in Doxygen"
+++

There are probably few C/C++ programmers that haven't heard of Doxygen. It would have to be the most well known documentation generator for those two languages. For me, my first experience with documentation systems was Java's humble javadoc. Whilst I've written a fair amount of C/C++ code (and document'd it), and hadn't really bothered to learn too much of the underlying documentation system.

Recently I was documenting some code I had written and wanted to add a seperate tab to show some licencing information. It took me a bit of hunting around to find out how to do this - it is mentioned in Doxygen, but some of the information around it is a little unintuitive. 

Now, adding a tab itself is actually quite simple, but I wanted doxygen to be able to reference this tab. In my preceding example, I wanted to be able to include 

<pre><code>
\licence

Here lies some licencing code

</code></pre>

Into my main doxygen file and have it know to create a seperate licencing section and link it to the tab.

To do this, you need to edit the doxygenlayout.xml file. At the top, you will have a navindex directive at the top of it, which will probably look something like this:

<pre><code>
&lt;navindex&gt;
    &lt;tab type=&quot;mainpage&quot; visible=&quot;yes&quot; title=&quot;Home&quot;/&gt;
    &lt;tab type=&quot;modules&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
    &lt;tab type=&quot;namespaces&quot; visible=&quot;yes&quot; title=&quot;&quot;&gt;
      &lt;tab type=&quot;namespacelist&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
      &lt;tab type=&quot;namespacemembers&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
    &lt;/tab&gt;
    &lt;tab type=&quot;classes&quot; visible=&quot;yes&quot; title=&quot;&quot;&gt;
      &lt;tab type=&quot;classlist&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
      &lt;tab type=&quot;classindex&quot; visible=&quot;$ALPHABETICAL_INDEX&quot; title=&quot;&quot;/&gt; 
      &lt;tab type=&quot;hierarchy&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
      &lt;tab type=&quot;classmembers&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
    &lt;/tab&gt;
    &lt;tab type=&quot;files&quot; visible=&quot;yes&quot; title=&quot;&quot;&gt;
      &lt;tab type=&quot;filelist&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
      &lt;tab type=&quot;globals&quot; visible=&quot;yes&quot; title=&quot;&quot; intro=&quot;&quot;/&gt;
    &lt;/tab&gt;
  &lt;/navindex&gt;
</code></pre>

Lets add something to it:

<pre><code>
&lt;tab type=&quot;user&quot; url=&quot;@ref licence&quot; visible=&quot;yes&quot; title=&quot;Licence&quot;/&gt;
</code></pre>

This add a "user" defined tab, that is visible (obviously) and is displayed as Licence. The url variable "@ref licence" lets doxygen know to create the licence html page and that any \licence directive will reference that page.