# Zen and the Art of System Monitoring

System monitoring is an essential but often-overlooked part of production software deployment - it's as critical as security, but rarely given the same attention.  

By overlooked, we don't necessarily mean ignored - even novice operations folks know that monitoring is needed, and most environments do have some basic alarms in place (even if it's the sales department screaming "The website's down!" - effective, but perhaps not optimal.)

There isn't (yet) a standard methodology for monitoring, and there really ought to be.

So let's change this.

At Scalyr, we're a group of experienced operations engineers who - despite what you may think of the craft - don't love being yelled at by the sales department, and certainly don't love being woken up at 3 am by our own alerts.  We've learned a lot from our years on the front lines: Monitoring is critical to your application's health, and effective monitoring is critical to your own.

In this guide, we're going to explore exactly what we mean by "effective monitoring" and offer a practical framework of best practices that you can use to structure your monitoring and alerts. You'll learn how to:

 * Minimize production issues;
 * Avoid false alarms; and
 * Do all this with as little effort as possible.

## Challenges

Three historical challenges to effective monitoring are a false sense of security, a lack of cohesive tools, and the wrong mindset.  These are related to one another. 

The false sense of security looks like this:  "I care if my site goes down.  A ping test will let me know if my site goes down, so all I need is a ping test!" (A ping test is a simple monitor that pings your site and alerts if it doesn't respond.)

But a ping test alone won't keep you safe.  You'll only know about problems once they've spilled over and caused a crash.  And you'll miss subtler problems (say, for example, your site stays up but hackers replace all of your images with pictures of tiny kittens playing tiny violins.)

Similarly, most hosting providers will give you some default graphs and alerts on your dashboard:  Basic traffic and CPU metrics, an email alert if the server fails, and maybe some raw log access.  And you'll be forgiven if you think "Ah ha - now they've got me covered."  These defaults are great and useful.  But not enough!  If your web server instance runs out of memory, starts swapping to disk, and site performance drops to nil, you might not notice for a long time.

Even if you combine a ping monitor, your hosting provider's dashboards and alerts, and maybe some additional outside services - log management or exception monitoring - your mental monitoring model (say that three times fast) is spread all around.  What happens when a problem hits and all three systems start peppering you with alerts?

This leads to the third major challenge - and perhaps the most important:  The wrong mindset.  "This monitoring thing is a pain."  This isn't dissimilar to a novice developer's attitude toward testing (an attitude which often evolves quickly as the application grows, changes become more frequent and the value of good tests becomes apparent.)  Monitoring isn't spiritually dissimilar from testing: tests help identify problems before production; monitoring helps identify problems in production.

## Our Tenets

We address these challenges with a cohesive approach to monitoring.  This approach will help ensure that all your bases are covered, and put you in a mindset of monitoring confidence and security.

Our core tenets:

  1.  **Identify as many problems as possible**.  Good monitoring doesn't just tell you when your site is completely down.
  2.  **Identify problems as early as possible**.  The sooner you know about an issue, the better chance you'll have to address it before it affects users.  Longer lead times are your friend.
  3.  **Generate as few false alarms as possible**.  False alarms (aka false positives) can lead to "alert fatigue" - where you get so used to seeing a noisy alert that it starts to carry less psychological weight.  This alert fatigue can, paradoxically, lead to increased downtime.  Do not, in other words, allow your monitoring system to cry wolf.
  4.  **Do it all with as little work as possible.**  We are engineers, after all.

Apply this philosophy as you would approach any engineering problem - deconstruct the problem into smaller pieces (which we'll call "monitoring layers"), and use the right tools for each piece. Concretely, we'll deconstruct monitoring into four steps:

  1.  **Monitor Potential Bad Things** (Alert before they happen);
  2.  **Monitor Actual Bad Things** (Alert when they do happen, which is, unfortunately, inevitable);
  3.  **Monitor Good Things** (Alert when they stop happening); and
  4.  **Tune and Continuously Improve** (Iterate!)

Let's dive in...

## The Tools

Our core components are alerts, graphs, and logs.  These work together to help you identify and troubleshoot issues easily and quickly.

  - **Alerts** notify you when things change beyond specified thresholds.  Good, bad, or otherwise, an alert is the beginning of an analysis sequence that lets you know it's time to dig deeper into your graphs and logs.  Setting the proper threshold is key to successful alerting (we address this in depth in our companion guide, appropriately named [How To Set Alerts](how-to-set-alerts.md).)  Used properly, alerts can warn you well in advance of a disaster and enable you to fix problems on a convenient schedule and without affecting users.

  - **Graphs** are the visual representations of your data.  Graphs help you see trends that might not be obvious in raw form.  We humans have remarkably good visual-pattern-recognition systems, and graphs enable us to identify anomalous data quickly.  They also look super cool on a huge flatscreen TV dashboard mounted in the middle of your office.  

  - **Logs** hold your raw operational data, recording events in the operating system or application.  Whereas graphs summarize an
  overall trend (potentially hiding important details), logs will always have the finest level of detail and be the "base source of truth."

## <a name="layers"></a> Where to Monitor

To best deploy our tools, let's think of our system as a multi-layer stack, with your application on top, and the components and services that support it forming layers below.

By intelligently monitoring the right metrics in each layer, we'll get a complete view of the system and maximize monitoring coverage.

In a production environment, there will likely be multiple systems we want to monitor (e.g. the web server, the application server, and the database server), so plan to apply this full-stack approach to **each** system in your deployment.  There will be some overlap -- your web server and application server may share a physical machine -- so don't feel the need to follow this too religiously.  You can consolidate as you move down the stack.

### Layer 0:  The Application

The first layer of metrics we care about are those generated from, and specific to, the application itself.  Examples include a web server's active connections, an application server's worker health, or a database's I/O utilization.  We're specifically **not** looking at process- or system-level metrics like CPU usage or network bandwidth.  Those come lower down in the stack.

### Layer 1:  The Process

While the specifics of the process model are slightly different between UNIX and Windows, the concept is the same - a process is the running instance of an application within the operating system.  An application may have multiple processes running at any given time, and there can be dozens to hundreds of processes running on a server.

The metrics we're interested in here are those that report on process state ("created", "running", "terminated"), uptime, and consumed resources (CPU, memory, disk and network.)

### Layer 2:  The Server

One way of looking at a server is simply as a container for your processes.  At this layer, we care about the health and resource usage of the overall system.  This becomes particularly important if that system hosts multiple applications (either yours or other users') since their behavior can have an impact on your target application's performance.

Server metrics include system-wide resource usage data (CPU, memory, disk and network usage), summary metrics (total # of processes, load average, socket state and availability) and hardware state and health (disk health, memory health, physical-port access and use, CPU temperature, and fan speed.)

### Layer 3:  The Hosting Provider

Your servers live within your hosting provider.  Depending on the provider, available metrics can range from broad ("we're up" or "we're down") to latency and availability by physical datacenter and availability zone.  

Also included in this layer are the associated hosted services that support your application.  These may include databases, load balancers, and other "cloud" services your application implicitly or explicitly depends on.

As more and more critical services move away from local servers and into cloud-based services, monitoring the status of those services becomes increasingly important, particularly when the health of those services is out of your immediate control.

When lower-level services fail, many higher-level alerts will trigger (and loudly.)  But for troubleshooting purposes, you shouldn't skip setting alerts on the lower levels.  The extra alert telling you that your hosting provider is down will save you steps in the analysis stage.

### Bonus Layer!  Layer 4:  External Dependencies

Very often overlooked but absolutely critical.  Several are common to nearly all production web services, and failures here can lead to unexpected (and painfully long) downtime. Common dependencies:

  * **Domain Names** - DNS renewal dates can creep up and cause severe headaches if forgotten ([just ask Microsoft](http://news.cnet.com/2100-1023-234907.html)).  Mark your calendars!
  * **SSL certificates** - these also expire, with consequences almost as severe as DNS, and certificate providers are not as proactive with expiration warnings as DNS providers are.

### Layer 5:  The User

Finally, we get to the last (but certainly not least) layer in our stack - the user.  This is who it's all about anyway, right?  

At the user layer, we're interested in the behavior of the full stack as a user would experience it (similar to an integration test in TDD terms.)  In practice, this can be as simple as an external HTTP monitor that looks for a 200 OK response from an API endpoint, or a series of automated browser tests that check a sequence of pages for specific responses.

This can also include wider-ranged metrics that happen to roll up a number of lower-level behaviors and implicitly test them.  At Google, for example, internal lore held that the most important alert in the entire operations system measured “advertising dollars earned per second.”  This is a great integration metric to watch, because a change could point to anything from a data center connectivity problem to a code bug to mistuning in the AdWords placement algorithms.

## Monitoring Strategies

For each layer of the stack, you can apply four core monitoring strategies:

### 1.  Monitor Potential Bad Things

Most problems are detectable in advance if you look carefully enough.  Since one of our stated goals is to do this all with "as little work as possible" - and firefighting is a lot more work than casual maintenance - averting problems is a good place to start.

Many production incidents have as their root cause “something that filled up.”  In 2010, Foursquare suffered a widely publicized 11-hour outage because one shard of their MongoDB database outgrew the available memory.  “Filling up” can take many forms: CPU overload, network saturation, queue overflows, disk exhaustion, worker processes falling behind, or API rate limits being exceeded.

These failures rarely come out of nowhere. The effect may appear very suddenly, but the cause creeps up gradually. This gives you time to spot the problem before it reaches a tipping point.

**Action Items for Monitoring Potential Bad Things**

  * Identify the resources in each layer of your stack that can fill-up / overflow;
  * Understand the tipping point of each resource.  A "full" CPU might not be 100% - a process could show meaningful degradation at a level below that;
  * Understand the consequences of an overflow.  CPU overload might slow things down a bit; RAM overload can take you down hard. The more serious the consequences, the more aggressive your alert should be; and
  * Set alerts that will fire before the tipping point is reached -- say, when a disk is 90% full.

Any form of storage can fill up. Monitor unused memory on each server, free space in each garbage-collected heap, available space on each disk volume, and slack in each fixed-size buffer, queue, or storage pool.

Any throughput-based resource can become oversubscribed.  Monitor CPU load, network load, and I/O operations. If you’re using rate limiters anywhere, monitor those as well.

A background thread or fixed-size thread pool can choke even if the server has CPU to spare. Use variable-sized pools, or monitor the thread’s workload.

Lock contention can also cause a tipping point on an otherwise lightly loaded server. It’s difficult to monitor lock usage directly, but you can try to monitor something related. Internally at Scalyr, we generate a log message each time we acquire or release certain critical locks, allowing our log analyzer to compute the aggregate time the lock is held.

### 2. Monitor Actual Bad Things 

Despite our best efforts, things will break.  

When they do, your best defense is to be prepared and respond quickly.  To respond quickly, you need to know about the problem as soon as it occurs.

**Action Items for Monitoring Actual Bad Things**

  * Identify the resources in each layer of your stack that can stop functioning correctly;
  * Understand the effects that each potential failure can have on the rest of the system; 
  * Make sure alerts are set to trigger when those resources fail, and that those triggers reach you with an appropriate level of urgency; and
  * Have tools in place to move quickly from alert to graph to log (as necessary) in order to identify the root cause of a failure.

It's important to ensure thorough alert coverage in both breadth and depth.  In [a famous incident in 2012](http://www.paritynews.com/2012/10/13/432/oracle-website-down-says-hello-world), for over an hour, Oracle.com's home page said simply "Hello, World".  Presumably a monitor that tested for a 200 OK response from the home page would not have triggered an alert in that failure scenario, so it's important to understand the scope of potential failures and cover them accordingly.

**Exception Monitoring**

Exception monitoring is worth special mention here.  Exceptions come in several flavors -- most commonly as application exceptions that wind up as a stack trace in your application log.

Application exceptions provide some of the best monitoring bang for your buck, since they generally correlate with an application bug, and can surface edge-case issues that are hard to spot in the high level statistics used by the rest of your monitoring. A bonus is that, by closely watching exceptions, you get to impress users by proactively reaching out to them before they even complain about an error.

(If you've ever gotten an email from a company along the lines of - "we saw you were having trouble with...." - that's a good sign they're closely monitoring exceptions.  It's a neat technique.)

### 3.  Monitor Good Things (When They Stop Happening)

This idea is deceptively simple:  alert when normal, desirable user actions suddenly drop.  It's a surprisingly effective way to increase alerting coverage (the percentage of problems for which you’re promptly notified), while minimizing false alarms.

Even in the best-intentioned setups, there will be gaps in the "potential bad things" and "actual bad things" strategies outlined above.  It's hard to anticipate every possible point of failure, and there are cases where operations that simply fail to complete won't trigger an error (but won't trigger success either.)  

The specifics will depend on your application, but you might look for a dropoff in page loads or invocations of important actions.  It’s a good idea to cover a variety of actions. For instance, if you were in charge of operations for Twitter, you might start by monitoring the rate of new tweets, replies, clickthroughs on search results, accounts created, and successful logins.  Think about each important subsystem you’re running, and make sure that you’re monitoring at least one user action which depends on that subsystem.

**Action Items for Monitoring Good Things**

  * Identify the key user actions and system behaviors that indicate a "green" system status; and
  * Set alerts to notify you when these values fall out of line.

It’s often best to look for a sudden drop, rather than comparing to a fixed threshold. You might alert if the rate of events over the last 5 minutes is 30% lower than the average over the preceding half hour. This avoids false positives or negatives due to normal variations in usage.

### 4. Tuning and Continuous Improvement

Monitoring is an iterative process - you're not expected to get it all right the very first time around.  

After each incident, ask yourself how you could have seen it coming.  Sometimes, the hints are already there in your log and you just need to add an alert.  Other times, you might need to make a code change to log additional information.

Start with tight thresholds and broad, aggressive alerts and iteratively narrow them to eliminate false alarms.

Pay particular attention to noisy alerts (false positives) and work on quieting them.  Noisy alerts play on a feature (or bug, depending on your perspective) in the human brain that allows us to grow tolerant of repetition.

If a particularly noisy alert keeps at it, you'll start to assign less value to it (and other alerts in general), and your reaction will be muted.

In the early, pre-tuned stages of a new monitoring setup, it's important to pay close attention to your alerts, graphs and logs (that is to say, even closer attention than you otherwise might.)  You'll want to keep an eye out for missed issues, noisy alerts, and other areas for refinement.  Watch, iterate, watch some more, and in relatively short order you'll have a setup in which you have total confidence.

## Further Reading

If you found this guide useful, be sure to look at some of the other material on best practices in the [Scalyr Community](http://www.scalyr.com/community).  We're building a library of specific instructions for applying these ideas to a wide variety of applictations and systems.

## Summary

Properly monitoring your systems and setting up thorough alert coverage requires effort.  But with the right tools, a good understanding of your application stack, and a broad approach that covers problems before, during and after they occur, we expect you'll be monitoring very effectively...

...and sleeping very well.