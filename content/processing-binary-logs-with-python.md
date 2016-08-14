+++
date = "2014-07-16"
title = "Processing binary logs with python"
+++

Long ago, in a time when you had nothing but mechanical systems, condition monitoring was much simpler. You had the benefit of being able to physically inspect such a a system for wear and tear to work out whether the particular component was degraded. Unfortunately, we aren't afforded this luxury with many electronic systems (though if you can sense defects by looking at silicon I think you are probably going to be very rich). 

As we move to using more and more electronic condition monitoring systems, both for industry and the general consumer (i.e. the internet of things), being able to parse logs and keep up with the ensuing tidle wave of data is incredibly important. We do not want to miss important events that can explain the cause of a particular fault, nor do we want to overlook data that can give us hints to what might happen in the future.

The following details some recent adventures I've had with parsing binary log files with Python, and therein trying to get the best possible performance I could. None of what I am going to explain is new or groundbreaking, but it might assist others with a framework for writing binary parsers. 

On that note though, I do believe a language like C/C++ is better for this particular task (which I will indeed address later on). However, python is great for prototyping, and the overhead required to learn certain features in C/C++ is much greater than the Python equivalent.

#### Motivation

I currently work at a railway that provides a protection system deployed on-board it's locomotives. This system provides supervision, preventing the driver from exceeding speed restrictions, entering sections they do authority to enter, as well as providing various bits of feedback. This is vital to ensuring safe railway operation, and I have little doubt this and systems just like it around the world have prevented many potential accidents.

Knowing this, we can come to impacts that occur regarding an unhealthy state in this system.

1. The system fails to acknowledge a sequence of events that could lead to an incident, thus greatly increasing the chance of that event occuring.

2. The second is in which the system is overzealous resulting in network delays at no added safety, resulting in reduced performance.

Neither of these conditions is acceptable and engineers should be vigilent in identifying these issues whilst attempting to understand and eradicate their prevalence. Fortunately for us, systems such as this are generally SIL (safety integrity level) rated and the second case is far more likely than the first.

#### Understanding and Improving the System

This brings us to how we can improve such a system. It's pretty to safe to assume that for the most part, the system will follow it's original specification. But occasionally we will hit an edge case, where the planets align in such a way that the system seems to act contrary to the design. The only way to understand this is through evaluation of historic log data; to see if we can piece together the set of circumstances that cause the issue.

These types of events quite often end up being 'needles-in-the-hay-stack', so you often need a large quantity of data to sift through to find a handfull of events. This obviously neccessitates some sort of automatic processing. 

With all that in mind, we are stepping towards a list of requirements for our log parsing system. It should be expected that it the parsing system:

a) **Be accurate**. It should throw away informaiton that is unreliable, incomplete, or just plain useless for the set of situations that we want to investigate.

b) **Be fast**. We might have to sift through a few times, changing the end-database schema as a result. Requirements change, and you want to be able to transform an existing set to the new set as quickly as possible.

c) **Be modifiable**. Those changes need to able to be implemented quickly by the developer. What if the format of the log files changes at some point and new data types/codes are added?

I'll be benchmarking my implementation of this in Python, but most of results are applicable in other languages. I will, of course, discuss various elements of Python itself that both help and hinder this goals, which I believe is important given the popularity and penetration of Python.

#### Extract-Transform-Load

Those familiar with databases will recognise what I'm doing as part of the ETL Holy Trinity. In particular, what I discuss is here is essentially the transform part. The safety system has already extracted the data it needs into log files - We are simply *transforming* them into a suitable format for storage and interrogation.

#### Our pipeline

Our data processing pipeline consists of the following abstractions.

##### The main routine

This is the general area of execution, which corals the various files into and out of main memory, creates whatever objects we may need to do the processing, and outputs our results. Consider this our log parsing 'manager'.

##### The Decoder

The decoder will get passed a binary string and will do the actual decoding of the particular message, and place the neccessary information into some structure that is passed to it.

