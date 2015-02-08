# How to Set Alerts

In our guide [Zen and the Art of Server Monitoring](zen-and-the-art-of-server-monitoring.md), we cover the three main tools you should be using to monitor your systems effectively:  Logs, graphs and alerts.

In this guide we're going to take a closer look at alerts, and, more specifically, how alerts are constructed, when they should trigger, and how they should notify you.

There's more to alerting than, well, just setting an alert.  In fact there's an unexpected amount of complexity when you peek below the surface, so we're going to do our best to keep things as simple and straightforward as possible while still covering the topic in-depth.

Below, we'll prescribe a specific set of techniques that will help you build alerts intelligently, minimize false alarms, missed incidents and, ultimately, let you sleep better at night (both figuratively and literally - nobody wants the 3am wake-up call unless things are _really_ borked.)

Before we dive into the heart of our guide, though, it's important to cover some background.  What we'll cover:

  * Iteration & Tradeoffs
  * How an Alert is Structured
  * Alert Thresholds
  * Metrics and the Alerts that Love Them
  * How an Alert Notifies You

So let's dive in...

## A Note on Iteration & Tradeoffs

First, it's worth mentioning that there is no definitive _initial_ prescription for how to build your alerts - it's a process of iteration.  You'll want to pick a logical starting point for each metric and, over the course of a few weeks of active use, refine and tune.

During the tuning process, your goal is to find a balance between "noisy" and "quiet" alerts.

There's a tradeoff here -- tighter trigger thresholds will make for noisier alerts -- they'll tell you about more problems more frequently, but at the cost of more false positives.  Looser thresholds will make for quieter alerts, but at the increased risk of missing real incidents.  So a balance is needed.  

(A good analogy here for anyone who's ever played with a shortwave radio:  Radios have a 'squelch' control that keeps you from hearing background static until a loud signal comes through.  If your squelch is too high, you won't hear any static but you may miss some weak signals.  If your squelch is too low, you'll hear all the signals but also lots of static - which can be super annoying.)

Noisy alerts also open you up to potential "alert exhaustion" -- where you become so used to seeing a particular alert that - often unconsciously - you start to take it less seriously.  This can have the paradoxical effect of making the alert less effective despite its vocality (since you'll be more likely to ignore it when there's a real incident - the DevOps version of crying wolf.)

The optimal strategy is to find the right balance between a tolerable number of false positives and the security that you're not missing anything major.

We recommend erring on the side of tighter thresholds to start.  Despite the risk of alert exhaustion, it's generally preferrable to face the "this alert is too noisy" problem than "I didn't know about the server failure for 30 minutes" problem.

## How an Alert is Structured

If we were to define a "universal template" for an alert, it would look like this:

    [Metric] [Comparator] [Threshold]

The **metric** is the value (either direct or computed) we're monitoring.  The **comparator** is the comparison operator, i.e. "greater than", "less than", "equals".  And the **threshold** is the value to be compared against (we'll explore thresholds in more depth in a later section.)  If the entire expression evaluates as truthy, the alert will fire.  

Let's take a simple example:

    CPU usage > 90%

Our **metric** is "CPU usage", our **comparator** is >, and our **threshold** is 90%.  This is an example of what we call a Fixed Threshold - an absolute value.

A slightly more complex example:

    Average CPU usage over the past 15 minutes > Average CPU Usage over the past 12 hours 

Both the metric ("Average CPU usage over the past 15 minutes") and the threshold ("Average CPU usage over the past 12 hours") are what we call compound values -- a base metric ("CPU Usage") modified by a function ("average") over a time period ("15 minutes" and "12 hours", respectively.)  The threshold here is what we call a historical threshold -- it's based not on a fixed / absolute value but instead on a rolling average over a certain period of time in the past.

## Alert Thresholds

It's worth going into some depth on the various types of alert thresholds in a typical arsenal, as it's important to understand what each type means and for which metrics it's most useful.  Most monitoring tools should support these (if not, we have [a suggestion](http://www.scalyr.com)...).  

We'll then follow this with a review of the different metric types you're likely to encounter along with the appropriate thresholds to apply to each.

### State-based Threshold

A state-based threshold is defined by discrete changes in state.  Let's use an example to illustrate:

    Alert if Process Status changes from "Running" to "Stopped"  

You'd use an alert like this to notify you if your application process stops running (which certainly merits an alert.)  In this case state refers to one of the several possible discrete process values, which under Linux includes "Running", "Stopped", "Zombie", etc.

### Fixed Threshold

Used in our first example above ("CPU usage > 90%"), a fixed threshold is used to trigger an alert based on the comparison of a metric to a fixed numeric value, which in the case of our example is 90%.

