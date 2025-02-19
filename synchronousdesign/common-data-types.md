---
layout: default
title: Common Data Types
parent: Synchronous-API Design
nav_order: 4
---

Definitions of data objects that are good candidates for wider usage:

{MUST} Use a Common Money Object
================================

Use the following common money structure:

    Unresolved directive in common-data-types.adoc - include::../money.yaml[]

APIs are encouraged to include a reference to the global schema for Money.

    SalesOrder:
      properties:
        grand_total:
          $ref: 'https://opensource.zalando.com/restful-api-guidelines/money.yaml#/Money'

Please note that APIs have to treat Money as a closed data type, i.e. it’s not meant to be used in an inheritance hierarchy. That means the following usage is not allowed:

    {
      "amount": 19.99,
      "currency": "EUR",
      "discounted_amount": 9.99
    }

Cons
----

-   Violates the [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)

-   Breaks existing library support, e.g. [Jackson Datatype Money](https://github.com/zalando/jackson-datatype-money)

-   Less flexible since both amounts are coupled together, e.g. mixed currencies are impossible

A better approach is to favor [composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance):

    {
        "price": {
          "amount": 19.99,
          "currency": "EUR"
        },
        "discounted_price": {
          "amount": 9.99,
          "currency": "EUR"
        }
    }

Pros
----

-   No inheritance, hence no issue with the substitution principle

-   Makes use of existing library support

-   No coupling, i.e. mixed currencies is an option

-   Prices are now self-describing, atomic values

Notes
-----

Please be aware that some business cases (e.g. transactions in Bitcoin) call for a higher precision, so applications must be prepared to accept values with unlimited precision, unless explicitly stated otherwise in the API specification.

Examples for correct representations (in EUR):

-   `42.20` or `42.2` = 42 Euros, 20 Cent

-   `0.23` = 23 Cent

-   `42.0` or `42` = 42 Euros

-   `1024.42` = 1024 Euros, 42 Cent

-   `1024.4225` = 1024 Euros, 42.25 Cent

Make sure that you don’t convert the "amount" field to `float` / `double` types when implementing this interface in a specific language or when doing calculations. Otherwise, you might lose precision. Instead, use exact formats like Java’s [`BigDecimal`](https://docs.oracle.com/javase/8/docs/api/java/math/BigDecimal.html). See [Stack Overflow](http://stackoverflow.com/a/3730040/342852) for more info.

Some JSON parsers (NodeJS’s, for example) convert numbers to floats by default. After discussing the pros and cons we’ve decided on "decimal" as our amount format. It is not a standard OpenAPI format, but should help us to avoid parsing numbers as float / doubles.

{MUST} Use common field names and semantics
===========================================

There exist a variety of field types that are required in multiple places. To achieve consistency across all API implementations, you must use common field names and semantics whenever applicable.

Generic Fields
--------------

There are some data fields that come up again and again in API data:

-   {id}: the identity of the object. If used, IDs must be opaque strings and not numbers. IDs are unique within some documented context, are stable and don’t change for a given object once assigned, and are never recycled cross entities.

-   {xyz\_id}: an attribute within one object holding the identifier of another object must use a name that corresponds to the type of the referenced object or the relationship to the referenced object followed by `_id` (e.g. `customer_id` not `customer_number`; `parent_node_id` for the reference to a parent node from a child node, even if both have the type `Node`)

-   {created\_at}: when the object was created. If used, this must be a `date-time` construct. Originally named {created} before adding the [naming conventions for date/time properties](#235).

-   {modified\_at}: when the object was updated. If used, this must be a `date-time` construct. Originally named {modified} before adding the [naming conventions for date/time properties](#235).

-   {type}: the kind of thing this object is. If used, the type of this field should be a string. Types allow runtime information on the entity provided that otherwise requires examining the Open API file.

-   {etag}: the [ETag](#182) of an [embedded sub-resource](#158). It may be used to carry the {ETag} for subsequent {PUT}/{PATCH} calls (see [???](#etag-in-result-entities)).

Example JSON schema:

    tree_node:
      type: object
      properties:
        id:
          description: the identifier of this node
          type: string
        created_at:
          description: when got this node created
          type: string
          format: 'date-time'
        modified_at:
          description: when got this node last updated
          type: string
          format: 'date-time'
        type:
          type: string
          enum: [ 'LEAF', 'NODE' ]
        parent_node_id:
          description: the identifier of the parent node of this node
          type: string
      example:
        id: '123435'
        created: '2017-04-12T23:20:50.52Z'
        modified: '2017-04-12T23:20:50.52Z'
        type: 'LEAF'
        parent_node_id: '534321'

These properties are not always strictly necessary, but making them idiomatic allows API client developers to build up a common understanding of Zalando’s resources. There is very little utility for API consumers in having different names or value types for these fields across APIs.

Link Relation Fields
--------------------

To foster a consistent look and feel using simple hypertext controls for paginating and iterating over collection values the response objects should follow a common pattern using the below field semantics:

-   {self}:the link or cursor in a pagination response or object pointing to the same collection object or page.

-   {first}: the link or cursor in a pagination response or object pointing to the first collection object or page.

-   {prev}: the link or cursor in a pagination response or object pointing to the previous collection object or page.

-   {next}: the link or cursor in a pagination response or object pointing to the next collection object or page.

-   {last}: the link or cursor in a pagination response or object pointing to the last collection object or page.

Pagination responses should contain the following additional array field to transport the page content:

-   {items}: array of resources, holding all the items of the current page ({items} may be replaced by a resource name).

To simplify user experience, the applied query filters may be returned using the following field (see also {GET-with-body}):

-   {query}: object containing the query filters applied in the search request to filter the collection resource.

As Result, the standard response page using [pagination links](#161) is defined as follows:

    ResponsePage:
      type: object
      properties:
        self:
          description: Pagination link pointing to the current page.
          type: string
          format: uri
        first:
          description: Pagination link pointing to the first page.
          type: string
          format: uri
        prev:
          description: Pagination link pointing to the previous page.
          type: string
          format: uri
        next:
          description: Pagination link pointing to the next page.
          type: string
          format: uri
        last:
          description: Pagination link pointing to the last page.
          type: string
          format: uri

         query:
           description: >
             Object containing the query filters applied to the collection resource.
           type: object
           properties: ...

         items:
           description: Array of collection items.
           type: array
           required: false
           items:
             type: ...

The response page may contain additional metadata about the collection or the current page.

Address Fields
--------------

Address structures play a role in different functional and use-case contexts, including country variances. All attributes that relate to address information should follow the naming and semantics defined below.

    addressee:
      description: a (natural or legal) person that gets addressed
      type: object
      properties:
        salutation:
          description: |
            a salutation and/or title used for personal contacts to some
            addressee; not to be confused with the gender information!
          type: string
          example: Mr
        first_name:
          description: |
            given name(s) or first name(s) of a person; may also include the
            middle names.
          type: string
          example: Hans Dieter
        last_name:
          description: |
            family name(s) or surname(s) of a person
          type: string
          example: Mustermann
        business_name:
          description: |
            company name of the business organization. Used when a business is
            the actual addressee; for personal shipments to office addresses, use
            `care_of` instead.
          type: string
          example: Consulting Services GmbH
      required:
        - first_name
        - last_name

    address:
      description:
        an address of a location/destination
      type: object
      properties:
        care_of:
          description: |
            (aka c/o) the person that resides at the address, if different from
            addressee. E.g. used when sending a personal parcel to the
            office /someone else's home where the addressee resides temporarily
          type: string
          example: Consulting Services GmbH
        street:
          description: |
            the full street address including house number and street name
          type: string
          example: Schönhauser Allee 103
        additional:
          description: |
            further details like building name, suite, apartment number, etc.
          type: string
          example: 2. Hinterhof rechts
        city:
          description: |
            name of the city / locality
          type: string
          example: Berlin
        zip:
          description: |
            zip code or postal code
          type: string
          example: 14265
        country_code:
          description: |
            the country code according to
            [iso-3166-1-alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2)
          type: string
          example: DE
      required:
        - street
        - city
        - zip
        - country_code

Grouping and cardinality of fields in specific data types may vary based on the specific use case (e.g. combining addressee and address fields into a single type when modeling an address label vs distinct addressee and address types when modeling users and their addresses).
