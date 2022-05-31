- Start Date: 2022-05-25
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: (fill me with the URL of the CppMicroServices issue that necessisated this RFC, otherwise leave blank)

# Integration of BundleTracker into Core Framework

## Summary

This document describes the design and integration of the BundleTracker into the CppMicroServices core framework. The BundleTracker functionality is described in the [OSGI BundleTracker Specification](http://docs.osgi.org/specification/osgi.core/7.0.0/util.tracker.html#d0e52020), which is mapped to an API definition in C++ described in this document. This document discusses the need for adding BundleTracker to the core framework. Namely, there is existing compendium code that uses workarounds for lack of a BundleTracker in the core framework (e.g. bundle extension creation/destruction in DeclarativeServices), that would be made simpler and more robust with a proper BundleTracker implementation. Additionally, a chosen design and alternative implementations for BundleTracker are discussed.

## Motivation

Generally, it is useful to be able to react to state changes in bundles. The core framework offers a BundleListener to do this, but it is limited in its functionality on its own.

The main issue with the BundleListener is that it only gives the user information on the bundle state changes _after_ the listener is initialized, not the full history of bundle states. Therefore, the user will not find out about any bundles that have been in a certain state since before the BundleListener was started. The BundleTracker, on the other hand, provides the full history of states for bundles (initial, transitory, and current).

To discover this prior information without a BundleTracker, a user might loop through the bundles either before or after starting the BundleListener. Both methods have unexpected results in a multithreaded context, and solving the race conditions while avoiding deadlock is nontrivial.

Currently, DeclarativeServices gets around this issue by starting a BundleListener and then calling tracker methods on each bundle in a controlled way (in the SCRActivator). However, this is a fragile workaround that is easy to misimplement, leading to race conditions or deadlock. As such, it seems appropriate to implement a BundleTracker based on the OSGi specification to handle this problem inside of the core framework.

__Note__: The issues with the BundleListener necessitating a BundleTracker are mirrored by the ServiceListener and the ServiceTracker, which has been implemented in the CppMicroServices core framework.

## Requirements

> Shane added this section to talk about requirements

## Functional design

> Shane added this section to talk about the API

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing CppMicroServices patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the CppMicroServices guides must be
re-organized or altered? Does it change how CppMicroServices is taught to new users
at any level?

> How should this feature be introduced and taught to existing CppMicroServices
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching CppMicroServices,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
