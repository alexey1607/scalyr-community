# How to Monitor NGINX:  The Essential Guide

NGINX is an increasingly popular open-source HTTP and reverse proxy server that's known for its high concurrency, high performance, and low memory usage.  It's become the second most popular public-facing web server (as of December 2014) among the top 1M busiest sites online, and it's pretty awesome.

But, like any long-running software (or a small child), it can get into trouble if left completely unattended.

This guide will take you through a series of recommendations and best practices for monitoring a production NGINX deployment.  It will make no recommendations as to how to monitor a small child.

While NGINX's open source variant (NGINX F/OSS, or "plain 'ol NGINX") is the most popular, a commercial version (NGINX Plus) is also available and offers load balancing, session persistence, advanced management, and finer-grained monitoring metrics.  This guide uses "NGINX" in the universal sense and refers to both versions.  The metrics discussed in this guide are also available in both versions.

## Why Write This Guide?

While there's a body of conventional wisdom and some basic material on the web, we found no definitive resource on the subject.

At Scalyr, our approach to monitoring is based on years of frontline operational experience.  Good monitoring will help you more detect problems more quickly, spot incipient / slow-burn issues before they become full-scale outages, and ultimately save you (and your users) headaches.

## Background Material

This guide can be read as-is and is designed to provide quick, actionable recommendations and best practices.

If you want to dig deeper, take a spin through [Zen and the Art of System Monitoring](zen-and-the-art-of-system-monitoring.md).  It will give you a systematic framework for thinking about system monitoring.

Next, read [How to Set Alerts](how-to-set-alerts.md) to get a deeper understanding of how to build intelligent alerts and set notification thresholds properly.  The primary goal here is to minimize false alarms without missing real incidents.

Finally, take a look at our [In-Depth Guide to NGINX Metrics](an-in-depth-guide-to-nginx-metrics.md).  Think of that as an addendum / appendix to this guide.  It examines the complete list of available NGINX metrics, what exactly they measure, and what exactly they mean.

## The 15 Essential NGINX Metrics to Monitor

