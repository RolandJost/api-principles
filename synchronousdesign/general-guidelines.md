---
layout: default
title: General Guidelines
parent: Synchronous-API Design
nav_order: 1
---

The titles are marked with the corresponding labels: {MUST}, {SHOULD}, {MAY}.

{MUST} Follow API First Principle
=================================

You must follow the [API First Principle](#api-first), more specifically:

-   You must define APIs first, before coding its implementation, [using OpenAPI as specification language](#101)

-   You must design your APIs consistently with this guidelines; use our [API Linter Service \[internal link](https://zally.zalando.net/)\] for automated rule checks.

-   You must call for early review feedback from peers and client developers, and apply [our lightweight API review process \[internal link](https://github.bus.zalan.do/ApiGuild/ApiReviewProcedure)\] for all component external APIs, i.e. all apis with `x-api-audience =/= component-internal` (see [API Audience](#219)).

{MUST} Provide API Specification using OpenAPI
==============================================

We use the [OpenAPI specification](http://swagger.io/specification/) as standard to define API specification files. API designers are required to provide the API specification using a single **self-contained YAML** file to improve readability. We encourage to use **OpenAPI 3.0** version, but still support **OpenAPI 2.0** (a.k.a. Swagger 2).

The API specification files should be subject to version control using a source code management system - best together with the implementing sources.

You [must / should publish](#192) the component [external / internal](#219) API specification with the deployment of the implementing service, and, hence, make it discoverable for the group via our [API Portal \[internal link](https://apis.zalando.net/)\].

**Hint:** A good way to explore **OpenAPI 3.0/2.0** is to navigate through the [OpenAPI specification mind map](https://openapi-map.apihandyman.io/) and use our [Swagger Plugin for IntelliJ IDEA](https://plugins.jetbrains.com/search?search=swagger+Monte) to create your first API. To explore and validate/evaluate existing APIs the [Swagger Editor](https://editor.swagger.io/) or our [API Portal](https://apis.zalando.net) may be a good starting point.

{MUST} only use Durable and Immutable Remote References
=======================================================

Normally, API specification files must be **self-contained**, i.e. files should not contain references to local or remote content, e.g. `../fragment.yaml#/element` or `$ref: 'https://github.com/zalando/zally/blob/master/server/src/main/resources/api/zally-api.yaml#/schemas/LintingRequest'`. The reason is, that the content referred to is *in general* **not durable** and **not immutable**. As a consequence, the semantic of an API may change in unexpected ways.

However, you may use remote references to resources accessible by the following service URLs.

-   `https://infrastructure-api-repository.zalandoapis.com/` (internal repository of APIs)

-   `https://opensource.zalando.com/problem/` (see [???](#176))

-   `https://zalando.github.io/problem/` (deprecated alias for [???](#176))

As we control these URLs, we ensure that their content is **durable** and **immutable**. This allows to define API specifications by using fragments published via this sources, as suggested in [???](#151).

{SHOULD} Provide API User Manual
================================

In addition to the API Specification, it is good practice to provide an API user manual to improve client developer experience, especially of engineers that are less experienced in using this API. A helpful API user manual typically describes the following API aspects:

-   API scope, purpose, and use cases

-   concrete examples of API usage

-   edge cases, error situation details, and repair hints

-   architecture context and major dependencies - including figures and sequence flows

The user manual must be published online, e.g. via our documentation hosting platform service, GHE pages, or specific team web servers. Please do not forget to include a link to the API user manual into the API specification using the `#/externalDocs/url` property.

{MUST} Write APIs in U.S. English
=================================
