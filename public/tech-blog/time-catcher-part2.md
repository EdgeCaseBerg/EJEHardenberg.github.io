So the work on my [newest project] is at a point where I felt comfortable enough for an alpha release. It's pretty simple. The basics are done. You can start tasks, finish them, switch between them and view information for one or all tasks.

The best part? In my opinion it's a combination of the following:
 

- It's all in C.
- It's memory leak free and has no unintialized value runtime errors.
- The Bash Completion of it's commands
- It's only 1600 lines of code. Refactored from ~2600 just today.
- It doesn't consume your process space by having a running timer


So let's see. What were some cool things I probably want to touch on. Oh right, how about the way it tracks time? 

The algorithm is actually pretty simple. It depends on a couple of things. First off, each task has a sequence file and an information file. These are stored in the .tc directory the program creates in the home directory. The information files are only important for the view --all command. But the sequence files are the heart of a task. In each file is a simple format:


<pre> 
  &lt;seq num&gt; &lt;state&gt; &lt;epoch time&gt;  
  &lt;seq num&gt; &lt;state&gt; &lt;epoch time&gt;  
</pre>


 So you might see where this is going. A task can be in 1 of 5 possible states. Only 3 of which are ever recorded in the sequence file: Started, paused, and finished (The other two states are for error handling).

 The algorithm to read information is simple:

<pre>
	while( read 3 fields stated above) 
		if seqNum = 0
			startTime = seqTime;
		else 
			if( priorState == STARTED and (state = PAUSED or state = TC_FINISHED) ) 
				runningTime = runningTime + (seqTime - priorTime);
			
		priorTime = seqTime;
		priorState = state;
			
	if runningTime = 0 and state == STARTED 
		runningTime =  time(0) - startTime;

 </pre>

Basically all we're doing is computing the time between when the task started being in progress and when it was finished. Taking into affect that if it is current in progress we'll use the current time as the ending time (not shown here). One little tricky bit is that the running time is affected at the very end after the loop if the runningTime is zero and we're in a started state. We do this because of the special case of when a task is first started and there's only a single entry in the sequence file.

There's far more interesting things going on in the program besides this little algorithm. But it seems like since it's a time tracking program it's appropriate to mention it at the very least. Some more interesting tidbits are:

- This program calls it's own main.

What's that? Did your head just explode? Did your pedentic sense of justice to the C++ C99 standard come raging forth? Reality check. I'm compiling with cc and ansi pedantic. The C (C! not C++! PURE C!) standard doesn't say I can't call my own main if I want to. And guess what. There's no loss (on my machine at least) of the current stack or anything. I return from the call to main as you'd expect and carry on my way to free the memory allocated in the calling function. If you're interested, the recursive call is in the tc-start.c file. 

-  The program creates a directory in your home directory

How does it do this? The wonderful world of environmental variables! Believe it or not, if you give a path to fopen with a tilde... it's not going to like it. Why? Because the tilde is really a shell expansion for your home directory. So you either have to grab the environment or use wordexp to do word expansion on the tilde itself. I do both. If you're curious how it's done, check out tc-directory.c

- It runs through the tasks directory within the .tc directory and finds the filename's using the opendir commands. 

It was my first time getting a chance to play with the dirent library so it was a lot fun. The code's in tc-view.c around where the --all command is parsed out. 

- Bash Completion

Ok, so this is just cool. I didn't know how to do completion for my own programs before and I found it it's not that hard using all the excellent tutorials around for it. There is [one in particular]  that everyone links to. Probably because you can easily modify the example scripts to get what you'd like out of it. Perhaps my favorite part of this whole process was this clever bash command:

 <pre> cat ~/.tc/indexes/*.index | cut -d ' ' -f 2- | uniq | grep -v  [[:space:]]*8 | rev | cut -d ' ' -f 2- | rev` </pre>

This is part of the completion script, specifically, this grabs the list of tasks that aren't in progress. How? Easy. We cat the index files to retrieve the task names and the states of them. Then we grab the 2nd field onward with cut (using space as our delimiter). Do an inverse grep for the state that corresponds to the in progress state. Then we reverse the string (wait why you might ask). Then we remove the first field of the reversed string (which happens to be the state). Then we flip the string again and we're only left with the task name itself. Cool right?


This has been a very fun project to do, and I think I'll probably add a few more commands before I'm done with it. But while I do, I'll definitely be using it to keep track of how long I spend on it! (Talk about dog fooding)


[newest project]:https://github.com/EdgeCaseBerg/timecatcher
[one in particular]:http://www.debian-administration.org/articles/316 