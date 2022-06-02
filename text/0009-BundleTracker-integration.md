- Start Date: 2022-05-25
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: (fill me with the URL of the CppMicroServices issue that necessitated this RFC, otherwise leave blank)

# Integration of BundleTracker into Core Framework

## Summary

This document describes the design and integration of the `BundleTracker` into the CppMicroServices core framework. The `BundleTracker` functionality is described in the [OSGI BundleTracker Specification](http://docs.osgi.org/specification/osgi.core/7.0.0/util.tracker.html#d0e52020), which is mapped to an API definition in C++ described in this document. This document discusses the need for adding `BundleTracker` to the core framework. Namely, there is existing compendium code that uses workarounds for lack of a BundleTracker in the core framework (e.g. bundle extension creation/destruction in DeclarativeServices), that would be made simpler and more robust with a proper `BundleTracker` implementation. Additionally, a chosen design and alternative implementations for `BundleTracker` are discussed.

## Motivation

Generally, it is useful to be able to react to state changes in bundles. The core framework offers a `BundleListener` to do this, but it is limited in its functionality on its own.

The main issue with the `BundleListener` is that it only gives the user information on the bundle state changes _after_ the listener is initialized, not the full history of bundle states. Therefore, the user will not find out about any bundles that have been in a certain state since before the `BundleListener` was started. The BundleTracker, on the other hand, provides the full history of states for bundles (initial, transitory, and current).

To discover this prior information without a `BundleTracker`, a user might loop through the bundles either before or after starting the `BundleListener`. Both methods have unexpected results in a multithreaded context, and solving the race conditions while avoiding deadlock is nontrivial.

Currently, Declarative Services gets around this issue by starting a `BundleListener` and then calling tracker methods on each bundle in a controlled way (in the `SCRActivator`). However, this is a fragile workaround that is easy to implement incorrectly, leading to race conditions or deadlock. As such, it seems appropriate to implement a `BundleTracker` based on the OSGi specification to handle this problem inside of the core framework.

__Note__: The issues with the `BundleListener` necessitating a `BundleTracker` are mirrored by the `ServiceListener` and the `ServiceTracker`, which has been implemented in the CppMicroServices core framework.

## Requirements Analysis

### User Roles and Goals

<table>
<thead>
<tr class="header">
<th><p>ID.</p></th>
<th><p>Priority</p></th>
<th><p>User Roles</p></th>
<th><p>User Goals (What and Why)</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p>1</p></td>
<td><p>High</p></td>
<td><p>Casual User of Declarative Services</p></td>
<td><p><em>What:</em> Trust that Declarative Services will always detect bundles started before starting DS.<br />
<em>Why:</em></p>
<ol>
<li>Preserve the abstraction that Declarative Services provides to the casual Microservices user</li>
<li>Improve software maintainability and simplify product deployment</li>
</ol></td>
</tr>
<tr class="even">
<td><p>2</p></td>
<td><p>High</p></td>
<td><p>Advanced Microservices Developer</p></td>
<td><p><em>What:</em> Create services with that leverage resources stored in bundles.<br />
<em>Why:</em></p>
<ol>
<li>Separate resources from resource-users</li>
<li>Handle resource absence gracefully</li>
<li>Create scalable, extensible and reactive systems</li>
</ol></td>
</tr>
</tbody>
</table>

### Use Cases

#### UC1: Casual User of Declarative Services

Lucas is a casual user who uses DS to handle the loading and unloading of components for his bundle. Lucas trusts that DS will always perform this operation correctly. However, since there is no `BundleTracker` implemented in the core framework, Declarative Services could sometimes mishandle other bundles that change state before or during the startup of DS. This can create issues that break the abstraction that DS provides for users of Declarative Services like Lucas (PP1).

#### UC2: Advanced Microservices Developer

Diana is an advanced microservices developer who wants to develop a service that serves resources to a user, where these resources are stored in bundles. In this application, it is not guaranteed that the provider service will be started before the resource bundles. If a `BundleListener` is used to track the addition and removal of resources, Diana's application will not track bundles started before the service (PP2). Additionally, attempted resolutions to this issue in user code, like looping through the already started bundles to add them, create various possible race conditions (PP3). As an alternative, Diana would like to be able to add her own tracked objects and callbacks through a core framework construct (PP4).

### Pain Point Summary

PP1: Declarative Services currently has a workaround for the lack of a `BundleTracker` in its service component activation code, which can fail if bundles are changing state while DS is starting.

PP2: Listeners on their own are insufficient in creating an up-to-date record of bundle states for user code, since bundle states that have not changed since the addition of the listener will not be tracked.

PP3: Leaving the proper tracker implementation to the user creates many opportunities for race conditions or deadlock.

PP4: For a tracker to be useful, it is helpful to include methods of customization and extension, so that custom wrapper objects and callbacks can be included.

### Requirements

<table>
<thead>
<tr class="header">
<th><p>ID</p></th>
<th><p>Statement</p></th>
<th><p>Pain Point ID / Rationale</p></th>
<th><p>Priority</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>R1_Previous_States</td>
<td>The solution must be able to track bundles started before initialization.</td>
<td>PP1, PP2</td>
<td>Must have</td>
</tr>
<tr class="even">
<td>R2_Concurrency</td>
<td>The solution must handle concurrency when grabbing states of bundles existing during tracker initialization.</td>
<td>PP3</td>
<td>Must have</td>
</tr>
<tr class="odd">
<td>R3_Customization</td>
<td>The solution should allow for the user to customize callbacks through a customizer object or subclassing the BundleTracker.</td>
<td>PP4</td>
<td>Must have</td>
</tr>
<tr class="even">
<td>R3_Extension</td>
<td>The BundleTracker should be generic, such that a wrapper class can be defined to pass additional information through the tracking system.</td>
<td>PP4</td>
<td>Must have</td>
</tr>
<tr class="odd">
<td>R5_Standardization</td>
<td>The solution should map to C++ from the OSGi specification for BundleTracker.</td>
<td><a href="http://docs.osgi.org/specification/osgi.core/7.0.0/util.tracker.html#d0e52020">OSGi Spec</a></td>
<td>Must have</td>
</tr>
<tr class="even">
<td>R6_Quality</td>
<td>The solution should be robust and bug-free. This is doubly crucial as Declarative Services would then depend on the BundleTracker.</td>
<td>PP1</td>
<td>Must have</td>
</tr>
<tr class="odd">
<td>R7_Efficiency</td>
<td>The solution should be able to scale to many bundles without slowing the core framework.</td>
<td><a href="http://docs.osgi.org/specification/osgi.core/7.0.0/util.tracker.html#d0e52020"></a></td>
<td>Must have</td>
</tr>
</tbody>
</table>

## Functional design

### Functional design summary

The API for the `BundleTracker` closely mirrors that of the OSGi specification, mapped to an implementation in C++. The `ServiceTracker`, which has been implemented in the core framework, is used as a guide to ensure uniformity. However, certain enhancements are provided (e.g. smart pointers) to the `BundleTracker` API to simplify lifetime and ownership of certain objects.

The `BundleTracker` is templated to allow for custom object types to be tracked, containing extra data or functionality.

### Creating a BundleTracker

In this implementation, the `BundleTracker` is templated, where `T` corresponds with the wrapper object type. This defaults to `Bundle`, and is not necessary if a wrapper object is not being used. The `BundleTracker` offers one constructor:

```cpp

/**
 * Create a BundleTracker that tracks bundles through states covered by the state mask.
 * 
 * @param context The BundleContext from which tracking occurs.
 * @param stateMask The bit mask which defines the bundle states to be tracked.
 * @param customizer The customizer to call when bundles are added, modified, or removed.
 *                   If the customizer is nullptr, then the callbacks in this BundleTracker will 
 *                   be used instead (default or can be overridden).
 */
BundleTracker(const BundleContext& context,
              uint32_t stateMask,
              std::shared_ptr<BundleTrackerCustomizer> customizer = nullptr)
```

### Using a BundleTracker

Once a `BundleTracker` is opened, there are API methods to gain information and monitor bundles:

```cpp

/**
 * Return an array of all the tracked bundles.
 *
 * @return A vector of Bundles (could be empty).
 */ 
std::vector<Bundle> GetBundles()

/**
 * Returns the custom object for the given Bundle if the given Bundle is tracked. Otherwise null.
 *
 * @param bundle The Bundle paired with the object
 * @return The custom object paired with the given Bundle or null if the Bundle is not being tracked.
 */
std::shared_ptr<T> GetObject(const Bundle& bundle)

/**
 * Remove a bundle from this BundleTracker.
 *
 * @param bundle the Bundle to be removed
 */
void Remove(const Bundle&)

/**
 * Return the number of bundles being tracked by this BundleTracker.
 *
 * @return The number of tracked bundles.
 */
int Size()

/**
 * Returns the tracking count for this BundleTracker.
 *
 * The tracking count is set to 0 when the BundleTracker is opened.
 * The tracking count increases by 1 anytime a Bundle is added, modified, or removed from the BundleTracker.
 * Tracking counts from different times can be compared to determine whether any bundles have changed.
 *
 * @return The current tracking count.
 */
unsigned int GetTrackingCount()

/**
 * Return true if and only if this BundleTracker is tracking no bundles.
 *
 * @return true if and only if this BundleTracker is empty.
 */
bool IsEmpty()

/**
 * Return an unordered map from all of the currently tracked Bundles to their custom objects.
 *
 * @return An unordered map from all of the Bundles currently tracked by this BundleTracker to their custom objects.
 */
std::unordered_map<Bundle, std::shared_ptr<T>> GetTracked()
```

### Customizing a BundleTracker

The behavior of the `BundleTracker` can be customized in one of two ways:

- Subclassing the `BundleTrackerCustomizer` class, implementing its pure virtual functions, and passing a pointer into the constructor for `BundleTracker`
- Subclassing `BundleTracker` and overriding its methods instead of supplying a `BundleTrackerCustomizer`

The `BundleTrackerCustomizer` defines three callback methods, which also exist as defaults in `BundleTracker`:

```cpp

/**
 * Called when a Bundle is being added to the BundleTracker.
 * When the BundleTracker detects a Bundle that should be added to the tracker based on the search parameters (state mask, context, etc.),
 * this method is called. This method should return the object to be tracked for the specified Bundle if the BundleTracker is being extended.
 * Otherwise, return the Bundle itself. If the return is nullptr, the Bundle is not tracked.
 *
 * @param bundle The Bundle being added to the BundleTracker.
 * @param event the BundleEvent which was caught by the BundleTracker.
 *
 * @return The object to be tracked for the specified Bundle object or nullptr to avoid tracking the Bundle.
 */
std::shared_ptr<T> AddingBundle(const Bundle& bundle, const BundleEvent& event)

/**
 * Called when a Bundle is modified that is being tracked by the BundleTracker.
 * When a tracked bundle changes states, this method is called.
 *
 * @param bundle The tracked Bundle whose state has changed.
 * @param event The BundleEvent which was caught by the BundleTracker.
 *
 * @tparam object The tracked object corresponding to the tracked Bundle (returned from AddingBundle).
 */
void ModifiedBundle(const Bundle& bundle, const BundleEvent& event, std::shared_ptr<T> object)

/**
 * Called when a Bundle is being removed from the BundleTracker
 *
 * @param bundle The tracked Bundle that is being removed.
 * @param event The BundleEvent which was caught by the BundleTracker.
 *
 * @tparam object The tracked object corresponding to the tracked Bundle (returned from AddingBundle).
 */
void RemovedBundle(const Bundle& bundle, const BundleEvent& event, std::shared_ptr<T> object)

```

### Extending a BundleTracker

The [OSGi specification](http://docs.osgi.org/specification/osgi.core/7.0.0/util.tracker.html#d0e52020) permits extension of the `BundleTracker` by using wrapper objects. This behavior is mapped to C++ using a template parameter for the `BundleTracker`. By specifying a wrapper object of type `T` to the `BundleTracker`, the user can make the tracker hold on to custom objects through the tracking callbacks that contain extra data or functionality.

__TODO: Add example of the Extender pattern__

## Detailed design

__TODO: Add architectural design and implementation details__

## How we teach this

The `BundleTracker` concept is best presented as a parallel to the `ServiceTracker`, which already exists and has the same customizer and extender patterns. A relevant difference between the constructs besides what is being tracked, is the method of filtering. Services offer class names and filters to target certain services. On the contrary, the `BundleTracker` uses a bit mask for `Bundle` states. This difference should be explained in comparing the `BundleTracker` to the `ServiceTracker`.

Given the acceptance of this RFC, it would be worthwhile to include an example of the `BundleTracker` being used in a CppMicroServices example. It could also be included in a "Best Practices" document to urge existing and new users to use the `BundleTracker` implementation instead of creating their own solutions to the tracking problem.

Since the `BundleTracker` is not a feature exposed in a beginner's workflow, it is not urgent to include it in a "Getting Started" guide.

## Drawbacks

__TODO: Add implementation drawbacks__

## Alternatives

__TODO: Add alternative paths (not implementing this, other possible implementations, etc.)__

## Unresolved questions

- What type of pointer should be used for the `BundleTrackerCustomizer` in the `BundleTracker` constructor, `std::unique_ptr` or `std::shared_ptr`?
