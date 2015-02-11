# How to Set Alerts

In our guide [Zen and the Art of System Monitoring](zen-and-the-art-of-system-monitoring.md), we cover our monitoring and alerting philosophy and discuss practical steps for putting that philosophy into practice.  The three main tools for doing so are logs, graphs, and alerts.

In this guide, we're going to take a closer look at alerts, and, more specifically, how alerts are constructed, when they should trigger, and how they should notify you.

There's more to alerting than, well, just setting an alert.  In fact, there's an unexpected amount of complexity when you peek below the surface, so we're going to do our best to keep things as straightforward as possible while covering the topic in-depth.

We'll prescribe a specific set of techniques that will help you build alerts intelligently, minimize false alarms and missed incidents and, ultimately, let you sleep better at night. (Both figuratively and literally - nobody wants the 3am wake-up call unless things are _really_ borked.)

Before we dive into recipies for building good alerts, it's important to cover some background.  In this guide, we'll discuss:

  * Iteration & Tradeoffs
  * How an Alert is Structured
  * Alert Thresholds
  * Metrics and the Alerts that Love Them
  * How an Alert Notifies You

So let's dive in...

## Iteration & Tradeoffs

It's worth starting with that fact that there is no definitive _initial_ prescription for how to build your alerts - it's a process of iteration.  You'll want to pick a logical starting point for each metric and, over the course of a few weeks of active use, refine and tune.

During the tuning process, your goal is to find a balance between "noisy" and "quiet" alerts.

There's an explicit tradeoff here -- tighter trigger thresholds will make for noisier alerts. They'll catch more problems, but at the cost of false alarms.  Looser thresholds will make for quieter alerts, but at the increased risk of missing real incidents.  A balance is needed.

