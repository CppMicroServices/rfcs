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

### Functional design summary

The API for the BundleTracker closely mirrors that of the OSGi specification, mapped to an implementation in C++. The ServiceTracker, which has been implemented in the core framework, is used as a guide to ensure uniformity. However, certain enhancements are provided (e.g. smart pointers) to the BundleTracker API to simplify lifetime and ownership of certain objects.

The BundleTracker is templated to allow for object Types

### Creating a BundleTracker

In this implementation, the BundleTracker is templated, where T corresponds with the wrapper object type. This defaults to Bundle, and is not necessary if a wrapper object is not being used. The BundleTracker offers one construtor:

```cpp

// Construct a BundleTracker
// int corresponds with a bit mask over bundle states, such that
// a state is tracked iff it is in the mask
// Use nullptr for the customizer if using default implementation
// or subclassing BundleTracker to override methods
BundleTracker(const BundleContext&, int, std::shared_ptr<BundleTrackerCustomizer>)
```

### Using a BundleTracker

Once a BundleTracker is opened, there are API methods to gain information and monitor bundles:

```cpp

// Return an array of all the tracked bundles
// TODO: determine the proper return type (pointers to Bundles?)
std::vector<std::shared_ptr<Bundle>> GetBundles()

// Return the tracked object associated with a provided Bundle
// This object is defined from the `addingBundle` method
std::shared_ptr<T> GetObject(Bundle&)

// Remove a bundle from the tracked bundles
// When the Bundle is not in the tracked map, 
// calls `removedBundle' method
// TODO: what if the bundle already isn't tracked?
void Remove(Bundle&)

// Returns the number of bundles being tracked
int Size()

// Returns the tracking count
// Tracking count is a positive integer that ticks up
// every time the BundleTracker is changed
int GetTrackingCount()

// Detects whether the BundleTracker has no tracked bundles
bool IsEmpty()

// Return the tracked objects
std::vector<std::shared_ptr<T>> GetTracked()
```

### Customizing a BundleTracker

The behavior of the BundleTracker can be customized in one of two ways:

- Subclassing the BundleTrackerCustomizer class, implementing its pure virtual funtions, and passing a pointer (TODO: what kind of p?) into the constructor for BundleTracker
- Subclassing BundleTracker and overriding its methods instead of supplying a BundleTrackerCustomizer

The BundleTrackerCustomizer defines three callback methods, which also exist as defaults in BundleTracker:

```cpp

// Called when bundle is added to tracker. This method returns a pointer to a wrapper, which is a Bundle by default. If null is returned, the Bundle is no longer tracked
std::shared_ptr<T> AddingBundle(const Bundle&, const BundleEvent&)

// Called when a tracked bundle is modified
// (T is the wrapper object)
void ModifiedBundle(const Bundle&, const BundleEvent&, const T&)

// Called when a tracked bundle is removed
// (T is the wrapper object)
void RemovedBundle(const Bundle&, const BundleEvent&, const T&)

```

### Extending a BundleTracker



## Detailed design

> This is the bulk of the RFC.

The BundleTracker design posed in this document follows the Pointer to Implementation (PImpl) pattern in order to hide class members and method implementations from the user. This echoes the behavior 

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
