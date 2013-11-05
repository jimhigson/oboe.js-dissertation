---
title:  'Oboe.js: An approach to I/O for REST clients which is neither batch nor stream; nor SAX nor DOM'
date: November 2013
author:
- Jim Higson, Kellogg College
abstract: |
   A new design is presented for HTTP client libraries which incorporates HTTP streaming,
   pattern matching, and incremental parsing, with the aim of improving
   performance, fault tolerance, and encouraging a greater degree of loose
   coupling between programs. A Javascript client library capable of
   progressively parsing JSON resources is presented targeting both Node.js
   and web browsers. Loose coupling is particularly considered in light of
   the application of Agile methodologies to REST and SOA, providing a
   framework in which it is acceptable to partially restructure the JSON
   format of a resource while maintaining compatibility with dependent
   systems.
   
   A critique is made of current practice under which resources are
   entirely retrieved before items of interest are extracted
   programmatically. An alternative model is presented allowing the
   specification of items of interest using a declarative syntax similar to
   JSONPath. The identified items are then provided incrementally while the
   resource is still downloading.
      
   In addition to a consideration of performance in absolute terms, the
   usability implications of an incremental model are also considered with
   regards to developer ergonomics and user perception of performance.
...