(A good analogy here for anyone who's ever played with a shortwave radio:  Radios have a 'squelch' control that keeps you from hearing background static until a loud signal comes through.  If your squelch is too high, you won't hear any static but you may miss some weak signals.  If your squelch is too low, you'll hear all the signals but also lots of annoying static.)

Noisy alerts open you up to "alert fatigue" -- where you become so used to seeing a particular alert that - often unconsciously - you start to take it less seriously.  This can have the paradoxical effect of making the alert _less_ effective - the DevOps version of crying wolf.

For each alert, your ideal balance point is where false alarms are rare but you're still confident that real incidents won't be missed.  When you first set an alert, we recommend erring on the side of a tighter threshold.  If you get false alarms, raise the threshold before alert fatigue can set in.  The converse strategy -- starting with a loose threshold -- can lead to the "no one noticed the server was down" problem, which is worse than "this new alert is a bit annoying" problem.

A new, untested alert should usually not be able to wake you up at 3:00 AM. Start with more passive forms of notification (such as an email.)  Once an alert has been in the field for a few weeks without generating false alarms, it can "graduate" to more urgent forms of notification, such as an SMS message or phone call.  (See the section [How an Alert Notifies You](#notification).)

If you're not able to find a reasonable balance point, you should try a different approach to the alert. Use a different kind of threshold, add a grace period, or base your alert on a different metric.  You'll learn about these things as you read this guide.

## How an Alert is Structured

At its simplest, an alert is structured as follows:

    [Metric] [Comparator] [Threshold]

The **metric** is the value we're monitoring, whether it's a direct measurement or a computed value.  The **comparator** is the comparison operator, i.e. "greater than", "less than", "equals".  And the **threshold** is the value to be compared against. (We'll explore thresholds in depth in a later section.)  If the entire expression evaluates as truthy, the alert will fire.

Let's take a simple example:

    CPU usage > 90%

Our **metric** is "CPU usage", our **comparator** is ">", and our **threshold** is "90%".  This is an example of what we call a fixed threshold - 90% doesn't change.

A slightly more complex example:

    Average CPU usage over the past 15 minutes > Average CPU Usage over the past 12 hours 

Both the metric ("Average CPU usage over the past 15 minutes") and the threshold ("Average CPU usage over the past 12 hours") are what we call compound values. The base metric ("CPU Usage") is modified by a function ("average") over some time period.  The threshold here is what we call a historical threshold -- it's not based on a fixed value, but on a rolling average of past measurements.

## Alert Thresholds

There are three types of thresholds you can apply to your alerts.  Most monitoring tools should support these (and if not, hey, [you can try ours](http://www.scalyr.com)).

### 1.  Fixed Threshold

A fixed threshold is used to trigger an alert based on the comparison of a metric to a fixed numeric value.  The metric may be the result of an average or a more complex calcluation, but the threshold it's being compared to is fixed.

Fixed thresholds make sense when you're dealing with a metric that has a hard, well-characterized limit.  For instance, CPU usage can't exceed 100%, free disk space can't go below 0, and open files can't go above the operating system's limit on file descriptors. When monitoring this type of metric, it's easy to set a fixed threshold.

### 2.  State-based Threshold

Some metrics have discrete values that identify changes in system state.  For metrics like this, it usually makes sense to alert when the metric changes from one particular value to another. For example:

    Alert if Process Status changes from "Running" to "Stopped"  

You'd use an alert like this to notify you if your application process stops running.

### <a name="historical-threshold"></a> 3.  Historical Threshold

Historical thresholds are used to compare a metric's current value with past values of the same metric.  This is also known as a "sliding window" threshold, since they typically look at a specified span (aka window) of time over a period that moves as the clock ticks.  An example:

    15-min Average Network Usage > 110% of 15-min Average Network Usage One Week Ago

This alert will fire if the 15-minute trailing average of network usage exceeds 110% of the same measurement from one week earlier.  This type of threshold is useful for metrics that follow a regular, cyclical schedule. By comparing the current value to the value at a similar previous time, you can use a tighter bound than would be possible with a simple fixed threshold.

For example, suppose traffic spikes every Monday but is relatively quiet over the weekend. With a fixed threshold, if you set the threshold high enough to not be triggered by routine Monday spikes, you might miss an abornmal increase in network usage over the weekend. If you set the threshold lower, then it will trigger every Monday, risking alert fatigue.

Different types of historical threshold make sense for different metrics. For highly cyclic metrics, such as traffic to a web site, a day-ago or week-ago threshold often makes sense, but can be thrown off by unusual events such as holidays or promotions. You can also base your threshold on the immediate past, for instance:

    5-min Average Error Rate > 150% of 60-min Average Error Rate

This will detect sudden jumps in a metric, without being thrown off by things like holidays. However, it will not detect a "slow creep" trend, such as CPU usage slowly creeping up due to a memory leak. Day-ago or week-ago comparisons are better at detecting a slow creep.

### Grace Period

A "grace period" prevents an alert from firing unless it's triggered for a sustained period of time. This is useful if you want to disregard the behavior of a metric in small bursts, but be alerted to sustained periods of that behavior.

For example, when monitoring CPU usage on a low-traffic server, you might see occasional spikes to 100% for benign reasons.  But if CPU usage remains at 100% for 15 minutes, that could indicate a stuck process or other serious problem. It's difficult to distinguish this scenario using the tools we've discussed so far, but a grace period makes it easy:

    CPU Usage > 90% for 15 minutes or more

This specific alert, by the way, is particularly helpful if you're using some sort of metered service such as Amazon's T2 "burstable" instances.  In those environments, a stuck CPU will deplete your usage credits and render your system unusable, so this is something you definitely want to know about.

### Advanced Techniques

There exists a sub-field of statistical and data mining research devoted to more sophisticated [anomaly detection](http://en.wikipedia.org/wiki/Anomaly_detection) techniques.  Examples include the Holt-Winters algorithm or a heuristic called the Western Electric Rules.  These are outside the scope of this article.

## Metrics and the Alerts that Love Them

Ok, now that we've covered the various threshold types, we're ready to prescribe some best practices for setting alerts.  We'll organize the discussion according to the type of metric used in the alert.

### Capacity Metrics

These measure things with a fixed, known capacity -- like disk space, free memory, or open file handles.  If something can "fill up" or "run out", it's generally measured with a capacity metric.  In particular, capacity metrics usually imply that bad things will happen (to stability, performance, or both) when that capacity is reached.

  * **Alert Threshold to Use**:  Fixed
  * **How To Set Threshold**:  Ask the following questions:
    * What is the capacity of this resource (e.g. how much disk space do I have?);
    * What happens when I reach that capacity (how bad is it if I run out of disk space?); and
    * How quickly could I potentially reach that capacity?

Set a fixed threshold that is somewhat below maximum capacity. Reasonable thresholds may range anywhere from "50% full" to "90% full". Use a lower threshold when the consequences of hitting capacity are especially severe, and if usage is spiky. (If your system consumes disk space slowly, then even when it's 90% full, you'll still have some time to take action. But if it can chew up large amounts of space in short period of time, then you might need to use a 70% threshold to give yourself enough time to react. So it's important understand the volatility of the metric you're monitoring.)

### Bandwidth Metrics

These measure flows, such as network utilization, CPU usage, and disk throughput (IOPS).

  * **Alert Threshold to Use**: Fixed
  * **How to Set Threshold**:  Ask the following questions:
    * What is the maximum possible throughput? (how much network bandwidth do I have?)
    * What happens when I reach that capacity? (does the network just slow down, or does it cease to operate?)
    * What does my usage pattern look like?  How spiky is it?

Bandwidth metrics can be very spiky, so you may want to base your alert on a running average.  For instance, you might alert based on average network usage over a 15-minute period, rather just using the latest measurement.  Then, set your threshold in the same way that you would for a capacity metric.

### State Metrics

A state metric is one with discrete values that identify changes in system state.  These are usually "status"-type metrics.  Examples include process state, server status and hosting provider status.

  * **Alert Threshold to Use**:  State-based
  * **How to Set Threshold**:  The state threshold is based on which state changes you care about.  If you want to know when a process terminates, you'll monitor for a state change from "running" to "stopped".  Likewise, if you want to know when your host is down, you'll set an alert to trigger when status goes from "up" to "down".

### Rate Metrics

Rate metrics measure the rate at which some event occurs, i.e. the number of events per unit of time.  These are similar to bandwidth metrics, but measure discrete events.  Examples include "requests per second", "errors per second", "login attempts per minute", or "HTTP 501 responses per hour."

  * **Alert Threshold to Use**:  Fixed or Historical
  * **How to Set Alert Threshold**:  Rate metrics are slightly more complicated to alert.  Ask the following questions:
    * What is normal? 
    * What does a deviation from normal look like?
    * How predictable is such a deviation?
    * How dangerous is it when that deviation happens?

Remember - iteration is your friend here.  Don't get too hung up on setting the perfect starting threshold - just get close enough, keep an eye on things, and iterate as you get more data.

"Normal" and "deviation from normal" will depend on the metric.  For some error types, normal is (hopefully!) 0, and you can set a fixed threshold. Other metrics may follow the cycle of traffic to your site, and a historical threshold wil work best to highlight anomalies.

The more predictable a metric, the easier it is to set thresholds.  On a predictable ("smooth") metric, you'll want to look for sudden deviations from historical norms.  On a less-predictable ("spikey") metric, you may be better off setting a fixed threshold based on historical minimum or maximum values.

  * **Protip**:  You can use smoothing (averaging) to turn a spikey metric into a smoother metric, at the cost of some delay in detecting changes.
        
Finally, think about how "dangerous" it is when a deviation happens.  If you can handle 5x your normal load, then a 2x jump in requests per second may not be worth alerting. You can use a threshold that's well above the normal value. But for a metric like "error rate", where a 2x jump is less acceptable, you would use a tighter threshold.

### Event Parameter Metrics

These are metrics based on a quantitative attribute of an event, such as request size or response time.  Usually, the metric will report an average value for all events in some time bucket, e.g. average response time for requests in each minute. Depending on your monitoring tools, you might also be able to work with minimum, maximum, and percentile values.  (Don't confuse _response time_, which measures the time a server spent processing a request, with _response rate_, which is a rate metric reporting the total number of responses per time period.)

  * **Alert Threshold to Use**:  Fixed or Historical
  * **How to Set Alert Threshold**:  Ask the same questions as for a rate metric:
    * What is normal?
    * What does a deviation from normal look like?
    * How predictable is such a deviation?
    * How dangerous is it when that deviation happens?

A deviation in response time can be normal - perhaps a user is downloading a large file - but a sustained deviation can be something you'll want to know about (is your server overloaded?).  Set your threshold as you would for a rate metric: use a historical threshold for predictable metrics, and a fixed threshold for spiky metrics.  Use averaging or grace periods for especially spiky metrics, and rely on iteration to get things right.

An event parameter metric can be especially spiky, as a single large value (say, a user downloading a large video file) can throw off the entire average.  A time-based average may not be able to smooth away these spikes.  Hence, grace periods are especially useful when alerting on an event parameter metric.

## <a name="notification"></a> How an Alert Notifies You

In the good 'ol days of operations, there weren't many options for how you could be notified.  If you were in the datacenter, you'd keep an eye out for the blinking light of doom.  If you were at home sleeping, someone might call you to tell you about the blinking light of doom.  Pagers then brought a new dimension to the process -- the blinking light of doom could now wake you up directly!

In modern day environments, we have a variety of options.  Alerts can be stored in databases for review on demand, batched and emailed to you daily, emailed to you immediately, or trigger an SMS or even a phone call.  Take advantage of these options to stay on top of issues without interrupting yourself unnecessarily.

The more urgent or severe a problem an alert might indicate, the more immediate / interruptive form of notification you should use.  It often makes sense to "stack" alerts on the same metric, at different thresholds and with different notification methods (increasing urgency as the situation becomes more "dire".)  For example:

    Available Disk < 50% (Log Warning)
    Available Disk < 25% (Email Notification)
    Available Disk < 10% (SMS or Push Notification)
    Available Disk < 1% (Phone Call)

As mentioned earlier, you might also start with less-interruptive notifications for a new alert, and then move up to more-interruptive notifications once the alert has been seen to not generate false alarms.

## A Final Note on Iteration & Tuning

We hope this has been a useful exploration of the (surprisingly complex!) art of setting alerts.  We want to emphasize again that setting thresholds should be an iterative process - digest this guide and set reasonable starting points to the best of your ability, but don't agonize.

If an alert is too noisy, you can relax the threshold, add a grace period, or try a sliding window to smooth out the underlying metric.  

If you've found that you're missing important events, look for places to tighten your threshold or perhaps add an additional alert.

And finally, we hope you've found this reference valuable, enlightening, and perhaps even fun...(ok, ok, we won't push it.)

Good luck and happy alerting!
