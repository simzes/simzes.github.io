---
layout: post
author: simon
title: Simple Server Caching for the Harvard Book Store
caption: Fixes and a software sketch for caching a legacy web application.
---
If one visits the Harvard Book Store's events webpage, it presents this gray, spinning wheel for several seconds before the events come into place. On a good day, it might be two or three seconds; on a bad day, it might take more than 10. Or 13.

![my life.](public/images/hbs/events-page-waiting-annot.png)

Clicking on an event, and then navigating back... the same thing happens, with the spinning wheel occupying the space for several seconds. It should remember some content...

Peeking under the hood, a giant scab of an ajax script emerges from the browser's timings chart:

![](public/images/hbs/request-headers-no-cache-annot.png)

What. the cluck. If Ann Patchett is coming to town, we should all find out sooner. As soon as possible.

## What to do, what to do
Emailing with the bookstore's IT person, it sounded like there weren't great configuration options from within the software system; the system is old. Really old, and doesn't have a manual around.

Fixing it would likely involve writing some php for an ancient application. The code would have to follow the logic that the events page doesn't need to be updated until the soonest event occurs, or until the page's content is changed. For client-side caching, the server should also keep the page on a relatively short leash, in case of updates; maybe 15 minutes. Perhaps aligned to the hour, because all their events are on the half or quarter-hour.

This script is definitely not doing any caching, or anything smart: from a data-rate perspective, the content is only 7kb, and takes a second and a half to finish. That's crawling along at ~4kb a second, compared to 2mb in 4.5 seconds for the whole page--400kb a second on average. (And much more in practice, given the portion of time taken up by latency and server-side processing.)

Better yet, the timings chart for the script helpfully informs us that the request spent all but 1 millisecond waiting for a response from the server:

![](public/images/hbs/request-timing-ajax-cropped.png)

So there's no server caching. And the headers don't allow for client-side caching, either:
* `Expires` has a 1997 date
* `Cache-Control` is set with a max-age of 1 (second), with a must-revalidate directive
* `Last-Modified` is always the server time, current at the time of the request.

On refresh, if the browser goes to look for the script contents in its cache, it will find that it's already expired or too old. Even if the client sends an `If-Modified-Since` using the last `Last-Modified` time (and it does), the server doesn't issue a `304 Not Modified`--presumably because it doesn't have an internal concept of when the output changed. So it will run the script again and send the contents. This looks like a cheap way of getting a calendar where everyone always has a consistent, up-to-date view.

---
Looking at the script's content, as one might expect, it returns the html for the structured data embedded in the events calendar: event times and places, book details, and book/author image links. Right after the request returns, the browser issues a flurry of requests for all of the linked pictures--the author pictures from the same server, and the book pictures from an amazon site. Once these come in, the page is rendered. This script holds up everything.

The server does do the right thing with image headers. On a new request, the `Cache-Control` header is set to public, with a large max-age. `Last-Modified` has a permanent date. On refresh, the server issues a 304 and the browser loads the image from cache.

