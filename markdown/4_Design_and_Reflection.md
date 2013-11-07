Design and Reflection
=====================

The REST workflow is more efficient if we do not wait until we have
everything before we start using the parts that we do have. The main
tool to achieve this is the SAX parser whose model presents poor
developer ergonomics because it is not usually convenient to think on
the markup's level of abstraction. Using SAX, a programmer may only
operate on a convenient abstraction after inferring it from a lengthy
series of callbacks. In terms of ease of use, DOM is generally preferred
because it provides the resource whole and in a convenient form. It is
possible to duplicate this convenience and combine it with progressive
interpretation by removing one restriction: that the node which is given
is always the document root. From a hierarchical markup such as XML or
JSON, when read in order, sub-trees are fully known before we fully know
their parent tree. We may select pertinent parts of a document and
deliver them as fully-formed entities as soon as they are known, without
waiting for the remainder of the document to arrive. This approach
combines most of the desirable properties from SAX and DOM parsers into
a new, hybrid method.

The interesting parts of a document may be identified before it is
complete if we turn the established model for drilling-down inside-out.
Under asynchronous I/O the programmer's callback traditionally receives
the whole resource and then, inside the callback, locates the sub-parts
that are required for a particular task. Inverting this process, the
locating logic currently found inside the callback can be extracted from
it, expressed as a selector language, and used it to declare the cases
in which the callback should be notified. The callback will receive
complete fragments from the response once they have been selected
according to this declaration.

Javascript will be used to implement the software deliverables because
it has good support for non-blocking I/O and covers both environments
where this project will be most useful: web browser and web server.
Focusing on the MVP, parsing will only be implemented for one mark-up
language. Although this technique could be applied to any text-based,
tree-shaped markup, JSON best meets the project goals because it is
widely supported, easy to parse, and defines a single n-way tree, making
it more amenable to selectors which span multiple format versions.

JSONPath is well suited for selecting nodes while the document is being
read because it specifies only constraints on paths and 'contains'
relationships. Because of the top-down serialisation order, on
encountering any node in a serialised JSON stream we will have already
seen enough of the prior document to know its full path. JSONPath would
not be so amenable if it expressed sibling relationships because there
is no similar guarantee of having seen other nodes on the same level
when any particular node is encountered. A new implementation of the
language is required because the existing JSONPath library is
implemented only as a means to search through already gathered objects
and is too narrow in applicability to be useful in a streaming context.

Given that we are selecting specifically inside a REST resource it is
unlikely that we will be examining a full model. Rather, the selectors
will be applied to a subset that we requested and was assembled on our
behalf according to parameters that we supplied. We can expect to be
interested in all of the content belonging to a particular category so
search-style selections such as 'books costing less than X' are less
useful than queries which identify nodes because of their type and
position such as 'all books in the discount set', or, because we know we
are examining `/books/discount`, simply 'all books'. In creating a new
JSONPath implementation the existing language is followed somewhat
loosely, specialising the matching by adding features which are likely
to be useful when detecting entities in REST resources while avoid
unnecessary code by dropping others. Later adding new features to a
language is easier than removing them once a userbase has built up so
where the utility isn't clear the default position is to not include. It
is difficult to anticipate all real-world matching requirements but it
should be possible to identify a core 20% of features that are likely to
be useful in 80% of cases. For the time being any functionality which is
not included may be implemented by registering a more permissive
selection and then further filtering programmatically from inside the
callback. Patterns of programmatic filtering which arise from use in the
wild can later mined and added to the selection language.

Detecting types in JSON
-----------------------

As seen in the 'all books' example above, it is intuitive to support
identifying sub-trees according to a categorisation by higher-level
types. JSON markup describes only a few basic types. On a certain level
this is also true for XML -- most nodes are either of type Element or
Text. However, the XML metamodel provides tagnames; essentially, a
built-in type system for subclassifying the elements. JSON has no
similar notion of types beyond the basic constructs: array, object,
string, number. To understand data written in JSON's largely typeless
model it is often useful if we think in terms of a more complex type
system. This imposition of type is the responsibility of the observer
rather than of the observed. The reader of a document is free to choose
the taxonomy they will use to interpret it and this decision will vary
depending on the purposes of the reader. The required specificity of
taxonomy differs by the level of involvement in a field; whereas 'watch'
may be a reasonable type for most data consumers, to a horologist it is
likely to be unsatisfactory without further sub-types. To serve
disparate purposes, the JSONPath variant provided for node selection
will have no inbuilt concept of type, the aim being to support
programmers in creating their own.

