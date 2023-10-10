- Start Date: 2023-09-18
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/20
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/491

# Supporting GetBundleContext inside Declarative Services

## Summary

Currently, the freestanding `GetBundleContext` function generally does not work when used inside a declarative service. This is for two reasons: the supporting content is usually not added to the bundle object, and even when present, it is often never set up at runtime. This RFC is for the resolution of these two deficiencies.

## Motivation

As Declarative Services is a layer on top of the Core Framework, developers most likely would not expect features of the CF like `GetBundleContext` to stop working when using DS. This issue also does not always occur, and is not noted in the documentation. From a bundle developer's perspective, this is a bug in CppMicroServices.

## Detailed design

Declarative Services offers two benefits that are relevant to this issue: it generates boilerplate code for the user, and it allows lazy loading to optimize memory usage. Unfortunately, some things were missed when implementing these features. Specifically:

1. Use of the macro that provides the pointer `GetBundleContext` retrieves is discouraged when using Declarative Services, but its functionality has not been replaced.
2. A logic error in the immediate bundle loader means that this pointer is never assigned to unless `"bundle.activator"` is set to `true` in the bundle manifest.
3. The lazy bundle loader, which is the default loader, does not set the pointer under any circumstances.

The global `BundleContextPrivate` pointer and its accessor functions that allow `GetBundleContext` to work are provided when the user adds the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro to their bundle code (`BundleInitialization.h`). While this is required without DS, our current documentation shows it not being used with DS. Because the DS Code Generation Tool already provides a facility for inserting code into the bundle, the tool will be modified to insert the necessary code, providing the contents of the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro for the developer automatically.

The immediate bundle loader already has the necessary logic for initializing the context pointer, but it is guarded by a conditional that it should not be. The resolution for this component is to simply pull the initialization outside of that conditional. (`BundlePrivate::Start0`)

As the lazy bundle loader completely lacks initialization code, it will need to be copied from the immediate bundle loader and be placed such that it runs once, after the bundle is loaded into memory but before the bundle activator is run. (`scrimpl::GetComponentCreatorDeletors`)

If initialization fails, for example because the symbols do not exist in the bundle, an error will be logged and initialization will continue. The bundle signature check should prevent corrupted bundles from loading. Not failing will allow existing DS bundles that were built before this fix to still be loaded and used.

Tests for the DS Code Generation changes will be added to `SCRCodegenTests`, and tests for the bundle loaders will be added to `usDeclarativeServicesTests`.

## How we teach this

1. A brief explanation of this resolution will be included in the changelog.
2. Documentation will be updated to note that the use of `CPPMICROSERVICES_INITIALIZE_BUNDLE` with DS is not permitted, and will result in multiple definition linker errors for "_us_bundle_context_instance" symbols.

## Drawbacks

1. Because the contents of the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro are always inserted into the bundle, if the bundle developer has already included this macro, they will encounter a multiple definition error during linking.
2. The linker error messages are described in terms of implementation details, making the cause unclear to the bundle developer.

## Alternatives

1. Not adding the contents of the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro to the bundle would avoid introducing a multiple definition error for developers that already include the macro. However, this would mean including the macro is required for the function of `GetBundleContext`. This is contrary to existing documentation and the purpose of Declarative Services.
2. Catastrophically failing when a bundle without the symbols needed for `GetBundleContext` to function would provide an early sign to bundle developers that their bundle file was corrupted, or that an error occurred in the Declarative Services Code Generation Tool. However, this would cause some existing bundles to stop working, which is not optimal. The bundle signature check should catch corrupted bundles without the need to trigger a failure here.

## Unresolved questions
- With the `CPPMICROSERVICES_INITIALIZE_BUNDLE` macro no longer used in DS, should `cppmicroservices_init.cpp` be omitted entirely, or is there a remaining use of the file?
