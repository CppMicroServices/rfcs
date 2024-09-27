- Start Date: 2019-10-02
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/196

# Complete implementation of HTTPService

## Summary

> The current implementation of HTTPService is not complete and prevent CppMicroServices to be used as base for a fully-fledged REST backend.

## Motivation

> We are using CppMicroServices as a backend for a desktop/network application which use a Javascript frontend. Hence we need support for all HTTP verbs and also support for file upload.

## Detailed design

> Complete POST, GET and PUT HTTP methods
> Implement management of file upload using POST/multipart, following the Servlet(c) API
> Allow to provide options to CivetWeb when starting a ServletContainer
> Implement unit/functional tests for HttpService
> Allow to run GTest in a HttpService-enabled environment

## How we teach this

> This enhancement must not break existing usage of HttpService. Say, we add features and no not break existing one.

## Drawbacks

> We expect changes in core code to allow fine-grained testing of HTTP layer without launching a full container. This implies to implement new layer on top of some components to allow them to be mocked.

## Alternatives

> Alternative would be to drop current implementation, which rely on CivetWeb server[1] and use more REST friendly library. But this would break the general web purpose of HTTPService.

## Unresolved questions

> Some core code has been modified in the current WIP branch. This could be an issue. We did that to allow fine-grained testing of our implementation.

[1]: https://civetweb.github.io/civetweb/