(Curiously, the requests for the author pictures from the bookstore site are "blocked" for upwards of hundreds of milliseconds; even when the image is revalidated, it still takes the server 50+ milliseconds to decide this is the case, and the requests back up in the browser. The aws pictures are snappy and don't back up. Some server...)

## Adding some apache header directives
Since changing the source code isn't viable, mucking with the server configuration might improve some things. Apache's header directives can drop/set the right headers for getting some caching, somewhere.

For request headers, apache should drop any client cookies; on the server side, allowing client cookies will show up as misrepresenting and undercounting the visitor population, as only requests that miss the cache will make it through. There's also already a cookie on the base events page, and a tidy correspondence between loads of `/events` and `/ajax/events/upcoming`.

For response headers, apache should drop or set the expires, last-modified, and cache-control headers noted above to get some server-side caching in. Any headers for setting the cookies should also be dropped, as these might be cached and re-served to multiple clients; these may be re-used in later browsing, polluting the visitor analytics.

### A test setup
Without the ability to login to someone's server, it can be hard to test changes. Luckily, the events page can be easily mocked out with a python script that emulates the server's behavior; only the headers matter in this scenario. And the configuration changes should only apply to the ajax script anyway, so they are limited in scope.

A python script that returns the current time as the contents (this makes it easy to see if the content has changed), with mimicked headers, is straightforward: <sup>[1](#myfootnote1)</sup>
```
    # ... setup python
    status = '200 OK'

    html = str(time.time()) + '\n'

    start_response(status, [
        ('Content-Type', 'text/plain'),
        ('CacheControl', 'private, must-revalidate, max-age=1'),
        ('Pragma', 'no-cache'),
        ('Last-Modified', datetime.datetime.now(tz=tz.gettz('GMT')).strftime('%a, %d %b %Y %H:%M:%S GMT')),
        ('Expires', datetime.datetime(year=1997, month=7, day=5, hour=12, tzinfo=tz.gettz('GMT')).strftime('%a, %d %b %Y %H:%M:%S GMT')),
        ('Set-Cookie', 'random_test_cookie=' + str(time.time())),
        ])

    return [html]
```

Linking the script into apache, a test request shows matching headers coming back.

![](public/images/hbs/test-environment-headers-cropped.png)

### Apache header directives

To remove and set the headers as described in this emulated environment, directives from [mod_headers](https://httpd.apache.org/docs/current/mod/mod_headers.html) fix up the `/cacheme` path as described: <sup>[2](#myfootnote2)</sup>

```
<Location "/cacheme">
    # ... boilerplate setup ...

    # client cookies remove
    RequestHeader unset Cookie

    # server cookies remove
    Header unset Set-Cookie

    # set/unset server -> client caching headers
    Header set CacheControl "public, max-age=60"
    Header unset Last-Modified
    Header unset Pragma
    Header unset Expires
</Location>
```

Any requests under `/cacheme` now get this modified response:
![](public/images/hbs/test-environment-cached-headers2-cropped.png)

With these changes, server-side disk caching now works; adding the `CacheHeader` and `CacheDetailHeader` flags add hit/miss debug headers that show up on the client-side.

(Trying to configure a memory cache via apache's [`mod_socache`](https://httpd.apache.org/docs/2.4/mod/mod_cache_socache.html) did not work with the header modifications. Disabling gzipping entirely worked, and re-ording the cache/deflate filters worked for server-side caching, but did not work with these mocked-out headers; the headers directives were then applied after caching, and the cache refused the content because of the expired header.)<sup>[3](#myfootnote3)</sup>

### Client-side caching

Server-side caching is an improvement, but this didn't enable client-side caching. The [mozilla web docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) are a great resource for cache-control directives. But in practice, browser caching didn't work as described. Even with "only-if-cached" set with a positive max-age, the browser wouldn't cache any requests under `/cacheme`.

The mocked-out page did need to be changed, so that a cached page was embedded in another page; when a site is refreshed, a browser refetches the current page with `max-age=0` set. Embedding the `/cacheme` path within another also gets closer to the events page layout, in how it embeds a cachable page's content: <sup>[4](#myfootnote4)</sup>

```
def application(environ, start_response):
   request_url = environ['REQUEST_URI']

    html = "<p>" + str(time.time()) + "</p>"
    if not request_url.startswith('/cacheme'):
        html += '<p><a href="/b">stuff link</a></p>'
        html += "<embed src=/cacheme/c>"

    start_response(status, [
        ('Content-Type', 'text/html'), # changed to html
        # ...
```

Including a link to a page that also embeds the same, (hopefully) cacheable content (`...<a href="/b">...`) is a convenient way of seeing how the browser will handle this content without the browser issuing a refresh.

This still didn't work. Client-side caching worked well if `Expires` was set to some date in the future. But without this, the `Cache-Control: max-age=60` header set by apache was ignored, until the `Last-Modified` header was set in the server response; if this timestamp was set to the current time, it wasn't cached. Trailing by ten seconds, it was cached sporadically. Trailing by half an hour, it was cached; and for much longer than the `max-age` specified by the server.<sup>[5](#myfootnote5)</sup> Curious.

Both of these changes had to be applied from the python script; these headers can't be changed in this way from the apache configuration, as `mod_headers` and `mod_headers` can't set `Last-Modified` to a date relative to the current time, and `mod_expires` can't override an existing `Expires` header (even if it has been removed by mod_headers).

---
## Legacy caching

This falls into an odd corner of web software. Most server-side caching plugins are designed to work with the software--not bludgeon it into something workable. Content delivery networks can overwrite and modify headers on the fly, according to a separate configuration.

But there could be software that lives on, and works with the server, providing an efficient and easy-to-configure way to update the headers and cache content. A small, simple system that lives between the web server and a legacy system might allow the server and client to keep up with newer, more efficient techniques, without the need to modify a legacy system.

For the events page, the server could set the `Last-Modified` date to match when the content has actually changed; this would enable efficient client-side checks to see if it needs the page again. The server could also issue an `Expires` date that lets the client hold onto the page for longer. The images could be using the new `Cache-Control: immutable` header, and not need requests from the browser to double-check that everything is OK; if these requests did come in, the server should be able to issue a 304 status much faster.

### legacy-webcache
The [webcache.wsgi file](https://github.com/simzes/legacy-webcache/blob/main/src/webcache.wsgi) contains the result of a sketch for this software.

In an apache configuration, the webcache is mounted as an overlay of cacheable resources. Inbound requests for these resources are rerouted to the webcache script; internally-issued requests for the same resource are passed through to the original application.

State is held by url, and stored in memcached. Requests for a resource with no cache entry are sourced from a request to origin, and written into the cache on the way out. Requests for a resource with a valid cache entry can be sourced from the cache, by issuing a response with full content (drawn from the last successful origin response), or by issuing a 304 status. If the cache entry is not valid, it can be replaced or merged with a new origin response: if the origin's content has changed, the entry is wiped; if it has not changed, the `Last-Modified` status is preserved.

A backoff/contest algorithm handles races for the same resource from multiple threads. On a per-url basis, multiple requests are linearized with atomic cache updates, so that each thread attempting to update the cache with a request to origin are assigned a token unique to both the existing cache entry and the thread's request to renew the resource. In a contest to update a cache entry, only one thread "wins," and can make an origin request immediately. All threads that lose wait additional time, if necessary, before making an origin request, where this additional time is governed by the count of other, known threads participating in the contest.

---
## Footnotes

<a name="myfootnote1">1</a>: mod_wsgi python script for setting headers and time: <https://github.com/simzes/legacy-webcache/blob/main/basic_headers_rewrite/mock_headers.wsgi>

<a name="myfootnote2">2</a>: full apache configuration for disk caching: <https://github.com/simzes/legacy-webcache/blob/main/basic_headers_rewrite/disk_cache_access.conf>

<a name="myfootnote3">3</a>: <https://stackoverflow.com/questions/15323619/django-response-always-chunked-with-text-html-cannot-set-content-length/53642877#53642877>

<a name="myfootnote4">4</a>: mod_wsgi python script for setting headers, time, and embedding a resource under `/cacheme`: <https://github.com/simzes/legacy-webcache/blob/main/linked_content_wsgi/linked_content_mockout.wsgi>

<a name="myfootnote5">5</a>: mod_wsgi python script for setting headers for browser caching/memcache experiments: <https://github.com/simzes/legacy-webcache/blob/main/linked_content_options_wsgi/linked_content_options.wsgi>

