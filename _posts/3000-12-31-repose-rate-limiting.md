---
layout: post
title: "Repose - Rate Limiting"
date: 2015-12-30 00:00
comments: true
author: Bill Scheidegger
authorIsRacker: true
authorAvatar: https://secure.gravatar.com/avatar/a90582eafe040d70f11d28b39078a1a6
bio: Bill is a Software Developer at Rackspace that is currently working on the open source Repose project
    and has almost two decades of professional experience in Software and Systems design.
    Prior to joining Rackspace and after eight years of active duty service in the USAF,
    he worked primarily on Iridium Based Friendly Force Tracking Solutions for the US Military.
published: true
categories:
    - Repose
    - OpenStack
    - Rate Limiting
---

![Repose logo]({% asset_path repose_logo.png %})

The [Repose][repose] team has been diligently trying to get the latest major release out and the Ninja's on Duty have been extremely busy helping folks out.
That said, it has been more than a few sprints since we were able to post to this blog.

The last [Repose][repose] Ninja on Duty post discussed
[WSGI Middleware On The JVM](https://developer.rackspace.com/blog/wsgi-middleware-on-the-jvm)
and the last time I [posted](https://developer.rackspace.com/blog/repose-ninja-097)
I went over the [Repose Hello World project](https://github.com/rackerlabs/repose-hello-world) that walks you through writing your own custom filter (bundle) for [Repose][repose].
This time I'm going to take a little bit of a deeper dive into one of the most commonly used features of [Repose][repose], **Rate Limiting**.  

I'll cover:

* How Rate Limiting works in [Repose][repose].
* How to configure Rate Limiting in [Repose][repose].
* What else you'll need to bring it all together.

<!-- more -->

# How does Rate Limiting work in Repose

I think everyone reading this is at least familiar with the concept of Rate Limiting, but lets just define it for the sake of making sure we are all on the same page.
Rate Limiting is allowing at most a configured maximum number of requests to reach an Origin Service within a given time period.
This can be further refined to a particular entity based on account, IP address, or any other consistently identifiable criterion.
Repose's Rate Limiting Filter uses the value of the `X-PP-User` request header as the unique identifier and the value(s) of the `X-PP-Groups` request header in the rate limit determination.

At the most basic level, the `rate-limiting.cfg.xml` configuration file contains a series of `limit-group`'s each containing a series of `limit`'s.
Each `limit` has the following attributes:
 
* a unique name used for identification (`id`)
* a human readable version of the RegEx used for system messages including logging (`uri`)
* a RegEx that is applied to the URI to determine a match (`uri-regex`)
* a list of HTTP Methods (e.g. GET, PUT, POST) to apply this `limit` (`http-methods`)
* the time unit (e.g. HOUR, MINUTE, SECOND) in which to apply this `limit` (`unit`)
* the number of matches allowed through in the period of time (`value`)

The `X-PP-User` is used as the entity to apply `limit`'s to and `X-PP-Groups` is used for comparison of the `limit-group` elements' `groups` attribute to determine which rates apply. 
The first `limit-group`, in the order they are listed in the configuration file, that has a `limit` that matches (i.e. URI/RegEx/Method) will be the only `limit-group` effected.
Each limit count in the `limit-group` is effected until one of the limits exceeds the value within the time frame.
If no `limit` is exceeded and so the request has not already been rejected, then the optional `default` `limit-group` and `global-limit-group` are processed in turn and either can potentially reject the request.
If any `limit` is exceeded by the current request, then the response will be an error status code and will contain a Retry After header.

# How to configure Rate Limiting in Repose

Keep in mind that only the first `limit-group` that matches is effected and only the `limit`'s in that group up to and including the `limit` that is being triggered are processed.
This typically means that more tightly defined `limit`'s are earlier in the configuration while broader limits are placed later.
However, it is completely dependent on the desired outcome as to what order the `limit`'s are placed within a `limit-group`.

Let's use the following example `rate-limiting.cfg.xml` configuration:

{% codeblock lang:xml %}
<?xml version="1.0" encoding="UTF-8"?>
<rate-limiting xmlns="http://docs.openrepose.org/repose/rate-limiting/v1.0">
    <limit-group id="group-one" groups="groups">
        <limit id="limit-one" uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="GET POST" unit="MINUTE" value="1"/>
        <limit id="limit-two" uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="ALL"      unit="MINUTE" value="5"/>
    </limit-group>
</rate-limiting>
{% endcodeblock %}

Based on the order of the two `limit`s, all `GET` and `POST` requests exceeding the one per minute threshold would not increment _limit-two_.

For this example, let's assume the `X-PP-User` header has the same value for all of the requests and the `X-PP-Groups` header has the value _groups_.

| Method  | Time (secs) | _limit-one_ | _limit-two_ | Pass/ID     |
|:--------|:------------|:-----------:|:-----------:|:------------|
|         | 00:00       | 0           | 0           | PASS        |
| GET     | 00:00 + 0   | 1           | 1           | PASS        |
| DELETE  | 00:00 + 1   | 1           | 2           | PASS        |
| POST    | 00:00 + 2   | 1           | 2           | _limit-one_ |
| PUT     | 00:00 + 3   | 1           | 3           | PASS        |
| PATCH   | 00:00 + 4   | 1           | 4           | PASS        |
| HEAD    | 00:00 + 5   | 1           | 5           | PASS        |
| OPTIONS | 00:00 + 6   | 1           | 5           | _limit-two_ |
| CONNECT | 00:00 + 7   | 1           | 5           | _limit-two_ |
| TRACE   | 00:00 + 8   | 1           | 5           | _limit-two_ |
[Example 1][table-1]

Note the `POST` does not increment _limit-two_ since _limit-one_ already triggered and would have already rejected the request.

If the two `limit`s were reversed, then the first five per minute would also be checked against the `GET` and `POST` `limit`.

| Method  | Time (secs) | _limit-one_ | _limit-two_ | Pass/ID     |
|:--------|:------------|:-----------:|:-----------:|:------------|
|         | 00:00       | 0           | 0           | PASS        |
| GET     | 00:00 + 0   | 1           | 1           | PASS        |
| DELETE  | 00:00 + 1   | 1           | 2           | PASS        |
| POST    | 00:00 + 2   | 1           | 3           | _limit-one_ |
| PUT     | 00:00 + 3   | 1           | 4           | PASS        |
| PATCH   | 00:00 + 4   | 1           | 5           | PASS        |
| HEAD    | 00:00 + 5   | 1           | 5           | PASS        |
| OPTIONS | 00:00 + 6   | 1           | 5           | _limit-two_ |
| CONNECT | 00:00 + 7   | 1           | 5           | _limit-two_ |
| TRACE   | 00:00 + 8   | 1           | 5           | _limit-two_ |
| GET     | 00:00 + 9   | 1           | 5           | _limit-one_ |
[Example 2][table-2]

Note the extra `GET` is still limited by _limit-one_ since it is still within the one minute time period.

Just to show an over exaggeration of the ordering behavior, let's expand the example configuration with another `limit-group`:

{% codeblock lang:xml %}
<?xml version="1.0" encoding="UTF-8"?>
<rate-limiting xmlns="http://docs.openrepose.org/repose/rate-limiting/v1.0">
    <limit-group id="group-one" groups="groups">
        <limit id="limit-one"   uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="GET POST" unit="MINUTE" value="1"/>
        <limit id="limit-two"   uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="ALL"      unit="MINUTE" value="5"/>
    </limit-group>
    <limit-group id="group-two" groups="groups">
        <limit id="limit-three" uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="GET"       unit="SECOND" value="1"/>
        <limit id="limit-four"  uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="POST"      unit="SECOND" value="2"/>
    </limit-group>
</rate-limiting>
{% endcodeblock %}

If we still assume the `X-PP-User` header has the same value for all of the requests and the `X-PP-Groups` header has the value _groups_,
then this would have the same results as [Example 1][table-1] because even though both `limit-group`'s would match, _group-two_ is never reached.

Until now, it was assumed the value of the `X-PP-GROUPS` header was _groups_.
For this example lets expand the test to change this value so we exercise the `groups` attribute. 

Now let's update the example configuration a little more:

{% codeblock lang:xml %}
<?xml version="1.0" encoding="UTF-8"?>
<rate-limiting xmlns="http://docs.openrepose.org/repose/rate-limiting/v1.0">
    <limit-group id="group-one" groups="group-one">
        <limit id="limit-one"   uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="GET POST" unit="MINUTE" value="1"/>
        <limit id="limit-two"   uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="ALL"      unit="MINUTE" value="5"/>
    </limit-group>
    <limit-group id="group-two" groups="group-two">
        <limit id="limit-three" uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="GET"       unit="SECOND" value="1"/>
        <limit id="limit-four"  uri="/service/*" uri-regex="/service/([\d^/]*)/.*" http-methods="POST"      unit="SECOND" value="2"/>
    </limit-group>
</rate-limiting>
{% endcodeblock %}

Based on the order of the two `limit`s, all `GET` and `POST` calls exceeding the one per minute threshold would not increment _limit-two_.

| Method  | X-PP-Groups | Time (secs) | _limit-one_ | _limit-two_ | _limit-three_ | _limit-four_ | Pass/ID     |
|:--------|:------------|:-----------:|:-----------:|:-----------:|:-------------:|:------------:|:------------|
|         |             | 00:00       | 0           | 0           | 0             | 0            | PASS        |
| GET     | _group-one_ | 00:00 + 0   | 1           | 1           | 0             | 0            | PASS        |
| GET     | _group-two_ | 00:00 + 0   | 1           | 1           | 1             | 0            | PASS        |
| DELETE  | _group-one_ | 00:00 + 1   | 1           | 2           | 0             | 0            | PASS        |
| DELETE  | _group-two_ | 00:00 + 1   | 1           | 2           | 0             | 0            | PASS        |
| POST    | _group-one_ | 00:00 + 2   | 1           | 2           | 0             | 0            | _limit-one_ |
| POST    | _group-two_ | 00:00 + 2   | 1           | 2           | 0             | 1            | PASS        |
| PUT     | _group-one_ | 00:00 + 3   | 1           | 3           | 0             | 0            | PASS        |
| PUT     | _group-two_ | 00:00 + 3   | 1           | 3           | 0             | 0            | PASS        |
| PATCH   | _group-one_ | 00:00 + 4   | 1           | 4           | 0             | 0            | PASS        |
| PATCH   | _group-two_ | 00:00 + 4   | 1           | 4           | 0             | 0            | PASS        |
| HEAD    | _group-one_ | 00:00 + 5   | 1           | 5           | 0             | 0            | PASS        |
| HEAD    | _group-two_ | 00:00 + 5   | 1           | 5           | 0             | 0            | PASS        |
| OPTIONS | _group-one_ | 00:00 + 6   | 1           | 5           | 0             | 0            | _limit-two_ |
| OPTIONS | _group-two_ | 00:00 + 6   | 1           | 5           | 0             | 0            | PASS        |
| CONNECT | _group-one_ | 00:00 + 7   | 1           | 5           | 0             | 0            | _limit-two_ |
| CONNECT | _group-two_ | 00:00 + 7   | 1           | 5           | 0             | 0            | PASS        |
| TRACE   | _group-one_ | 00:00 + 8   | 1           | 5           | 0             | 0            | _limit-two_ |
| TRACE   | _group-two_ | 00:00 + 8   | 1           | 5           | 0             | 0            | PASS        |
[Example 3][table-3]





There is also a special `limit-group` named `global-limit-group`.
It is configured the same way as any other `limit-group`, but it is applied to all incoming requests.
The `global-limit-group` is applied after the regular `limit-group`'s in an effort to avoid miscounts and DOS attacks.
The `global-limit-group` is only applied if another `limit` has not already rejected the request.
The `global-limit-group` is intended as a safety feature to protect the Origin Service from to many incoming requests.
For this reason, the `global-limit-group` configuration should be set to the absolute maximums that the Origin Service can be guaranteed to support in order to prevent unnecessarily rejecting incoming requests.

# Bringing it all together

Just to recap all the important takeaways:

* The `limit-group`'s are applied in the order they are defined in the configuration file.
* If no `limit-group` `groups` match and a `limit-group` is identified as the default, then it is used as if the `groups` matched.
* If a `global-limit-group` is defined and no other `limit` has already rejected the request, then it is also applied.
* It is important to specify a default `limit-group` and/or a `global-limit-group` in order to prevent the opportunity to bypass the rate limiting configuration entirely. 
* If a regular limit is exceeded and the `overLimit-429-responseCode` attribute is `false` or is not present, then the response status code will be **Request Entity Too Large** (413). 
* If a regular limit is exceeded and the `overLimit-429-responseCode` attribute is `true`, then the response status code will be **Too Many Requests** (429). 
* If a global limit is exceeded, then the response status code will be **Service Unavailable** (503).

I hope this helps to better understand the capabilities of this great feature in [Repose][repose] and at the same time informed you how to better configure it.

Always remember, [Repose][repose] is Open Source and we are more than happy to accept external [Pull Requests][github].
Also, if you come across something that you would like us to go into in a future [Repose][repose] Ninja on Duty post or
you just want to chat all things [Repose][repose], then please contact us on [IRC][irc] ([WebChat][webchat]),
[email][email], or the [Wiki][wiki].

[repose]: http://www.OpenRepose.org/
[wiki]: http://wiki.OpenRepose.org/
[github]: https://github.com/rackerlabs/repose
[email]: mailto:ReposeCore@Rackspace.com
[webchat]: http://webchat.freenode.net/?channels=repose
[irc]: irc://irc.freenode.net:6667/repose
[narwhal]: https://www.worldwildlife.org/stories/unicorn-of-the-sea-narwhal-facts