We recommend a [layered approach](zen-and-the-art-of-system-monitoring.md#layers) to monitoring, starting from the application layer and moving down through process, server, hosting provider, external services, and user activity.  By monitoring the metrics listed here, you'll get good coverage for both active and incipient problems with your NGINX site.

### 1.  Requests Per Second (RPS)

The essential job of NGINX is to serve content to client devices.  That content is delivered in response to requests from those clients, so it makes sense that the first metric we care about is **requests per second** - the rate at which NGINX is doing its job. 

Spikes in RPS can indicate benign events (increased customer activity) or malignant ones (a DDoS attack).  A spike can also be connected to errors from upstream servers -- if NGINX is load balancing a set of servers and those servers go down, NGINX will return errors very quickly.  

Drops in RPS, on the other hand, can be signs of network connectivity issues or saturation of an essential system resource like CPU or RAM.

Whatever the cause - significant changes in RPS are events you'll want to know about and investigate further.

  * **Monitor** requests per second (RPS) by sampling [`requests`](an-in-depth-guide-to-nginx-metrics.md#requests) from [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html).
  * **Set an alert** if RPS crosses a threshold based your traffic pattern.  We generally recommend a [historical threshold](how-to-set-alerts.md#historical-threshold) for cyclical metrics like RPS, but your needs may vary.  See [How to Set Alerts](how-to-set-alerts.md) for details on finding the right value.  It's best to set two alerts: one for abnormally high traffic, and one for abnormally low traffic.

### 2.  Response Time

Response time measures how quickly requests are being handled - one of the primary indicators of application performance.

  * **Monitor** response time by adding the [`$request_time`](an-in-depth-guide-to-nginx-metrics#request_time) variable to your NGINX log configuration.  This measures the elapsed time for NGINX to receive the full client request, process the request, and transmit the response.
  * **Set an alert** if response time exceeds a fixed threshold that makes sense for your application.  Pick a baseline based on your site's observed performance, and fine-tune as needed.  Response time can fluctuate with changes in the pattern of requests to your site, so you may need to use a fairly loose threshold.  For additional coverage, if your monitoring tool supports it, you might consider adding alerts on median response time, 99th percentile response time, or response time for specific pages.

### 3.  Active Connections

There is a (configurable) hard limit on the total number of [connections](an-in-depth-guide-to-nginx-metrics.md#connections) that NGINX can handle.  It's important to know that limit and alert before the limit is reached.

The limit is equal to the # of [`worker_connections`](http://nginx.org/en/docs/ngx_core_module.html#worker_connections) * [`worker_processes`](http://nginx.org/en/docs/ngx_core_module.html#worker_processes) in your NGINX configuration.  Once the limit is reached, connections will be dropped and users will see errors.  Note that this limit includes all connections (from clients and to upstream servers).

  * **Monitor** active connections by reading [`Active connections`](an-in-depth-guide-to-nginx-metrics.md#active_connections) from [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html).
  * **Set an alert** as your active connections approach the maximum connection limit.  We recommend setting the alert to 70% of the limit to give yourself room to adjust without setting off false positives.
  * **Monitor** both connections accepted and connections handled by reading [`accepts`](an-in-depth-guide-to-nginx-metrics.md#accepts) and [`handled`](an-in-depth-guide-to-nginx-metrics.md#handled) respectively from [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html).  Under normal circumstances, these should be equal.
  * **Set an alert** if [`handled`](an-in-depth-guide-to-nginx-metrics.md#handled) falls below [`accepts`](an-in-depth-guide-to-nginx-metrics.md#accepts) - this means that NGINX is dropping connections before completion and is an indication that a resource limit has been reached.

### 4.  Connection Backlog Queue

NGINX accepts connections very quickly, but in extremely high-traffic situations, a connection backlog can still happen at the system level (which is a distinct bottleneck from the application-level connection handling described in #3 above.)  When this occurs, new connections will be refused.

  * **Monitor** the syslog for backlog-related error messages
  * **Monitor** the output of `netstat -s` for "SYNs to LISTEN sockets dropped” and "times the listen queue of a socket overflowed" values.
  * **Set an alert** if a backlog queue begins to form.  

The connection queue size can be increased by modifying the `somaxconn` and `tcp_max_syn_backlog` kernel variables.  Details of those are beyond the scope of this article, but [this kernel.org documentaion](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) contains more information. 

### 5.  Response Codes

NGINX logs the [HTTP response code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) returned for each request, and this can be a rich source of information about the health of both NGINX and your upstream servers. 

  * **Monitor** the NGINX access logs for the relative distribution of 2xx/3xx/4xx/5xx response codes.  Use this distribution to determine the typical fraction of 4xx and 5xx responses served by your site.
  * **Set an alert** if the fraction of 5xx responses rises above a fixed threshold.  The appropriate threshold is very depending on the nature of your site, so you'll have to look at historical data to set an appropriate threshold.  Use the techniques in [How to Set Alerts](how-to-set-alerts.md); in particular: start with a relatively tight threshold, adjust the threshold if there are too many false alarms, and consider adding a grace period to ignore brief spikes.
  * **Set an alert** if the fraction of 4xx responses rises above a fixed threshold.  In theory, a 4xx response is the user's fault, not the fault of your server.  However, if users are suddenly requesting missing pages, there's a good chance it's due to something you've done (or at least something you'd like to know about.)

### 6.  Process Open File Handles

NGINX uses a system [file descriptor](http://en.wikipedia.org/wiki/File_descriptor) for each connection it handles - one for each client and one for each upstream connection.  The number of simultaneous connections, therefore, cannot exceed the system's limit on open files.  When there are no more file handles available, NGINX drop new connection requests.  

The maximum number of open files can is defined in several places:

  * **System Limit**:  `sys.fs.file_max` (or a close variation thereof depending on your UNIX flavor) is a kernel variable that defines the system-wide maximum number of open files.
  * **User Limit**: `/etc/security/limits.conf` defines a `nofile` entry which sets the maximum number of open files per user.
  * **Process Limit**: `ulimit -n` defines a per-process limit.

You'll want to know each of these values, but the smallest one will be the limiting value.  By monitoring the number of open files relative to this limiting value, you'll know how much headroom your system has for additional connections.

  * **Monitor** the number of open file handles for each process.
  * **Set an alert** when the number of open file handles reaches 70% of the smallest limit.  This means it's time to increase the limit before it reaches capacity and connections are dropped.

A note on increasing the limit:  If you change the system / user / process limit, you'll normally have to restart the NGINX master process to apply the change.  If for some reason you cannot allow a restart, NGINX provides a workaround in the [`worker_rlimit_nofile`](http://nginx.org/en/docs/ngx_core_module.html#worker_rlimit_nofile) directive.  You can change the value of this directive to match the new system / user / process limit, do a configuration reload, and NGINX will apply thew new limit without needing a restart.  

### 7.  Process State

NGINX spawns multiple OS processes - a master process and a separate process for each worker.  If you've enabled caching, there's an additional process for that too...("You get a process!  And you get a process!  And YOU get a process!")  It's critical to keep an eye on these processes and make sure they stay healthy.

  * **Monitor** the uptime for each process through a process monitoring agent.  Uptime should be continuously-increasing so long as the process is active.
  * **Monitor** each process' status.  Possible states vary according to UNIX flavor, but generally are one of `Running`, `Sleeping`, `Idle`, `Defunct`/`Zombie`, or `Stopped`/`Exited`.
  * **Set an alert** if the master process leaves the `running` state.  Note that if a worker process terminates, the master process will restart it automatically, so it's less important to alert on failure of a worker process.  (However, worker failure could be a symptom of a deeper issue, so you might still want to monitor it.)

### 8.  Server Status

Complete server monitoring is topic beyond the scope of this guide, but we'll cover a few key high-level metrics.

First, the big simple picture - you want to monitor each box's status.  Whether you're running NGINX on dedicated hardware or a cloud-based VPS, your hosting provider will likely provide you with an overall status indication.

  * **Monitor** Server status.
  * **Set an alert** when the server leaves the "running" state.

### 9.  Server Load Average

Load average is a metric provided by UNIX systems that summarizes CPU and disk usage into a single number.  The number indicates how many threads or processes are attempting to do work at any given time.  Loosely speaking, if the number is equal to the number of CPU cores plus the number of disk drives in your server, then the system is completely occupied.  Higher numbers indicate that the system is overloaded and some threads are waiting.  Lower numbers indicate spare capacity.

This is a loose rule, because it's possible that your CPUs are fully occupied while some disks are idle, or vice versa.  As a rule of thumb, if your system doesn't use much disk I/O, then "fully occupied" load average is equal to the number of CPU cores.  If it does a lot of disk I/O, then add the number of disk rdrives to the number of CPU cores.

Most systems report three metrics: the 1-minute running load average, the 5-minute running average, and the 15-minute running average.  Which of these you should monitor is a matter of taste, but if in doubt, try the 5-minute average.  (The 1-minute average is sensitive to spikes; the 15-minute average can be slow to react to changes.)

  * **Monitor** Server Load Average -- at least one of the three metrics.
  * **Set an alert** when Load Average exceeds a fixed threshold.  The exact threshold depends on the number of CPU cores and/or disk drives in your system, and your tolerance for performance slowdowns.  A reasonable rule of thumb is to alert when the load average exceeds 75% of the maximum capacity of your system -- e.g. 3.0 on a 4-core machine that doesn't do much I/O, or 6.0 on a 4-core, 4-disk machine that does a lot of I/O.

### 10.  Server Network Usage

Most servers are not constrained by network bandwidth, but if you do happen to overuse your network, it can lead to confusing problems that are hard to track down to a root cause.  To properly monitor network usage, you need to understand how much bandwidth you have available.  If your server is primarily communicating over the public Internet, e.g. with browsers on your users' machines, then your effective limit is probably determined by your hosting provider -- not by the 1Gps Ethernet card in the server.

  * **Monitor** Inbound and outbound network usage.
  * **Set an alert** if network usage exceeds a fixed threshold; say, 60% of available bandwidth.  You may want to use a bit of a running average (e.g. over 5 minutes), or a brief grace period, to smooth out spikes.

### 11.  Server Disk Space

Running out of disk space is a common source of "mystery" system failures.  Logs can grow rapidly during periods of high usage (or DDoS attacks) and if they eat up all available space, chaos can ensue.

  * **Monitor** available disk space.  If your system has multiple disks, it's particularly important to monitor partitions that contain logs, caches, or other mutable data.
  * **Set an alert** if free space falls below a fixed threshold.  This can be a good place to use [escalating notifications](how-to-set-alerts.md#notification).  A reasonable rule of thumb is to generate a low-urgency alert (e.g. email message) when the disk is 70% full, and a high-urgency alert (e.g. SMS) when the disk is 90% full.

### 12.  Hosting Provider status.

Your servers live within a hosting provider, so you'll want to know about big connectivity or availability issues.

  * **Monitor** Hosting provider status.  How will depend on your provider -- some will report status via an API while others will simply provide a status page (usually something like 'status.yourprovider.com').  Configure your monitoring tools to watch and report on the appropriate status endpoint.
  * **Set an alert** if the provider experiences an outage.

### 13. DNS expiration

[Don't make Hotmail's mistake](http://news.cnet.com/2100-1023-234907.html)!  If users can't reach your site because of a DNS expiration, you're in trouble.  Your domain registrar will probably start bugging you about renewal at the 60- and 30-day marks, but in case those alerts are missed (or go to someone else's email address), your monitoring tools can be a valuable backstop.

  * **Monitor** domain expiration dates.
  * **Set an alert** 30 days before the expiration date to give yourself ample time to renew.

### 14.  SSL Certificate expiration

An expired SSL certificate can also wreak havoc.  Certificate authorities are not always as proactive with expiration alerts, so it's a good idea to use your monitoring platform as a backstop.

  * **Monitor** SSL certificate expiration dates.  
  * **Set an alert** 60 days before the expiration date.  We recommend a 60-day alert in particular because certificate issuance/renewal can take longer than other services (in the case of Extended Validation certificates, for example.)

### 15.  User Activity

Monitoring availability of key pages and their corresponding user activity is the last and perhaps most important part of the picture.  We suggest a two-pronged approach:

  1.  An HTML or ping test to monitor the availability of key pages from the outside; and 
  2.  Log analysis to monitor request volume and activity of key pages from the inside.

We recommend monitoring static pages (that only NGINX responds to) as well as dynamic pages handled by upstream servers so you can better isolate any issues.

  * **Monitor** key pages for 2xx (successful) or 3xx (redirect) responses.  This should be done with an external HTML / ping service that contacts your NGINX server.
  * **Set an alert** if response codes change or are otherwise not as-expected.  If possible, also use a substring test to verify that the response includes the correct content.
  * **Monitor** the request rate for key pages on your site, by looking for those pages in the NGINX request log.
  * **Set an alert** if request volume drops significantly.  You'll probably want to use a running average to smooth out spikes in user activity; the smaller your site, the more volatile traffic will be.  As a starting point, you might alert if the average request rate over the last 10 minutes, is 40% lower than the average over the preceeding 10 minutes.

## A Note on Lower-Level Alerts & Duplication

One of the side effects of our layered approach is that, when there is a problem at a lower level of the system, alerts may trigger at multiple levels.  For example, if your hosting provider fails, that will trigger alerts at the system, process, and application level.  But by monitoring every level, you'll be able to more quickly hone in on problems.

## Digging in Deeper

With these alerts in place, you can rest easy(er) knowing you've covered a majority of failure scenarios. Of course every environment is different, so your specific needs may vary (and batteries won't be included, etc. etc.)

Be sure to check out our [In-Depth Guide to NGINX Metrics](an-in-depth-guide-to-nginx-metrics.md) to learn about the complete list of measurable NGINX metrics so you can customize your monitoring to suit.

Are there any metrics that you like to monitor that we've left out?  Are there any NGINX monitoring tips or tricks you think we should add?  Let us know in the comments!