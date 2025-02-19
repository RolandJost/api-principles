---
layout: default
title: JSON Guidelines
parent: Synchronous-API Design
nav_order: 14
---

These guidelines provides recommendations for defining JSON data at Zalando. JSON here refers to {RFC-7159}\[RFC 7159\] (which updates {RFC-4627}\[RFC 4627\]), the "application/json" media type and custom JSON media types defined for APIs. The guidelines clarifies some specific cases to allow Zalando JSON data to have an idiomatic form across teams and services.

The first some of the following guidelines are about property names, the later ones about values.

{MUST} Property names must be ASCII snake\_case (and never camelCase): `^[a-z_][a-z_0-9]*$`
===========================================================================================

Property names are restricted to ASCII strings. The first character must be a letter, or an underscore, and subsequent characters can be a letter, an underscore, or a number.

(It is recommended to use `_` at the start of property names only for keywords like `_links`.)

Rationale: No established industry standard exists, but many popular Internet companies prefer snake\_case: e.g. GitHub, Stack Exchange, Twitter. Others, like Google and Amazon, use both - but not only camelCase. It’s essential to establish a consistent look and feel such that JSON looks as if it came from the same hand.

{SHOULD} Define Maps Using `additionalProperties`
=================================================

A "map" here is a mapping from string keys to some other type. In JSON this is represented as an object, the key-value pairs being represented by property names and property values. In OpenAPI schema (as well as in JSON schema) they should be represented using additionalProperties with a schema defining the value type. Such an object should normally have no other defined properties.

