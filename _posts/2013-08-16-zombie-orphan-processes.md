---
layout: post
title: Zombie & Orphan process
date: 2016-10-11
excerpt: "A more practical guide to understanding Zombie & Orphan processes, that arise due to improper termination of parent and child processes, also explaining the way UNIX deals with such cases, and the correct way to program to avoid such cases."
tags: [unix, process, C, OS]
comments: true
---

**Zombie**, Let's run,,they are gonna eat us.
<figure>
  <a href="http://i.imgur.com/vwND4Ca.jpg"><img src="http://i.imgur.com/vwND4Ca.jpg"></a>
</figure>
Hey!!,,,wait,,wait,,wait,, This is not a post about Resident Evil or Walking Dead containing images about vicious monsters eating raw flesh, leaving destruction on it's path. It's just a terminology used in UNIX to categorize certain processes that we are gonna understand. Also, don't relate 'Orphan' to any movie. This post ain't any fun unless you are an UNIX junkie. \\
A natural question that comes after we fork processes is that what happens after process termination? As an example, if the child processes terminates first and it returns it's exit status as in [Exit/Return]({% post_url 2016-09-15-Exit %}) system calls, but the parent hasn't reached the point where it accepts the status. Where does the status go? \\
The opposite case is more strange, what if the Parent terminates first?, What happens to the child processes. Who is their parent? (NOTE: They can't be perfectly orphan, as one might execute [getppid()]({% post_url 2016-09-15-Fork %}) call. What shall it return then?
Let's dive a little more deep with actual snippets of code:-

### Child terminates First
{% highlight c %}
#include <unistd.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <stdio.h>
int main()
{
  int pid=fork(),status;
  if (pid < 0) {
    //Fork call failed.
  }
  else if (pid == 0) {
    printf ("FROM CHILD\nChild ID: %d, of Parent: %d\n", getpid(), getppid());
    exit(5);              //Exit status is 5
  }
  else {
    sleep(100);
    wait(&status);        //Reap Child, but after 100 secs.
  }
  return 0;
}
{% endhighlight %}
The corresponding output is:-
{% highlight io %}
FROM CHILD
Child ID: 5571, of Parent: 5570
{% endhighlight %}
During the time the parent was sleeping (for a whopping 100 secs), it gave me enough time to execute "ps -a" to check out the active processes, and the results were as:-
{% highlight io %}
 PID TTY          TIME CMD
4094 pts/21   00:00:19 bundle
5570 pts/22   00:00:00 a.out
5571 pts/22   00:00:00 a.out <defunct>
5582 pts/23   00:00:00 ps
{% endhighlight %}
We can se an odd 'a.out defunct' entry marked with PID: 5571, that exactly matches the process id of the child process. But, we know that by the time parent kept executing, the child process has already terminated. Then, why is the entry still present in the process table (that's why displayed in 'ps -a' call)? \\
Well, simply speaking, UNIX doesn't remove a process entry and some other info until it's reaped by it's parent. Hence, in this case, until the parent reaches the 'wait' statement, the process entry corresponding to the child process exists. This state that the child process enters is what is known as a _Zombie/Defunct_ process. \\
This zombie process is ridden off of it's data and all the memory allocated. i.e, it's resources can be reallocated to other processes. However, the process ID and the exit status/ return code still persists to be fetched by the parent (reaping of child). \\
**NOTE:** At a point of time, all processes beconme zombie. (Before it gets reaped, there is an amount of time when it remains DEFUNCT). \\
Now, having understood what is a zombie in UNIX terms, let's look at the other case:-

### Parent Terminates First
{% highlight c %}
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <stdio.h>
int main()
{
  int pid=fork(),status;
  if (pid < 0) {
    //Fork call failed.
  }
  else if (pid == 0) {
    printf ("FROM CHILD\nProcess ID: %d, of Parent: %d\n", getpid(), getppid());
    sleep(100);
    printf ("FROM CHILD\nProcess ID of Parent: %d\n", getppid());
    exit(5);        //Exit status is 5
  }
  else {
    //Do parent stuff that doesn't take up much time- "DON'T REAP. i.e BLOCK"
  }
  return 0;
}
{% endhighlight %}

In this case, the parent executes prior to child termination, since there is no blocking wait call. Thus, what I ought to say is that the child process remains parent-less. i.e. an Orphan. It keeps on executing until trouble comes at the point where it exits. Who should fetch the exit code for this child. It's biological parent is dead, hence, UNIX deals with this situation by getting the status reaped by the **'init'** process. The whole modus operandi of this operation occurs as follows:- \\
Once a process terminates, the OS goes through all the processes in it's process table and assigns the init process as the parent to all those processes that are the child of the terminating process. In simpler ternms, if a parent is about to terminate, it iterates through all process to check whether it is a child of the terminating parent or not. if it is, then init is assigned as the new parent. *The init process has the process id 1.* \\
Hence, when the child finally terminates, it is immediately reaped by the init process. \\
**NOTE:** Do process adopted by init (due to biological parent terminating prior to itself) become zombies? \\
The answer is 'NO'. As soon as an init adopted process terminates, it is immediately reaped. In this manner, init prevents the system from getting clogged by zombies. Ultimately, all processes are gonna get reaped by the init, freeing the process id. \\
The output speaks for the same:-

{% highlight io %}
FROM CHILD
Process ID: 4248, of Parent: 4247

FROM CHILD
Process ID of Parent: 1359
{% endhighlight %}
A similar behaviour can be seen here. After a painful waiting of 100 secs, before exiting, we observe that getppid() in child returns 1359. (But it should return 1, right?). We execute 'ps -A \| grep 1359', to get the following output:-
{% highlight io %}
1359 ?        00:00:00 upstart
{% endhighlight %}
As stated in the [UPSTART](http://upstart.ubuntu.com/cookbook/) cookbook, upstart process is the "init" replacement for the traditional UNIX "System V" init system. I don't know what the hell System V means but it is quite clear that in modern systems such as Ubuntu (like mine), upstart is the front for init, and it performs all tasks that init performs. One can think of upstart as comprising init, although the truth of such a theory is out of scope of this post. \\
So, we pretty much have a clear idea of a zombie and an init process.
### Avoiding Zombie & Orphan situations
<figure>
  <a href="http://i.imgur.com/ihy2g5g.jpg"><img src="http://i.imgur.com/ihy2g5g.jpg"></a>
</figure>
Zombie processes are undesirable as they are a resource leak, and hold up process id that could be otherwise be allocated to some other process. The [wait/waitpid]({% post_url 2016-10-12-Wait %}) system calls prevent this situation. The wait call as already shown will not let a parent terminate before it reaps up it's child, ensuring that zombies or orphans are created. Since, it is a blocking call (if wait is used), it doesn't let the parent terminate until the child is reaped. (So no orphan is created). Therefore, the proper way to program is to use wait calls so that ulimately all child processes are reaped, and there is no situation of zombie or orphan.
