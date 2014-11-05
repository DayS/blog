---
layout: post
title:  "Too many open files in system"
date:   2014-11-06 14:00:00
categories: java
tags: java wtf
image: /assets/article_images/java-cover2.jpg
---
I'm currently working on a big Spring/Hibernate project with a complex administration back-end.
Everything was working fine until IntelliJ started crashing with no visible reasons. The only error I had was this weird message, just before my Mac also started going crazy :

> Error=21, Too many open files in system

{{ post.excerpt }}

# Investigation

A quick research reminds me that every OS limits the maximum number of opened files in system and per process. This security prevents processes to abusively consume resources.

[Almost](https://rtcamp.com/tutorials/linux/increase-open-files-limit/) [all results](https://stackoverflow.com/questions/20901518/ubuntu-too-many-open-files) [I could](https://stackoverflow.com/questions/26325851/intellij-too-many-files-open-error) [find](https://superuser.com/questions/433746/is-there-a-fix-for-the-too-many-open-files-in-system-error-on-os-x-10-7-1) suggested to tweak system configuration in order to increase this limit. To do so you can use these commands :

{% highlight bash %}
sysctl -w kern.maxfiles=20480 			# or whatever number you choose
sysctl -w kern.maxfilesperproc=18000 	# or whatever number you choose
ulimit -S -n 2048 						# or whatever number you choose
{% endhighlight %}

Even if this solution should have worked, I wouldn't resign myself to do that because of these main reasons :

* Limits are usually here for a good reason;
* The application have to be deployed on the client's servers and I don't want to force them to tweak their own systems because of this;
* I work on MacOS, servers will be on Unix and my client works on Windows. I don't have time to find and check tweak commands for all these OS.

# Debugging

During my research I found [an article](http://qupera.blogspot.fr/2012/07/solving-too-many-open-files-under-linux.html) describing how to measure the number of open files used by process with this simple command :

{% highlight bash %}
lsof -p $PROCESS_ID | wc -l
{% endhighlight %}

At this point I was wondering how to debug my issue.
Given this command and the others found in previous section, I came to write the following bash script.
**Here some explanations:** It starts by listing processes matching the given name pattern. Then, for each result it prints the associated opened files count. And if this value is greater than a given threshold (default is half the max open file set in system) the full list of open files is dump in a log file.

{% highlight bash %}
#!/bin/sh

# This script prints number of files opened for the given process pattern
# Arguments :
#	1. process pattern
#	2. opened files threshold before dumping list of opened files (default is half on system's maxfilesperproc)

PROCESS_PATTERN=$1

OPENED_FILES_THRESHOLD=`sysctl kern.maxfilesperproc | awk '{ print $2 / 2 }'`
if [ "$#" -gt 1 ]; then
	OPENED_FILES_THRESHOLD=$2
fi

# change IFS to support spaces in array
OLD_IFS=$IFS
IFS=$'\n'
PROCESS_LIST=(`ps aux | grep $PROCESS_PATTERN | sort`)

echo "Scanning opened files with process pattern $PROCESS_PATTERN and opened files threshold $OPENED_FILES_THRESHOLD"

for PROCESS in ${PROCESS_LIST[@]}; do
	PID=`echo $PROCESS | awk '{ print $2 }'`
	OPENED_FILES=`lsof -p $PID | wc -l`
	PROCES_NAME=`echo $PROCESS | awk '{ print ($11,$12,$13,$14) }'`

	echo "pid=$PID\topenedFiles=$OPENED_FILES\tprocess=$PROCES_NAME"

	if [ "$OPENED_FILES" -gt "$OPENED_FILES_THRESHOLD" ]; then
		DUMP_FILE="opened_files-$PID-$(date "+%Y.%m.%d-%H.%M.%S").log"
		lsof -p $PID &gt; "$DUMP_FILE"
		echo "Opened filed threshold reached. Dumping list in $DUMP_FILE..."
	fi
	# echo $PROCESS
done

# Restore IFS
IFS=$OLD_IFS
{% endhighlight %}

With this in my hands, all I had to do was to run this script (with the command `./trace_opened_files.sh IntelliJ`) right before and after reproducing the bug.
Well that's the theory... Because, once all file descriptors are used, it can't even be executed. In fact all the system starts acting weird..
After killing some useless processes and freed enough descriptors I was finally able to execute the script, which gave me something like this :

{% highlight console %}
Scanning opened files with process pattern IntelliJ and opened files threshold 5120
pid=18967	openedFiles=1726	process=/Applications/IntelliJ IDEA 13.app/Contents/MacOS/idea
pid=19009	openedFiles=9    	process=/Applications/IntelliJ IDEA 13.app/Contents/bin/fsnotifier
pid=19055	openedFiles=9970	process=/System/Library/Java/JavaVirtualMachines/1.6.0.jdk/Contents/Home/bin/java -d64 -Djava.awt.headless=true -Didea.version==13.1.5...
Opened filed threshold reached. Dumping list in opened_files-19055-2014.10.22-14.18.38.log...
{% endhighlight %}

We can easily see that the issue wasn't caused by IntelliJ but it was probably due to process `19055` (Tomcat instance) with **9970 opened files**. As the threshold was reached, the script dumped every opened files of this process in order to analyze what's going on. The result was clear enough :

{% highlight console %}
// ...
java    14030 dvilleneuve   98r  PSXSEM     0t0    kcms000036CE1AAC9000
java    14030 dvilleneuve   99r  PSXSEM     0t0    kcms000036CE1AAC9000
// ...  [about 9000 other lines like these] ...
java    14030 dvilleneuve 9144r  PSXSEM     0t0    kcms000036CE1AAC9000
{% endhighlight %}

It seems that my webapp somehow created almost **10.000 PSXSEM** files!
But what's PSXSEM ? One more time, Google was a good friend and the first result was a [StackOverlfow thread](http://stackoverflow.com/a/16677752/815737) of a guy having the exact same issue I was facing.

*By the way, PSXSEM states for POSIX Semaphore*.

# Solution

After all this research, I finally found the source of this issue.
It appears that `ImageIO.read(ImageInputStream)`  method is buggy on **Java 1.7.4** and creates tones of semaphore files without cleaning them. This was fixed on Java 8 and back-ported on 1.7.6.

In the end, **the solution was simply to update to Java 1.7.6+.** but the investigation was interesting :)