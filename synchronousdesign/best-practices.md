---
layout: default
title: Best Practices
parent: Synchronous-API Design
nav_order: 3
---

The best practices presented in this section are not part of the actual guidelines, but should provide guidance for common challenges we face when implementing RESTful APIs.

Optimistic Locking in RESTful APIs
==================================

Introduction
------------

Optimistic locking might be used to avoid concurrent writes on the same entity, which might cause data loss. A client always has to retrieve a copy of an entity first and specifically update this one. If another version has been created in the meantime, the update should fail. In order to make this work, the client has to provide some kind of version reference, which is checked by the service, before the update is executed. Please read the more detailed description on how to update resources via {PUT} in the [HTTP Requests Section](#put).

A RESTful API usually includes some kind of search endpoint, which will then return a list of result entities. There are several ways to implement optimistic locking in combination with search endpoints which, depending on the approach chosen, might lead to performing additional requests to get the current version of the entity that should be updated.

`ETag` with `If-Match` header
-----------------------------

An {ETag} can only be obtained by performing a {GET} request on the single entity resource before the update, i.e. when using a search endpoint an additional request is necessary.

Example:

    < GET /orders

    > HTTP/1.1 200 OK
    > {
    >   "items": [
    >     { "id": "O0000042" },
    >     { "id": "O0000043" }
    >   ]
    > }

    < GET /orders/BO0000042

    > HTTP/1.1 200 OK
    > ETag: osjnfkjbnkq3jlnksjnvkjlsbf
    > { "id": "BO0000042", ... }

    < PUT /orders/O0000042
    < If-Match: osjnfkjbnkq3jlnksjnvkjlsbf
    < { "id": "O0000042", ... }

    > HTTP/1.1 204 No Content

Or, if there was an update since the {GET} and the entity’s {ETag} has changed:

    > HTTP/1.1 412 Precondition failed

#### Pros

-   RESTful solution

#### Cons

-   Many additional requests are necessary to build a meaningful front-end

`ETags` in result entities
--------------------------

The ETag for every entity is returned as an additional property of that entity. In a response containing multiple entities, every entity will then have a distinct {ETag} that can be used in subsequent {PUT} requests.

In this solution, the `etag` property should be `readonly` and never be expected in the {PUT} request payload.

Example:

    < GET /orders

    > HTTP/1.1 200 OK
    > {
    >   "items": [
    >     { "id": "O0000042", "etag": "osjnfkjbnkq3jlnksjnvkjlsbf", "foo": 42, "bar": true },
    >     { "id": "O0000043", "etag": "kjshdfknjqlowjdsljdnfkjbkn", "foo": 24, "bar": false }
    >   ]
    > }

    < PUT /orders/O0000042
    < If-Match: osjnfkjbnkq3jlnksjnvkjlsbf
    < { "id": "O0000042", "foo": 43, "bar": true }

    > HTTP/1.1 204 No Content

Or, if there was an update since the {GET} and the entity’s {ETag} has changed:

    > HTTP/1.1 412 Precondition failed

#### Pros

-   Perfect optimistic locking

#### Cons

-   Information that only belongs in the HTTP header is part of the business objects

Version numbers
---------------

The entities contain a property with a version number. When an update is performed, this version number is given back to the service as part of the payload. The service performs a check on that version number to make sure it was not incremented since the consumer got the resource and performs the update, incrementing the version number.

Since this operation implies a modification of the resource by the service, a {POST} operation on the exact resource (e.g. `POST /orders/O0000042`) should be used instead of a {PUT}.

In this solution, the `version` property is not `readonly` since it is provided at {POST} time as part of the payload.

Example:

    < GET /orders

    > HTTP/1.1 200 OK
    > {
    >   "items": [
    >     { "id": "O0000042", "version": 1,  "foo": 42, "bar": true },
    >     { "id": "O0000043", "version": 42, "foo": 24, "bar": false }
    >   ]
    > }

    < POST /orders/O0000042
    < { "id": "O0000042", "version": 1, "foo": 43, "bar": true }

    > HTTP/1.1 204 No Content

or if there was an update since the {GET} and the version number in the database is higher than the one given in the request body:

    > HTTP/1.1 409 Conflict

#### Pros

-   Perfect optimistic locking

#### Cons

-   Functionality that belongs into the HTTP header becomes part of the business object

-   Using {POST} instead of PUT for an update logic (not a problem in itself, but may feel unusual for the consumer)

`Last-Modified` / `If-Unmodified-Since`
---------------------------------------

In HTTP 1.0 there was no {ETag} and the mechanism used for optimistic locking was based on a date. This is still part of the HTTP protocol and can be used. Every response contains a {Last-Modified} header with a HTTP date. When requesting an update using a {PUT} request, the client has to provide this value via the header {If-Unmodified-Since}. The server rejects the request, if the last modified date of the entity is after the given date in the header.

This effectively catches any situations where a change that happened between {GET} and {PUT} would be overwritten. In the case of multiple result entities, the {Last-Modified} header will be set to the latest date of all the entities. This ensures that any change to any of the entities that happens between {GET} and {PUT} will be detectable, without locking the rest of the batch as well.

Example:

    < GET /orders

    > HTTP/1.1 200 OK
    > Last-Modified: Wed, 22 Jul 2009 19:15:56 GMT
    > {
    >   "items": [
    >     { "id": "O0000042", ... },
    >     { "id": "O0000043", ... }
    >   ]
    > }

    < PUT /block/O0000042
    < If-Unmodified-Since: Wed, 22 Jul 2009 19:15:56 GMT
    < { "id": "O0000042", ... }

    > HTTP/1.1 204 No Content

Or, if there was an update since the {GET} and the entities last modified is later than the given date:

    > HTTP/1.1 412 Precondition failed

#### Pros

-   Well established approach that has been working for a long time

-   No interference with the business objects; the locking is done via HTTP headers only

-   Very easy to implement

-   No additional request needed when updating an entity of a search endpoint result

#### Cons

-   If a client communicates with two different instances and their clocks are not perfectly in sync, the locking could potentially fail

Conclusion
----------

We suggest to either use the *{ETag} in result entities* or *{Last-Modified} / {If-Unmodified-Since}* approach.
