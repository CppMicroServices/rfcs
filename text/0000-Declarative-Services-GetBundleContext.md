- Start Date: 2023-09-18
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/20
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/491

# Supporting GetBundleContext inside Declarative Services

## Summary

Currently, when used inside a declarative service (DS), the freestanding `GetBundleContext` function only works under the following specific circumstances:

1. The bundle developer uses the CPPMICROSERVICES_INITIALIZE_BUNDLE macro, even though our documentation does not advise its use
2. The bundle is set to load immediately, which is not the default, or
3. The bundle has a bundle activator, which is not always needed

Under other circumstances, either the supporting content for `GetBundleContext` is not added to the bundle object, or it is never set up at runtime. This RFC is for the resolution of the latter deficiency.

## Motivation

As Declarative Services is a layer on top of the Core Framework (CF), developers most likely would not expect features of the CF like `GetBundleContext` to stop working when using DS. This issue also does not always occur, and is not noted in the documentation. From a bundle developer's perspective, this is a bug in CppMicroServices.

## Detailed design

Declarative Services offers two benefits that are relevant to this issue: it removes the need for common boilerplate code in the bundle, and it allows lazy loading to optimize memory usage. Unfortunately, some things were missed when implementing these features. Specifically:

1. The CF's immediate bundle loader does not set the bundle context pointer unless `bundle.activator` is set to `true` in the bundle manifest. This is because CF bundles without an activator are never loaded. This assumption is not true for DS bundles, which use a separate loader.
2. The DS's delayed bundle loader, which is used for DS bundles without an activator, does not set the bundle context pointer. Bundles that use this loader will have a nonfunctional `GetBundleContext` function.

The necessary `BundeContextPrivate` pointer for initializing the global context is currently only present privately in the CF. As it needs to be accessed both from the CF and from DS, an undocumented getter function will be added to the `BundleContext` class and exported.

Code will be added to `scrimpl::GetComponentCreatorDeletors` (`BundleLoader.cpp`) to ensure the bundle context is initialized for DS bundles without activators. This code will use the new `BundleContextPrivate` getter function.

The core framework's bundle loader (`BundlePrivate::Start0`), which is also responsible for DS bundles with activators, already has the necessary logic for initializing the context pointer, and is known to work. This will be used as a model for the code being added to the DS bundle loader.

If initialization fails, for example because the symbols do not exist in the bundle, an error will be logged and initialization will continue. This matches current behavior. The bundle signature check should prevent corrupted bundles from loading.

Tests for bundle initialization will be added to `usDeclarativeServicesTests`. These tests will ensure that the various ways of accessing the bundle context produce consistent results.

## How we teach this

A brief explanation of this resolution will be included in the changelog.

## Drawbacks

The new `BundleContextPrivate` getter function violates encapsulation.

## Alternatives

1. The global `BundleContextPrivate` pointer and its accessor functions that allow `GetBundleContext` to work are provided when the user adds the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro to their bundle code, but current documentation does not show its use with DS. While it would be possible to provide the contents of this macro for the developer automatically using existing DS features, the removal of one line of code is not worth the backward compatibility issues this would introduce.
2. The introduction of a `BundleContextPrivate` getter function can be avoided, and thus encapsulation preserved, by either altering the bundle context setter to take a `BundleContext` instead, or by introducing an additional setter that accepts a `BundleContext`.
	1. Altering the one existing setter function would break both backward and forward compatibility between bundles and the core framework. This is not acceptable for large-scale uses of CppMicroServices.
	2. Introducing an additional setter raises memory management complications. The `BundleContext` object is currently only used by value, but would need to be passed by reference through a C function interface. This requires an extra constructor that uses `new` to return a reference with unbounded lifetime. This constructor has the risk of creating leaks when used by bundle authors unfamiliar with manual memory management.
3. Catastrophically failing when a bundle without the symbols needed for `GetBundleContext` to function would provide an early sign to bundle developers that their bundle file was corrupted, or that an error occurred in the Declarative Services Code Generation Tool. However, this would cause some existing bundles to stop working, which is not optimal. The bundle signature check should catch corrupted bundles without the need to trigger a failure here.

## Unresolved questions

None.