Both the metric and the fixed threshold can be compound values (such as averages over time), but the defining feature is that it's a singular, fixed value.

Fixed thresholds make sense when you're dealing with well-characterized, hard limits such as CPU usage, network usage, disk space or process count.  

### <a name="historical-threshold"></a> Historical Threshold

Historical thresholds are based on the comparison of a metric to past values.  We also refer to these thresholds as "sliding windows" since they typically look at a specified width (aka window) of time over a period that moves as the clock ticks.  An example:

    15-min Average Network Usage > 110% of 15-min Average Network Usage One Week Ago

This alert will fire if the 15-minute moving average network usage exceeds 110% of the same 15-minute moving average one week ago.  This type of alert is useful if your network usage fluctuates on a regular cyclical weekly schedule that would make a fixed value fire too often.

If, say, traffic spikes every Monday but is relatively quiet over the weekend, you don't want an alert every time the spike hits (risk of alert exhaustion), nor would you want to set the threshold to a fixed value above the normal Monday spike because then you might miss a smaller (but still relatively important) spike over the weekend.

Varying the distance in the past you look at can have benefits -- a longer-term threshold, for example, can be used to detect a "slow creep" (i.e. CPU usage slowly creeping up due to a memory leak) that might not otherwise be visible in shorter scale.  Be aware that this sort of alert can be thrown off by confounding factors such as holidays or promotions.

#### Grace Period

A grace period is an alert modifier that's slightly outside the bounds of our standard model, but can be incredibly useful.  Employing a grace period means an alert won't fire unless it's triggered for a sustained period of time.  This is useful if you want to disregard the behavior of a metric in small bursts but be alerted to a longer period of that behavior.  

For example, when monitoring CPU usage on certain low or medium volume servers, the occasional spike to 100% is not notable.  But if the CPU usage remains at 100% for 15 minutes, that could indicate a stuck process, so an alert as follows would be great:

    CPU Usage > 90% for 10 minutes or more

This specific alert, btw, is particularly helpful if you're using some sort metered service such as Amazon's T2 "burstable" instances.  In those environments, a stuck CPU will deplete your usage credits and render your system unusable, so this is something you definitely want to know about.

#### Advanced Techniques

There exists a sub-field of statistical and data mining research devoted to more sophisticated "anomaly detection" techniques.  Examples include the Holt-Winters algorithm or a heuristic called the Western Electric Rules.  These are outside the scope of this article, however it's worth noting that we expect to see these sorts of advanced techniques make their way into professional alerting tools.  [Wikipedia has some more details here](http://en.wikipedia.org/wiki/Anomaly_detection).

## Metrics and the Alerts that Love Them

Ok, now that we've covered the various threshold types, let's explore the most common metric types you'll be working with and prescribe some best practices for setting alerts.

### Capacity Metrics

These measure things with a fixed, known capacity like disk space, free memory, or open file handles.  If something can "fill up" or "run out", it's generally measured with a capacity metric.  In particular, capacity metrics usually imply that bad things will happen (performance- and stability-wise) when that capacity is reached.

  * **Alert Threshold to Use**:  Fixed
  * **How To Set Threshold**:  Ask the following questions for each metric you're monitoring:  
    * What is the capacity of this value (i.e. how much disk space do I have?);
    * What happens when I reach that capacity (i.e. how severe is it if I run out of disk space?); and
    * How quickly could I potentially reach that capacity from an arbitrary point in the process?  

The last point is important - if you're monitoring disk usage and set an alert to fire at 90% capacity, great.  But if that last 10% can fill up in a matter of minutes, less-great, so it's important understand the volatility of what you're monitoring.  In the case of our rapidly-filling disk, it would be wise to set a lower threshold to give yourself more time to react.  

### Bandwidth Metrics

These are akin to measurements of flow and include network utilization, CPU usage, and disk throughput (IOPS).

  * **Alert Threshold to Use**: Fixed or Historical
  * **How to Set Threshold**:  Ask the following questions:
    * What is the maximum possible throughput? (i.e. how much network bandwidth do I have?)
    * What happens when I reach that capacity? (i.e. does the network just slow down, or does it cease to operate?)
    * What does my usage pattern look like?   

The last question, of course, can only be answered after a period of observation.  A "spikey" pattern (observed in lower-usage metrics with potential for bursts of increased activity) will merit a smoother, historical threshold (perhaps with a grace period) to avoid false positives from local maxima.  A higher-volume ("smooth") usage pattern will be better served with a fixed threshold.  

### State Metrics

