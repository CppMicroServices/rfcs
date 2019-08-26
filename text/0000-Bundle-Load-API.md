- Start Date: 2019-07-26
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/7
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/376

# Add Load API in Bundle class 

## Summary

Currently there is no API function in Bundle class to load bundles/dynamic libraries at runtime.
This document discusses need for such API in the particular use case of declarative services (referred as DS henceforth in this document) and its possible design/solution.

## Motivation

Developers using DS rely on DS's service component runtime to delay/lazily load the bundles.
Since DS hides all the boilerplate code of bundle lifetime (install => start => stop => uninstall), service consumers can not directly use framework APIs (e.g. Start) to load bundles

Moverover, one could argue that framework's Start() function could be modified to accomodate this functionality instead of having a new member function in the Bundle class.
However, this not possible as DS runtime is designed such that it only loads the service class constructors/destructors by calling extern "C" functions instead of framework's Start function.
As such, framework's Start API can not solve this specific use case for DS.

Because of these reasons, DS uses OS specific dlopen/LoadLibraryW calls for loading bundles/shared objects.

This loading of libraries/bundles increases the overhead in the declarative services while doing the symbol resoution and ref count increment done in the system.

Adding to that, this functionality to load bundles/shared objects is currently implemented at two places

1) Declarative services runtime
2) framework

During the application run, these different callsites may become out of sync.

Given this it seems appropriate to have a Bundle::BundleLoad API for CppMicroServices library.

## Requirements

For the specific use case of DS (and similar extender clients) ,

1) implement Bundle::LoadBundle such that it encapsulates shared library loading and symbol resolution, not exposing OS specific data types
2) become a single function in the CppMicroServices eco system that loads the bundles/shared libraries
3) reduce the amount of OS specific code a client needs to write

## Detailed design

Bundle::LoadBundle() API will be declared in the file CppMicroServices/framework/include/cppmicroservices/Bundle.h as below
The output of above API will be in the wrapper class which will be as belows.

```
/**
* @brief  This class holds the output of BundleLoad API.
*
* It acts as a wrapper over the handles returned by the OS APIs that loads the library.
* Clients have to supply the function type as template argument to correctly get the desired symbol resolved using
* GetSymbol function. See the usage workflow for more details.
*
*/

class BundleHandle
{
    public:
        BundleHandle():handle(nullptr) {}
        ~BundleHandle() { handle = nullptr; }
        BundleHandle(void *handle) : handle(symHandle) {}

        template<typename T>
        T Getsymbol(const std::string& symname);

    private:
        void * symHandle;
}

/**
* Loads this bundle using platform APIs and return a tuple of BundleSymbol class object
* It uses following OS APIs to achieve the same
*
* On POSIX   : dlopen/dlsym
* On Windows : LoadLibraryW/GetProcAddress
* 
* A framework event of type FRAMEWORK_ERROR will fired with following information
* 1) error type
* 2) bundle object
* 3) message
* 4) exception object
* 
* @return A function pointer symbols of type T supplied by the user
*
* @throws std::runtime_error if this bundle is not initialized correctly, i.e if dlopen or LoadLibraryW fails
*
*/

BundleHandle LoadBundle() const;

```
As mentioned above, the API will call the platform APIs to open the shared objects.It will supply the name of the bundle using Bundle class's GetLocation() function to these APIs.

There will be a static cache which will hold the map of Bundle Location as key and handle returned (from above platform APIs) as value.
API will first check the cache to know if the handle is already loaded or not, if yes, it will return the handle, if not, it will open the fresh one
and add it to the cache and then return that handle. 

Once the bundle is opened, it will fetch the required symbols e.g. in case of DS, it will be the symbols of extern "C" functions containing constructor/destructors calls to services.
It will then populate the BundleHandle class object with the library handle. 
BundleHandle object in turn can be queried for a particular symbol by passing in a symbol name.

If the platform APIs fails, it will thrown an exception with relevant error information for ex. platform specific error code and if possible, error string.

A typical usage workflow will be as below
```
Void SomeDeclarativeServiceUser(const cppmicroservices::Bundle& bd)
{
    try
    {
        BundleHandle bundle_handle = bd.LoadBundle();

        auto handleCon = bundle_handle.GetSymbol<ComponentInstance*(*)(void)>(constructor_name);
        auto handleDes = bundle_handle.GetSymbol<void(*)(ComponentInstance*)>(destructor_name);

        // Do stuff with the handleCon and handleDes
    }
    catch(...)
    {
        some::Log << "Error loading bundle\n" ;
    }    
}
```

## How we teach this

If the proposal is accepted, the CppMicroServices guides will have to be modified to reflect the new functionality.
LoadBundle API function will be specific to extenders of Cppmicroservices such as DS and is not catered for every client of CppMicroServices.
Clients except for DS must follow the usual bundle lifetime (install => start => stop => uninstall) and avoid using LoadBundle API.

## Drawbacks

This allows one (framework API user) to load bundles anytime bypassing the usual bundle life cycle in the form of install => start => stop => uninstall.

## Unresolved questions
