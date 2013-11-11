Background
==========

![**Labelling nodes in an n-tier architecture**. Although network
topology is often split about client and server side, for our purposes
categorisation as data, middle, and presentation tier is the more
meaningful distinction. According to this split the client- and
server-side presentation layer serve the same purpose, generating
mark-up based on aggregated data prepared by the middle tier.
\label{architecture}](images/architecture.png)

The web as an application platform
----------------------------------

Application design has historically charted an undulating path pulled by
competing approaches of thick and thin clients. Having evolved from a
document viewing system to the preferred application platform for all
but the most specialised interfaces, the web perpetuates this narrative
by resisting categorisation as either mode.

While the trend is generally towards more client scripting and for many
sites a Javascript runtime is now requisite, there are also
counter-trends. In 2012 Twitter reduced load times to one fifth of their
previous design by moving much of their rendering back to the
server-side, commenting that "The future is coming and it looks just
like the past" [@newTwitter]. Under this architecture short,
fast-loading pages are generated on the server-side but Javascript also
provides progressive enhancements. Although it does not generate pages
anew, the Javascript must know how to create most of the interface
elements so one weakness of this architecture is that much of the
presentation layer logic must be expressed twice.

Despite client devices taking on responsibilities which would previously
have been performed on a server, there is a limit to how much of the
stack may safely be offloaded in this direction. The client-side
ultimately falls under the control of the user so no important business
decisions should be taken here. A banking site should not allow loan
approval to take place in the browser because for the knowledgeable user
any decision would be possible. Separated from data stores by the public
internet, the client is also a poor place to perform data aggregation or
examine large data sets. In non-trivial applications these restrictions
encourage a middle tier to execute business logic and produce aggregate
data.

While REST may not be the only communications technology employed by an
application architecture, for this project we should examine where REST
client libraries may fit into the picture. REST is used by the
presentation layer to pull data from the middle tier regardless of where
the presentation resides. Likewise, rather than connect to databases
directly, for portability the middle tier will often communicate with a
thin REST layer which wraps the data store. This suggests three uses:

-   From web browser to middle tier
-   From server-side presentation layer to middle tier
-   From middle tier to nodes in the data tier

Fortunately, each of these contexts requires a similar performance
profile. The work done is computationally light and answering a request
involves more time waiting than processing. As a part of an interactive
system low latency is important whereas throughput can be increased
relatively cheaply by adding more hardware, especially in a cloud hosted
environment. As demand for the system increases the total work required
grows but the complexity in responding to any one of the requests
remains constant. Although serving any particular request might be done
in series, the workload as a whole is embarrassingly parallelisable.

Node.js
-------

Node.js [@nodejs] is a general purpose tool for executing Javascript
outside of a browser. It has the aim of low-latency I/O and is used
mostly for server applications and command line tools. It is difficult
to judge to what degree Javascript is a distraction from Node's design
as a tool for I/O and to what degree the language defines the platform.

For most imperative languages the thread is the basic unit of
concurrency, whereas Node presents the programmer with a single-threaded
abstraction. Threads are an effective means to share parallel
computation over multiple cores but are less well suited to scheduling
concurrent tasks which are mostly I/O dependent. Safely programming
threads to share mutable objects requires great care and experience,
otherwise the programmer is liable to create race conditions. Consider
for example a Java HTTP aggregator; because we wish to fetch in parallel
each request is assigned to a thread. These 'requester' tasks are
computationally simple: make a request, wait for a complete response,
and then participate in a Barrier while the other requesters complete.
Each thread consumes considerable resources but during its multi-second
lifespan requires only a fraction of a millisecond on the CPU. It is
unlikely that any two requests will return closely enough in time to be
processed in parallel, shedding threading's chief advantage, that it may
process simultaneously utilising multiple cores. Even if requests do
return proximately, the actual CPU time required in making an HTTP
request is so short that any concurrent processing is a pyrrhic victory.