State metrics relate to a discrete state value of something and pair with state-based thresholds.

  * **Alert Threshold to Use**:  State-based
  * **How to Set Threshold**:  The state threshold is based on which state changes you care about.  If you want to know when a process terminates, you'll monitor for a state change from "running" to "stopped".

### Rate Metrics

Rate metrics measure the rate at which underlying events happen and typically have a "per unit of time" component.  These are similar to bandwidth metrics mathematically, but distinct in the context of monitoring.  While bandwidth metrics measure the rate of flow of an underlying resource, rate metrics measure the rate at which discrete monitorable events take place.

Examples include "requests per second", "errors per second", "login attempts per minute", or "5xx requests per hour."

  * **Alert Threshold to Use**:  Fixed or Historical
  * **How to Set Alert Threshold**:  Rate metrics are slightly more complicated to alert.  Ask the following questions:
    * What is normal? 
    * What is deviation from normal?
    * How predictable is this deviation?
    * How dangerous is it when deviation happens?

Remember - iteration is your friend here.  Don't get too hung up on setting the perfect starting threshold - just get close enough, keep an eye on things, and iterate as you get more data.

"Normal" and "deviation from normal" will depend on the metric.  For some error types, normal is 0, so any deviation is something you'll want to know about, and you'll want to set a fixed threshold to alert if that number becomes greater than 0.  

For requests per second, your traffic might vary on a regular cyclical weekly pattern (as we described in our example earlier), so a sliding window comparing against the previous week will work best to highlight anomalies.

The more predictable a metric, the easier it is to set thresholds.  On a predictable ("smooth") metric, you'll want to look for sudden deviations from historical norms.  On a less-predictable ("spikey") metric, you may be better off setting a fixed threshold based on historical minimum or maximum values.

  * **Protip**:  You can use smoothing (averaging) to turn a spikey metric into a smoother metric, at the cost of some delay.
        
Finally, think about how "dangerous" it is when a deviation happens -- this will be instructive to how your alert threshold should be set.  If your requests per second quintuples (and you can handle the load), you'll want to know about the change, but it may not be high-urgency.  In that case you can employ a sliding window threshold to smooth out the change.  If your error rate quintuples, however, you'll want to know right away so a tighter (and perhaps noisier) threshold will be best.

### Event Parameter Metrics

These are metrics for an objective quantitative attribute like size or time (i.e. request or response size, response time, page rendering time, etc.)  Note that response time - which is an objective parameter usually measured in ms - is not to be confused with response rate, which is a rate metric.  Although to be fair, the two are similar and the difference is subtle.

  * **Alert Threshold to Use**:  Fixed or Historical
  * **How to Set Alert Threshold**:  Ask a similar set of questions:
    * What is normal?
    * What does deviation from normal look like?

A deviation in response size can be normal - perhaps a user is downloading a large file - but a sustained deviation can be something you'll want to know about (is the user scraping your entire site?) - so a grace period + average over the past 15 minutes would alert to that issue.  But a rapid increase in response time could be an early indication of problems elsewhere in your stack, so an alert based on the average response time over the past 5 minutes compared with the past week would highlight a rapid change quickly.

## <a name="notification"></a> How an Alert Notifies You

In the good 'ol days of operations, there weren't many options for how you could be notified.  If you were in the datacenter, you'd keep an eye out for the blinking light of doom.  If you were at home sleeping, someone might call you to tell you about the blinking light of doom.  Pagers then brought a new dimension to the process -- the blinking light of doom could now wake you up directly!

In modern day environments, several degrees of notification types can parallel several degrees of escalation urgency.  Alerts can be stored in databases for review on demand, batched and emailed to you daily, emailed to you immediately, texted, or pushed via native phone notifications.  And of course the traditional phone call still works.

It's important to consider these degrees of escalation when organizing your alert structure.  One idea to consider is "stacking" alerts on the same metric, but at different thresholds and via different notification methods (with increasing urgency as the situation becomes more "dire".)

An example of an alert escalation stack:

    Available Disk < 50% (Log Alert)
    Available Disk < 25% (Send Email)
    Available Disk < 10% (Send Push Notification)
    Available Disk < 1% (Phone Call)

## A Final Note on Iteration & Tuning

We hope this has been a solid exploration of (surprisingly complex!) art of setting alerts.  We want to emphasize again that setting thresholds should be an iterative process - digest this guide and set reasonable starting points to the best of your ability, but don't agonize.

If an alert is too noisy, you can relax the threshold, add a grace period, or try a sliding window to smooth out the underlying metric.  

If you've found that your missing important events, look for places to tighten your threshold or perhaps add an additional alert.

And finally, we hope you've found this reference valuable, enlightening, and perhaps even fun...(ok, ok, we won't push it.)

Good luck and happy alerting!