~~~~ {.xml}
<!--  
  XML leaves no doubt as to the labels we give to an Element's type 
  type. Although we might further interpret, this is a 'person'
-->
<person  name='...' gender="male"
         age="45" height="175cm" profession="architect">
</person>
~~~~

~~~~ {.javascript}
/* JSON meanwhile provides no built-in type concept. 
   This node's type might be 'thing', 'animal', 'human', 'male',
   'man', 'architect', 'artist' or any other of many overlapping
   impositions depending on our reason for examining this data
*/
{  "name":"...", "gender":"male", "age":"45" 
   "height":"172cm" "profession":"architect">
}         
~~~~

In the absence of node typing beyond categorisation as objects, arrays
and various primitives, the key immediately mapping to an object is
often taken as a loose marker of its type. In the below example we may
impose the the type 'address' on two nodes prior to examining their
contents because of the field name which maps to them from the parent
node.

~~~~ {.javascript}
{
   "name": "..."
,  "residence": {
      "address": [
         "47", "Cloud street", "Dreamytown"
      ]
   }
,  "employer": {
      "name": "Mega ultra-corp"
   ,  "address":[
         "Floor 2", "The Offices", "Alvediston", "Wiltshire"      
      ]
   }   
}
~~~~

This means of imposing type is simply expressed as JSONPath. The
selector `address` would match all nodes whose parent maps to them via
an address key.

As a loosely typed language, Javascript gives no protection against
lists which store disparate types but by sensible convention this is
avoided. Likewise, in JSON, although type is a loose concept, the items
in a collection will generally be of the same type. From here follows a
sister convention illustrated in the example below, whereby each item
from an array is typed according to the key in the grandparent node
which maps to the array.

~~~~ {.javascript}
{
   "residences": {
      "addresses": [
         ["10", "Downing street", "London"]
      ,  ["Chequers Court", "Ellesborough", "Buckinghamshire"]      
      ,  ["Beach Hut", "Secret Island", "Bahamas"]
      ]
   }
}
~~~~

In the above markup, `addresses.*` would correctly identify three
address nodes. The pluralisation of field names such as 'address'
becoming 'addresses' is common when marshaling from OO languages because
the JSON keys are extracted from getter names which reflect the method's
cardinality: `public Address getAddress()` or
`public List<Address> getAddresses()`. To identify members of a type
held singularly or plurally it might help if a system that understands
natural language pluralisation such as Ruby on Rails were investigated.
Unions were also considered as a simpler solution, resembling
`address|addresses.*`. It was decided that until the usefulness is
better demonstrated, with no obvious best solution, it is simplest to
handle plurals outside of the JSONPath language by expecting the
programmer to register two selection specifications against the same
handler function.

In the below example types may not be easily inferred from ancestor
keys.

~~~~ {.javascript}
{
   "name": "..."
,  "residence": {
      "number":"...", "street":"...", "town":"..." 
   }
,  "employer":{
      "name": "..."
   ,  "premises":[
         { "number":"...", "street":"...", "town":"..." }
      ,  { "number":"...", "street":"...", "town":"..." }
      ,  { "number":"...", "street":"...", "town":"..." }
      ]
   ,  "registeredOffice":{
         "number":"...", "street":"...", "town":"..."
      }
   }
}  
~~~~

Here, the keys which map onto addresses are named by the relationship
between the parent and child nodes rather than by the type of the child.
The type classification problem could be solved using an ontology with
'address' subtypes 'residence', 'premises', and 'office' but this
solution feels quite heavyweight for a simple selection language.
Instead the idea of *duck typing* was imported from Python, as named in
a 2000 usenet discussion:

> In other words, don't check whether it IS-a duck: check whether it
> QUACKS-like-a duck, WALKS-like-a duck, etc, etc, depending on exactly
> what subset of duck-like behaviour you need [@pythonduck]