Node builds on a model of event-based, asynchronous I/O which was
established by browser Javascript execution. Although Javascript in a
browser may be performing multiple tasks simultaneously, for example
requesting several resources from the server side, it does so from
within a single-threaded virtual machine. Node facilitates concurrency
by managing an event loop of queued tasks and providing exclusively
non-blocking I/O. Unlike Erlang, Node does not swap tasks out
preemptively, it always waits for a task to complete before moving onto
the next. This means that each task must complete quickly to avoid
holding up others. *Prima facie* this might seem like an onerous
requirement to put on the programmer but in practice with only
non-blocking I/O available each task naturally exits quickly without any
special effort. Accidental non-terminating loops or heavy
number-crunching aside, with no reason for a task to wait it is
difficult to write a node program in which the tasks do not complete
quickly. In production environments Node deployments usually take
advantage of multiple cores by creating one Node instance per processor
code. The separate instances act independently and do not normally use
shared RAM.

Each task in Node is simply a Javascript function. Node is able to swap
its single Javascript thread between these tasks efficiently while
providing the programmer with an intuitive interface because of
closures. Utilising closures, the responsibility of maintaining state
between issuing an asynchronous call and receiving the callback is
removed from the programmer by folding the storage invisibly into the
language. This implicit data store requires no syntax and feels so
natural and inevitable that it is often not obvious that the
responsibility exists at all.

Consider the example below. The code schedules three tasks, each of
which are very short and exit quickly allowing Node to finely interlace
them between other concurrent concerns. The `on` method is used to
attach functions as listeners to streams. However sophisticated and
performant this style of programming, to the developer it is hardly any
more difficult an expression than if blocking I/O were used. It is
certainly harder to make mistakes programming in this way than managing
synchronised access to mutable objects that are shared between threads.

~~~~ {.javascript}
function printResourceToConsole(url) {

   http.get(url)
      .on('response', function(response){
      
         // This function will be called when the response starts.
         // It logs to the console, adds a listener and quickly 
         // exits.
         
         // Because it is captured by a closure we are able to 
         // reference the URL parameter after the scope that 
         // declared it has finished.            
         console.log("The response has started for", url);
      
         response.on('data', function(chunk) {      
            // This function is called each time some data is
            // received from the HTTP request. The task writes
            // the response to the console and quickly exits.
            console.log('Got some response', chunk);
                   
         }).on('end', function(){
            console.log('The response is complete');
         })
         
      }).on("error", function(e){
         
         console.log("There was an error", e.message);
      });      
   console.log("The request has been made");
}   
~~~~

> "Node Stream API, which is the core I/O abstraction in Node.js (which
> is a tool for I/O) is essentially an abstract in/out interface that
> can handle any protocol/stream that also happens to be written in
> JavaScript." [@nodeStream]

In Node I/O is performed using a unified data streaming interface
regardless of the source. The streams fit comfortably with the wider
event-driven model by implementing Node's EventEmitter interface, a
generic dispatcher capable of supporting any event type. Although the
abstraction provided by streams is quite a thin layer on top of the host
system's sockets, it forms a powerful and intuitive interface. For many
tasks it is preferable to program in a 'plumbing' style by joining one
stream's output to another's input. In the example below a resource from
the internet is written to the local filesystem.

~~~~ {.javascript}
http.get(url)
   .on('response', function(response){
      response.pipe(fs.createWriteStream(pathToFile));
   });
~~~~

Following Node's lead, traditionally thread-based environments are
beginning to embrace asynchronous, single-threaded servers. The Netty
project [@netty] can be thought of as roughly an equivalent of Node for
the Java Virtual Machine.