##### The Parser

The parser has a few different jobs. It will:

1. Hold the current state of the system
2. Get the next binary message in the log file, and pass it to the decoder (which will then update the state variable).
3. Write the current state to some variable ready to written to output (this could be on every update, when some particular variable is read from the logfile, or every so and so bytes. It's up the programmer)

For those are more visual, this diagram roughly explains what it is happening -

{<1>}![](/content/images/2014/Jul/atp_logic_flow.png)

This design covers my 'be modifiable' goal quite well - any extra codes that might get added are handled by updating the decoder object accordingly, without needing to touch the main method or parser object.

#### Python and Performance

Python is a great programming language. It is similar in syntax and structure to psuedo-code, that beloved language of whiteboards the world over. There are drawbacks to this. Whereas a language like C exposes a small but flexible amount of syntax close to the bare metal, Python has a many different ways to do something within easy access of the beginner with varying effects on speed and code clarity. So what are some pitfalls the beginner Python programmer might make when designing something to parse log files?

##### If-Then-Else Chaining

The If-this-Do-This-Else-Do-This is the work horse of programming world. On a first pass of building a prototype parsing system it will be incredibly temping to just keeping adding another else-if block everytime you want to add a new decoding routine. This creates two main issues. The resulting source code is going to become incredibly long and ugly. The second is the execution cost. When falling to the last else-if block  if will be required to execute every testing condition in the if statement before it reaches the last statement. This is terribly time consuming if your log file is of reasonable complexity, and even if you knew statistically what the most common codes were and structured the chain accordingly, I doubt the performance would be much better. At any rate, if such probabilities changed it would result in error-prone re-ordering of code.

<pre><code>

if (code == something)

else if (code == somethingelse)

else if (code == somethingelse2)

</code></pre>

##### Hashing Functions

An alternative to these chains of if-else statements is the use of function arrays or hashed functions. In doing so, we assign a a function to the index of an array or Python dictionary (the same functionality can be obtained in C/C++ through the use of function pointers). We then call the function using the id of the message as the index to the array. This has significant beneftis. Firstly, it is much faster then if-else chaining and guarantees we branch to the decoding routine with the same speed as a any other decoding routine. Profiling the different methods proves this. For the test set of log files, this processing step went from a sluggish 2.19 seconds per call to a much faster 0.245 seconds per call. Additionally, the resulting code is clean; it it small, and encourages the developer to seperate the decoding logic from the rest of the code. 

<pre><code>

# A quick example demonstrating hashed functions

state = {}

def decode1(msg,state):
	# do some decoding, chuck it in the state
    
def decode2(msg,state):
	# do some other decoding, chuck in the state

decoder[1] = decode1
decoder[2] = decode2

for msg in msgs:
	decoder[msg.id]
    
</code></pre>

##### Parallel Processing

Up to now, I've really only been talking about processing one log file at a time, in the context of a single processor system. Today multiprocessor systems are the norm rather than the exception. Most user-commodity hardware has anywhere between 2-8 CPU's, so it would be appear to make use of all the resources available to us. Generally, when we discuss concurrent execution, there are two models available to use - threading and multiprocessing. The latter is what we shall investigate here.

Concurrency with multiprogramming is a difficult concept, but fortunately for our purposes we develop a useful rule of thumb from the functional programming world. Imagine two types of functions; those that mutate some outside state and those that don't, the latter being termed 'pure' functions. As pure functions do not mutate some global state, they have no  side effects and in most instances, sets of them be parallel computed.

Google published a paper in 2006 detail a method of parallel computation they called MapReduce, which is once again borrowed from the functional programming paradigm. In this, we *map* several objects to an execution content, and output the results of these subproblems. These subproblems are then *reduced* to the final solution. Essentially, you can think of this as breaking a problem into subproblems which are then solved and combined to give the answer to the super-problem. If your problem breaks up nicely like this, it may be a good candidate for MapReduce. In a way, this is kind of similar to dynamic programming, but differs in that more independent the subproblems are from each other, the better the gains will be.

Moving back to the task at hand, we'll treat each log file as one unit of execution independent from all others, and combine the output of each processed log file into the end-solution. As the log files in my case are essentially independent of each, this works well enough. We realise this by creating a seperate parser object for each log file, which the structure of the program accomodates nicely.

Code for this will follow the rough idiom below.

<pre><code>
from multiprocessing import Pool

def create_parser(log):
	# create your parser object that contains the log
    # file here

def process_logs(parser):
	output = parser.readlog()
    return output
    
parser_list = [for log in logs create_parser(log)]

pool = Pool(processes=4)

log_outputs = pool.map(process_logs, parser_list, chunksize=4)
	
</code></pre>

The Pool(processes=x) tells multiprocessing to create x number of contexts to process our logs. Generally, the best number to pick is going to be around your processor size. The below table demonstrates the results for different pool sizes. The computer used for this example is an i7 quad core.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
.tg .tg-e3zv{font-weight:bold}
</style>
<table class="tg">
  <tr>
    <th class="tg-e3zv">Processing Pools</th>
    <th class="tg-e3zv">Execution Time (seconds)</th>
  </tr>
  <tr>
    <td class="tg-031e">1</td>
    <td class="tg-031e">20.85</td>
  </tr>
  <tr>
    <td class="tg-031e">2</td>
    <td class="tg-031e">11.017</td>
  </tr>
  <tr>
    <td class="tg-031e">3</td>
    <td class="tg-031e">10.304</td>
  </tr>
  <tr>
    <td class="tg-031e">4</td>
    <td class="tg-031e">12.44</td>
  </tr>
  <tr>
    <td class="tg-031e">6</td>
    <td class="tg-031e">13.48</td>
  </tr>
  <tr>
    <td class="tg-031e">8</td>
    <td class="tg-031e">16.13</td>
  </tr>
</table>

##### Context Switching Overhead vs Memory Usage

It should be noted that by using a mapping function for parallel computing you will incur some overhead as objects and functions are marshalled to the correct processing context. You should be expected to pay this cost everytime you set up the mapping stage. To pay up as little as possible, you could bring everything into main memory and call the map function, but you may not have the RAM to afford this - you will also want to avoid swapping objects in and out of main memory. The options have to be carefully weighed; determine how much memory you can afford on objects at any one time, and size your processing 'chunks' accordingly. Alternatively, hold off onto reading into main memory until inside the mapping stage. The below tables demonstrates the effects of different chunk sizes on processing speed.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
</style>
<table class="tg">
  <tr>
    <th class="tg-031e">Chunk Size</th>
    <th class="tg-031e">Execution Time (seconds)</th>
  </tr>
  <tr>
    <td class="tg-031e">36</td>
    <td class="tg-031e">7.57</td>
  </tr>
  <tr>
    <td class="tg-031e">30</td>
    <td class="tg-031e">10.59</td>
  </tr>
  <tr>
    <td class="tg-031e">24</td>
    <td class="tg-031e">10.26</td>
  </tr>
  <tr>
    <td class="tg-031e">18</td>
    <td class="tg-031e">10.22</td>
  </tr>
  <tr>
    <td class="tg-031e">12</td>
    <td class="tg-031e">13.72</td>
  </tr>
</table>

##### Multiprocessing in Python - A caveat

It wouldn't be a discussion on multiprogramming in Python without mentioning the GIL - the Global Interpreter Lock. In Python, You can only execute one thread at any given time per interpreter. The multiprogramming library in Python gets around this by (and I'm simplifying this explanation a bit) serializing the required objects, launching a new interpreter for each process, and then rebuilding the serialized objects/functions in the new interpreters. If you don't understand all of this, that's fine, but a consequence of this is that if an object cannot be serialized (or 'pickled' in Python terminology), then the multprocessing library will fail. I discovered the hard way that functions assigned to an array cannot be pickled with Python's default serialization library. You can get around this by keeping functions declared inside a class file, but outside the class definition. This way the functions themselves don't require pickling because they are automatically visible to anything that imports the class file. This is a little messy in terms of design patterns and proper encapsulation, but it does get the job done.