An address 'duck-definition' for the above JSON would say that any
object which has number, street, and town properties is an address.
Applied to JSON, duck typing takes an individualistic approach by
deriving type from the node itself rather than the situation in which it
is found. As discussed in section \ref{jsonpathxpath}, JSONPath's syntax
is designed to resemble the equivalent Javascript accessors but
Javascript has no syntax for a value-free list of object keys. The
closest available Javascript notation is that for object literals so a
derivative duck-type syntax was created by omitting the values,
quotation marks, and commas. The address type described above would be
written as `{number street town}`. Field order is insignificant so
`{a b}` and `{b a}` are equivalent.

It is difficult to generalise but when selecting items it is often
useful if subtypes, nodes which are covariant with the given type, are
also matched. We may consider that there is a root duck type `{}` which
matches any node, that we create a sub-duck-type if we add to the list
of required fields, and a super-duck-type if we remove from it. Because
in OOP extended classes may add new fields, this idea of the attribute
list expanding for a sub-type applies neatly to resources marshaled from
an OO representation. To conform to a duck-type a node must have all of
the required fields but could also have any others.

Importing CSS4's explicit capturing to JSONPath
-----------------------------------------------

JSONPath naturally expresses a 'contained in' relationship using the dot
notation but no provision is made for the inverse 'containing'
relationship. *Cascading Style Sheets*, CSS, the web's styling language,
has historically shared this restriction but a proposal for extended
selectors which is currently at Editor's Draft stage [@css4] introduces
an elegant solution. Rather than add an explicit 'containing'
relationship, the draft observes that CSS has previously always selected
the element conforming to the right-most of the selector terms, allowing
only the deepest mentioned element to be styled. This restriction is
lifted by allowing terms to be prefixed with `$` in order to make them
explicitly capturing; a selector without an explicit capturing term
continues to work as before. The CSS selector
`form.important input.mandatory` selects mandatory inputs inside
important forms but `$form.important input.mandatory` selects important
forms with mandatory fields.

CSS4 selector capturing will be incorporated into this project's
JSONPath implementation. By duplicating a syntax which the majority of
web developers should become familiar with over the next few years the
learning curve should appear more gradual. Taking on this feature, the
selector `person.$address.town` would identify an address node with a
town child, or `$people.{name, dob}` can be used to locate the same
people array repeatedly whenever a new person is added to it. Javascript
frameworks such as d3.js and Angular are designed to work with whole
models as they change. Consequently, the interface they present
converses more fluently with collections than individual entities. If we
are downloading data to use with these libraries the integration is more
convenient with explicit capturing because we can hand over the
collection as it expands.

Parsing the JSON response
-------------------------

While SAX parsers provide an unappealing interface to application
developers, as a starting point to handle low-level parsing in
higher-level libraries they work very well -- most XML DOM parsers are
built in this way. The pre-existing Clarinet project [@clarinet] is well
tested, liberally licenced, and compact, meeting our needs perfectly.
The name of this project, Oboe.js, was chosen in tribute to the value
delivered by Clarinet, itself named after the **SAX**ophone.

API design
----------

Everything that Oboe is designed to do can already be achieved by
combining a SAX parser with imperatively coded node selection. This has
not been widely adopted because it requires verbose, difficult
programming in a style which is unfamiliar to most programmers. With
this in mind it is a high priority to design a public API for Oboe which
is concise, simple, and resembles other commonly used tools. If Oboe's
API is made similar to common tools a lesser modification should be
required to switch existing projects to streaming HTTP.

For some common use cases it should be possible to create an API which
is a close enough equivalent to popular tools that it can be used as a
direct drop-in replacement. Although used in this way no progressive
loading would be enacted, when refactoring towards a goal the first step
is often to create a new expression of the same logic [@cleancode p.
212]. By giving basic support for non-progressive downloading the door
is open for apps to incrementally refactor towards a progressive
expression. Allowing adoption as a series of small, easily manageable
steps rather than a single leap is especially helpful for teams working
under Scrum because all work must be self-contained and fit within a
fairly short timeframe.

jQuery is by far the most popular library for AJAX today. The basic call
style for making a GET request is as follows:

~~~~ {.javascript}
jQuery.ajax("resources/shortMessage.txt")
   .done(function( text ) {
      console.log( "Got the text:", text ); 
   }).
   .fail(function() {
      console.log( "the request failed" );      
   });
~~~~

The jQuery API is callback-based, it does not wrap asynchronously
retrieved content in event objects, and event types are expressed by the
name of the method used to add the listener. These names, `done` and
`fail`, follow generic phrasing and are common to all asynchronous
functionality that jQuery provides. Promoting brevity, the methods are
chainable so that several listeners may be added from one statement.
Although Javascript supports exception throwing, for asynchronous
failures a fail event is used instead. Exceptions are not applicable to
non-blocking I/O because at the time of the failure the call which
provoked the exception will already have been popped from the stack.

`jQuery.ajax` is overloaded so that the parameter may be an object,
allowing more detailed information to be given:

~~~~ {.javascript}
jQuery.ajax({ "url":"resources/shortMessage.txt",
              "accepts": "text/plain",
              "headers": { "X-USER-ID": "123ABC" }
           });
~~~~

This pattern of passing arguments as object literals is common in
Javascript for functions which take a large number of parameters,
particularly if some are optional. This avoids having to pad unprovided
optional arguments in the middle of the list with null values and,
because the purpose of the values is given at the call site, avoids an
anti-pattern where a call may only be understood after counting the
position of the arguments.

Taking on this style while extending it to incorporate progressive
parsing, we arrive at the following API:

~~~~ {.javascript}
oboe("resources/people.json")
   .node( "person.name", function(name) {
      console.log("There is somebody called", name);   
   })
   .done( function( wholeJson ) {
      console.log("That is everyone!");
   })
   .fail( function() {
      console.log("There might may be more people but",
                  "we don't know who they are yet.");
   });
~~~~

In jQuery the whole content is given back at once so usually only one
`done` handler is added to a request. Under Oboe each separately
addressed area of interest inside the JSON resource requires its own
handler so it is helpful to provide a shortcut style for adding several
selector-handler pairs at a time.

~~~~ {.javascript}
oboe("resources/people.json")
   .node({  
      "person.name": function(personName, path, ancestors) {
         console.log("You will hear about", name, "...");
      },
      "person.address.town": function(townName, path, ancestors) {
         console.log("...they live in", townName);
      }
   });
~~~~

Note the `path` and `ancestors` parameters in the example above. These
provide additional information regarding the context in which the
identified node was found. Consider the following JSON:

~~~~ {.javascript}
{ 
   "event": "Mens' 100m sprint",
   "date": "5 Aug 2012",
   "medalWinners": {
      "gold":     {"name": "Bolt",    "time": "9.63s"},
      "silver":   {"name": "Blake",   "time": "9.75s"},
      "bronze":   {"name": "Gatlin",  "time": "9.79s"}
   }
}  
~~~~

In this JSON we may extract the runners using the pattern `{name time}`
or `medalWinners.*` but nodes alone are insufficient because their
location communicates information which is as important as their
content. The `path` parameter provides the location as an array of
strings plotting a descent from the JSON root to the found node. For
example, Bolt has path `['medalWinners', 'gold']`. Similarly, the
`ancestors` array lists the ancestors starting with the JSON root node
and ending at the immediate parent of the found node. For all but the
root node, which in any case has no ancestors, the nodes given by the
ancestor list will have been only partially parsed.

~~~~ {.javascript}
oboe("resources/someJson.json")
   .node( "medalWinners.*", function(person, path) {
      let metal = lastOf(path);
      console.log( person.name, "won the", metal, 
        "medal with a time of ", person.time );
   });
~~~~

Being loosely typed, Javascript does not enforce that ternary callbacks
are used as selection handlers. Before a callback is made the
application programmers must have provided a JSONPath selector
specifying the locations in the document that they are interested in.
The programmer will already be aware enough of the node location so for
most JSON formats the content alone will be sufficient, the API
purposefully orders the callback parameters so that in most cases a
unary function can be given.

Under Node.js the code style is more obviously event-based. Listeners
are normally added using an `.on` method where the event name is a
string given as the first argument. Adopting this style, Oboe's API
design also allows events to be added as:

~~~~ {.javascript}
oboe("resources/someJson.json")
   .on( "node", "medalWinners.*", function(person) {
   
      console.log( "Well done", person.name );
   });
~~~~

While allowing both styles creates an API which is larger than it needs
to be, the dual interface is designed to encourage adoption on the
client and server side. The two styles are similar enough that a person
familiar with one should be able to work with the other without
difficulty. Implementing the duplicative parts of the API should require
only a minimal degree of extra coding because they may be expressed in
common and specialised using partial completion. Because `'!'` is the
JSONPath for the root of the document, for some callback `c`, `.done(c)`
is a equal to `.node('!', c)`. Likewise, `.node` is easily expressible
as a partial completion of `.on` with `'node'`.

When making PUT, POST or PATCH requests the API allows the body to be
given as an object and serialises it as JSON because it is expected that
REST services which emit JSON will also accept it.

~~~~ {.javascript}
oboe.doPost("http://example.com/people", {
   "body": {
      "name":"Arnold", "location":"Sealands"
   }
});
~~~~

Earlier callbacks when paths are found prior to nodes
-----------------------------------------------------

Following the project's aim of giving callbacks as early as possible,
sometimes useful work can be done when a node is known to exist but
before we have the contents of the node. Under Oboe each node found in
the JSON document can potentially trigger notifications at two stages:
when it is first addressed and when it is complete. The API facilitates
this by providing a `path` event following much the same style as
`node`.

~~~~ {.javascript}
oboe("events.json")
   .path( "medalWinners", function() {
      // We don"t know the winners yet but we know we have some 
      // so let's start drawing the table:    
      gui.showMedalTable();
   })
   .node( "medalWinners.*", function(person, path) {    
      let metal = lastOf(path);
      gui.addPersonToMedalTable(person, metal);
   })
   .fail( function(){
      // That didn"t work. Revert!
      gui.hideMedalTable();
   });
~~~~

Implementing path notifications requires little extra code, only that
JSONPath expressions can be evaluated when items are found in addition
to when they are completed.

Choice of streaming data transport
----------------------------------

As discussed in section \ref{browserstreamingframeworks}, current
techniques to provide streaming over HTTP encourage a dichotomous split
of traffic as either stream or download. This split is not a necessary
consequence of the technologies used and streaming may instead be viewed
as the most efficient means of downloading. Streaming services
implemented using push pages or websockets are not REST. Under these
frameworks a stream has a URL but the data in the stream is not
addressable. This is similar to STREST, the *Service Trampled REST*
anti-pattern [@strest] in which HTTP URLs are viewed as locating
endpoints for services rather than the actual resources. Being
unaddressable, the data in the stream is also uncacheable: an event
which is streamed live cannot later, when it is historic, be retrieved
from a cache which was populated by the stream. Like SOAP, these
frameworks use HTTP as the underlying transport but do not follow HTTP's
principled design.

Although Oboe is not designed for live events, it is interesting to
speculate whether it could be used as a REST-compatible bridge to unify
live-ongoing feeds with ordinary REST resources. Consider a REST service
which gives per-constituency results for UK general elections. If
historic results are requested the data is delivered in JSON format much
as usual. Requesting the results for the current year on the night of
the election, an incomplete JSON with the constituencies known so far
would be immediately sent, followed by the remainder dispatched
individually as the results are called. When all results are known the
JSON would finally close leaving a complete resource. A few days later,
somebody wishing to fetch the results would use the *same URL for the
historic data as was used on the night for the live data*. This is
possible because the URL refers only to the data that is required, not
to whether it is current or historic. Because it eventually forms a
complete HTTP response, the data that was streamed is not incompatible
with HTTP caching and a cache which saw the data while it was live could
later serve it from cache as historic. More sophisticated caches located
between client and service would recognise when a new request has the
same URL as an already ongoing request, serve the response received so
far, and then continue by giving both inbound requests the content as it
arrives from the already established outbound request. Hence, the
resource would be cacheable even while the election results are
streaming and a service would only have to provide one stream to serve
the same live data to multiple users fronted by the same cache. An
application developer programming with Oboe would not have to handle
live and historic data as separate cases because the node and path
events they receive are the same. Without branching, the code which
displays results as they are announced would automatically be able to
show historic data.

