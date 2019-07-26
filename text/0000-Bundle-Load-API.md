- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/376

# Add Load API in Bundle class 

## Summary

Currently there is no API function in Bundle class to load bundles/dynamic libraries at runtime.
This document discusses the need for such API , its possible design followed by solution.
It should be analogues to Bundle::loadClass in the OSGi spec
https://osgi.org/specification/osgi.core/7.0.0/framework.api.html#org.osgi.framework.Bundle.loadClass-String-

## Motivation

Clients of CppMicroServices such as declarative services rely on calling OS specific dlopen/LoadLibraryW calls for loading bundles/shared objects.
This increases the overhead in the declarative services while doing the symbol resoution and ref count increment done in the system.
Moreover, framework APIs such as BundleActivator also call these library loading APIs while invoking BundleActivator.
Having this functionality to load bundles/shared objects at two places namely

1) CppMicroServices clients such as DS
2) framework 

seems like repetition and can be consolidated at one place, i.e in Bundle class.
Given this it seems appropriate to have a Bundle::Load API for CppMicroServices library.

## Detailed design

Bundle::Load() API will be declared in the file CppMicroServices/framework/include/cppmicroservices/Bundle.h as below 

```
/**
* Loads this bundle using platform APIs 
* On POSIX   : dlopen
* On Windows : LoadLibraryW
* 
* @return Handle to the bundle in the form of void *
*
* @throws std::runtime_error if this bundle is not initialized correctly, i.e if dlopen or LoadLibraryW fails
*
*/
void* Load() const;

```
As mentioned above, the API will call the platform APIs to open the shared objects.It will suplly the name of the bundle using Bundle class's GetLocation() function to these APIs.
There will be a static cache which will hold the map of Bundle Location as key and handle returned (from above platform APIs) as value.
So the Load will first check the cache to know if the handle is already loaded or not, if yes, it will return the handle, if not, it will open the fresh one
and add it to the cache and then return that handle.
If these platform APIs fails, it will thrown an exception with relevant error information.

A typical usage workflow for ex. will be as below
```
Void SomeDeclarativeServiceUser(const cppmicroservices::Bundle& bd)
{
    try
    {
        void * handle = nullptr;
        handle = bd.Load();

        // Do stuff such as symbol resolution here and what not
    }
    catch(...)
    {
        some::Log << "Error loading bundle\n" ;
    }    
}
```

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing CppMicroServices patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the CppMicroServices guides must be
re-organized or altered? Does it change how CppMicroServices is taught to new users
at any level?

> How should this feature be introduced and taught to existing CppMicroServices
users?

If the proposal is accepted, the CppMicroServices guides will have to modified to reflect the new functionality.
Existing users such as declarative services, would have to modify their code to use new API function.
New users will be able to use this API function to load their bundle as and when required.

## Drawbacks

Currently none.


## Unresolved questions

1) If a service consumer doesnt use BundleActivator, can she load the bundle(s) using this new API function whenever required ?