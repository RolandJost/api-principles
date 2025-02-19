---
layout: default
title: Events
parent: Synchronous-API Design
nav_order: 10
---

Zalando’s architecture centers around decoupled microservices and in that context we favour asynchronous event driven approaches. The guidelines in this section focus on how to design and publish events intended to be shared for others to consume.

**Events, Event Types and Categories.**

Events are defined using an item called an *Event Type*. The Event Type allows events to have their structure declared with a schema by producers and understood by consumers. An Event Type declares standard information, such as a name, an owning application (and by implication, an owning team), a schema defining the event’s custom data, and a compatibility mode declaring how the schema will be evolved. Event Types also allow the declaration of validation and enrichment strategies for events, along with supplemental information such as how events can be partitioned in an event stream.

Event Types belong to a well known *Event Category* (such as a data change category), which provides extra information that is common to that kind of event.

Event Types can be published and made available as API resources for teams to use, typically in an *Event Type Registry*. Each event published can then be validated against the overall structure of its event type and the schema for its custom data.

The basic model described above was originally developed in the [Nakadi project](https://github.com/zalando/nakadi), which acts as a reference implementation of the event type registry, and as a validating publish/subscribe broker for event producers and consumers.

{MUST} Treat Events as part of the service interface
====================================================

Events are part of a service’s interface to the outside world equivalent in standing to a service’s REST API. Services publishing data for integration must treat their events as a first class design concern, just as they would an API. For example this means approaching events with the "API first" principle in mind as described in the [???](#introduction).

{MUST} Make Event schema available for review
=============================================

Services publishing event data for use by others must make the event schema as well as the event type definition available for review.

{MUST} Ensure Event schema conforms to Open API Schema Object
=============================================================

To align the event schema specifications to API specifications, we use the Schema Object as defined by the Open API Specifications to define event schemas. This is particularly useful for events that represent data changes about resources also used in other APIs.

The [Open API Schema Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#schemaObject) is an **extended subset** of [JSON Schema Draft 4](http://json-schema.org/). For convenience, we highlight some important differences below. Please refer to the [Open API Schema Object specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#schemaObject) for details.

As the Open API Schema Object specification *removes* some JSON Schema keywords, the following properties **must not** be used in event schemas:

-   `additionalItems`

-   `contains`

-   `patternProperties`

-   `dependencies`

-   `propertyNames`

-   `const`

-   `not`

-   `oneOf`

On the other side Schema Object *redefines* some JSON Schema keywords:

-   `additionalProperties`: For event types that declare compatibility guarantees, there are recommended constraints around the use of this field. See the guideline [{SHOULD} Avoid in event type definitions](#210) for details.

Finally, the Schema Object *extends* JSON Schema with some keywords:

-   `readOnly`: events are logically immutable, so `readOnly` can be considered redundant, but harmless.

-   `discriminator`: to support polymorphism, as an alternative to `oneOf`.

-   `^x-`: patterned objects in the form of [vendor extensions](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#vendorExtensions) can be used in event type schema, but it might be the case that general purpose validators do not understand them to enforce a validation check, and fall back to must-ignore processing. A future version of the guidelines may define well known vendor extensions for events.

{MUST} Ensure that Events are registered as Event Types
=======================================================

In Zalando’s architecture, events are registered using a structure called an *Event Type*. The Event Type declares standard information as follows:

-   A well known event category, such as a general or data change category.

-   The name of the event type.

-   The definition of the [event target audience](#219).

-   An owning application, and by implication, an owning team.

-   A schema defining the event payload.

-   The compatibility mode for the type.

Event Types allow easier discovery of event information and ensure that information is well-structured, consistent, and can be validated.

Event type owners must pay attention to the choice of compatibility mode. The mode provides a means to evolve the schema. The range of modes are designed to be flexible enough so that producers can evolve schemas while not inadvertently breaking existing consumers:

-   `none`: Any schema modification is accepted, even if it might break existing producers or consumers. When validating events, undefined properties are accepted unless declared in the schema.

-   `forward`: A schema `S1` is forward compatible if the previously registered schema, `S0` can read events defined by `S1` - that is, consumers can read events tagged with the latest schema version using the previous version as long as consumers follow the robustness principle described in the guideline’s [???](#api-design-principles).

-   `compatible`: This means changes are fully compatible. A new schema, `S1`, is fully compatible when every event published since the first schema version will validate against the latest schema. In compatible mode, only the addition of new optional properties and definitions to an existing schema is allowed. Other changes are forbidden.

The compatibility mode interact with revision numbers in the schema `version` field, which follows semantic versioning (MAJOR.MINOR.PATCH):

-   Changing an event type with compatibility mode `compatible` can lead to a PATCH or MINOR version revision. MAJOR breaking changes are not allowed.

-   Changing an event type with compatibility mode `forward` can lead to a PATCH or MINOR version revision. MAJOR breaking changes are not allowed.

-   Changing an event type with compatibility mode `none` can lead to PATCH, MINOR or MAJOR level changes.

The following examples illustrate this relations:

-   Changes to the event type’s `title` or `description` are considered PATCH level.

-   Adding new optional fields to an event type’s schema is considered a MINOR level change.

-   All other changes are considered MAJOR level, such as renaming or removing fields, or adding new required fields.

The core Event Type structure is shown below as an Open API object definition:

    EventType:
      description: |
        An event type defines the schema and its runtime properties. The required
        fields are the minimum set the creator of an event type is expected to
        supply.
      required:
        - name
        - category
        - owning_application
        - schema
      properties:
        name:
          description: |
            Name of this EventType. The name must follow the functional naming
            pattern `<functional-name>.<event-name>` to preserve global
            uniqueness and readability.
          type: string
          pattern: '[a-z][a-z0-9-]*\.[a-z][a-z0-9-]*'
          example: |
            transactions.order.order-cancelled
            customer.personal-data.email-changed
        audience:
          type: string
          x-extensible-enum:
            - component-internal
            - business-unit-internal
            - company-internal
            - external-partner
            - external-public
          description: |
            Intended target audience of the event type, analogue to audience definition for REST APIs
            in rule #219 -- see https://opensource.zalando.com/restful-api-guidelines/#219
        owning_application:
          description: |
            Name of the application (eg, as would be used in infrastructure
            application or service registry) owning this `EventType`.
          type: string
          example: price-service
        category:
          description: Defines the category of this EventType.
          type: string
          x-extensible-enum:
            - data
            - general
        compatibility_mode:
          description: |
            The compatibility mode to evolve the schema.
          type: string
          x-extensible-enum:
            - compatible
            - forward
            - none
          default: forward
        schema:
          description: The most recent payload schema for this EventType.
          type: object
          properties:
            version:
              description: Values are based on semantic versioning (eg "1.2.1").
              type: string
              default: '1.0.0'
            created_at:
              description: Creation timestamp of the schema.
              type: string
              readOnly: true
              format: date-time
              example: '1996-12-19T16:39:57-08:00'
            type:
              description: |
                 The schema language of schema definition. Currently only
                 json_schema (JSON Schema v04) syntax is defined, but in the
                 future there could be others.
              type: string
              x-extensible-enum:
                - json_schema
            schema:
              description: |
                  The schema as string in the syntax defined in the field type.
              type: string
          required:
            - type
            - schema
        ordering_key_fields:
          type: array
          description: |
            Indicates which field is used for application level ordering of events.
            It is typically a single field, but also multiple fields for compound
            ordering key are supported (first item is most significant).

            This is an informational only event type attribute for specification of
            application level ordering. Nakadi transportation layer is not affected,
            where events are delivered to consumers in the order they were published.

            Scope of the ordering is all events (of all partitions), unless it is
            restricted to data instance scope in combination with
            `ordering_instance_ids` attribute below.

            This field can be modified at any moment, but event type owners are
            expected to notify consumer in advance about the change.

            *Background:* Event ordering is often created on application level using
            ascending counters, and data providers/consumers do not need to rely on the
            event publication order. A typical example are data instance change events
            used to keep a slave data store replica in sync. Here you have an order
            defined per instance using data object change counters (aka row update
            version) and the order of event publication is not relevant, because
            consumers for data synchronization skip older instance versions when they
            reconstruct the data object replica state.

          items:
            type: string
            description: |
              Indicates a single ordering field. This is a JsonPointer, which is applied
              onto the whole event object, including the contained metadata and data (in
              case of a data change event) objects. It must point to a field of type
              string or number/integer (as for those the ordering is obvious).

              Indicates a single ordering field. It is a simple path (dot separated) to
              the JSON leaf element of the whole event object, including the contained metadata and data (in
              case of a data change event) objects. It must point to a field of type
              string or number/integer (as for those the ordering is obvious), and must be
              present in the schema.
            example: "data.order_change_counter"
        ordering_instance_ids:
          type: array
          description: |
            Indicates which field represents the data instance identifier and scope in
            which ordering_key_fields provides a strict order. It is typically a single
            field, but multiple fields for compound identifier keys are also supported.

            This is an informational only event type attribute without specific Nakadi
            semantics for specification of application level ordering. It only can be
            used in combination with `ordering_key_fields`.

            This field can be modified at any moment, but event type owners are expected
            to notify consumer in advance about the change.
          items:
            type: string
            description: |
              Indicates a single key field. It is a simple path (dot separated) to the JSON
              leaf element of the whole event object, including the contained metadata and
              data (in case of a data change event) objects, and it must be present in the
              schema.
           example: "data.order_number"
        created_at:
          description: When this event type was created.
          type: string
          pattern: date-time
        updated_at:
          description: When this event type was last updated.
          type: string
          pattern: date-time

APIs such as registries supporting event types, may extend the model, including the set of supported categories and schema formats. For example the Nakadi API’s event category registration also allows the declaration of validation and enrichment strategies for events, along with supplemental information, such as how events are partitioned in the stream (see [{SHOULD} Use the hash partition strategy for Data Change Events](#204)).

{MUST} Ensure Events conform to a well-known Event Category
===========================================================

An *event category* describes a generic class of event types. The guidelines define two such categories:

-   General Event: a general purpose category.

-   Data Change Event: a category used for describing changes to data entities used for data replication based data integration.

The set of categories is expected to evolve in the future.

A category describes a predefined structure that event publishers must conform to along with standard information about that kind of event (such as the operation for a data change event).

**The General Event Category.**

The structure of the *General Event Category* is shown below as an Open API Schema Object definition:

    GeneralEvent:
      description: |
        A general kind of event. Event kinds based on this event define their
        custom schema payload as the top level of the document, with the
        "metadata" field being required and reserved for standard metadata. An
        instance of an event based on the event type thus conforms to both the
        EventMetadata definition and the custom schema definition. Previously
        this category was called the Business Category.
      required:
        - metadata
      properties:
        metadata:
            $ref: '#/definitions/EventMetadata'

Event types based on the General Event Category define their custom schema payload at the top-level of the document, with the [metadata](#event-metadata) field being reserved for standard information (the contents of [metadata](#event-metadata) are described further down in this section).

In the example fragment below, the reserved [metadata](#event-metadata) field is shown with fields "a" and "b" being defined as part of the custom schema:

Note:

-   The General Event in a previous version of the guidelines was called a *Business Event*. Implementation experience has shown that the category’s structure gets used for other kinds of events, hence the name has been generalized to reflect how teams are using it.

-   The General Event is still useful and recommended for the purpose of defining events that drive a business process.

-   The Nakadi broker still refers to the General Category as the Business Category and uses the keyword "business" for event type registration. Other than that, the JSON structures are identical.

See [{MUST} Use the General Event Category to signal steps and arrival points in business processes](#201) for more guidance on how to use the category.

**The Data Change Event Category.**

The *Data Change Event Category* structure is shown below as an Open API Schema Object:

    DataChangeEvent:
      description: |
        Represents a change to an entity. The required fields are those
        expected to be sent by the producer, other fields may be added
        by intermediaries such as a publish/subscribe broker. An instance
        of an event based on the event type conforms to both the
        DataChangeEvent's definition and the custom schema definition.
      required:
        - metadata
        - data_op
        - data_type
        - data
      properties:
        metadata:
          description: The metadata for this event.
          $ref: '#/definitions/EventMetadata'
        data:
          description: |
            Contains custom payload for the event type. The payload must conform
            to a schema associated with the event type declared in the metadata
            object's `event_type` field.
          type: object
        data_type:
          description: name of the (business) data entity that has been mutated
          type: string
          example: 'sales_order.order'
        data_op:
          type: string
          enum: ['C', 'U', 'D', 'S']
          description: |
            The type of operation executed on the entity:

            - C: Creation of an entity
            - U: An update to an entity.
            - D: Deletion of an entity.
            - S: A snapshot of an entity at a point in time.

The Data Change Event Category is structurally different to the General Event Category. It defines a field called `data` for placing the custom payload information, as well as specific information related to data changes in the `data_type`. In the example fragment below, the fields `a` and `b` are part of the custom payload housed inside the `data` field:

See the following guidelines for more guidance on how to use the Data Change Event Category:

-   [{SHOULD} Ensure that Data Change Events match API representations](#205)

-   [{MUST} Use Data Change Events to signal mutations](#202)

-   [{SHOULD} Use the hash partition strategy for Data Change Events](#204)

**Event Metadata.**

The General and Data Change event categories share a common structure for *metadata*. The metadata structure is shown below as an Open API Schema Object:

    EventMetadata:
      type: object
      description: |
        Carries metadata for an Event along with common fields. The required
        fields are those expected to be sent by the producer, other fields may be
        added by intermediaries such as publish/subscribe broker.
      required:
        - eid
        - occurred_at
      properties:
        eid:
          description: Identifier of this event.
          type: string
          format: uuid
          example: '105a76d8-db49-4144-ace7-e683e8f4ba46'
        event_type:
          description: The name of the EventType of this Event.
          type: string
          example: 'example.important-business-event'
        occurred_at:
          description: When the event was created according to the producer.
          type: string
          format: date-time
          example: '1996-12-19T16:39:57-08:00'
        received_at:
          description: |
            When the event was seen by an intermediary such as a broker.
          type: string
          readOnly: true
          format: date-time
          example: '1996-12-19T16:39:57-08:00'
        version:
          description: |
            Version of the schema used for validating this event. This may be
            enriched upon reception by intermediaries. This string uses semantic
            versioning.
          type: string
          readOnly: true
        parent_eids:
          description: |
            Event identifiers of the Event that caused the generation of
            this Event. Set by the producer.
          type: array
          items:
            type: string
            format: uuid
          example: '105a76d8-db49-4144-ace7-e683e8f4ba46'
        flow_id:
          description: |
            A flow-id for this event (corresponds to the X-Flow-Id HTTP header).
          type: string
          example: 'JAh6xH4OQhCJ9PutIV_RYw'
        partition:
          description: |
            Indicates the partition assigned to this Event. Used for systems
            where an event type's events can be sub-divided into partitions.
          type: string
          example: '0'

Please note than intermediaries acting between the producer of an event and its ultimate consumers, may perform operations like validation of events and enrichment of an event’s [metadata](#event-metadata). For example brokers such as Nakadi, can validate and enrich events with arbitrary additional fields that are not specified here and may set default or other values, if some of the specified fields are not supplied. How such systems work is outside the scope of these guidelines but producers and consumers working with such systems should look into their documentation for additional information.

{MUST} Ensure that Events define useful business resources
==========================================================

Events are intended to be used by other services including business process/data analytics and monitoring. They should be based around the resources and business processes you have defined for your service domain and adhere to its natural lifecycle (see also [???](#139) and [???](#140)).

As there is a cost in creating an explosion of event types and topics, prefer to define event types that are abstract/generic enough to be valuable for multiple use cases, and avoid publishing event types without a clear need.

{MUST} Events must not provide sensitive customer personal data
===============================================================

Similar to API permission scopes, there will be Event Type permissions passed via an OAuth token supported in near future. In the meantime, teams are asked to note the following:

-   Sensitive data, such as (e-mail addresses, phone numbers, etc) are subject to strict access and data protection controls.

-   Event type owners **must not** publish sensitive information unless it’s mandatory or necessary to do so. For example, events sometimes need to provide personal data, such as delivery addresses in shipment orders (as do other APIs), and this is fine.

{MUST} Use the General Event Category to signal steps and arrival points in business processes
==============================================================================================

When publishing events that represent steps in a business process, event types must be based on the General Event category.

All your events of a single business process will conform to the following rules:

-   Business events must contain a specific identifier field (a business process id or "bp-id") similar to flow-id to allow for efficient aggregation of all events in a business process execution.

-   Business events must contain a means to correctly order events in a business process execution. In distributed settings where monotonically increasing values (such as a high precision timestamp that is assured to move forwards) cannot be obtained, the `parent_eids` data structure allows causal relationships to be declared between events.

-   Business events should only contain information that is new to the business process execution at the specific step/arrival point.

-   Each business process sequence should be started by a business event containing all relevant context information.

-   Business events must be published reliably by the service.

At the moment we cannot state whether it’s best practice to publish all the events for a business process using a single event type and represent the specific steps with a state field, or whether to use multiple event types to represent each step. For now we suggest assessing each option and sticking to one for a given business process.

{MUST} Use Data Change Events to signal mutations
=================================================

When publishing events that represents created, updated, or deleted data, change event types must be based on the Data Change Event category.

-   Change events must identify the changed entity to allow aggregation of all related events for the entity.

-   Change events [{SHOULD} Provide a means for explicit event ordering](#203).

-   Change events must be published reliably by the service.

{SHOULD} Provide a means for explicit event ordering
====================================================

Some common error cases may require event consumers to reconstruct event streams or replay events from a position within the stream. Events *should* therefore contain a way to restore their partial order of occurrence.

This can be done – among other ways – by adding

-   a strictly monotonically increasing entity version (e.g. as created by a database) to allow for partial ordering of all events for an entity, or

-   a strictly monotonically increasing message counter.

In the event type definition, the `ordering_key_fields` property should be used to indicate which field(s) contains the ordering key, if any.

System timestamps are not necessarily a good choice, since exact synchronization of clocks in distributed systems is difficult, two events may occur in the same microsecond and system clocks may jump backward or forward to compensate drifts or leap-seconds. If you use system timestamps to indicate event ordering, you must carefully ensure that your designated event order is not messed up by these effects.

Also, if using timestamps, the producer **must** make sure that they are formatted for all events in the UTC time zone, to allow for a simple string-based comparison.

**Note** that basing events on data structures that can be converged upon in a distributed setting (such as [CRDTs](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type), [logical clocks](https://en.wikipedia.org/wiki/Logical_clock) and [vector clocks](https://en.wikipedia.org/wiki/Vector_clock)) is outside the scope of this guidance.

{SHOULD} Use the hash partition strategy for Data Change Events
===============================================================

The `hash` partition strategy allows a producer to define which fields in an event are used as input to compute a logical partition the event should be added to. Partitions are useful as they allow supporting systems to scale their throughput while provide local ordering for event entities.

The `hash` option is particulary useful for data changes as it allows all related events for an entity to be consistently assigned to a partition, providing a relative ordered stream of events for that entity. This is because while each partition has a total ordering, ordering across partitions is not assured by a supporting system, thus it is possible for events sent across partitions to appear in a different order to consumers that the order they arrived at the server.

When using the `hash` strategy the partition key in almost all cases should represent the entity being changed and not a per event or change identifier such as the `eid` field or a timestamp. This ensures data changes arrive at the same partition for a given entity and can be consumed effectively by clients.

There may be exceptional cases where data change events could have their partition strategy set to be the producer defined or random options, but generally `hash` is the right option - that is while the guidelines here are a "should", they can be read as "must, unless you have a very good reason".

{SHOULD} Ensure that Data Change Events match API representations
=================================================================

A data change event’s representation of an entity should correspond to the REST API representation.

There’s value in having the fewest number of published structures for a service. Consumers of the service will be working with fewer representations, and the service owners will have less API surface to maintain. In particular, you should only publish events that are interesting in the domain and abstract away from implementation or local details - there’s no need to reflect every change that happens within your system.

There are cases where it could make sense to define data change events that don’t directly correspond to your API resource representations. Some examples are -

-   Where the API resource representations are very different from the datastore representation, but the physical data are easier to reliably process for data integration.

-   Publishing aggregated data. For example a data change to an individual entity might cause an event to be published that contains a coarser representation than that defined for an API

-   Events that are the result of a computation, such as a matching algorithm, or the generation of enriched data, and which might not be stored as entity by the service.

{MUST} Permissions on events must correspond to API permissions
===============================================================

If a resource can be read synchronously via a REST API and read asynchronously via an event, the same read-permission must apply: We want to protect access to data, not the way data is accessed.

{MUST} Indicate ownership of Event Types
========================================

Event definitions must have clear ownership - this can be indicated via the `owning_application` field of the EventType.

Typically there is one producer application, which owns the EventType and is responsible for its definition, akin to how RESTful API definitions are managed. However, the owner may also be a particular service from a set of multiple services that are producing the same kind of event.

{MUST} Define Event Payloads in accordance with the overall Guidelines
======================================================================

Events must be consistent with other API data and the API Guidelines in general.

Everything expressed in the [???](#introduction) to these Guidelines is applicable to event data interchange between services. This is because our events, just like our APIs, represent a commitment to express what our systems do and designing high-quality, useful events allows us to develop new and interesting products and services.

What distinguishes events from other kinds of data is the delivery style used, asynchronous publish-subscribe messaging. But there is no reason why they could not be made available using a REST API, for example via a search request or as a paginated feed, and it will be common to base events on the models created for the service’s REST API.

The following existing guideline sections are applicable to events:

-   [???](#general-guidelines)

-   [???](#api-naming)

-   [???](#data-formats)

-   [???](#common-data-types)

-   [???](#hypermedia)

{MUST} Maintain backwards compatibility for Events
==================================================

Changes to events must be based around making additive and backward compatible changes. This follows the guideline, "Must: Don’t Break Backward Compatibility" from the [???](#compatibility) guidelines.

In the context of events, compatibility issues are complicated by the fact that producers and consumers of events are highly asynchronous and can’t use content-negotiation techniques that are available to REST style clients and servers. This places a higher bar on producers to maintain compatibility as they will not be in a position to serve versioned media types on demand.

For event schema, these are considered backward compatible changes, as seen by consumers -

-   Adding new optional fields to JSON objects.

-   Changing the order of fields (field order in objects is arbitrary).

-   Changing the order of values with same type in an array.

-   Removing optional fields.

-   Removing an individual value from an enumeration.

These are considered backwards-incompatible changes, as seen by consumers -

-   Removing required fields from JSON objects.

-   Changing the default value of a field.

-   Changing the type of a field, object, enum or array.

-   Changing the order of values with different type in an array (also known as a tuple).

-   Adding a new optional field to redefine the meaning of an existing field (also known as a co-occurrence constraint).

-   Adding a value to an enumeration (note that [`x-extensible-enum`](#112) is not available in JSON Schema)

{SHOULD} Avoid `additionalProperties` in event type definitions
===============================================================

Event type schema should avoid using `additionalProperties` declarations, in order to support schema evolution.

Events are often intermediated by publish/subscribe systems and are commonly captured in logs or long term storage to be read later. In particular, the schemas used by publishers and consumers can  
drift over time. As a result, compatibility and extensibility issues that happen less frequently with client-server style APIs become important and regular considerations for event design. The guidelines recommend the following to enable event schema evolution:

-   Publishers who intend to provide compatibility and allow their schemas to evolve safely over time **must not** declare an `additionalProperties` field with a value of `true` (i.e., a wildcard extension point). Instead they must define new optional fields and update their schemas in advance of publishing those fields.

-   Consumers **must** ignore fields they cannot process and not raise errors. This can happen if they are processing events with an older copy of the event schema than the one containing the new definitions specified by the publishers.

The above constraint does not mean fields can never be added in future revisions of an event type schema - additive compatible changes are allowed, only that the new schema for an event type must define the field first before it is published within an event. By the same turn the consumer must ignore fields it does not know about from its copy of the schema, just as they would as an API client - that is, they cannot treat the absence of an `additionalProperties` field as though the event type schema was closed for extension.

Requiring event publishers to define their fields ahead of publishing avoids the problem of *field redefinition*. This is when a publisher defines a field to be of a different type that was already being emitted, or, is changing the type of an undefined field. Both of these are prevented by not using `additionalProperties`.

See also rule [???](#111) in the [???](#compatibility) section for further guidelines on the use of `additionalProperties`.

{MUST} Use unique Event identifiers
===================================

The `eid` (event identifier) value of an event must be unique.

The `eid` property is part of the standard [metadata](#event-metadata) for an event and gives the event an identifier. Producing clients must generate this value when sending an event and it must be guaranteed to be unique from the perspective of the owning application. In particular events within a given event type’s stream must have unique identifiers. This allows consumers to process the `eid` to assert the event is unique and use it as an idempotency check.

Note that uniqueness checking of the `eid` might be not enforced by systems consuming events and it is the responsibility of the producer to ensure event identifiers do in fact distinctly identify events. A straightforward way to create a unique identifier for an event is to generate a UUID value.

{SHOULD} Design for idempotent out-of-order processing
======================================================

Events that are designed for [idempotent](#149) out-of-order processing allow for extremely resilient systems: If processing an event fails, consumers and producers can skip/delay/retry it without stopping the world or corrupting the processing result.

To enable this freedom of processing, you must explicitly design for idempotent out-of-order processing: Either your events must contain enough information to infer their original order during consumption or your domain must be designed in a way that order becomes irrelevant.

As common example similar to data change events, idempotent out-of-order processing can be supported by sending the following information:

-   the process/resource/entity identifier,

-   a [monotonically increasing ordering key](#203) and

-   the process/resource state after the change.

A receiver that is interested in the current state can then ignore events that are older than the last processed event of each resource. A receiver interested in the history of a resource can use the ordering key to recreate a (partially) ordered sequence of events.

{MUST} Follow Naming Convention for Event Type Names
====================================================

Event type names must (or should, see [???](#223) for details and definition) conform to the functional naming depending on the [audience](#219) as follows:

    <event-type-name>       ::= <functional-event-name> | <application-event-name>

    <functional-event-name> ::= <functional-name>.<event-name>

    <event-name>            ::= [a-z][a-z0-9-]* -- free event name (functional name)

The following application specific legacy convention is **only** allowed for [internal](#223) event type names:

    <application-event-name> ::= [<organization-id>.]<application-id>.<event-name>
    <organization-id>  ::= [a-z][a-z0-9-]* -- organization identifier, e.g. team identifier
    <application-id>   ::= [a-z][a-z0-9-]* -- application identifier

**Note:** consistent naming should be used whenever the same entity is exposed by a data change event and a RESTful API.

{MUST} Prepare for duplicate Events
===================================

Event consumers must be able to process duplicate events.

Most message brokers and data streaming systems offer "at-least-once" delivery. That is, one particular event is delivered to the consumers one or more times. Other circumstances can also cause duplicate events.

For example, these situations occur if the publisher sends an event and doesn’t receive the acknowledgment (e.g. due to a network issue). In this case, the publisher will try to send the same event again. This leads to two identical events in the event bus which have to be processed by the consumers. Similar conditions can appear on consumer side: an event has been processed successfully, but the consumer fails to confirm the processing.
