---
author:
- 'Jim Higson, May 2014'
title: 'Book Proposal: REST and Streaming'
...

Make it fast
============

There is a common belief, sometimes true, that creating fast systems
requires writing fast algorithms. Servicing a request in this age of
cheap, fast CPUs and architectures composed from loosely coupled, remote
resources will normally consume more time waiting for I/O than any other
activity. Can it be right that we create processes only to spend their
lives waiting?

We have now a generation of highly asynchronous servers: Node.js, Netty,
NginX. In Node the stream, not the whole resource, is the primary
abstraction. However, today's popular REST clients are yet to embrace
streaming and still wait until a resource arrives completely before
using it. We should want to act early: in most situations there is no
real-world difference between being reacting *earlier* and being
*faster*.

On a Journey: reactivity and fault tolerance
--------------------------------------------

A passenger checks her email on her phone. As the train moves through
the countryside the reception is lost and reestablished many times.
Let's look inside the email webapp. The web developer's standard toolkit
is structured so that connections that are terminated early but were
partially successful are considered wholly unsuccessful. Popular AJAX
libraries automatically parse JSON and XML responses before passing back
to the application but if there is an early disconnection, provide no
facility to gather the partial response. With applications following a
path set out by their tooling, our email webapp disregards partial data
without inspection. For our passenger, even if 90% of her inbox had been
retrieved, if her signal is lost the application behaves as if it
received nothing. Later, when the network returns her inbox will be
downloaded from scratch, repeating the 90% which was already
successfully transferred.

By integrating streaming into REST we step back from a dichotomy of
messages as wholly successful or unsuccessful. The message is
conceptualised as having many parts which are useful in themselves, in
which the successful delivery of each part is handled immediately on
delivery. When an early disconnection occurs the content previously
delivered has already been handled: no special cases are needed to
salvage the remains.

A Vote: caching and distribution
--------------------------------

Election results are provided by a news organisation through a REST
service. When clients request historical data, static data is delivered
much as we would expect. For data representing an ongoing vote the best
information so far as is known can be immediately sent: an incomplete
resource with the results so far, followed by the remainder dispatched
live as the polls are called. While the data is sent as a stream, when
all results are known, after-the-fact the JSON closes as usual and
leaves a standard, cacheable complete resource. Once the polls have
closed a client wishing to fetch the results would use the *same URL for
the historic data as was used during the event for the live stream*.
This is possible because cool URLs locate data by its meaning,
indifferent to the time when the request is made.

An application developer receiving streaming REST does not have to
handle live and historic data as separate cases. They may concentrate on
handling the receipt of data regardless of the timing. Without providing
divergent cases, the code which displays results as they are announced
may also be applied to historic data.
