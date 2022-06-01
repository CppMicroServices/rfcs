- Start Date: 2022-05-25
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: (fill me with the URL of the CppMicroServices issue that necessitated this RFC, otherwise leave blank)

# Integration of BundleTracker into Core Framework

## Summary

This document describes the design and integration of the BundleTracker into the CppMicroServices core framework. The BundleTracker functionality is described in the [OSGI BundleTracker Specification](http://docs.osgi.org/specification/osgi.core/7.0.0/util.tracker.html#d0e52020), which is mapped to an API definition in C++ described in this document. This document discusses the need for adding BundleTracker to the core framework. Namely, there is existing compendium code that uses workarounds for lack of a BundleTracker in the core framework (e.g. bundle extension creation/destruction in DeclarativeServices), that would be made simpler and more robust with a proper BundleTracker implementation. Additionally, a chosen design and alternative implementations for BundleTracker are discussed.

## Motivation

Generally, it is useful to be able to react to state changes in bundles. The core framework offers a BundleListener to do this, but it is limited in its functionality on its own.

The main issue with the BundleListener is that it only gives the user information on the bundle state changes _after_ the listener is initialized, not the full history of bundle states. Therefore, the user will not find out about any bundles that have been in a certain state since before the BundleListener was started. The BundleTracker, on the other hand, provides the full history of states for bundles (initial, transitory, and current).

To discover this prior information without a BundleTracker, a user might loop through the bundles either before or after starting the BundleListener. Both methods have unexpected results in a multithreaded context, and solving the race conditions while avoiding deadlock is nontrivial.

Currently, Declarative Services gets around this issue by starting a BundleListener and then calling tracker methods on each bundle in a controlled way (in the SCRActivator). However, this is a fragile workaround that is easy to implement incorrectly, leading to race conditions or deadlock. As such, it seems appropriate to implement a BundleTracker based on the OSGi specification to handle this problem inside of the core framework.

__Note__: The issues with the BundleListener necessitating a BundleTracker are mirrored by the ServiceListener and the ServiceTracker, which has been implemented in the CppMicroServices core framework.

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

Lucas is a casual user who uses DS to handle the loading and unloading of components for his bundle. Lucas trusts that DS will always perform this operation correctly. However, since there is no BundleTracker implemented in the core framework, Declarative Services could sometimes mishandle other bundles that change state before or during the startup of DS. This can create issues that break the abstraction that DS provides for users of Declarative Services like Lucas (PP1).

#### UC2: Advanced Microservices Developer

Diana is an advanced microservices developer who wants to develop a service that serves resources to a user, where these resources are stored in bundles. In this application, it is not guaranteed that the provider service will be started before the resource bundles. If a BundleListener is used to track the addition and removal of resources, Diana's application will not track bundles started before the service (PP2). Additionally, attempted resolutions to this issue in user code, like looping through the already started bundles to add them, create various possible race conditions (PP3). As an alternative, Diana would like to be able to add her own tracked objects and callbacks through a core framework construct (PP4).

### Pain Point Summary

PP1: Declarative Services currently has a workaround for the lack of a BundleTracker in its service component activation code, which can fail if bundles are changing state while DS is starting.

PP2: BundleListeners on their own are insufficient in creating an up-to-date record of bundle states for user code, since bundle states that have not changed since the addition of the listener will not be tracked.

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
