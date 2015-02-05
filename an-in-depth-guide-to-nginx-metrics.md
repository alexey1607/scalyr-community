# An In-Depth Guide to Nginx Metrics

In our guides [Zen and the Art of Server Monitoring]() and [How to Monitor Nginx: The Complete Guide](), we cover our monitoring philosophy and recommend a specific series of metrics to monitor and alerts to set for maximum Nginx happiness.

Here, we'd like to dive into the nitty-gritty of the essential Nginx metrics and discuss more about what exactly they mean and why they're important.  This will also serve as a primer for those of you who want a bit more familiarity with some of the (perhaps esoteric) terminology associated with web servers.

You can think of this as a companion to both the official Nginx documentation on metrics (available [here]()) and an appendix to our [Nginx monitoring guide]().

For now, this guide covers only the metrics available via [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html) and variables associated with the F/OSS version of Nginx.  More comprehensive metrics are available via [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html), which is included with the commercial version (Nginx Plus).  A later revision of this guide will be expanded to include those Nginx Plus metrics.

So roll up you sleeves, grab a slanket or snuggie, and let's talk Nginx metrics...

## Data Sources

Metrics are available from two sources:

  *  **Nginx Status Modules** - The most direct way to get the goods.  Data is available either through polling a configurable web page (such as /status) or via embedded variables that can be output to log files.  Note:  Polling is the preferred method of access, as Nginx does not provide embedded variables for all of the metrics available via the status module.

  *  **Log Files** - Additional metrics can be synthesized by reading certain Nginx variables from the logs and performing calculations on them.  This is a slighly less-direct technique and requires an outside service to do those calculations (rates, averages, minimums, maximums, etc.), but it's a perfectly workable approach.  You accomplish this by defining a custom log format using the [`log_format`](http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format) directive and writing the variables you care about to the logs.  Your monitoring service then reads the logs and runs the necessary calculations.

## It's All About The Connections

The status module metrics are mostly about **connections**, so it's worth diving into some depth about what those are exactly.

An Nginx server generally plays two roles.  Its primary role, as the name "server" would imply, is to serve content (web pages, images, video, or raw API data) to clients.  Clients historically have been simple remote users with web browsers, but now include a variety of endpoints, including mobile devices, "apps", and even other servers.  

Nginx's secondary role is as middleman (or proxy) between clients and servers farther up the chain of command (called upstream servers).  Upstream servers do more of the heavy lifting for a given request.  They run application servers (like Unicorn for Ruby on Rails apps or Tomcat for Java apps) and are the "dynamic" part of dynamic web pages.

As a proxy, Nginx can do a few things really, really well -- it can serve local static content (images and non-dynamic HTML files) super quickly, it can act as a cache for content from upstream servers, and it can load-balance requests among several upstream servers.

All this communication by and between clients, Nginx, and upstream servers is done through **connections** -- the network-level establishment of formal communications between two programs.  Once a network-level connection is established, application-level data can flow via the various protocols that Nginx supports (most commonly HTTP, but also WebSockets, SPDY, TCP streams, and several mail protocols.)