Taking this idea one step further, Oboe might be used for infinite data
which intentionally never completes. In principle this is not
incompatible with HTTP caching although more research would have to be
done into how well current caches handle requests which do not finish. A
REST service which provides infinite length resources would have to
confirm that it is delivering to a streaming client, perhaps with a
request header. Otherwise, if a non-streaming REST client were to use
the service it would try to get 'all' of the data and never complete its
task.

Supporting only XHR as a transport unfortunately means that on older
browsers which do not fire progress events (section
\ref{xhrsandstreaming}) a progressive conceptualisation of the data
transfer is not possible. Streaming workarounds such as push tables will
not be used because they would result in a client which is unable to
connect to the majority of REST services. Degrading gracefully, the best
compatible behaviour is to wait until the document completes and then
interpret the whole content as if it were streamed. Because nothing is
done until the request is complete the callbacks will be fired later
than on a more capable platform but will have the same content and be in
the same order. By reverting to non-progressive AJAX on legacy
platforms, an application author will not have to write special cases
and the performance should be no worse than with traditional AJAX
libraries such as jQuery. On legacy browsers Oboe could not be used to
receive live data -- in the election night example no constituencies
would be shown until they had all been called.

One benefit of a unified model for streamed and short-lived content is that
it allows a simpler security model. Because the demands of the transport are
different, streaming security is usually implemented separately from
other HTTP requests. Schneier argues that "complexity is the worst enemy
of security" [@simpleschneier Software Complexity and Security] and in
one online debate paints a buildings analogy [@schneierdoors]:

> More specifically, simplicity tends to completely remove potential
> avenues of attack. An easy example might be to think of a building.
> Adding a new door is an additional complexity, and requires additional
> security to secure that door. This leads to an analysis of door
> materials, lock strength, and so on. The same building without that
> door is inherently more secure, and requires no analysis or
> assumptions about how it will be secured. Of course, this isn’t to say
> that buildings with doors are insecure, only that it takes more work
> to secure them. And it takes more work to secure a building with ten
> doors than with one door.

Unifying two means of data transfer into a single model is analogous to
a building having only one entrance. A better level of security should
be possible given the same resources.

Node's standard HTTP library provides a view of the response as a
standard ReadableStream so there will be no problems programming to a
streaming interpretation of HTTP. In Node because all streams provide a
common API regardless of their origin allowing arbitrary sources to be
read is no extra work. Although Oboe is intended primarily as a REST
client, under Node it will be capable of reading data from any source.
Oboe might be used to read from a local file, an ftp server, a
cryptography source, or the process's standard input.

Handling transport failures
---------------------------

Oboe cannot know the correct behaviour when a connection is lost so this
decision is left to the containing application. Generally on request
failure one of two behaviours are expected: if the actions performed in
response to data so far remain valid in the absence of a full
transmission their effects will be kept and a new request made for just
the missed part; alternatively, if all the data is required for the
actions to be valid, the application should take an optimistic locking
approach and perform rollback.

Oboe.js as a micro-library
--------------------------

HTTP traffic is often compressed using gzip so that it transfers more
quickly, particularly for entropy-sparse text formats such as
Javascript. When measuring a library's download footprint it usually
makes more sense to compare post-compression. For the sake of adoption
smaller is better because site creators are sensitive to the download
size of their sites. Javascript micro-libraries are listed at
[microjs.com](http://microjs.com), which includes this project. A
library qualifies as being *micro* if it is delivered in 5kb or less,
5120 bytes but micro-libraries also tend to follow the ethos that it is
better for an application developer to gather together several tiny
libraries than find one with a one-size-fits-all approach, perhaps
echoing the unix command line tradition for small programs which each do
exactly one thing. As well as being small, in the spirit of a
micro-library a project should impose as few restrictions as possible on
its use and be agnostic as to which other libraries or programming
styles it will be combined with. Oboe feels on the edge of what is
possible to elegantly do as a micro-library so while the limit is
somewhat arbitrary, keeping below this limit whilst writing readable
code should provide an interesting extra challenge.