##### Sorting the input set

In the event you do settle on several map-reduce steps, you should consider that any mapping step will not finish until all processes have finished. This means that you handle a small file and large file at the same time, the small file will finish before the large file. It will then block that processor (provided there isn't any more processing to do in that mapping step). If lightnining strikes and all your large files end up queued on one processor and all the small files on the other, you might find you waste a lot of time waiting for the large files to finish processing. To that end, you should consider sorting your files so that they are spread evenly across your processors. On my test set of files, sorting the files cut processing time down by roughly half a second.Nothing amazing, but still something to consider.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
</style>
<table class="tg">
  <tr>
    <th class="tg-031e">Processing Pools</th>
    <th class="tg-031e">Execution Time (seconds)</th>
    <th class="tg-031e">Sorted Execution Time (seconds)</th>
  </tr>
  <tr>
    <td class="tg-031e">1</td>
    <td class="tg-031e">20.85</td>
    <td class="tg-031e">20.85</td>
  </tr>
  <tr>
    <td class="tg-031e">2</td>
    <td class="tg-031e">11.017</td>
    <td class="tg-031e">10.86</td>
  </tr>
  <tr>
    <td class="tg-031e">3</td>
    <td class="tg-031e">10.304</td>
    <td class="tg-031e">10.06</td>
  </tr>
  <tr>
    <td class="tg-031e">4</td>
    <td class="tg-031e">12.44</td>
    <td class="tg-031e">12.25</td>
  </tr>
  <tr>
    <td class="tg-031e">6</td>
    <td class="tg-031e">13.48</td>
    <td class="tg-031e">13.18</td>
  </tr>
  <tr>
    <td class="tg-031e">8</td>
    <td class="tg-031e">16.13</td>
    <td class="tg-031e">15.58</td>
  </tr>
</table>
	
##### Optimisation with Cython

One big thing Python has going for it is it's application of dynamic typing, which simplifies writing code. Unfortunately, dynamic typing incurs an overhead, as you relying on the interpreter to infer the meaning of every variable you use. Static typing is always much faster. You can gain some of these benefits by utilising Cython. Cython enables Python modules to statically type variables, and it can compile Python modules down to C. This can buy the programmer some small performance boosts with very little effort on their part. Using Cython, I was able to get the processing speed down to roughly 6 seconds, and by sizing the input sets approriately, was able to get the processing speed down to 3.7 seconds.

#### Profiling Summary

The below tables illustrate the speed improvements. Essentially, by starting with a naive implementation we manage to process the log files in roughly 23 seconds. A better implementation utilising the features discussed drops this number down to 3.7 seconds - this is roughly a 621% improvement in speed!

That is much better in terms of my 'be fast' requirement.

#### A quick note on other languages

To be honest, there are other languages that will likely be a lot more efficient than Python for this particular task, both in size and speed. I have translated the Python program into C, and it is certainly a lot faster, and that is before adding in multiprocessing capability. However, with C/C++ comes a bit more boilerplate and verbosity, so it is up to the author to decide whether the trade offs are worth it.

#### Summary

I hope this helps someone out in their attempts to process files, hopefully you learnt some new tricks that can be applied to your own efforts. When it comes to speed, remember the following tips to increase speed:

1. Reduce unneccesary branching - can you use hashed functions?
2. Make use of the multiprocessing library, if appropriate.
3. Chunk sizes can have a reasonable effect on processing speed.
4. Choose an appropriate size for the processing pool.
5. Try compiling your module with Cython.

'til next time.



