# How to Monitor Nginx:  The Complete Guide

Nginx is an increasingly popular open-source HTTP and reverse proxy server that's known for its high concurrency, high performance, and low memory usage.  It's become the second most popular public-facing web server (as of December 2014) among the top 1M busiest sites online, and it's pretty awesome.

But, like any piece of long-running software (or a small child), it can get into trouble if left completely unattended.  

This guide will take you through a series of recommendations on how to properly setup monitoring and alerting for a production Nginx deployment.  It will make no recommendations on how to monitor a small child.

While Nginx's open source variant (Nginx F/OSS, or "plain 'ol Nginx") is the most popular, a commercial version (Nginx Plus) is also available and offers load balancing, session persistence, advanced management, and finer-grained monitoring metrics.  This guide uses "Nginx" in the universal sense and refers to both versions.  The metrics discussed in this guide are also available in both versions.

## Why Write This Guide?

While there's a body of conventional wisdom and some basic material on the web, we found no definitive resource on the subject, so we decided to produce one.  

Scalyr's monitoring approach is based on years of frontline operations experience and we're confident this approach will help you more quickly detect problems, efficiently spot incipient / slow-burn issues before they become full-scale outages, and ultimately save you (and your users) headaches.  

## Background Material

This guide can be read as-is and is designed to provide quick, actionable recommendations.

If you want to dig deeper, take a spin through [Zen and the Art of Server Monitoring](zen-and-the-art-of-server-monitoring.md).  It explores, in-depth, our strategy and philosophy.

Next, you can read about [How to Set Alerts](how-to-set-alerts.md) to get a deeper understanding of how to build intelligent alerts and set notification thresholds properly.  The primary goal there is to strike the right balance between false positives and missed incidents.

Finally, take a look at our [In-Depth Guide to Nginx Metrics](an-in-depth-guide-to-nginx-metrics.md).  Think of that as an addendum / appendix to this guide.  It examines the complete list of available Nginx metrics, what exactly they measure, and what exactly they mean.

## The 16 Essential Nginx Metrics to Monitor

