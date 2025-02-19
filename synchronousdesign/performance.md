---
layout: default
title: Performance
parent: Synchronous-API Design
nav_order: 18
---

{SHOULD} Reduce Bandwidth Needs and Improve Responsiveness
==========================================================

APIs should support techniques for reducing bandwidth based on client needs. This holds for APIs that (might) have high payloads and/or are used in high-traffic scenarios like the public Internet and telecommunication networks. Typical examples are APIs used by mobile web app clients with (often) less bandwidth connectivity. (Zalando is a 'Mobile First' company, so be mindful of this point.)

Common techniques include:

-   compression of request and response bodies (see [{SHOULD} Use gzip Compression](#156))

-   querying field filters to retrieve a subset of resource attributes (see [{SHOULD} Support Partial Responses via Filtering](#157) below)

-   {ETag} and {If-Match}/{If-None-Match} headers to avoid re-fetching of unchanged resources (see [???](#182))

-   {Prefer} header with `return=minimal` or `respond-async` to anticipate reduced processing requirements of clients (see [???](#181))

-   [???](#pagination) for incremental access of larger collections of data items

-   caching of master data items, i.e. resources that change rarely or not at all after creation (see [{MUST} Document Cachable , , and Endpoints](#227)).

Each of these items is described in greater detail below.

{SHOULD} Use gzip Compression
=============================

Compress the payload of your API’s responses with gzip, unless there’s a good reason not to — for example, you are serving so many requests that the time to compress becomes a bottleneck. This helps to transport data faster over the network (fewer bytes) and makes frontends respond faster.

Though gzip compression might be the default choice for server payload, the server should also support payload without compression and its client control via {Accept-Encoding} request header — see also {RFC-7231}\#section-5.3.4\[RFC 7231 Section 5.3.4\]. The server should indicate used gzip compression via the {Content-Encoding} header.

{SHOULD} Support Partial Responses via Filtering
================================================

Depending on your use case and payload size, you can significantly reduce network bandwidth need by supporting filtering of returned entity fields. Here, the client can explicitly determine the subset of fields he wants to receive via the `fields` query parameter. (It is analogue to [GraphQL `fields`](https://graphql.org/learn/queries/#fields) and simple queries, and also applied, for instance, for [Google Cloud API’s partial responses](https://cloud.google.com/storage/docs/json_api/v1/how-tos/performance#partial-response).)

Unfiltered
----------

    GET http://api.example.org/users/123 HTTP/1.1

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "id": "cddd5e44-dae0-11e5-8c01-63ed66ab2da5",
      "name": "John Doe",
      "address": "1600 Pennsylvania Avenue Northwest, Washington, DC, United States",
      "birthday": "1984-09-13",
      "friends": [ {
        "id": "1fb43648-dae1-11e5-aa01-1fbc3abb1cd0",
        "name": "Jane Doe",
        "address": "1600 Pennsylvania Avenue Northwest, Washington, DC, United States",
        "birthday": "1988-04-07"
      } ]
    }

Filtered
--------

    GET http://api.example.org/users/123?fields=(name,friends(name)) HTTP/1.1

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "name": "John Doe",
      "friends": [ {
        "name": "Jane Doe"
      } ]
    }

The query `field` value determines the fields returned with the response payload object. For instance, `(name)` returns `users` root object with only the `name` field, and `(name,friends(name))` returns the `name` and the nested `friends` object with only its `name` field.

OpenAPI doesn’t support you in formally specifying different return object schemes depending on a parameter. When you define the field parameter, we recommend to provide the following description: `Endpoint supports filtering of return object fields as
described in [Rule #157](https://opensource.zalando.com/restful-api-guidelines/#157)`

The syntax of the query `field` value is defined by the following [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) grammar.

    <fields>            ::= [ <negation> ] <fields_struct>
    <fields_struct>     ::= "(" <field_items> ")"
    <field_items>       ::= <field> [ "," <field_items> ]
    <field>             ::= <field_name> | <fields_substruct>
    <fields_substruct>  ::= <field_name> <fields_struct>
    <field_name>        ::= <dash_letter_digit> [ <field_name> ]
    <dash_letter_digit> ::= <dash> | <letter> | <digit>
    <dash>              ::= "-" | "_"
    <letter>            ::= "A" | ... | "Z" | "a" | ... | "z"
    <digit>             ::= "0" | ... | "9"
    <negation>          ::= "!"

{SHOULD} Allow Optional Embedding of Sub-Resources
==================================================

Embedding related resources (also know as *Resource expansion*) is a great way to reduce the number of requests. In cases where clients know upfront that they need some related resources they can instruct the server to prefetch that data eagerly. Whether this is optimized on the server, e.g. a database join, or done in a generic way, e.g. an HTTP proxy that transparently embeds resources, is up to the implementation.

See [???](#137) for naming, e.g. "embed" for steering of embedded resource expansion. Please use the [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) grammar, as already defined above for filtering, when it comes to an embedding query syntax.

Embedding a sub-resource can possibly look like this where an order resource has its order items as sub-resource (/order/{orderId}/items):

    GET /order/123?embed=(items) HTTP/1.1

    {
      "id": "123",
      "_embedded": {
        "items": [
          {
            "position": 1,
            "sku": "1234-ABCD-7890",
            "price": {
              "amount": 71.99,
              "currency": "EUR"
            }
          }
        ]
      }
    }

{MUST} Document Cachable `GET`, `HEAD`, and `POST` Endpoints
============================================================

Caching has to take many aspects into account, e.g. general [cacheability](#cacheable) of response information, our guideline to protect endpoints using SSL and [OAuth authorization](#104), resource update and invalidation rules, existence of multiple consumer instances. As a consequence, caching is in best case complex, e.g. with respect to consistency, in worst case inefficient.

As a consequence, client side as well as transparent web caching should be avoided, unless the service supports and requires it to protect itself, e.g. in case of a heavily used and therefore rate limited master data service, i.e. data items that rarely or not at all change after creation.

As default, API providers and consumers should always set the {Cache-Control} header set to {Cache-Control-no-store} and assume the same setting, if no {Cache-Control} header is provided.

**Note:** There is no need to document this default setting. However, please make sure that your framework is attaching this header value by default, or ensure this manually, e.g. using the best practice of Spring Security as shown below. Any setup deviating from this default must be sufficiently documented.

    Cache-Control: no-cache, no-store, must-revalidate, max-age=0

If your service really requires to support caching, please observe the following rules:

-   Document all [???](#cacheable) {GET}, {HEAD}, and {POST} endpoints by declaring the support of {Cache-Control}, {Vary}, and {ETag} headers in response. **Note:** you must not define the {Expires} header to prevent redundant and ambiguous definition of cache lifetime. A sensible default documentation of these headers is given below.

-   Take care to specify the ability to support caching by defining the right caching boundaries, i.e. time-to-live and cache constraints, by providing sensible values for {Cache-Control} and {Vary} in your service. We will explain best practices below.

-   Provide efficient methods to warm up and update caches, e.g. as follows:

    -   In general, you should support [`ETag` Together With `If-Match`/ `If-None-Match` Header](#182) on all [???](#cacheable) endpoints.

    -   For larger data items support {HEAD} requests or more efficient {GET} requests with {If-None-Match} header to check for updates.

    -   For small data sets provide full collection {GET} requests supporting {ETag}, as well as {HEAD} requests or {GET} requests with {If-None-Match} to check for updates.

    -   For medium sized data sets provide full collection {GET} requests supporting {ETag} together with [???](#pagination) and {entity-tag} filtering {GET} requests for limiting the response to changes since the provided {entity-tag}. **Note:** this is not supported by generic client and proxy caches on HTTP layer.

**Hint:** For proper cache support, you must return {304} without content on a failed {HEAD} or {GET} request with [`If-None-Match: <entity-tag>`](#182) instead of {412}.

    components:
      headers:
      - Cache-Control:
          description: |
            The RFC 7234 Cache-Control header field is providing directives to
            control how proxies and clients are allowed to cache responses results
            for performance. Clients and proxies are free to not support caching of
            results, however if they do, they must obey all directives mentioned in
            [RFC-7234 Section 5.2.2](https://tools.ietf.org/html/rfc7234) to the
            word.

            In case of caching, the directive provides the scope of the cache
            entry, i.e. only for the original user (private) or shared between all
            users (public), the lifetime of the cache entry in seconds (max-age),
            and the strategy how to handle a stale cache entry (must-revalidate).
            Please note, that the lifetime and validation directives for shared
            caches are different (s-maxage, proxy-revalidate).

          type: string
          required: false
          example: "private, must-revalidate, max-age=300"

      - Vary:
          description: |
            The RFC 7231 Vary header field in a response defines which parts of
            a request message, aside the target URL and HTTP method, might have
            influenced the response. A client or proxy cache must respect this
            information, to ensure that it delivers the correct cache entry (see
            [RFC-7231 Section
            7.1.4](https://tools.ietf.org/html/rfc7231#section-7.1.4)).

          type: string
          required: false
          example: "accept-encoding, accept-language"

**Hint:** For {ETag} source see [???](#182).

The default setting for {Cache-Control} should contain the `private` directive for endpoints with standard [OAuth authorization](#104), as well as the `must-revalidate` directive to ensure, that the client does not use stale cache entries. Last, the `max-age` directive should be set to a value between a few seconds (`max-age=60`) and a few hours (`max-age=86400`) depending on the change rate of your master data and your requirements to keep clients consistent.

    Cache-Control: private, must-revalidate, max-age=300

The default setting for {Vary} is harder to determine correctly. It highly depends on the API endpoint, e.g. whether it supports compression, accepts different media types, or requires other request specific headers. To support correct caching you have to carefully choose the value. However, a good first default may be:

    Vary: accept, accept-encoding

Anyhow, this is only relevant, if you encourage clients to install generic HTTP layer client and proxy caches.

**Note:** generic client and proxy caching on HTTP level is hard to configure. Therefore, we strongly recommend to attach the (possibly distributed) cache directly to the service (or gateway) layer of your application. This relieves from interpreting the {vary} header and greatly simplifies interpreting the {Cache-Control} and {ETag} headers. Moreover, is highly efficient with respect to caching performance and overhead, and allows to support more [advanced cache update and warm up patterns](#cache-support-patterns).

Anyhow, please carefully read {RFC-7234}\[RFC 7234\] before adding any client or proxy cache.
