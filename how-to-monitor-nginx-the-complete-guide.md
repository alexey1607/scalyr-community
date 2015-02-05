# How to Monitor Nginx:  The Complete Guide

Nginx is an increasingly popular open source HTTP and reverse proxy server that's known for its high concurrency, high performance, and low memory usage.  It's become the second most popular public-facing web server (as of December 2014) among the top 1M busiest sites online, and it's pretty awesome.

But, like any piece of long-running software (or small child), it can get into trouble if left completely unattended.  

This guide will take you through a series of recommendations on how to properly setup monitoring and alerting for a production Nginx deployment.  It will make no recommendations on how to monitor a small child.

While Nginx's open source variant (Nginx F/OSS, aka plain 'ol Nginx) is the most popular, a commercial version (Nginx Plus) is also available and offers load balancing, session persistence, advanced management, and finer-grained monitoring metrics.  This guide uses "Nginx" in the universal sense and refers to both the F/OSS and commercial versions.  The metrics discussed in this guide are also available in both versions.

## Why Write This Guide?

While there's a body of conventional wisdom and some basic resources on the web, there's no definitive guide to monitoring Nginx.  At Scalyr, we've developed a monitoring approach based on years of frontline operations experience that we're confident will help you more quickly detect problems, efficiently spot incipient / slow-burn issues before they become full-scale outages, and ultimately save you (and your users) headaches.

