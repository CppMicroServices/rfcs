- Start Date: 2023-09-18
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/20
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/491

# Supporting GetBundleContext inside Declarative Services

## Summary

Currently, when used inside a declarative service (DS), the freestanding `GetBundleContext` function only works under the following specific circumstances:

1. The bundle developer uses the CPPMICROSERVICES_INITIALIZE_BUNDLE macro, even though our documentation does not advise its use
2. The bundle is set to load immediately, which is not the default, or
3. The bundle has a bundle activator, which is not always needed

Under other circumstances, either the supporting content for `GetBundleContext` is not added to the bundle object, or it is never set up at runtime. This RFC is for the resolution of these two deficiencies, so that the function can always be used.

## Motivation

As Declarative Services is a layer on top of the Core Framework (CF), developers most likely would not expect features of the CF like `GetBundleContext` to stop working when using DS. This issue also does not always occur, and is not noted in the documentation. From a bundle developer's perspective, this is a bug in CppMicroServices.

## Detailed design

Declarative Services offers two benefits that are relevant to this issue: it removes the need for common boilerplate code in the bundle, and it allows lazy loading to optimize memory usage. Unfortunately, some things were missed when implementing these features. Specifically:

1. Use of the macro that provides the pointer `GetBundleContext` retrieves is discouraged when using Declarative Services, but its functionality has not been replaced.
2. The CF's immediate bundle loader does not set the bundle context pointer unless `bundle.activator` is set to `true` in the bundle manifest. This is because CF bundles without an activator are never loaded. This assumption is not true for DS bundles, which use a separate loader.
3. The DS's delayed bundle loader, which is used for DS bundles without an activator, does not set the bundle context pointer. Bundles that use this loader will have a nonfunctional `GetBundleContext` function.

The global `BundleContextPrivate` pointer and its accessor functions that allow `GetBundleContext` to work are provided when the user adds the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro to their bundle code (`BundleInitialization.h`). While this is required without DS, our current documentation shows it not being used with DS. Because the DS Code Generation Tool already provides a facility for inserting code into the bundle, the tool will be modified to insert the necessary code, providing the contents of the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro for the developer automatically.

The necessary infrastructure for initializing the global context pointer is currently only present privately in the CF. As it needs to be invoked both from the CF and from DS, it will be moved to a shared location, and any dependencies on the CF will be severed.

This code will then be used in the following locations to initialize the bundle context pointer:

1. Replacing the original code in `BundlePrivate::Start0`, but not changing existing functionality. `GetBundleContext` will continue working for non-DS bundles, as well as DS bundles with activators.
2. Adding to `scrimpl::GetComponentCreatorDeletors`, ensuring initialization for DS bundles without activators.

The core framework's bundle loader (`BundlePrivate::Start0`), which is also responsible for DS bundles with activators, already has the necessary logic for initializing the context pointer. The `BundleUtils` functions it relies on are private to the CF, so they will be moved to the `util` folder to make them accessible to DS.

This logic also requires the `BundleContextPrivate` class, which cannot be exposed, to be passed to the bundle to set the context pointer. To allow both the CF and DS to  set the context pointer, the bundle will be modified to offer two setter functions: one that takes a `BundleContextPrivate`, and one that takes a `BundleContext`. Having two setters simplifies calling code and allows for old versions of the CF to be forward compatible with new bundles.

If initialization fails, for example because the symbols do not exist in the bundle, an error will be logged and initialization will continue. This matches current behavior. It also allows for an updated CF to be backward compatible with old bundles that lack the additional setter. The bundle signature check should prevent corrupted bundles from loading.

Tests for the DS Code Generation changes will be implemented by adding to or modifying `SCRCodegenTests`, and tests for bundle initialization will be added to `usDeclarativeServicesTests`. Code generation testing will ensure that the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro and its requisite `#include` are emitted. Bundle initialization testing will ensure that the various ways of accessing the bundle context produce consistent results. The changes to `BundlePrivate::Start0` will not be tested as they are trivial.

## How we teach this

1. A brief explanation of this resolution will be included in the changelog.
2. Documentation will be updated to note that the use of `CPPMICROSERVICES_INITIALIZE_BUNDLE` with DS is not permitted, and will result in multiple definition linker errors for "_us_bundle_context_instance" symbols.

## Drawbacks

1. Because the contents of the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro are always inserted into the bundle, if the bundle developer has already included this macro, they will encounter a multiple definition error during linking.
2. The linker error messages are described in terms of implementation details, making the cause unclear to the bundle developer.

## Alternatives

1. Not adding the contents of the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro to the bundle would avoid introducing a multiple definition error for developers that already include the macro. However, this would mean including the macro is required for the function of `GetBundleContext`. This is contrary to existing documentation and the purpose of Declarative Services.
2. Catastrophically failing when a bundle without the symbols needed for `GetBundleContext` to function would provide an early sign to bundle developers that their bundle file was corrupted, or that an error occurred in the Declarative Services Code Generation Tool. However, this would cause some existing bundles to stop working, which is not optimal. The bundle signature check should catch corrupted bundles without the need to trigger a failure here.
3. Using a single setter function for the bundle's context would simplify the interface, but complicate its use. It would need to be converted from taking a `BundleContextPrivate` to taking a `BundleContext`, which would break CF/bundle version compatibility bidirectionally. It would also complicate the calling of the function from within the CF, which works with `BundleContextPrivate` objects.

## Unresolved questions

None.