Connections are handled one layer down from the application data (at the TCP layer of networking) and negotiated through a "handshake" process.  The details of the TCP protocol & handshake are beyond the scope of this article, but some great details are available [on Wikipedia if you're so inclined](http://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment).  For more on the specifics of connections and how they related to HTTP, see [A Software Developer's Guide to HTTP - Connections](http://odetocode.com/articles/743.aspx)

Nginx connections can take one of several states:

  *  **Accepted** - After a completed TCP handshake between the client and Nginx, a connection is considered Accepted.  It then takes one one of 3 sub-states:

    *  **Idle / Waiting** - no data is currently being received by or sent to the client.  A connection will be in this state under two circumstances:  First, in between the completion of the request and the initiation of the response;  Second, in between a completed response and the next request.  As of HTTP 1.1, all connections are considered persistent unless declared otherwise, so a connection will remain open until it is closed or times out.

    *  **Reading** - Nginx is reading data from the client.

    *  **Writing** - Nginx is writing data to the client.

  *  **Handled** - Nginx has finished writing data to the client and the request has been completed successfully and closed.
  
  *  **Dropped** - Nginx ended the connection prior to successfully completing the request.  (This usually happens because of a resource or configuration limit.)

With this background information in mind, let's now take a look at the specific metrics:

## Nginx Status Module Metrics

#### `Active connections`

The _current_ number of active (accepted) connections from clients, which includes all connections with the status **Idle / Waiting**, **Reading**, and **Writing**.

#### `accepts`

The _total_ number of accepted connections from clients since the Nginx master process started.  Note that reloading configuration or restarting worker processes does not reset this metric, but terminating and restarting the master process does.

#### `handled`

The _total_ number of handled connections from clients since the Nginx master process started.  This will be lower than `accepts` only in cases where a connection is dropped before it is handled.

  * **Open Question**:  Will `handled` be lower than `accepts` by the number of currently "in-flight" requests that are being processed by upstream servers?

#### `requests`

The _total_ number of requests from clients since the Nginx master process started.  A request is an application-level (HTTP, SPDY, etc.) event and is defined as a client requesting a resource via the application protocol.  A single connection can (and often does) make multiple requests, so this number will generally be larger than the number of accepted/handled connections.

#### `Reading`

The _current_ number of (accepted) connections from clients where Nginx is reading the request (at the time the status module was queried.)

#### `Writing`

The _current_ number of connections from clients where Nginx is writing a response back to the client.

#### `Waiting`

The _current_ number of connections from cluents that are in the **Idle / Waiting** state, waiting for a request.

## Selected Log File Variables

It's beyond the scope of this guide to dive into _every_ Nginx log variable.  Instead, we're going to take a close look at a few variables that are of particular interest in the context of monitoring.

(A note about this:  If we've missed something or you feel otherwise about our choices below, let us know.  [Submit a pull request on GitHub]() or leave a comment below.)

### Default Variables

The default Nginx access log (used by declaring `log_format combined` in your configuration file) uses the following variables.  A description is provided along with suggestion on possible inclusion in metrics.

#### `$body_bytes_sent`

The number of bytes sent to a client, not including the response header.  This can help you monitor individual response size (on its own) or a portion of outgoing bandwidth used (in aggregate).

#### `$http_referer`

The value of the `HTTP referer` header from the incoming request.  This is determined by the requesting browser and identifies the URL of the webpage that linked to the resource being requested.  Two interesting notes on this:  
  1.  The word "referrer", at least in English, is spelled with two Rs, but the original mis-spellling from the HTTP spec ([RFC 1945](http://tools.ietf.org/html/rfc1945)) managed to stick around.

  2.  Since the header is set by the client making the request, it, like any data packet, can be modified.  Lately this has resulted in what's known as "referer spam", where malicious clients will spoof their referer headers to instead show spammy websites for consumers of analytics or monitoring software to see, be tempted by, and (they hope), visit.
  
#### `$http_user_agent`

The `User-Agent` HTTP request header.  This string provides identifying (in a general sense) information about the client making a request.  The format is slightly different between human-operated web browsers and automated agents (bots), but the theme is the same.  [Wikipedia has some awesome nitty-gritty details on user agent strings.](http://en.wikipedia.org/wiki/User_agent)
  
#### `$remote_addr`

The [IP address](http://en.wikipedia.org/wiki/IP_address) of the client making the request.  Note that if the physical client device is behind a proxy or NAT server, the public-facing address of that server (not the internal IP address of the client) will be shown here. 
  
#### `$remote_user`

The user name supplied if [HTTP Basic authentication](http://en.wikipedia.org/wiki/Basic_access_authentication) is used for the request.
  
#### `$request`

The complete original request line.  An example of a (familiar) request line is as follows:

````
GET /community/guides/an-in-depth-guide-to-nginx-metrics/ HTTP/1.1  
````

The request line is actually a compound variable is composed of 3 sub-variables (each of which is accessible individually if needed)

````
$request_method $request_uri $server_protocol
````
    
A breakdown of these variables:

  * `$request_method` - The [HTTP method](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) of the request.  Most common methods used by browsers supporting HTTP 1.1 are GET, POST, PUT and DELETE, but the spec also includes OPTIONS, HEAD, TRACE and CONNECT (details on each available at the preceeding link.)

  * `$request_uri` - The original URI requested (with arguments).  Note that if the ultimate resource returned is different from the one requested (due to redirects or other changes), that will not be logged here -- this is what the client asked for originally.

  * `$server_protocol` - The application-level protocol and version used in the request.  You'll most commonly see HTTP/1.0 or HTTP/1.1, but Nginx supports SPDY, WebSockets and several mail protocols as well.  

#### `$time_local`

The local (server) time in the [Common Log Format](http://en.wikipedia.org/wiki/Common_Log_Format).  That time format is `dd/mm/yyyy:hh:mm:ss UTC-offset`.
  
#### `$status`

The numeric [HTTP Status Code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) of the response.  This is an important variable to log as your monitoring tools can read these entries and calculate the relative good-to-error ratios (and rates) accordingly.

### Other Variables of Interest

These variables are not included in the default `combined` log format.

#### `$args`

The arguments of the request from the client, also known as the [query string](http://en.wikipedia.org/wiki/Query_string).

#### `$bytes_sent`

The number of bytes sent to the client in the response.  This is a particularly useful variable to log as you can use this to calculate outbound bandwidth used.

#### `$connection`

The connection serial number.  This can be used to identify unique connections (and monitor the number of types of requests made by through each connection.)  This serial number resets when the master Nginx process is terminated.  

#### `$connection_requests`

The number of requests made through a the `$connection` that's being referenced or logged.

#### `$content_length` & `$request_length`

`$content_length` returns the HTTP Content-Length request header field.  This is the total size (in bytes) of the body of the request being made by the client, as reported by the client.

`$request_length` is the full request length (in bytes) - including the request line, header and body, as calculated by Nginx.  

We're highlighting these both mostly to avoid confusion:  If you're interested in monitoring the size of incoming requests or overall incoming bandwidth, use `$request_length`.

Because `$content_length` is drawn from a request header, it's calculcated by the client and therefore has the potential to be spoofed (in the case of a DDoS attack, for example.)

#### `$gzip_ratio`

If Gzip is enabled on responses, the compression ratio of the response (the ratio between the original and compressed response sizes.)

Gzip compression is a feature that Nginx provides (through [`ngx_http_gzip_module`](http://nginx.org/en/docs/http/ngx_http_gzip_module.html)) that pipes responses through the [gzip](http://www.gzip.org/) compression software before sending them to the client.  This can reduce the size of responses by 50% or more and provide a significant outbound bandwidth savings.  Gzip is extremely fast, but there is still material overhead in the compression process (both in terms of CPU usage and response time), though some of this overhead is recovered in the transfer-time savings from the smaller file.

It's a delicate balance to strike so be sure to monitor your resource usage closely if you decide to use gzip compression.

Further details about compression and decompression [are available from Nginx directly](http://nginx.com/resources/admin-guide/compression-and-decompression/).

#### `$host`

Either the hostname of the server (as resolved by the client making the request and presented in the `Host` HTTP header), or if that header is not available or empty, the name of the server handling the request (as defined by the first server to appear using the Nginx `server_name` directive.)

#### `$http_`_name_ and `$upstream_http`_name_

Following the pattern of `$http_referer` and `$http_user_agent` above, Nginx actually allows you to access any of the [HTTP request headers](http://en.wikipedia.org/wiki/List_of_HTTP_header_fields) by referencing `$http_` and the header name (converted to lower case, with dashes replaced by underscores).

Similarly, you can access the headers as returned by any upstream servers by appending "upstream" to the front of the desired header name. 

#### `$msec`

The current time (in milliseconds).  This is useful to monitoring tools in calculating various rate metrics (requests per second, errors per second, etc.)

#### `$pid`

The process ID (PID) of the Nginx worker that handled the request.  When this variable is logged, it enables monitoring tools to track the workload and associated metrics of each worker. 

Since worker processes can (and do) crash or get reaped and restarted by the Nginx master process, their PID can change, so it's not safe to naively use this as an absolute mark of a particular worker.

#### `$request_time` & `$upstream_response_time`

`$request_time` is the total time taken for Nginx (and any upstream servers) to process a request and send a response (in seconds, with millisecond resolution).  This is the primary data used in order to calculate your server's response time metric.

The clock starts as soon as the first bytes are read from a client and stops after the last bytes have been sent.  Note that this _includes_ the processing time for upstream servers -- if you're interested in breaking out those metrics then you can use `$upstream_response_time`, which only measures the response time of the upstream server.

To clarify:  The language can be a bit confusing here -- even though the variable is "request" time, it actually measures the elapsed time of the full request-response cycle (from the Nginx's perspective.)

#### `$server_addr` and `$server_name`

The IP address (`$server_addr`) or name (`$server_name`) of the Nginx server that accepted a request.  This is useful in a multi-server (load-balanced) environment when you'll need to monitor which requests (and, therefore, which metrics) are handled by each server.

Note:  Computation of the IP address requires a system call unless you specifically bind an address using the `listen` directive in your configuration.  Keep this in mind as the addition of system calls can add significant overhead and impact server performance accordingly.  For this reason, $server_name may be a better choice for many installations unless the full IP is specifically required. 

#### `$uri`

The current URI of the request.  Internal to Nginx, this value may change during request processing (i.e. in the case of redirects), but if written to a log file for monitoring, it represents the final URI sent to the client.  

Note that this differs from `$request_uri` in that `$request_uri` represents the URI requested (ignoring any redirects or other rewrites), and `$uri` represents the final resource address returned in the response.

## Request for Feedback

We hope you've found this reference valuable, enlightening, and perhaps even fun (ok, ok, we won't push it.) If you didn't - let us know!  This, along with all of the Scalyr Community Guides, [is editable on Github]().  If we've gotten something wrong, please make a correction and submit a pull request or leave a comment below.  We are believers in [Cunningham's Law](http://meta.wikimedia.org/wiki/Cunningham%27s_Law) (best seen in [xkcd 386](http://xkcd.com/386/)) so we value and deeply appreciate feedback.