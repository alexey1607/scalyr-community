# Zen and the Art of Server Monitoring

Server and application monitoring is an essential but often-overlooked part of production software deployment - it's as critical as security, but rarely given the same attention.  

By overlooked, we don't necessarily mean ignored - even novice operations folks know that monitoring is needed, and most environments do have some basic alarms in place (even if it's the sales department screaming "The website's down!" - effective, but perhaps not optimal.)

There isn't (yet) a standard methodology for monitoring, and there really ought to be.

So let's change this.

At Scalyr, we're a group of experienced operations engineers who - despite what you may think of the craft - don't love being yelled at by the sales department, and certainly don't love being woken up at 3 am by our own alerts.  We've learned a lot from our years on the front lines (both in tending to other peoples' servers and, more recently, Scalyr's own servers):  Monitoring is critical to your application's health, and effective monitoring is critical to your own.

In this guide, we're going to explore exactly what we mean by "effective monitoring" and offer a framework in which to think about your own environments.  You'll learn to structure your monitoring and alerts in such a way that you're able to catch problems before they arise or, if you can't, identify and fix them as quickly and painlessly as possible.

## Challenges

Three historical challenges to effective monitoring are a false sense of security, a lack of cohesive tools, and the wrong mindset.  These are related to one another. 

The false sense of security looks like this:  "I care if my site goes down.  A ping test will let me know if my site goes down, so all I need is a ping test!" (A ping test is a simple monitor that pings your site and alterts accordingly - green = up, red = down.) 

But a ping test alone won't keep you safe.  You'll only know about problems once they've spilled over and caused a crash.  And you'll miss other problems (say, for example, if your site stays up but hackers replace all of your images with pictures of tiny kittens playing tiny violins.)

Similarly, most hosting providers will give you some default graphs and alerts on your dashboard:  Basic traffic and CPU metrics, an email alert if the server fails, and maybe some raw log access.  And you'll be forgiven if you think "Ah ha - now they've got me covered."  These defaults are great and useful.  But not enough!  If your web server instance runs out of memory, starts swapping to disk, and site performance drops to nil, you're likely to miss it.  And depending on the naggyness of your users, you could miss it for a while.

Even if you combine a ping monitor, your hosting provider's dashboards and alerts, and maybe some additional outside services - a log management service or an exception monitoring service - your tools (and mental monitoring model - say that three times fast) are now spread around.  What happens when they all start firing and sending you emails?

This leads to the third major challenge - and perhaps the most important:  The wrong mindset.  "This monitoring thing is a pain."  This isn't dissimilar to a novice developer's attitude toward testing (an attitude which often evolves - quickly - as the application grows, changes become more frequent and the value of good tests becomes apparent.)  Monitoring isn't spiritually dissimilar from testing -- tests help identify problems before production;  Monitoring helps identify problems in production.

## Our Prescription

We address these challenges by presenting to you our cohesive approach to monitoring.  This approach will give you the assurance that your bases are covered, won't spread you or your tools too thin, and will put you in a mindset of monitoring confidence and security.

At its core, our philosophy is as follows:

  1.  **Identify as many problems (and potential problems) as possible**.  The larger your monitoring surface, the better chance you have to spot issues that, left unchecked, could grow into bigger issues, and the lesser chance you'll miss those weird edge case problems.
  2.  **Identify problems (and potential problems) as early as possible**.  This is key - the sooner you know about something, the better chance you'll have to address it before it escalates.  Longer lead times are your friend. 
  3.  **Generate as few false alarms as possible**.  False alarms (aka false positives) can lead to "alert fatigue" - where you get so used to seeing a noisy alert that it starts to carry less psychological weight.  And this alert fatigue can - paradoxically - lead to more downtime if you accidentally disregard an actual event.  Do not, in other words, allow your monitoring system to cry wolf. 
  4.  **Do it all with as little work as possible.**  We are engineers, after all.

We apply this philosophy as one would approach any good engineering problem - by having the right tools, by deconstructing the problem domain into a series of layers ("the monitoring stack"), and by intelligently deploying our tools across the stack.

That approach results in these concrete monitoring steps:

  1.  **Monitor Potential Bad Things** (Alert before they happen);
  2.  **Monitor Actual Bad Things** (Alert when they do happen, which is, unfortunately, inevitable);
  3.  **Monitor Good Things** (Alert when they stop happening); and
  4.  **Tune and Continuously Improve** (Iterate!)

Let's dive in...

## The Tools

