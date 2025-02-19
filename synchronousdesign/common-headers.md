---
layout: default
title: Common Headers
parent: Synchronous-API Design
nav_order: 5
---

This section describes a handful of headers, which we found raised the most questions in our daily usage, or which are useful in particular circumstances but not widely known.

{MUST} Use `Content-*` Headers Correctly
========================================

Content or entity headers are headers with a `Content-` prefix. They describe the content of the body of the message and they can be used in both, HTTP requests and responses. Commonly used content headers include but are not limited to:

-   {Content-Disposition} can indicate that the representation is supposed to be saved as a file, and the proposed file name.

-   {Content-Encoding} indicates compression or encryption algorithms applied to the content.

-   {Content-Length} indicates the length of the content (in bytes).

-   {Content-Language} indicates that the body is meant for people literate in some human language(s).

-   {Content-Location} indicates where the body can be found otherwise ([{MAY} Use Header](#179) for more details\]).

-   {Content-Range} is used in responses to range requests to indicate which part of the requested resource representation is delivered with the body.

-   {Content-Type} indicates the media type of the body content.

{MAY} Use Standardized Headers
==============================

Use [this list](http://en.wikipedia.org/wiki/List_of_HTTP_header_fields) and mention its support in your OpenAPI definition.

{MAY} Use `Content-Location` Header
===================================

The {Content-Location} header is *optional* and can be used in successful write operations ({PUT}, {POST}, or {PATCH}) or read operations ({GET}, {HEAD}) to guide caching and signal a receiver the actual location of the resource transmitted in the response body. This allows clients to identify the resource and to update their local copy when receiving a response with this header.

The Content-Location header can be used to support the following use cases:

-   For reading operations {GET} and {HEAD}, a different location than the requested URI can be used to indicate that the returned resource is subject to content negotiations, and that the value provides a more specific identifier of the resource.

-   For writing operations {PUT} and {PATCH}, an identical location to the requested URI can be used to explicitly indicate that the returned resource is the current representation of the newly created or updated resource.

-   For writing operations {POST} and {DELETE}, a content location can be used to indicate that the body contains a status report resource in response to the requested action, which is available at provided location.

**Note**: When using the {Content-Location} header, the {Content-Type} header has to be set as well. For example:

    GET /products/123/images HTTP/1.1

    HTTP/1.1 200 OK
    Content-Type: image/png
    Content-Location: /products/123/images?format=raw

{SHOULD} Use `Location` Header instead of `Content-Location` Header
===================================================================

As the correct usage of {Content-Location} with respect to semantics and caching is difficult, we *discourage* the use of {Content-Location}. In most cases it is sufficient to direct clients to the resource location by using the {Location} header instead without hitting the {Content-Location} specific ambiguities and complexities.

More details in RFC 7231 {RFC-7231}\#section-7.1.2\[7.1.2 Location\], {RFC-7231}\#section-3.1.4.2\[3.1.4.2 Content-Location\]

{MAY} Consider to Support `Prefer` Header to Handle Processing Preferences
==========================================================================

The {Prefer} header defined in {RFC-7240}\[RFC 7240\] allows clients to request processing behaviors from servers. It pre-defines a number of preferences and is extensible, to allow others to be defined. Support for the {Prefer} header is entirely optional and at the discretion of API designers, but as an existing Internet Standard, is recommended over defining proprietary "X-" headers for processing directives.

The {Prefer} header can defined like this in an API definition:

    components:
      headers:
      - Prefer:
          description: >
            The RFC7240 Prefer header indicates that a particular server behavior
            is preferred by the client but is not required for successful completion
            of the request (see [RFC 7240](https://tools.ietf.org/html/rfc7240).
            The following behaviors are supported by this API:

            # (indicate the preferences supported by the API or API endpoint)
            * **respond-async** is used to suggest the server to respond as fast as
              possible asynchronously using 202 - accepted - instead of waiting for
              the result.
            * **return=<minimal|representation>** is used to suggest the server to
              return using 204 without resource (minimal) or using 200 or 201 with
              resource (representation) in the response body on success.
            * **wait=<delta-seconds>** is used to suggest a maximum time the server
              has time to process the request synchronously.
            * **handling=<strict|lenient>** is used to suggest the server to be
              strict and report error conditions or lenient, i.e. robust and try to
              continue, if possible.

          type: array
          items:
            type: string
          required: false

**Note:** Please copy only the behaviors into your {Prefer} header specification that are supported by your API endpoint. If necessary, specify different {Prefer} headers for each supported use case.

Supporting APIs may return the {Preference-Applied} header also defined in {RFC-7240}\[RFC 7240\] to indicate whether a preference has been applied.

{MAY} Consider to Support `ETag` Together With `If-Match`/`If-None-Match` Header
================================================================================

When creating or updating resources it may be necessary to expose conflicts and to prevent the 'lost update' or 'initially created' problem. Following {RFC-7232}\[RFC 7232 "HTTP: Conditional Requests"\] this can be best accomplished by supporting the {ETag} header together with the {If-Match} or {If-None-Match} conditional header. The contents of an `ETag: <entity-tag>` header is either (a) a hash of the response body, (b) a hash of the last modified field of the entity, or (c) a version number or identifier of the entity version.

To expose conflicts between concurrent update operations via {PUT}, {POST}, or {PATCH}, the `If-Match: <entity-tag>` header can be used to force the server to check whether the version of the updated entity is conforming to the requested {entity-tag}. If no matching entity is found, the operation is supposed a to respond with status code {412} - precondition failed.

Beside other use cases, `If-None-Match: *` can be used in a similar way to expose conflicts in resource creation. If any matching entity is found, the operation is supposed a to respond with status code {412} - precondition failed.

The {ETag}, {If-Match}, and {If-None-Match} headers can be defined as follows in the API definition:

    components:
      headers:
      - ETag:
          description: |
            The RFC 7232 ETag header field in a response provides the entity-tag of
            a selected resource. The entity-tag is an opaque identifier for versions
            and representations of the same resource over time, regardless whether
            multiple versions are valid at the same time. An entity-tag consists of
            an opaque quoted string, possibly prefixed by a weakness indicator (see
            [RFC 7232 Section 2.3](https://tools.ietf.org/html/rfc7232#section-2.3).

          type: string
          required: false
          example: W/"xy", "5", "5db68c06-1a68-11e9-8341-68f728c1ba70"

      - If-Match:
          description: |
            The RFC7232 If-Match header field in a request requires the server to
            only operate on the resource that matches at least one of the provided
            entity-tags. This allows clients express a precondition that prevent
            the method from being applied if there have been any changes to the
            resource (see [RFC 7232 Section
            3.1](https://tools.ietf.org/html/rfc7232#section-3.1).

          type: string
          required: false
          example: "5", "7da7a728-f910-11e6-942a-68f728c1ba70"

      - If-None-Match:
          description: |
            The RFC7232 If-None-Match header field in a request requires the server
            to only operate on the resource if it does not match any of the provided
            entity-tags. If the provided entity-tag is `*`, it is required that the
            resource does not exist at all (see [RFC 7232 Section
            3.2](https://tools.ietf.org/html/rfc7232#section-3.2).

          type: string
          required: false
          example: "7da7a728-f910-11e6-942a-68f728c1ba70", *

Please see [???](#optimistic-locking) for a detailed discussion and options.

{MAY} Consider to Support `Idempotency-Key` Header
==================================================

When creating or updating resources it can be helpful or necessary to ensure a strong [???](#idempotent) behavior comprising same responses, to prevent duplicate execution in case of retries after timeout and network outages. Generally, this can be achieved by sending a client specific *unique request key* – that is not part of the resource – via {Idempotency-Key} header.

The *unique request key* is stored temporarily, e.g. for 24 hours, together with the response and the request hash (optionally) of the first request in a key cache, regardless of whether it succeeded or failed. The service can now look up the *unique request key* in the key cache and serve the response from the key cache, instead of re-executing the request, to ensure [???](#idempotent) behavior. Optionally, it can check the request hash for consistency before serving the response. If the key is not in the key store, the request is executed as usual and the response is stored in the key cache.

This allows clients to safely retry requests after timeouts, network outages, etc. while receive the same response multiple times. **Note:** The request retry in this context requires to send the exact same request, i.e. updates of the request that would change the result are off-limits. The request hash in the key cache can protection against this misbehavior. The service is recommended to reject such a request using status code {400}.

**Important:** To grant a reliable [???](#idempotent) execution semantic, the resource and the key cache have to be updated with hard transaction semantics – considering all potential pitfalls of failures, timeouts, and concurrent requests in a distributed systems. This makes a correct implementation exceeding the local context very hard.

The {Idempotency-Key} header must be defined as follows, but you are free to choose your expiration time:

    components:
      headers:
      - Idempotency-Key:
          description: |
            The idempotency key is a free identifier created by the client to
            identify a request. It is used by the service to identify subsequent
            retries of the same request and ensure idempotent behavior by sending
            the same response without executing the request a second time.

            Clients should be careful as any subsequent requests with the same key
            may return the same response without further check. Therefore, it is
            recommended to use an UUID version 4 (random) or any other random
            string with enough entropy to avoid collisions.

            Idempotency keys expire after 24 hours. Clients are responsible to stay
            within this limits, if they require idempotent behavior.

          type: string
          format: uuid
          required: false
          example: "7da7a728-f910-11e6-942a-68f728c1ba70"

**Hint:** The key cache is not intended as request log, and therefore should have a limited lifetime, else it could easily exceed the data resource in size.

**Note:** The {Idempotency-Key} header unlike other headers in this section is not standardized in an RFC. Our only reference are the usage in the [Stripe API](https://stripe.com/docs/api/idempotent_requests). However, as it fit not into our section about [???](#proprietary-headers), and we did not want to change the header name and semantic, we decided to treat it as any other common header.