JSON and XML data transfer formats {#jsonxml}
----------------------------------

Both XML and JSON are text based, tree-shaped data formats with human
and machine readability. One of the design goals of XML was to simplify
SGML to the point that a graduate student could implement a full parser
in a week [@javatools p287]. Continuing this arc of simpler data
formats, JSON "The fat-free alternative to XML" [@jsonorg] isolates
Javascript's syntax for literal values into a stand-alone serialisation
language. For the graduate tackling JSON parsing the task is simpler
still, being expressible as fifteen context free grammars.

Whereas XML markup can be traced to document formats, JSON's lineage
is in a programming language. From these roots it isn't surprising that
JSON maps more directly to the metamodels that most programmers think in.
XML parsers produce Elements, Text, Attributes, ProcessingInstruction
which require extra translation before they are convenient to use inside
a programming language. Because JSON already closely resembles how a
programmer would construct a runtime model of their data, fewer steps
are required before using the deserialised form. The JSON nodes:
*strings*, *numbers*, *objects* and *arrays* will in many cases map
directly onto language types and, for loosely typed languages at least,
the parser output bears enough similarity to domain model objects that
it may be used directly without any further transformation.

~~~~ {.javascript}
{
   people: [
      {name: 'John', townOrCity:'London'},
      {name: 'Jack', townOrCity:'Bristol'}
      {townOrCity:'Cambridge', name: 'Sally'}
   ]
}
~~~~

Both JSON and XML are used to serialise orderless constructs but while
expressed as text the encoding is inevitably written according to some
serialisation order. XML specifically states that the order of
attributes is not significant [@xmlorder], JSON has no such detailed
specification but a similar order insignificance seems to be implied by
the JSON object's likeness to Javascript objects whose iteration order
is indeterminate [@ecma3 4.3.3]. In the example above the people objects
would probably have been written based on either a class with two public
properties or a hash map. On receiving this data the text would be
demarshalled into similar orderless structures and it would be quickly forgotten
that the data found an ordered expression during transport. When
viewing a document as a stream and interpreting while still incomplete
it is easier to mistakenly react differently according to field order.
If nodes from the example above were used when only the first field has
arrived Sally would find a different handling than John or Jack. Because
the serialisation will contain items which are written to follow an
indeterminate order it will be important to ensure that, despite the
streaming, the REST client does not encourage programming in a way that
gives different results depending on the order that fields are received.

Common patterns for connecting to REST services
-----------------------------------------------

For languages such as Javascript or Clojure which use a loosely-typed
representation of objects as generic key-value pairs, when a JSON REST
resource is received the output from the parser resembles the normal
object types closely enough that it is acceptable to use it directly
throughout the program. For XML this is not the case in any language and
some marshaling is required. In more strongly typed OO languages such as
Java or C\#, JSON's classless, relatively freeform objects are less
convenient. To smoothly integrate the example JSON from the previous
section, instances of a domain model Person class with methods such as
`getName()` and `getLocation()` would have to be initialised,
representing the remote objects no differently than if they had
originated locally. Automatic marshaling generalises this process by
providing a two-way mapping between the domain model and its
serialisation, either completely automatically or based on a declarative
specification. It is common in strongly typed languages for REST client
libraries to automatically demarshal as part of receiving a fetched REST
response. From the programmer's vantage it is as if the domain objects
themselves had been fetched. Another common design pattern intended to
give a degree of isolation between remote resources and the local domain
model is to demarshal automatically only so far as *Data Transfer
Objects* (DTOs). DTOs are instances of classes which implement no logic
other than storage, and from these DTOs an additional layer
programmatically instantiates the local domain model objects. DTOs are
more necessary when using XML. Reading resources encoded as JSON we
might say that the JSON objects are already DTOs.

The degree of marshaling that is used generally changes only the types
of the entities that the REST client library hands over to the
application developer without affecting the overall structure of the
message. Regardless of the exact types, having received the response
model the developer will usually start by locating the pertinent parts
of the response by drilling down into its structure using the
programming language itself.

~~~~ {.java}
// Java example - programmatic approach to domain model 
// interrogation
// The methods used to drill down to desired components 
// are all getters: getPeople, getName, and getTown.
 
void handleResponse( RestResponse response ) {

   for( Person p : response.getPeople() ) {
      addPersonToDb( p.getName(), p.getTown() );
   }   
}
~~~~

~~~~ {.javascript}
// equivalent Javascript - the programming follows the same basic
// process. This time using Javascript's dot operator.
function handleResponse( response ){

   response.people.forEach( function( person ){
      addPersonToDb( p.name, p.town );
   });
}
~~~~

One weakness of this method for locating resource parts is that the code
making the inspection is coupled to the precise structure of the thing
that it is inspecting. Taking the above example, if the resource being
fetched were later refactored such that the town concept were changed to
a fuller address structure as a street-town-county-country tuple, the
code addressing the structure would also have to change just to continue
to do the same thing. Although this kind of drill-down programming is
commonly practiced and not generally recognised as a code smell,
requiring knock-on changes when an unrelated system is refactored should
perhaps be seen as undesirable in relation to format structures as it
would be elsewhere. DTOs limit the spread of refactoring inside the
client because only the translation from DTO to domain object must be
updated but do not avoid change altogether if a service format is
refactored. In the *Red Queen's race* it took "all the running you can
do, to keep in the same place". Ideally a programmer should only have to
expend effort so that their code does something new, or perform better
something that it already did, not to stay still. Following an object
oriented encapsulation of data such that a caller does not have to
concern itself with the data structures behind an interface, the internal
implementation may be changed without disruptions to the rest of the
code base. However, when the structure of the inter-object composition
is revised, isolation from the changes is less often recognised as a
desirable trait. A method of programming which truly embraced extreme
programming would allow structural refactoring to occur without
disparate parts having to be modified in parallel.

Extraneous changes also dilute a VCS changelog, making it difficult to
later follow a narrative of updates to the logic expressed by the
program. It is therefore harder to later understand the thinking behind
a change or the reason for the change.

Little languages for descending into fetched data
-------------------------------------------------

\label{jsonpathxpath}

To address the problem of drilling down to pertinent fragments of a
message without tightly coupling to the format, consider if instead of
programmatically descending step-by-step, a language were used which
allows the right amount of specificity to be given regarding which parts
to select. In VirtualStudio LINQ [@linq], which is based on
lambda-calculus and resembles SQL, is used to drill-down into data
structures and may also modify the data that is found. However this
style of programming requires the application developer to write
significantly more code than in the simple programmatic access above so
it does not meet the aims of this project.

Given model interrogation logic which is tightly coupled so that the
model's structure cannot change, one suggested approach is Adaptive OOP
[@adaptive] in which no detailed class structure is committed to when
constructing the object oriented program. A REST resource could be
dynamically configured into an OO model according to a formal
specification in a specialisation that is capable of answering the
desired queries. The model that is constructed would be sufficient to
answer the queries without the programmer having to suppose any rigid
form.

Certain markup languages come with associated query languages whose
coupling is loose enough that not every node that is descended through
must be specified. The best known is XPATH but there is also JSONPath, a
JSON equivalent [@jsonpath]. As far as possible, JSONPath's syntax 
resembles the equivalent Javascript:

~~~~ {.javascript}
// in Javascript we can get the town of the second person as:
let town = subject.people[2].town

// the equivalent JSONPath expression is identical:
let townSelector = "people[2].town"

// We would be wise not to write overly-specific selectors.
// JSONPath also provides an ancestor notation which is not present
// in Javascript:
let betterTownSelector = "people[2]..town"
~~~~

Consider the resource below:

~~~~ {.javascript}
{
   people: [
      {name: 'John', town:'Oxford'},
      {name: 'Jack', town:'Bristol'}
      {town:'Cambridge', name: 'Sally'}
   ]
}
~~~~

The JSONPath `people.*..town` may be applied against the above JSON and
would continue to select correctly if the system were refactored to the
version below:

~~~~ {.javascript}
{
   people: [
      {  name: 'John', 
         address:{town:'Oxford', county:'Oxon', country:'uk'}
      },
      {  name: 'Jack',
         address:{town:'Bristol', county:'Bristol', country:'uk'}
      }
      {  address:{
            town:'Cambridge', county:'Cambridgeshire', 
            country:'uk'
         },
         name: 'Sally'
      }
   ]
}
~~~~

formats tend to grow, not shrink

Maintaining compatibility with unanticipated format revisions through
selector languages is easier with JSON than XML. The XML metamodel
contains overlapping representations of equivalent entities which a
format is liable to switch between when being refactored. Each XML element
has two distinct lists of child nodes, attribute children and node list
children. From one perspective attributes are child nodes of their
parent element but they can alternatively be considered as data stored
in the element. Because of this classification ambiguity an XML document
can't be said to form a single n-way tree. XML attributes may only
contain strings and have a lesser expressivity than child nodes which
allow recursive structure; it is a common refactor to change from
attributes to elements when a scalar value is upgraded to a compound.
XPath selectors written in the most natural way do not track this
change.

~~~~ {.xml}
<people>
   <person name="John" town="Oxford"></person>
</people>
~~~~

The XPath `//person@town` identifies the town in the XML above but because of
the switch from attribute to sub-element fails in the
revised version below.

~~~~ {.xml}
<people>
   <person>
      <name>
         John
      </name>
      <address>
         <town>Oxford</town> <county>Oxon</county>
      </address>
   </person>
</people>
~~~~

Reflecting its dual purpose for marking up documents or data, XML also
invites ambiguous interpretation of the whitespace between tags.
Whitespace is usually meaningful for documents but ignorable for data.
Strictly, whitespace text nodes are a part of the document model but in
practice many tree walkers discard them as insignificant. In the XML
above the `<person>` element may be enumerated as either the first or
second child of `<people>` depending on whether the whitespace before it
is considered. Likewise, the text inside `<name>` might be `'John'` or
`'(newline)(tab)(tab)John'`. Inheriting from its programming language
ancestry, in JSON there is no ambiguity. The space between tokens is
never significant.

Programming against a changing service is always going to present a
moving target but it would be easier to miss with XPATH than with
JSONPath. In the JSON metamodel each node has only one, unambiguous set
of children so the format author is not given a choice of logically
equivalent features that must be addressed through different mechanisms.
If a scalar value is updated to a compound only the node changes, the
addressing of the node is unaffected.

Generally in descriptive hierarchical data there is a trend for
ancestorship to signify the same relationship regardless of the number
of intermediate generations. In the example above, `town` transitioned
from a child to grandchild of `person` without disturbing the implicit
'lives in' relationship. In JSONPath the `..` operator provides matching
through zero or more generations, unperturbed when extra levels are
added. This trend does not hold for every way that message semantics may
be built because it is possible that an intermediate node on the path
from ancestor to descendant will change the nature of the expressed
relationship. A slightly contrived example might be if we expanded our
model to contain fuzzy knowledge:

~~~~ {.javascript}
{
   "people": [
      {  
         "name":     {"isProbably":"Bob"},
         "location": {"isNearTo":"Birmingham"}
      }
   ]   
}
~~~~

Considering the general case, it will not be possible to safely track
every refactoring. By necessity a resource consumer should limit their
ambitions to tracking ontology expansions which do not change the
meanings of existing concepts. In practice integration testing against
the beta version of a service will be necessary to be pre-warned of
upcoming, incompatible changes. If an incompatibility is found the
ability to then create an expression which is compatible with a present
and known future version would remain a valuable tool because it
decouples the consumer and provider update schedules, removing the need
for the client to march perfectly in sync with the service.

Browser XML HTTP Request (XHR)
------------------------------

Making HTTP requests from Javascript, commonly termed AJAX, was so
significant in establishing the modern web architecture that it is
sometimes used synonymously with Javascript-rich web applications.
Although AJAX is an acronym for **A**synchronous **J**avascript
(**a**nd) **X**ML, this reflects the early millennial enthusiasm for XML
as the one true data format and in practice any textual format may be
transferred. During the 'browser war' years web browsers competed by
adding non-standard features; Internet Explorer made AJAX possible in
2000 by exposing Microsoft's Active X *Xml Http Request* (XHR) class to
the Javascript sandbox. This was widely copied and near equivalents were
added to all major browsers. In 2006 the interface was eventually
formalised by the W3C [@xhrWorkingDraft]. XHR's slow progress to
standardisation reflected a period of general stagnation for web
standards. HTML4 reached Recommendation status in 2001 but having
subsequently found several evolutionary dead ends such as XHTML, there
would be no major updates until HTML5 started to gather pace some ten
years later.

Despite a reputation for being poorly standardised, as a language
Javascript enjoys consistent implementation. More accurately we would
say that browser APIs exposed to Javascript lack compatibility. Given
this backdrop of vendor extensions and lagging standardisation,
abstraction layers predictably rose in popularity. Various abstractions
competed primarily on developer ergonomics with the popular jQuery and
Prototype.js libraries promoting themselves as *"do more, write less"*
and *"elegant APIs around the clumsy interfaces of Ajax"*. Written
against the unadorned browser, Javascript applications read as a maze of
platform detection and special cases. Once applications were built using
abstractions over the underlying platform differences they could be
written purposefully and programmers were able to express more complex
ideas.

Today JSON is generally the preferred format, especially for resources
transmitted to client-side web applications. Javascript programmers
occupy a privileged position whereby their serialisation format maps
exactly onto the inbuilt types of their programming language. As such
there is never any confusion regarding which object structure to
de-serialise to. Should this advantage seem insubstantial, contrast with
the plethora of confusing and incompatible representations of JSON that
are output by the various Java parsers: JSON's Object better resembles
Java's Map interface than Java Objects, creating linguistic
difficulties, and the confusion between JSON null, Java null, and
Jackson's NullNode[^1] is a common cause of errors. Emboldened by
certainty regarding deserialisation, AJAX libraries directly integrate
JSON parsers, providing a call style for working with remote resources
so streamlined as to require hardly any additional effort.

~~~~ {.javascript}
ajax('http://example.com/people.json', function( people ) {

   // The parsing of the people JSON into a javascript object
   // feels so natural that it is easy to forget from looking 
   // at the code that parsing happens at all. 
   
   console.log('the first person is called', people[0].name);
});
~~~~

XHRs and streaming
------------------

\label{xhrsandstreaming}

Browser abstraction layers brought an improvement in expressivity to web
application programming but were ultimately limited to supporting the
lowest common denominator of available browser abilities. When the call
style above was developed the most popular browser barred access to
in-progress responses so the inevitable conceptualisation was drawn of
the response as a one-time event with no accommodation provided for
progressively delivered data.

The followup standard, XHR2 is now at Working Draft stage
[@xhr2progress]. Given ambitions to build a streaming REST client, of
greatest interest is the progress event:

> While the download is progressing, queue a task to fire a progress
> event named progress about every 50ms or for every byte received,
> whichever is least frequent.

The historic lack of streaming for data fetched using XHR stands
incongruously with the browser as a platform in which almost every other
remote resource is interpreted progressively. Examples include
progressive image formats, HTML, SVG, and video.

The progress event is supported by the latest version of all major
browsers. However, Internet Explorer only added support recently with
version 10 and there is a significant user base remaining on versions 8
and 9.

Browser streaming frameworks
----------------------------

\label{browserstreamingframeworks}

The web's remit is increasingly widening to encompass scenarios which
would have previously been the domain of native applications. In order
to use live data many current webapps employ frameworks which push
soft-real-time events to the client side. This kind of streaming
intersects only narrowly with the aims of the XHR2 progress event.
Whereas XHR2 enables downloads to be viewed as streams but does not
otherwise disrupt the sequence of HTTP's request-response model,
streaming frameworks facilitate an entirely different sequence, that of
perpetual data. Consider a webmail interface; initially the user's inbox
is downloaded via REST and although a streaming download might be used
to make its display more responsive, the inbox download is a standard
REST call and shares little in common with the push events which follow
to provide instant notification as new messages arrive.

**Push tables** sidestep the browser's absent data streaming abilities
by leaning on a resource that it can stream: progressive HTML. On the
client a page containing a table is hidden in an off-screen iframe. The
frame's content is served as an HTML page containing a table that never
completes, fed by a connection that never closes. When the server wishes
to push a message to the client it writes a new row to the table which
is then noticed by Javascript monitoring the iframe on the client. More
recently, **Websockets** provides a standardised streaming transport on
top of HTTP's chunked mode. Websockets requires browser implementation
and cannot be retrofitted to older browsers through Javascript. It is a
promising technology but for the time being patchy support means it
cannot be used without a suitable fallback.

These frameworks do not interoperate at all with REST. Because the
resources they serve never complete they may not be read by a standard
REST client. Unlike REST they also are not amenable to standard HTTP
mechanisms such as caching. A server which writes to an esoteric format
requiring a specific, known, specialised client also feels quite
anti-REST, especially when we consider that the format design reflects
the nature of the transport more so than the resource. This form of
streaming is not, however, entirely alien to a SOA mindset. The data
formats, while not designed primarily for human readability are
nonetheless text based and a person may take a peek inside the system's
plumbing simply by observing the traffic at a particular URL. For push
tables, because the transport is based on a presentation format, an
actual table of the event's properties may be viewed from a browser as
the messages are streamed.

Parsing: SAX and DOM
--------------------

From the XML world two standard parser types exist, SAX and DOM, with
DOM by far the more popular. Both styles of parsers are also available
for JSON. DOM performs a parse as a single evaluation and returns an
object model representing the whole of the document. Conversely, SAX
parsers are probably better considered as enhanced tokenisers, providing
a very low-level event driven interface that notifies the programmer of
each token separately as it is found. Working with DOM's level of
abstraction the markup syntax is a distant concern whereas for SAX each
element's opening and closing must be noted so the developer may not put
the data's serialisation aside. SAX comes with the advantages that it
may read a document progressively and has lower memory requirements
because it does not store the parsed tree. It is a
popular choice for embedded systems running on constrained hardware and may be
used to handle documents larger than the available RAM.

Suppose we have some JSON representing people and want to extract the
name of the first person. Given a DOM parser this may be written quite
succinctly:

~~~~ {.javascript}
function nameOfFirstPerson( myJsonString ) {

   // All recent browsers provide JSON.parse as standard. 
   var document = JSON.parse( myJsonString );
   return document.people[0].name; // that was easy!
}
~~~~

To contrast, the equivalent below uses the Javascript Clarinet SAX
parser, expressed in the most natural way for the technology[^2].

~~~~ {.javascript}
function nameOfFirstPerson( myJsonString, callbackFunction ){


   var clarinet = clarinet.parser(),
   
       // With a SAX parser it is the developer's responsibility 
       // to track where in the document the cursor currently is.
       // Several variables are required to maintain this state.        
       inPeopleArray = false,   
       inPersonObject = false,
       inNameAttribute = false,
       found = false;
   
   clarinet.onopenarray = function(){
      // For brevity we'll cheat by assuming there is only one
      // array in the document. In practice this would be overly
      // brittle.      
      inPeopleArray = true; 
   };
   
   clarinet.onclosearray = function(){
      inPeopleArray = false;
   };   
   
   clarinet.onopenobject = function(){
      inPersonObject = inPeopleArray; 
   };
   
   clarinet.oncloseobject = function(){
      inPersonObject = false;
   };   
      
   clarinet.onkey = function(key){
      inNameAttribute = (inPeopleObject && key == 'name');
   };

   clarinet.onvalue = function(value){
      if( !found && inNameAttribute ) {
         // finally!
         callbackFunction( value );
         found = true;
      }
   };      
   
   clarinet.write(myJsonString);   
}
~~~~

The developer pays a high price for progressive parsing, the SAX version
is considerably longer and more difficult to read. SAX's low-level
semantics require a lengthy expression and push onto the programmer the
responsibility for managing state regarding the current position in the
document and storing data extracted from previously seen nodes. This
maintenance of state tends to be programmed once per usage rather than
assembled as the composition of reusable parts. The ordering of the code
under SAX is also quite unintuitive; event handlers cover multiple
unrelated cases and each concern spans multiple handlers. This lends to
a style of programming in which separate concerns do not find separate
expression in the code. It is also notable that, unlike DOM, as the
depth of the document being interpreted increases, the length of the
programming required to interpret it also increases, mandating more
state be stored and an increased number of cases be covered per event
handler.

While SAX addresses many of the problems raised in this dissertation,
its unfriendly developer ergonomics have presented too high a barrier
for adoption for all but fringe use cases.

[^1]: See
    <http://jackson.codehaus.org/1.0.1/javadoc/org/codehaus/jackson/node/NullNode.html>

[^2]: For an example closer to the real world see
    https://github.com/dscape/clarinet/blob/master/samples/twitter.js
