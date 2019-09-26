- Start Date: 2019-07-26
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/7
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/376

# Add an API in Bundle class that can load shared libraries and retrieve desired symbols

## Summary

Currently there is no API function in Bundle class to load bundles/dynamic libraries and retrieve desired symbols at runtime.
This document discusses need for such API in the particular use case of declarative services (referred as DS henceforth in this document) and its possible design/solution.

## Motivation

Developers using DS rely on DS's SCR (service component runtime) to delay/lazily load the bundle's shared libraries. 
These bundles are loaded by calling extern "C" functions which internally call the native OS APIs.

One could argue that Bundle class's Start() function could be used OR modified to accomodate this functionality instead of adding a new member function in the Bundle class.
However, this is not possible because DS runtime can't call Bundle's Start() function as DS is waiting on the BUNDLE_STARTED event. This event is fired when the bundle is activated.
On a related note, since the DS bundles are marked as  "bundle.activator : false" , Start() function can not load the bundle's symbols from its shared library.

Given this, DS can't use bundle's Start() function to load the shared library associated with the bundle and needs a way to access the shared library's interface.
As such, DS uses OS specific API "load library" calls for retrieving bundles/shared objects and resolving symbols.

This loading of libraries/bundles increases the overhead in the declarative services while doing the symbol resoution and ref count increment done in the system.

Adding to that, this functionality to load bundles/shared objects is currently implemented at two places

1) Declarative services runtime
2) framework 

During the application run, these different callsites may become out of sync as CppMicroServices and DS are maintained.

Given this it seems appropriate to have a new Bundle::LoadSymbols API for CppMicroServices library.

## Requirements

For the specific use case of DS (and similar extender clients), have an API in the CppMicroServices that

1) loads shared library without dealing with OS specific data types
2) resolves symbols in a Bundle's shared library with the intent of calling exported functions
3) does not require the users of this API to write any OS specific code e.g DS would only call bundle.GetSymbol() instead of using native OS calls

## Detailed design

Bundle::GetSymbols() API will be declared in the file CppMicroServices/framework/include/cppmicroservices/Bundle.h as below
The output of above API will be in the wrapper class which will be as belows.

```
/**
* @brief  This class holds the output of GetSymbol API.
*
* It acts as a wrapper over the handles returned by the OS APIs that loads the library.
* Clients have to supply the function type as template argument to correctly get the desired symbol resolved using
* GetSymbols function. See the usage workflow for more details.
*
*/

namespace cppmicroservices {
ls ..libmwsplashscreen.so
class BundleHandle
{
    public:
        BundleHandle():handle(nullptr) {}
        ~BundleHandle() { handle = nullptr; }
        BundleHandle(const BundleHandle&);
        BundleHandle(BundleHandle&&);
        BundleHandle& operator=(const BundleHandle&);
        BundleHandle& operator=(BundleHandle&&);

        template<typename T>
        T ResolveSymbol(const std::string& symname);

    private:
        void * symHandle;
        BundleHandle(void *handle) : handle(symHandle) {}
        friend BundleHandle GetSymbols();
}

}

/**
* Retrieves the bundle symbol(s) using platform APIs and return a tuple of BundleHandle class object
* It uses following OS APIs to achieve the same
*
* On POSIX   : dlopen/dlsym
* On Windows : LoadLibraryW/GetProcAddress
* 
* A framework event of type FRAMEWORK_ERROR will fired if loading of symbols fail
* 
* @return A function pointer symbols of type T supplied by the user
*
* @throws std::runtime_error if this bundle is not initialized correctly, i.e if dlopen or LoadLibraryW fails
*
*/

BundleHandle GetSymbols();

```
As mentioned above, the API will call the platform APIs to open the shared objects.It will supply the name of the bundle using Bundle class's GetLocation() function to these APIs.
If these platform APIs fails, it will thrown an exception with relevant error information e.g. error code.

Once the bundle is opened, it will fetch the required symbols e.g. in case of DS, it will be the symbols of extern "C" functions containing constructor/destructors calls to services.
It will then populate the BundleHandle class object with the library handle. 
BundleHandle object in turn can be queried for a particular symbol by passing in a symbol name.

A typical usage workflow will be as below
```
Void SomeDeclarativeServiceUser(const cppmicroservices::Bundle& bd)
{
    try
    {
        BundleHandle bundle_handle = bd.GetSymbols();

        auto handleCon = bundle_handle.ResolveSymbol<ComponentInstance*(*)(void)>(constructor_name);
        auto handleDes = bundle_handle.ResolveSymbol<void(*)(ComponentInstance*)>(destructor_name);

        // Do stuff with the handleCon and handleDes
    }
    catch(...)
    {
        some::Log << "Error loading bundle\n" ;
    }    
}
```

## How we teach this

If the proposal is accepted, the CppMicroServices doxygen guide will be modified to reflect the new functionality.
GetSymbols API function will be specific to extenders of Cppmicroservices such as DS and is not catered for every client of CppMicroServices.
Clients except for DS must follow the usual bundle lifetime (install => start => stop => uninstall) and avoid using GetSymbols API.

## Drawbacks

This allows one (framework API user) to bypass a crucial step in the Bundle::Start() operation, the one where it loads the bundle's shared library.

## Unresolved questions
