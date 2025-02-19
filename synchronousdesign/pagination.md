---
layout: default
title: Pagination
parent: Synchronous-API Design
nav_order: 17
---

{MUST} Support Pagination
=========================

Access to lists of data items must support pagination to protect the service against overload as well as for best client side iteration and batch processing experience. This holds true for all lists that are (potentially) larger than just a few hundred entries.

There are two well known page iteration techniques:

-   [Offset/Limit-based pagination](https://developer.infoconnect.com/paging-results): numeric offset identifies the first page entry

-   [Cursor/Limit-based](https://dev.twitter.com/overview/api/cursoring) — aka key-based — pagination: a unique key element identifies the first page entry (see also [Facebook’s guide](https://developers.facebook.com/docs/graph-api/using-graph-api/v2.4#paging))

The technical conception of pagination should also consider user experience related issues. As mentioned in this [article](https://www.smashingmagazine.com/2016/03/pagination-infinite-scrolling-load-more-buttons/), jumping to a specific page is far less used than navigation via {next}/{prev} page links (See [{SHOULD} Use Pagination Links Where Applicable](#161)). This favours cursor-based over offset-based pagination.

**Note:** To provide a consistent look and feel of pagination patterns, you must stick to the common query parameter names defined in [???](#137).

{SHOULD} Prefer Cursor-Based Pagination, Avoid Offset-Based Pagination
======================================================================

Cursor-based pagination is usually better and more efficient when compared to offset-based pagination. Especially when it comes to high-data volumes and/or storage in NoSQL databases.

Before choosing cursor-based pagination, consider the following trade-offs:

-   Usability/framework support:

    -   Offset-based pagination is more widely known than cursor-based pagination, so it has more framework support and is easier to use for API clients

-   Use case - jump to a certain page:

    -   If jumping to a particular page in a range (e.g., 51 of 100) is really a required use case, cursor-based navigation is not feasible.

-   Data changes may lead to anomalies in result pages:

    -   Offset-based pagination may create duplicates or lead to missing entries if rows are inserted or deleted between two subsequent paging requests.

    -   If implemented incorrectly, cursor-based pagination may fail when the cursor entry has been deleted before fetching the pages.

-   Performance considerations - efficient server-side processing using offset-based pagination is hardly feasible for:

    -   Very big data sets, especially if they cannot reside in the main memory of the database.

    -   Sharded or NoSQL databases.

-   Cursor-based navigation may not work if you need the total count of results.

The {cursor} used for pagination is an opaque pointer to a page, that must never be **inspected** or **constructed** by clients. It usually encodes (encrypts) the page position, i.e. the identifier of the first or last page element, the pagination direction, and the applied query filters - or a hash over these - to safely recreate the collection. The {cursor} may be defined as follows:

    Cursor:
      type: object
      properties:
        position:
          description: >
            Object containing the identifier(s) pointing to the entity that is
            defining the collection resource page - normally the position is
            represented by the first or the last page element.
          type: object
          properties: ...

        direction:
          description: >
            The pagination direction that is defining which elements to choose
            from the collection resource starting from the page position.
          type: string
          enum: [ ASC, DESC ]

        query:
          description: >
            Object containing the query filters applied to create the collection
            resource that is represented by this cursor.
          type: object
          properties: ...

        query_hash:
          description: >
            Stable hash calculated over all query filters applied to create the
            collection resource that is represented by this cursor.
          type: string

      required:
        - position
        - direction

The page information for cursor-based pagination should consist of a {cursor} set, that besides {next} may provide support for {prev}, {first}, {last}, and {self} as follows (see also [???](#link-relation-fields)):

    {
      "cursors": {
        "self": "...",
        "first": "...",
        "prev": "...",
        "next": "...",
        "last": "..."
      },
      "items": [... ]
    }

**Note:** The support of the {cursor} set may be dropped in favor of [{SHOULD} Use Pagination Links Where Applicable](#161).

Further reading:

-   [Twitter](https://dev.twitter.com/rest/public/timelines)

-   [Use the Index, Luke](http://use-the-index-luke.com/no-offset)

-   [Paging in PostgreSQL](https://www.citusdata.com/blog/1872-joe-nelson/409-five-ways-paginate-postgres-basic-exotic)

{SHOULD} Use Pagination Links Where Applicable
==============================================

To simplify client design, APIs should support [simplified hypertext controls](#165) for pagination over collections whenever applicable. Beside {next} this may comprise the support for {prev}, {first}, {last}, and {self} as {link-relations}\[link relations\] (see also [???](#link-relation-fields) for details).

The page content is transported via {items}, while the {query} object may contain the query filters applied to the collection resource as follows:

    {
      "self": "http://my-service.zalandoapis.com/resources?cursor=<self-position>",
      "first": "http://my-service.zalandoapis.com/resources?cursor=<first-position>",
      "prev": "http://my-service.zalandoapis.com/resources?cursor=<previous-position>",
      "next": "http://my-service.zalandoapis.com/resources?cursor=<next-position>",
      "last": "http://my-service.zalandoapis.com/resources?cursor=<last-position>",
      "query": {
        "query-param-<1>": ...,
        "query-param-<n>": ...
      },
      "items": [...]
    }

**Note:** In case of complex search requests, e.g. when {GET-with-body} is required, the {cursor} may not be able to encode all query filters. In this case, it is best practice to encode only page position and direction in the {cursor} and transport the query filter in the body - in the request as well as in the response. To protect the pagination sequence, in this case it is recommended, that the {cursor} contains a hash over all applied query filters for pagination request validation.

**Remark:** You should avoid providing a total count unless there is a clear need to do so. Very often, there are significant system and performance implications when supporting full counts. Especially, if the data set grows and requests become complex queries and filters drive full scans. While this is an implementation detail relative to the API, it is important to consider the ability to support serving counts over the life of a service.