The map keys don’t count as property names in the sense of [rule 118](#118), and can follow whatever format is natural for their domain. Please document this in the description of the map object’s schema.

Here is an example for such a map definition (the `translations` property):

    components:
      schemas:
        Message:
          description:
            A message together with translations in several languages.
          type: object
          properties:
            message_key:
              type: string
              description: The message key.
            translations:
              description:
                The translations of this message into several languages.
                The keys are [IETF BCP-47 language tags](https://tools.ietf.org/html/bcp47).
              type: object
              additionalProperties:
                type: string
                description:
                  the translation of this message into the language identified by the key.

An actual JSON object described by this might then look like this:

    { "message_key": "color",
      "translations": {
        "de": "Farbe",
        "en-US": "color",
        "en-GB": "colour",
        "eo": "koloro",
        "nl": "kleur"
      }
    }

{SHOULD} Array names should be pluralized
=========================================

To indicate they contain multiple values prefer to pluralize array names. This implies that object names should in turn be singular.

{MUST} Boolean property values must not be null
===============================================

Schema based JSON properties that are by design booleans must not be presented as nulls. A boolean is essentially a closed enumeration of two values, true and false. If the content has a meaningful null value, strongly prefer to replace the boolean with enumeration of named values or statuses - for example accepted\_terms\_and\_conditions with true or false can be replaced with terms\_and\_conditions with values yes, no and unknown.

{MUST} Use same semantics for `null` and absent properties
==========================================================

Open API 3.x allows to mark properties as `required` and as `nullable` to specify whether properties may be absent (`{}`) or `null` (`{"example":null}`). If a property is defined to be not `required` and `nullable` (see [2nd row in Table below](#required-nullable-row-2)), this rule demands that both cases must be handled in the exact same manner by specification.

The following table shows all combinations and whether the examples are valid:

<table><colgroup><col style="width: 25%" /><col style="width: 25%" /><col style="width: 25%" /><col style="width: 25%" /></colgroup><thead><tr class="header"><th>{CODE-START}required{CODE-END}</th><th>{CODE-START}nullable{CODE-END}</th><th>{CODE-START}{}{CODE-END}</th><th>{CODE-START}{"example":null}{CODE-END}</th></tr></thead><tbody><tr class="odd"><td><p><code>true</code></p></td><td><p><code>true</code></p></td><td><p>{NO}</p></td><td><p>{YES}</p></td></tr><tr class="even"><td><p><code>false</code></p></td><td><p><code>true</code></p></td><td><p>{YES}</p></td><td><p>{YES}</p></td></tr><tr class="odd"><td><p><code>true</code></p></td><td><p><code>false</code></p></td><td><p>{NO}</p></td><td><p>{NO}</p></td></tr><tr class="even"><td><p><code>false</code></p></td><td><p><code>false</code></p></td><td><p>{YES}</p></td><td><p>{NO}</p></td></tr></tbody></table>

While API designers and implementers may be tempted to assign different semantics to both cases, we explicitly decide **against** that option, because we think that any gain in expressiveness is far outweighed by the risk of clients not understanding and implementing the subtle differences incorrectly.

As an example, an API that provides the ability for different users to coordinate on a time schedule, e.g. a meeting, may have a resource for options in which every user has to make a `choice`. The difference between *undecided* and *decided against any of the options* could be modeled as *absent* and `null` respectively. It would be safer to express the `null` case with a dedicated [Null object](https://en.wikipedia.org/wiki/Null_object_pattern), e.g. `{}` compared to `{"id":"42"}`.

Moreover, many major libraries have somewhere between little to no support for a `null`/absent pattern (see [Gson](https://stackoverflow.com/questions/48465005/gson-distinguish-null-value-field-and-missing-field), [Moshi](https://github.com/square/moshi#borrows-from-gson), [Jackson](https://github.com/FasterXML/jackson-databind/issues/578), [JSON-B](https://developer.ibm.com/articles/j-javaee8-json-binding-3/)). Especially strongly-typed languages suffer from this since a new composite type is required to express the third state. Nullable `Option`/`Optional`/`Maybe` types could be used but having nullable references of these types completely contradicts their purpose.

The only exception to this rule is JSON Merge Patch {RFC-7396}\[RFC 7396\]) which uses `null` to explicitly indicate property deletion while absent properties are ignored, i.e. not modified.

{SHOULD} Empty array values should not be null
==============================================

Empty array values can unambiguously be represented as the empty list, `[]`.

{SHOULD} Enumerations should be represented as Strings
======================================================

Strings are a reasonable target for values that are by design enumerations.

{SHOULD} Name date/time properties using the `_at` suffix
=========================================================

Dates and date-time properties should end with `_at` to distinguish them from boolean properties which otherwise would have very similar or even identical names:

-   {created\_at} rather than {created},

-   {modified\_at} rather than {modified},

-   `occurred_at` rather than `occurred`, and

-   `returned_at` rather than `returned`.

**Note:** {created} and {modified} were mentioned in an earlier version of the guideline and are therefore still accepted for APIs that predate this rule.

{SHOULD} Date property values should conform to RFC 3339
========================================================

Use the date and time formats defined by {RFC-3339}\#section-5.6\[RFC 3339\]:

-   for "date" use strings matching `date-fullyear "-" date-month "-" date-mday`, for example: `2015-05-28`

-   for "date-time" use strings matching `full-date "T" full-time`, for example `2015-05-28T14:07:17Z`

Note that the [OpenAPI format](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#data-types) "date-time" corresponds to "date-time" in the RFC) and `2015-05-28` for a date (note that the OpenAPI format "date" corresponds to "full-date" in the RFC). Both are specific profiles, a subset of the international standard {ISO-8601}\[ISO 8601\].

A zone offset may be used (both, in request and responses) — this is simply defined by the standards. However, we encourage restricting dates to UTC and without offsets. For example `2015-05-28T14:07:17Z` rather than `2015-05-28T14:07:17+00:00`. From experience we have learned that zone offsets are not easy to understand and often not correctly handled. Note also that zone offsets are different from local times that might be including daylight saving time. Localization of dates should be done by the services that provide user interfaces, if required.

When it comes to storage, all dates should be consistently stored in UTC without a zone offset. Localization should be done locally by the services that provide user interfaces, if required.

Sometimes it can seem data is naturally represented using numerical timestamps, but this can introduce interpretation issues with precision - for example whether to represent a timestamp as 1460062925, 1460062925000 or 1460062925.000. Date strings, though more verbose and requiring more effort to parse, avoid this ambiguity.

{MAY} Time durations and intervals could conform to ISO 8601
============================================================

Schema based JSON properties that are by design durations and intervals could be strings formatted as recommended by {ISO-8601}\[ISO 8601\] ({RFC-3339}\#appendix-A\[Appendix A of RFC 3339 contains a grammar\] for durations).

{MAY} Standards could be used for Language, Country and Currency
================================================================

-   {ISO-3166-1-a2}\[ISO 3166-1-alpha2 country\]

-   (It’s "GB", not "UK", even though "UK" has seen some use at Zalando)

-   {ISO-639-1}\[ISO 639-1 language code\]

-   {BCP47}\[BCP 47\] (based on {ISO-639-1}\[ISO 639-1\]) for language variants

-   {ISO-4217}\[ISO 4217 currency codes\]
