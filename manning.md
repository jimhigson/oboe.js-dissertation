---
author:
- 'Jim Higson, May 2014'
title: 'Book Proposal: REST and Streaming'
...

Make it fast
============

There is a common belief, sometimes true, that creating fast systems
requires writing fast algorithms. But completing a task in this age of
cheap, fast CPUs and loosely coupled, remote
resources we normally spend more time waiting for I/O than any other
activity. For internet computing, the efficient use
of I/O usually delivers more benefit than any other kind of optimisation.

We benefit today from a generation of highly asynchronous servers: Node.js, Netty,
NginX. In Node the stream, not the whole resource, is the primary
abstraction. However, our common REST clients are yet to embrace
streaming and do not use a resource until it has arrived completely.
We should try act earlier: in practice there is usually no
difference between being reacting *earlier* and being *faster*.

On a Journey: reactivity and fault tolerance
--------------------------------------------

A passenger checks her email on her phone. As the train moves through
the countryside her reception is lost and reestablished many times.
Let's look inside the email webapp. The web developer's standard toolkit
encourages us to consider connections that are terminated early but were
partially successful as if they were wholly unsuccessful. Popular AJAX
libraries automatically parse JSON and XML responses before passing back
to the application but if there is an early disconnection they provide no
facility to gather the partial response. With applications following the
path set out by their tooling, our email webapp disregards the partially
retrieved inbox without inspection. For our passenger, even if 90% of
her emails had been retrieved, the application behaves as if it
has nothing. Later, when the network returns her inbox will be
downloaded from scratch, repeating the 90% which was already
successfully transferred.

By integrating streaming into REST we step back from a dichotomy of
messages as wholly successful or unsuccessful. The resource is
conceptualised as having many parts which are useful in themselves, in
which the successful delivery of each part is handled immediately on
delivery. When an early disconnection occurs the content previously
delivered has already been handled: no special cases are needed to
salvage the remains.

A Vote: caching and distribution
--------------------------------

We wish to provide a REST service for election results. If a client
requests historical data, a static resource is delivered much as we would
expect. For data representing an ongoing vote we need not switch
REST for WebSockets. Under REST, the best information so far is immediately
sent, followed by the remainder dispatched live as the polls are called.
When all results are known, the JSON closes as usual to form a standard,
complete, cacheable resource. A client wishing to fetch results
after-the-fact requests the same URL for the historic data as was used
during the election for the live stream. This is possible because cool
URLs[1] are semantic: they locate a resource by its meaning, indifferent
to when the request is made.

Under streaming REST the unification of live and historic data frees the
application developer to handle either type
from a single transport layer without writing divergent cases.
For the election we might display colours on a map - the
results could be to-the-second or decades past, but through the
entire front-end stack we need only code once.


[1] http://something.com