We recommend a [layered approach](zen-and-the-art-of-server-monitoring.md#layers) to monitoring, starting from the application layer and moving up through process, server, hosting provider, external services, and user activity.

### 1.  Requests Per Second (RPS)

The essential job of Nginx is to serve content to client devices.  That content is delivered in response to requests from those clients, so it makes sense that the first metric we care about is **requests per second** - the rate at which Nginx is doing its job. Spikes in RPS can indicate benign events (increased customer activity) or malignant ones (a DDoS attack).  Drops in RPS can indicate a problem elsewhere in the stack.

  * **Monitor** requests per second (RPS) by sampling [`requests`](an-in-depth-guide-to-nginx-metrics.md#requests) from [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html).
  * **Set an alert** if RPS crosses a threshold based your traffic pattern.  We generally recommend a [historical threshold](how-to-set-alerts.md#historical-threshold) for cyclical metrics like RPS, but your needs may vary.  See [How to Set Alerts](how-to-set-alerts.md) for details on finding the right value.

### 2.  Response Time

Response time measures how quickly requests are being handled - one of the primary indicators of application performance.  Depending on the metric you m

  * **Monitor** response time by logging [`$request_time`](an-in-depth-guide-to-nginx-metrics#request_time).  This measures the elapsed time between Nginx receiving the first byte from a client's request and its completion of a log write following a response (this includes the time needed for upstream servers to respond.)  If you only need to measure upstream server performance, you can log [`$upstream_response_time`](an-in-depth-guide-to-nginx-metrics#request_time).
  * **Set an alert** if response time crosses a [historical threshold](how-to-set-alerts.md#historical-threshold) based on average application response time.  Pick a baseline based on recent data, and fine-tune as you get a better sense of your performance pattern.  Note:  Though it may seem counter-intuitive, you should actually care if response time suddenly drops.  Servers don't just start super-performing, so a sudden drop could mean an upstream process has failed.

### 3.  Active Connections

There is a (configurable) hard limit on the total number of [connections](an-in-depth-guide-to-nginx-metrics.md#connections) that Nginx can handle, so both knowing that limit and monitoring the current number of active connections is important.

The limit is equal to the # of [`worker_connections`](http://nginx.org/en/docs/ngx_core_module.html#worker_connections) * [`worker_processes`](http://nginx.org/en/docs/ngx_core_module.html#worker_processes).  Once the limit is reached, connections will be dropped and performance will drop.  Note that this limit includes all connections (from clients and to upstream servers), so be sure to set a high enough limit relative to the hardware and expected server load.

  * **Monitor** active connections by reading [`Active connections`](an-in-depth-guide-to-nginx-metrics.md#active_connections) from [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html).
  * **Set an alert** as your active connections approach the maximum connection limit.  We recommend setting the alert to 70% of the limit to give yourself room to adjust without setting off false positives.
  * **Monitor** both connections accepted and connections handled by reading [`accepts`](an-in-depth-guide-to-nginx-metrics.md#accepts) and [`handled`](an-in-depth-guide-to-nginx-metrics.md#handled) respectively from [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html).  Under normal circumstances, these should be equal.
  * **Set an alert** if [`handled`](an-in-depth-guide-to-nginx-metrics.md#handled) falls below [`accepts`](an-in-depth-guide-to-nginx-metrics.md#accepts) - this means that Nginx is dropping connections before completion and is an indication that a resource limit has been reached.

### 4.  Response Codes

Nginx logs the [HTTP response code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) returned for each request, and this can be a rich source of information about the health of both Nginx and your upstream servers. 

  * **Monitor** the Nginx access logs for the relative distribution of 2xx/3xx/4xx/5xx response codes.  Use this distribution to form a baseline threshold. 
  * **Set an alert** if 2xx or 3xx responses drop below that baseline.
  * **Set an alert** if 4xx or 5xx responses rise above that baseline.

### 5.  Process State

Nginx spawns multiple OS processes - a master process and a separate process for each worker.  If you've enabled caching, there's an additional process for that too...("You get a process!  And you get a process!  And YOU get a process!")  It's critical to keep an eye on these processes and make sure they stay healthy and consume an available amount of resources.

  * **Monitor** the uptime for each process through a process monitoring agent.  Uptime should be continuously-increasing so long as the process is active.
  * **Monitor** each process' status.  Possible states vary according to UNIX flavor, but generally are one of `Running`, `Sleeping`, `Idle`, `Defunct`/`Zombie`, or `Stopped`/`Exited`.
  * **Set an alert** if the master process leaves the `running` state.  Note that if a worker process terminates (due to a configuration change or crash), the master process will handle restarting it automatically, so it's less-important to alert on failure of those workers (though you may still want to know about them in case they're symptoms of another issue.)

### 6.  Process CPU usage

Web servers are constrained by CPU and network bandwidth more than any other system resource, so those are the most important to monitor.

  * **Monitor** CPU usage of each process.  Usage should scale linearly with connection load.  Make sure you configure the number of [`worker_processes`](http://nginx.org/en/docs/ngx_core_module.html#worker_processes) properly (the general rule is one per CPU core.)  
  * **Set an alert** if CPU usage deviates from its historical threshold.  A few hours worth of data should be enough to get a broad picture of your server's CPU usage profile.

### 7.  Process Network Usage

  * **Monitor** network usage of each process.  You'll want to look for both relative variations between worker processes (which could indicate an issue with load balancing) and combined Nginx process usage, which will show the application-level usage.  
  * **Set an alert** if a single worker's process usage changes significantly relative to other workers.
  * **Set an alert** when the combined network usage of all Nginx processes changes over a historical threshold.  

### 8.  Process Open File Handles

The number of simultaneous connections (including client connections and upstream server connections) cannot exceed Nginx's limit on the maximum number of open files, configured using the [`worker_rlimit_nofile`](http://nginx.org/en/docs/ngx_core_module.html#worker_rlimit_nofile) directive.  By monitoring the number of open files relative to the limit, you'll have an idea of how much headroom your system has for additional connections.

  * **Monitor** the number of open file handles for each process.
  * **Set an alert** when the number of open file handles reaches 70% of the [`worker_rlimit_nofile`](http://nginx.org/en/docs/ngx_core_module.html#worker_rlimit_nofile) limit - this means it's time to increase the limit before it reaches capacity and connections are dropped.  Tune this % as needed to minimize noisy alerts and to suit your deployment.

### 9.  Server Status 

Complete server monitoring is topic beyond the scope of this guide, but we'll cover a few key high-level metrics.

First, the big simple picture - you want to monitor each box's status.  Whether you're running Nginx on dedicated hardware or a cloud-based VPS, your hosting provider will likely provide you with an overall status indication.

  * **Monitor** Server status.
  * **Set an alert** when the server leaves the "running" state.

### 10.  Server Load Average

Load average is a compound metric provided by UNIX systems that combines overall CPU & disk usage into a single floating point number.  It measures how "loaded" the system is - how many processes are using vs. waiting for CPU or disk resources.  On a single-core system, a 5-min load average of 0.5 means that over the past 5 minutes, the CPU/Disk were idle 50% of the time and used 50% of the time.  A load average of 1.0 means the CPU/Disk were fully used by a process.  A load average of 2.0 means the CPU/Disk were fully used by a process _and_ a second process was waiting for use.  In other words, the system was "overloaded" by a factor of 2.

Note that on some systems load average does not scale automatically for multi-core CPUs.  So a 4-core CPU running at a 4.0 load average means that each core was fully used, not that the system was overloaded.  Keep this in mind when setting load average alerts.

  * **Monitor** Server Load Average over 1, 5 and 15 minutes.
  * **Set an alert** when Load Average over these periods deviates from a historical threshold.  Make sure you account for the number of logical cores your system has when setting the threshold.

### 11.  Server Network Usage

While process-level network usage is important to monitor for application performance, server-level network usage is also critical to see the bigger picture.  If other applications on the server eat up available network capacity, your Nginx application may be starved for bandwidth and process-level monitors won't catch those issues.

  * **Monitor** Network usage system-wide.
  * **Set an alert** if network usage crosses a fixed threshold relative to total network capacity.  Your specific system's threshold will vary according to setup -- some may run at full network capacity as a matter of course (in which case you might want to know if network usage drops, which could indicate problems.)  You will generally want to monitor a smoothed traffic graph over a recent time period (i.e., 15 minutes) to filter out short bursts of activity.

### 12.  Server Disk Space

Running out of disk space is a common source of "mystery" system failures.  Logs can grow rapidly during periods of high usage (or DDoS attacks) and if they eat up all available space, chaos can ensue.

  * **Monitor** available disk space.  If your system has multiple disks, it's particularly important to monitor partitions that contain logs, caches, or other mutable data.
  * **Set an alert** as disk usage increases.  Depending on how rapidly space can fill up, we recommend setting multiple [escalating](how-to-set-alerts.md#notification) alerts at 50%, 75%, and 90% capacity marks.

### 13.  Hosting Provider status.

Your servers live within a hosting provider, so you'll want to know about big connectivity or availability issues.

  * **Monitor** Hosting provider status.  How will depend on your provider -- some will report status via an API while others will simply provide a status page (usually something like 'status.yourprovider.com').  Configure your monitoring tools to watch and report on the appropriate status endpoint.
  * **Set an alert** if the provider's connectivity or availability changes. 

### 14. DNS expiration

[Don't be like Microsoft](http://news.cnet.com/2100-1023-234907.html)!  If users can't reach your site because of a DNS failure, you're in trouble.  Your domain registrar will probably start bugging you about renewal at the 60- and 30-day marks, but in case those alerts are missed (or go to someone else's email address), your monitoring tools can be a valuable backstop.

  * **Monitor** domain name expiration dates.  
  * **Set an alert** 30 days from the expiration date to give yourself ample time to renew.

### 15.  SSL Certificate expiration

Likewise, an expired SSL certificate can wreak just as much havoc.  The Certificate Authority from whom you purchased your certs will likely bug you plenty as your expiration dates near, but a backstop on your monitoring platform is also a good idea.  

  * **Monitor** SSL certificate expiration dates.  
  * **Set an alert** 30 days from the expiration date.  We recommend a 30-day alert in particular because certificate issuance/renewal can take longer than other services (in the case of Extended Validation certificates, for example.)

### 16.  User Activity

Monitoring availability of key pages and their corresponding user activity is the last and perhaps most important part of the Nginx monitoring picture.  We suggest a two-pronged approach:

  1.  An HTML or ping test to monitor the availability of key pages from the outside; and 
  2.  Log analysis to monitor request volume and activity of key pages from the inside.

We recommend monitoring static pages (that only Nginx responds to) as well as dynamic pages handled by upstream servers so you can better isolate any issues.

  * **Monitor** key pages for 2xx (successful) or 3xx (redirect) responses.  This should be done with an external HTML / ping service that contacts your Nginx server.
  * **Set an alert** if response codes change or are otherwise not as-expected.
  * **Monitor** the request volume of key pages.  This should be done internally by inspecting the Nginx logs.
  * **Set an alert** if request volume changes significantly.  Set an initial threshold based on historic page volume (drawn from your logs or as much activity history as you have access to) and iterate accordingly.

## A Note on Higher-Level Alerts & Duplication

One of the side effects of our layered approach is that there will be some level of alert duplication when higher-layer events trigger.

A failure of your hosting provider, for example, will trigger alerts in every layer of the stack beneath it (server status, request volume, etc.) But this is ok because those higher-level alerts will provide details that will help you pinpoint the issue more quickly.  

Your RPS may go to 0 for several reasons - and you'll want to know about it regardless - but if it's because of a hosting failure, that extra alert will prevent your wasting time chasing other potential causes.

## Digging in Deeper

These 16 metrics constitute the most critical Nginx metrics, and if you monitor and alert as we've outlined above, you can rest easy(er) knowing you've covered a majority of failure scenarios. Of course every environment is different, so your specific needs may vary (and batteries won't be included, etc. etc.)  

Be sure to check out our [In-Depth Guide to Nginx Metrics](an-in-depth-guide-to-nginx-metrics.md) to learn about the complete list of measurable Nginx metrics so you can customize your monitoring to suit.

And finally, once again, if there's something in this guide that's incorrect or if we've left out a glaring error, drop us a comment below or submit a pull request and contribute to the improvement of this guide.

Happy monitoring.