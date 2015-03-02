# An In-Depth Guide to Nginx Metrics

In our guides [Zen and the Art of System Monitoring](zen-and-the-art-of-system-monitoring.md) and [How to Monitor Nginx: The Essential Guide](how-to-monitor-nginx-the-essential-guide.md), we cover our monitoring philosophy and recommend a specific set of metrics to monitor and alerts to set for maximum Nginx happiness.

Here, we'd like to dive into the nitty-gritty of those essential Nginx metrics and discuss more about what exactly they mean and why they're important.  This will also serve as a primer for those of you who want a bit more familiarity with some of the (perhaps esoteric) terminology associated with web servers.

You can think of this as a companion to both the [official Nginx documentation](http://nginx.org/en/docs/) and an appendix to our [Nginx monitoring guide](how-to-monitor-nginx-the-essential-guide.md).

For now, this guide covers only the metrics available via [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html) and variables associated with the F/OSS version of Nginx.  More comprehensive metrics are available via [`ngx_http_status_module`](http://nginx.org/en/docs/http/ngx_http_status_module.html), which is included with the commercial version (Nginx Plus).  A later revision of this guide will be expanded to include those Nginx Plus metrics.

So roll up your sleeves, grab your Slanket or Snuggie, and let's talk Nginx metrics...

## Data Sources

Metrics are available from two sources:

  *  **Nginx Status Modules** - The most direct way to get the goods.  Data is available either through polling a configurable web page (such as /status) or via embedded variables that can be output to log files.  Note:  Polling is the preferred method of access, as Nginx does not provide embedded variables for all of the status module.

  *  **Log Files** - Nginx, like most web servers, maintains an "access log" with a record of each request handled.  Additional metrics can be synthesized from the access log if your monitoring tools are able to analyze log data. You can use the [`log_format`](http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format) directive to instruct Nginx to include [variable values](#variables) in each log record.  Your monitoring tools can then use those variables values as the inputs for your synthetic metrics.

## <a name="connections"></a> It's All About The Connections 

The status module metrics are mostly about **connections**, so it's worth diving into some depth about what those are exactly.

An Nginx server generally plays two roles.  Its primary role, as the name "server" would imply, is to serve content (web pages, images, video, or raw API data) to clients.  Clients historically have been web browsers, but now include native mobile apps and other servers that consume your APIs.

Nginx's secondary role is as middleman, or proxy between clients and "upstream servers".  Upstream servers do more of the heavy lifting for a given request.  They run application servers (like Unicorn for Ruby on Rails, or Tomcat for Java) and generally handle the "dynamic" part of dynamic web pages.

As a proxy, Nginx can do a few things really, really well. It can serve local static content (images and non-dynamic HTML files) super quickly, it can act as a cache for content from upstream servers, and it can load-balance requests among several upstream servers.

All this communication by and between clients, Nginx, and upstream servers is done through **connections** -- the network-level establishment of formal communications between two systems.  Once a network-level connection is established, application-level data can flow via the higher-level protocols that Nginx supports (most commonly HTTP, but also WebSockets, SPDY, TCP streams, and several mail protocols.)

Connections are handled one layer down from the application data (at the TCP layer of networking) and negotiated through a "handshake" process.  The details of the TCP protocol & handshake are beyond the scope of this article, but some great details are available [on Wikipedia](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment) if you're so inclined.  For even more on the specifics of connections and how they relate to HTTP, see [A Software Developer's Guide to HTTP - Connections](http://odetocode.com/articles/743.aspx)

Nginx connections can be in one of several states:

  *  **Accepted** - After a completed TCP handshake between the client and Nginx, a connection is considered Accepted.  It then takes one of 3 sub-states:

    *  **Idle / Waiting** - no data is currently being received by or sent to the client.  A connection will be in this state under two circumstances:  First, between the completion of the request and the initiation of the response; second, between a completed response and the next request.  As of HTTP 1.1, all connections are considered persistent unless declared otherwise, so a connection will remain open until it is closed or times out.

    *  **Reading** - Nginx is reading data from the client.

    *  **Writing** - Nginx is writing data to the client.

  *  **Handled** - Nginx has finished writing data to the client and the request has been completed successfully and closed.
  
  *  **Dropped** - Nginx ended the connection prior to successfully completing the request.  (This usually happens because of a resource or configuration limit.)

## Nginx Status Module Metrics

With this background information in mind, let's take a look at the metrics available via [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html):

-------
  **`Active connections`** <a name="active_connections"></a> - The _current_ number of active (accepted) connections from clients, which includes all connections with the status **Idle / Waiting**, **Reading**, and **Writing**.

-------
  **`accepts`** <a name="accepts"></a> - The _total_ number of accepted connections from clients since the Nginx master process started.  Note that reloading configuration or restarting worker processes does not reset this metric, but terminating and restarting the master process does.

-------
  **`handled`** <a name="handled"></a> - The _total_ number of handled connections from clients since the Nginx master process started.  This will be lower than `accepts` only in cases where a connection is dropped before it is handled.

  * **Open Question**:  Will `handled` be lower than `accepts` by the number of currently "in-flight" requests that are being processed by upstream servers?

-------
 **`requests`** <a name="requests"></a> - The _total_ number of requests from clients since the Nginx master process started.  A request is an application-level (HTTP, SPDY, etc.) event and is defined as a client requesting a resource via the application protocol.  A single connection can (and often does) make multiple requests, so this number will generally be larger than the number of accepted/handled connections.

-------
  **`Reading`** - The _current_ number of (accepted) connections from clients where Nginx is reading the request (at the time the status module was queried.)

-------
  **`Writing`** - The _current_ number of connections from clients where Nginx is writing a response back to the client.

-------
  **`Waiting`** - The _current_ number of connections from clients that are in the **Idle / Waiting** state (waiting for a request.)

-------
## <a name="variables"></a> Selected Log File Variables

It's beyond the scope of this guide to dive into _every_ Nginx log variable.  Instead, we're going to take a close look at a few variables that are of particular interest in the context of monitoring.

### Default Variables

The default Nginx access log (obtained by declaring `log_format combined` in your configuration file) uses the following variables:

  **`$body_bytes_sent`** - The number of bytes sent to the client as the response body (not including the response header).  This allows you to monitor individual response sizes. In aggregate, it also gives you a rough measure of outbound bandwidth.

-------
  **`$http_referer`** - The `HTTP Referer` header from the incoming HTTP request.  This is determined by the requesting browser and identifies the URL of the page that linked to the resource being requested.  Two interesting notes on this:

  1.  The word "referrer", at least in English, is spelled with two Rs, but the original misspelling from the HTTP spec ([RFC 1945](http://tools.ietf.org/html/rfc1945)) managed to stick around.

  2.  Since the header is set by the client making the request, it can't always be trusted.  Lately this has resulted in what's known as "referrer spam", where malicious clients will spoof their referrer headers to instead show spammy websites for consumers of analytics or monitoring software to see, be tempted by, and (they hope), visit.
  
-------
  **`$http_user_agent`** - The `User-Agent` header from the incoming HTTP request.  This identifies the specific browser, bot, or other software that issued the request, and may also include other information such as the client's operating system.  The format is slightly different between human-operated web browsers and automated agents (bots), but the theme is the same.  [Wikipedia has some awesome nitty-gritty details on user agent strings.](http://en.wikipedia.org/wiki/User_agent)

-------  
  **`$remote_addr`** - The [IP address](http://en.wikipedia.org/wiki/IP_address) of the client making the request.  If the request passed through an intermediate device, such as a NAT firewall, web proxy, or your load balancer, this will be the address of the last device to relay the request.
 
------- 

  **`$remote_user`** - The username supplied if [HTTP Basic authentication](http://en.wikipedia.org/wiki/Basic_access_authentication) is used for the request.
  
-------  
  **`$request`** - The raw HTTP request line.  An example of a (familiar) request line is as follows:

    GET /community/guides/an-in-depth-guide-to-nginx-metrics/ HTTP/1.1  

This is actually a compound variable, composed of 3 sub-variables (each of which is accessible individually if needed):

    $request_method $request_uri $server_protocol
 
A breakdown of these variables:

  * **`$request_method`** - The [HTTP method](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) of the request.  The most common methods used by browsers are GET and POST, but the spec also includes HEAD, PUT, DELETE, OPTIONS, TRACE and CONNECT (details on each available at the preceding link.)

  * **`$request_uri`** - The URI of the requested page, including query arguments.  Note that if the ultimate resource returned is different from the one requested (due to use of a module like mod_rewrite), this field will still log the originally requested URI.

  * **`$server_protocol`** - The application-level protocol and version used in the request.  You'll most commonly see HTTP/1.0 or HTTP/1.1, but Nginx supports SPDY, WebSockets, and several mail protocols as well.  

------- 
  **`$time_local`** - The local (server) time the request was received, in the format `dd/mm/yyyy:hh:mm:ss UTC-offset`.
 
-------   
  **`$status`** - The numeric [HTTP Status Code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) of the response.  This is an important variable to monitor, as it provides information regarding errors, missing pages, and other unusual events.

------- 
### Other Variables of Interest

These variables are not included in the default combined log format.

-------
  **`$bytes_sent`** - The total number of bytes sent to the client in the response, including headers.  This is similar to body_bytes_sent, but provides a more complete picture.

------- 
  **`$connection`** - The connection serial number.  This is a unique number assigned by nginx to each connection.  If multiple requests are received on a single connection, they will all have the same connection serial number.  Serial numbers reset when the master Nginx process is terminated, so they will not be unique over long periods of time.

------- 
  **`$connection_requests`** - The number of requests made through this `$connection`.

------- 
  **`$content_length`** - the HTTP Content-Length request header field.  This is the total size (in bytes) of the body of the request being made by the client, as reported by the client.

  **`$request_length`** - the full request length (in bytes) - including the request line, header, and body, as calculated by Nginx.  

If you're interested in monitoring overall incoming bandwidth, use `$request_length`.  Because `$content_length` is drawn from a request header, it's calculated by the client and, therefore, has the potential to be spoofed (in the case of a DDoS attack, for example.)

------- 
  **`$gzip_ratio`** - The compression ratio of the response (the ratio between the original and compressed response sizes.)  Applicable if you have enabled [gzip](http://www.gzip.org) response compression.

Gzip compression is a feature that Nginx provides (through [`ngx_http_gzip_module`](http://nginx.org/en/docs/http/ngx_http_gzip_module.html)) that pipes responses through gzip before sending them to the client.  This can reduce the size of responses by 50% or more and provide a significant outbound bandwidth savings.  Gzip is extremely fast, but there is still material overhead in the compression process (both in terms of CPU usage and response time), though some of this overhead is recovered in the transfer time savings from the smaller file.

It's a delicate balance to strike so be sure to monitor your resource usage closely if you decide to use gzip compression.

Further details about compression and decompression [are available from Nginx directly](http://nginx.com/resources/admin-guide/compression-and-decompression/).

------- 
  **`$host`** - The DNS name that the client used to find your server (as presented in the `Host` HTTP header). If that header is empty or missing, Nginx substitutes the name in the first `server_name` directive in your Nginx configuration.

------- 
  **`$http_HEADERNAME`** and **`$upstream_http_HEADERNAME`** - Following the pattern of `$http_referer` and `$http_user_agent` above, Nginx allows you to log any [HTTP request headers](http://en.wikipedia.org/wiki/List_of_HTTP_header_fields) by referencing `$http_` and the header name (converted to lowercase, with dashes replaced by underscores).

Similarly, you can access the headers as returned by any upstream servers by appending "upstream_" to the front of the desired header name.

------- 
  **`$msec`** - The current time, in milliseconds from the Unix epoch of 1/1/1970.  This allows you to determine the exact time at which a request took place.

  * **Open Question**:  Is this the time the request was received, or the time that the log record was emitted (i.e. the time the request was completed)?

------- 
  **`$pid`** - The process ID (PID) of the Nginx worker that handled the request.  This can be used to track the workload and associated metrics of each worker individually.

Note that worker processes can (and do) crash or be reaped and restarted by the Nginx master process, so PIDs can come and go.  Eventually, an old PID may even be reused.

------- 
  **`$request_time`** <a name="request_time"></a> and **`$upstream_response_time`** - `$request_time` is the total time taken for Nginx (and any upstream servers) to process a request and send a response.  Time is measured in seconds, with millisecond resolution.  This is the primary source you should use for your server's response time metric.

The clock starts as soon as the first bytes are read from a client and stops after the last bytes have been sent.  Note that this _includes_ the processing time for upstream servers -- if you're interested in breaking out those metrics then you can use `$upstream_response_time`, which only measures the response time of the upstream server.

To clarify:  The language can be a bit confusing here -- even though the variable is "request" time, it actually measures the elapsed time of the full request-response cycle (from the Nginx server's perspective.)

------- 
**`$server_addr`** and **`$server_name`** - The IP address (`$server_addr`) or name (`$server_name`) of the Nginx server that accepted a request.  This is useful in a multi-server (load-balanced) environment when you'll need to monitor which requests (and, therefore, which metrics) are handled by each server.

Note:  Computation of the IP address requires a system call unless you specifically bind an address using the `listen` directive in your configuration.  Keep this in mind as the addition of system calls can add significant overhead and impact server performance accordingly.  For this reason, $server_name may be a better choice for many installations unless the full IP is specifically required. 

------- 
**`$uri`** - The current URI of the request.  Internal to Nginx, this value may change during request processing (i.e. in the case of rewrites).  The value that is logged represents the URI of the resource that was ultimately sent to the client.

Tis differs from `$request_uri` in that `$request_uri` does not reflect any URL rewrites internal to the Nginx server.

------- 