Our core components are alerts, graphs, and logs.  These work together to help you identify and troubleshoot issues easily and quickly.

  - **Alerts** notify you when things change beyond specified thresholds.  Good, bad, or otherwise, an alert is the beginning of an analysis sequence that lets you know it's time to dig deeper into your graphs and logs.  Setting the proper threshold is key to successful alerting (we address this in depth in our companion guide, appropriately named [How To Set Alerts](how-to-set-alerts.md).)  Used properly, alerts can warn you well in advance of a disaster and enable you to fix problems on a convenient schedule and without affecting users.

  - **Graphs** are the visual representations of your data.  Graphs help you see trends that might not be obvious in raw form.  We humans have remarkably good visual-pattern-recognition systems, and graphs enable us to identify anomalous data quickly.  They also look super cool on a huge flatscreen TV dashboard mounted in the middle of your office.  

  - **Logs** hold your raw data.  They're traditionally stored as files (but can also live directly in database tables.)  They are the source records of event data from the operating system or application and are most commonly a linear time series.  Whereas graphs summarize (and, potentially, average away certain specifics), logs will always have the finest level of detail and be the "base source of truth."

## <a name="layers"></a> Where to Monitor

To best deploy our tools, let's first break our application into a multi-layer stack.  At the bottom is the application itself and each layer up represents the services and components that support the preceding layer (We'd make an onion / ogre joke here but c'mon - this is a serious technical article.)

By intelligently monitoring the right metrics in each layer, we'll get a complete view of the application and maximize monitoring coverage.

In a production environment, there will likely be multiple applications we want to monitor (i.e. the web server, the application server, and the database server), so plan to apply this full-stack approach to **each** application in your deployment.  There will be some overlap (i.e. your web server process and application server process may share a physical machine), so don't feel the need to follow this too religiously.  Consolidate as you move up the stack and higher layers encapsulate more lower-level services.

### Layer 0:  The Application

The first layer of metrics we care about are those generated from, and specific to, the application itself.  Examples include a web server's active connections, an application server's worker health, or a database row count.  We're specifically **not** looking at process- or system-level metrics like CPU usage or network bandwidth.  Those come higher up in the stack.  

### Layer 1:  The Process

While the specifics of the process model are slightly different between UNIX and Windows, the concept is the same - a process is the running instance of an application within the operating system.  An application may have multiple processes running at any given time, and there can be dozens to hundreds of processes running on a server.

The metrics we're interested in here are those that report on process state ("created", "running", "terminated"), uptime, and consumed resources (CPU, memory, disk and network.)

### Layer 2:  The Server