We hope you find this valuable.  And if you don't - let us know!  This, along with all of the Scalyr Community Guides, [is editable on Github]().  If we've gotten something wrong, please make a correction and submit a pull request or leave a comment below.  We are believers in [Cunningham's Law](http://meta.wikimedia.org/wiki/Cunningham%27s_Law) (best seen in [xkcd 386](http://xkcd.com/386/)) so we value and deeply appreciate feedback.

## Background Material

This guide can be read as-is and is designed to provide quick, actionable recommendations.

If you want to dig deeper, take a spin through [Zen and the Art of  Server Monitoring]().  It explores, in-depth, the strategy and philosophy behind what we recommend you monitor and why.

Next, you can read about [Setting Alert Thresholds]() (there's more to it than "if value X falls below threshold Y" -- what if threshold Y is different on weekends?  Or throughout the day?)

Finally, take a look at our [In-Depth Guide to Nginx Metrics]().  Think of that as an addendum / appendix to this guide.  It examines the complete list of available Nginx metrics, what exactly they measure, and what exactly they mean.

## The 15 Essential Nginx Metrics to Monitor

The monitors below are grouped by [application stack layer]().

### The Nginx Application Itself

#### 1.  Requests Per Second (RPS)

The essential job of Nginx is to serve content to clients, so it makes sense that the first metric we care about is requests per second - the rate at which we're doing that essential job.  Changes in RPS are most often associated with changes in client volume, and that's something we want to know about.  Spikes can indicate both benign events (sudden press coverage) or malignant ones (a DDoS attack.)  Drops can indicate a problem elsewhere in the stack.

  * **Monitor** requests per second (RPS) by sampling `requests` as reported by the `ngx_http_stub_status_module`.
  * **Set an alert** if RPS moves above *or* below your [threshold]().  Pick a baseline number based on recent RPS metrics, and iterate as you gather more data.  

#### 2.  Response Time

The companion to RPS.  Response Time measures how quickly requests are being handled - in other words, your application's performance.  Do note that response time can be held back by delays upstream, so it's not purely an Nginx metric.

  * **Monitor** response time by logging the `$request_time` log variable and monitoring logs accordingly.  This measures the elapsed time since the first bytes were read from the client and the log write after the last bytes were sent to the client.
  * **Set an alert** if response time moves above or below your [threshold]().  Pick a baseline based on recent data, and iterate as you gather more.  Note:  Though it may seem counter-intuitive, we do actually care if response time suddenly goes down.  Servers don't just start super-performing, so if your latency drops it could mean something that normally takes longer has broken.

#### 3.  Active Connections

There is a (configurable) hard limit on the total number of connections that Nginx can handle, so both knowing that limit and monitoring the current number of active connections is important.

The limit is equal to the # of [`worker_connections`]() * [`worker_processes`]().  Once the limit is reached, connections will be dropped and performance will suffer.  Note that this limit includes all connections (i.e. both connections from clients and connection to upstream servers), so be sure to set a high enough limit relative to the hardware and expected server load.

  * **Monitor** active connections by reading `Active connections` from `ngx_http_stub_status_module`.
  * **Set an alert** as your active connections approach the maximum connection limit.  We recommend setting the alert to 70% of the limit to give youself room to adjust without setting off false positives.
###  
  * **Monitor** both `connections handled` and `connections accepted` by reading their respective values from `ngx_http_stub_status_module`.  Under normal circumstances, these should be equal.
  * **Set an alert** if `connections handled` falls below `connections accepted` - this means that Nginx is dropping connections before completion and is an indication that a resource limit has been reached.

#### 4.  Response Codes

Nginx logs the [HTTP response code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) returned for each request, and this can be a rich source of information about the health of both Nginx and your upstream servers. 

  * **Monitor** HTTP response codes and the volume and relative distribution of 2xx/3xx/4xx/5xx responses through the Nginx access logs. 
  * **Set an alert** if 2xx or 3xx responses drop below your [threshold]() (an indication that a good thing has stopped happening.)
  * **Set an alert** if 4xx or 5xx responses rise above your threshold (an indication that a bad thing has started happening.)  Set your initial thresholds based on recent activity patterns.  Refine and iterate as you gather more data and gain more experience with your alert structure.

### The Nginx Processes

Nginx spawns multiple OS processes - a master process and a separate process for each worker.  If you've enabled caching, there's an additional process for that too  (You get a process!  And you get a process!  And YOU get a process!)  It's critical to keep an eye on these processes and make sure they stay healthy and consume an available amount of resources.

#### 5.  Process State

  * **Monitor** the uptime for each process through a process monitoring agent.  Uptime should be continuously-increasing so long as the process is active.
  * **Monitor** each process' status.  Possible states vary according to UNIX flavor, but generally are one of `Running`, `Sleeping`, `Idle`, `Defunct`/`Zombie`, or `Stopped`/`Exited`.
  * **Set an alert** if the master process leaves the `running` state.  Note that if a worker process terminates (due to a configuration change or crash), the master process will handle restarting it automatically, so it's less-important to alert on failure of those workers (though you may still want to know about them in case they're symptoms of another issue.) 

#### 6.  Process CPU usage

Nginx (and web servers in general) are constrained by CPU and network bandwidth more than any other system resource.

  * **Monitor** CPU usage of each process.  This should scale relatively linearly with connection load.  Make sure you configure the number of `worker_processes` properly (the general rule is one `worker_process` per CPU core.)  
  * **Set an alert** if CPU usage deviates from your [threshold]().  A few hours worth of monitoring data should be enough to get a broad picture of your server's CPU usage - pick your starting threshold based on that, and iterate accordingly.

#### 7.  Process Network Usage

  * **Monitor** network usage of each process.
  * **Set an alert** if network usage rises or falls below your [threshold]().  At this level you're mostly interested in network usage of the processes relative to each other, since in general Nginx workers handle client requests in a load-balanced way.

#### 8.  Process Open File Handles

The number of simultaneous connections (including client connections and upstream server connections) cannot exceed Nginx's limit on the maximum number of open files, configured using the `worker_rlimit_nofile` directive).  By monitoring the number of open files relative to the limit, you have an idea of how much headroom your system has for additional connections.

  * **Monitor** the number of open file handles.
  * **Set an alert** when the number of open file handles reaches 70% of the `worker_rlimit_nofile` limit - this means it's time to increase the limit before it reaches capacity and connections are dropped.  Tune this % as needed to minimize noisey alerts and to suit your deployment.

### The Server

In-depth server monitoring is a separate topic beyond the scope of this guide, but there are a number of key metrics to monitor overall system health.

#### 9.  Server Status 

First, the big picture.  Whether you're running Nginx on a physical machine or cloud-based VPS, your hosting provider will likely provide you with an overall status indication of the physical machine or instance.

  * **Monitor** Server status.
  * **Set an alert** when the server leaves the "running" state.

#### 10.  Server Load Average

Load average is a compound metric provided by UNIX systems that combines overall CPU & disk usage into a single (floating point) number.  It measures how "loaded" the system is - how many processes are using vs. waiting for CPU or disk resources.  On a single-core system, a 5-min load average of 0.5 , roughly, over the past 5 minutes, the CPU/Disk were idle 50% of the time and used 50% of the time.  Likewise, a load average of 2.0 means one process used all available resources and a second process was waiting for use, or the system was "overloaded by a factor of 2."

Note that on some systems, load average does not account for multi-core CPUs automatically, so on a 4-core system, a load average of 3.0 means the system was only at 75% resource capacity, not overloaded by a factor of 3.

  * **Monitor** Server Load Average over 1, 5 and 15 minutes.
  * **Set an alert** when Load Average deviates from its recent level or, if relevant, a fixed threshold.  Make sure you account for the number of logical cores your system has when setting the threshold.

#### 11.  Server Network usage

A layer up from process-level network usage, as it's also important to keep an eye on the overall system network usage, whether from Nginx or other shared processes on the machine.  If another process eats up your network capacity, Nginx performance will suffer.

  * **Monitor** Network usage system-wide.
  * **Set an alert** if network usage crosses your an alert threshold relative to total network capacity.  Your specific system's threshold will vary according to usage -- some setups may run at full network capacity as a matter of course (in which case you might want to know if network usage drops, which could indicate problems.)  You will generally want to monitor a smoothed traffic graph over a recent time period (i.e., 15 minutes) to filter out short bursts of activity.

### The Hosting Provider

Your servers live within a hosting provider, so you'll want to know about big connectivity or availability issues at this level.

#### 12.  Provider status.

  * **Monitor** Hosting provider status.  This will depend on your provider, but some will offer this via an API while others will simply provide a status page (such as 'status.yourprovider.com').  Configure your monitoring tools to watch and report on the appropriate status endpoint.
  * **Set an alert** if the provider's connectivity or availbility fails. 

### User Behavior

#### 13.  Key Pages 

Monitoring both the availability and usage of key pages on your site is an ideal way to simulate user behavior and test the entire Nginx stack.  An HTML or ping monitor is an ideal way to test the availability of key pages from the outside in and a monitor focused on the request volume for those pages pairs well to monitor from the inside out.

To avoid mixing concerns (and testing your entire application vs. just Nginx), we recommend isolating these monitors to pages that only Nginx responds to. Pages that are handled by upstream servers should be monitored separately.

  * **Monitor** key pages for 2xx (successful) or 3xx (redirect) responses.  This should be done with an external HTML / ping service that contacts your Nginx server.
  * **Set an alert** if the response code changes or is otherwise not as expected.
  * **Monitor** the request volume of these key pages.  This should be done internally by inspecting the Nginx logs.
  * **Set an alert** if request volume for these pages varies from your [calculated threshold]().  Set an initial threshold based on historic page volume (drawn from your logs or as much activity history as you have access to.)

### External Dependencies

#### 14. DNS expiration

[Don't be like Microsoft](http://news.cnet.com/2100-1023-234907.html).  If users can't reach your site because of a DNS failure, you're in trouble.  Your domain registrar will probably start bugging you about the renewal of your domain names at the 60- and 30-day marks, but in case those alerts are missed (or go to someone else's email address), your monitoring tools can be a valuable backstop.

  * **Monitor** domain name expiration dates.  
  * **Set an alert** 30 days from the expiration date (just to be on the safe side!)  

#### 15.  SSL Certificate expiration

Similar to above, the Certificate Authority from whom you purchased your certs will likely bug you plenty as your expiration date near, but a backstop on your monitoring platform is a good idea.  

  * **Monitor** SSL certificate expiration dates.  
  * **Set an alert** 30 days from the expiration date.  We recommend a 30-day alert in particular because certificate issuance/renewal can take longer than other services (in the case of Extended Validation certificates, for example.)

## A Note on Higher-Level Alerts & Duplication

One of the side effects of the layered approach is that because each layer encompasses its preceeding layers, there will be some level of duplication of alerts when higher-layers events trigger.

Your hosting provider failing, for example, will trigger alerts in every layer of the stack.  But this is ok, because those higher-level alerts will provide details that will help you pinpoint the issue more quickly.  

Your RPS may go to 0 for several reasons - and you'll want to know about it regardless - but if it's because of a hosting failure, that extra alert will prevent your wasting time hunting down other potential causes.  

## Digging in Deeper

These 15 metrics constitute the most critical Nginx metrics, and if you monitor and alert as we've outlined above, you can rest eas(ier) knowing you've covered a majority of failure scenarios. Of course every environment is different, so your specific needs may vary (and batteries won't be included, etc. etc.)  

Be sure to check out our [In-Depth Guide to Nginx Metrics]() to learn about the complete list of measurable Nginx metrics so you can customize your monitoring to suit.

And finally, once again, we want your feedback!  If there's something in this guide that's incorrect or if we've left out a glaring error, drop us a comment below or submit a pull request and contribute to the improvement of this guide.

Happy monitoring.