One way of looking at a server is simply as a container (either physical or virtual) for your processes.  At this layer, we care about the health and resource usage of the overall system.  This becomes particularly important if that system hosts multiple applications (either yours or other users'), since their behavior can have an impact on your target application's performance.

Server metrics include system-wide resource usage data (CPU, memory, disk and network usage), summary metrics (total # of processes, load average, socket state and availability) and hardware state and health (disk health, memory health, physical-port access and use, CPU temperature, and fan speed.)

### Layer 3:  The Hosting Provider

Your servers live within your hosting provider.  Depending on the provider, available metrics can range from broad ("we're up" or "we're down") to latency and availability by physical datacenter and availability zone.  

Also included in this layer are the associated hosted services that support your application's existence.  These may include hosted databases, caches, and other "cloud" services your application either implicitly or explicitly depends on.

As more and more critical services move away from local servers and into cloud-based services, monitoring the status of those services becomes increasingly important, particularly when the health of those services is out of your immediate control.

When higher-level services fail, many lower-level alerts will trigger (and loudly.)  You still want to have alerts set for these higher-level failures for troubleshooting purposes.  Even if you get 20 alerts about failures along every layer, the additional alert that says your hosting provider is down is one less step you have to take in the analysis stage.

### Bonus Layer!  Layer 4:  External Dependencies

Very often overlooked but absolutely critical.  Several are common to nearly all production web services, and failures here can lead to unexpected (and painfully long) downtime.

These dependencies include:

  * **Domain Names** - DNS renewal dates can creep up and cause severe headaches if forgotten ([just ask Microsoft](http://news.cnet.com/2100-1023-234907.html)).  Mark your calendars!
  * **SSL certificates** - Monitor both expiration dates as well as other deadlines (such as the [Google SHA-1 sunset date](http://googleonlinesecurity.blogspot.com/2014/09/gradually-sunsetting-sha-1.html)).
  
### Layer 5:  The User

Finally, we get to the last (but certainly not least) layer in our stack - the user.  This is who it's all about anyway, right?  

At the user layer, we're interested in the behavior of the full stack as a user would experience it (similar to an integration test in TDD terms.)  In practice, this can be as simple as an external HTTP monitor that looks for a 200 OK response from an API endpoint or a series of automated browser tests that check a sequence of pages for specific responses.

This can also include wider-ranged metrics that happen to roll up a number of lower-level behaviors and implicitly test them.  At Google, for example, internal lore held that the most important alert in the entire operations system was the measurement of “advertising dollars earned per second.”  This is a great integration metric to watch, because a change could point to anything from a data center connectivity problem to a code bug to mistuning in the AdWords placement algorithms.  

## What to Monitor

Now that we've got our theory covered, let's put it into practice and explore our four core monitoring strategies:

### 1.  Monitor Potential Bad Things

Most problems are detectable in advance if you look carefully enough.  Since one of our stated goals is to do this all with "as little work as possible" - and actual problems generally require work - it seems as if preventing problems in the first place is a good place to start.

Many production incidents have as their root cause “something that filled up.”  In 2010, Foursquare suffered a widely publicized 11-hour outage because one shard of their MongoDB database outgrew the available memory.  “Filling up” can take many forms: CPU overload, network saturation, queue overflows, disk exhaustion, worker processes falling behind, or API rate limits being exceeded.

These failures rarely come out of nowhere. The effect may appear very suddenly, but the cause creeps up gradually. This gives you time to spot the problem before it reaches a tipping point.

**Action Items for Monitoring Potential Bad Things**

  * Identify the resources in each layer of your stack that can fill-up / overflow;
  * Understand the tipping point of each resource.  A "full" CPU might not be 100% - a process could show meaningful degradation at a level below that;
  * Understand the consequences of an overflow.  We can't overstate how many hours we've spent tracking down a mysterious process crash to discover a full disk as the culprit; and
  * Set alerts that will fire before the tipping point is reached.  For instance, the library we use internally for rate limiting emits a log message whenever a limiter reaches 70% of the cutoff threshold.

Any form of storage can fill up. Monitor unused memory on each server, free space in each garbage-collected heap, available space on each disk volume, and slack in each fixed-size buffer, queue, or storage pool.

Any throughput-based resource can become oversubscribed.  Monitor CPU load, network load, and I/O operations. If you’re using rate limiters anywhere, monitor those as well.

A background thread or fixed-size thread pool can choke even if the server has CPU to spare. Use variable-sized pools, or monitor the thread’s workload.

Lock contention can also cause a tipping point on an otherwise lightly loaded server. It’s difficult to monitor lock usage directly, but you can try to monitor something related. For some locks, we generate a log message each time we acquire or release the lock, allowing Scalyr’s log analysis to compute the aggregate time spent holding the lock.

### 2. Monitor Actual Bad Things 

Despite our best efforts of prevention, things will break.  

When they, your best defense is to be prepared, respond quickly, gather as much data about the problem as possible, identify the cause as quickly as possible, and fix it.  

(In some cases, of course, that remedy is to wait for an external dependency to come back online, but there are still ways to mitigate problems with backup providers and failover rules.)

Your toolset should enable quick drill down from alert to graph to log to fix.  The more time you spend trying to identify the problem, the less time you have to remedy it and the longer the outage will last.  

**Action Items for Monitoring Actual Bad Things**

  * Identify the resources in each layer of your stack that can stop functioning correctly;
  * Understand the effects that each potential failure can have on the rest of the system; 
  * Make sure both monitors and alerts are set to trigger when those resources fail, and that those triggers reach you with an appropriate level of urgency; and
  * Have tools in place to move quickly from alert to graph to log (as necessary) in order to identify the root cause of a failure.

It's also important to ensure thorough alert coverage in both breadth and depth.  In [a famous incident in 2012](http://www.paritynews.com/2012/10/13/432/oracle-website-down-says-hello-world), Oracle.com's home page read only "Hello, World" for over an hour.  Presumably a monitor that tested for a 200 OK response from the home page would not have triggered an alert in that failure scenario, so it's important to understand the scope of potential failures and cover them accordingly.

**Exception Monitoring**

Monitoring exceptions is worth special mention here.  Exceptions are anomalous or exceptional conditions requiring special processing – often that change the normal flow of program execution.  They are generally handled by specialized programming language constructs or hardware mechanisms.

Exceptions come in several flavors -- most commonly as application exceptions (that are handled by the runtime and written to logs accordingly), but also as "out-of-band" exceptions from the processes themselves.

(For example, if your web server process is failing in some way that causes logs not to be written, then you've lost that dimension of monitoring - so catching exceptions from the server process itself is important.)

Application exceptions provide some of the best monitoring bangs for your buck, since they generally correlate with an application bug and are typically triggered by a user action or background task.

A bonus is that you get to impress users by proactively reaching out to them before they even complain about an error.

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

## Summary

Properly monitoring your systems and setting up thorough alert coverage is hard.  But with the right tools, a complete understanding of your application stack, and a broad approach that covers problems before, during and after they occur, we expect you'll be monitoring very effectively...

...and sleeping very well.
