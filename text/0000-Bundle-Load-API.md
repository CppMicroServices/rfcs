- Start Date: 2019-07-26
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/7
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/376

# Add an API in Bundle class that can load shared libraries and retrieve desired symbols

## Summary

Currently there is no API function in Bundle class to load bundles/dynamic libraries and retrieve desired symbols at runtime.
This document discusses need for such API in the particular use case of declarative services (referred as DS henceforth in this document) and its possible design/solution.

## Motivation

Developers using DS rely on DS's SCR (service component runtime) to delay/lazily load the bundle's shared libraries. 
The SCR loads the shared libraries and calls the extern "C" functions exported by the DS bundles.

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

Given this it seems appropriate to have a new Bundle::GetSymbol API for CppMicroServices library.

## Requirements

For the specific use case of DS (and similar extender clients), have an API in the CppMicroServices that

1) fetches a resolved symbols in a Bundle's shared library with the intent of calling exported functions
2) does not require the users of this API to write any OS specific code e.g DS would only call bundle.GetSymbol() instead of using native OS calls
3) does not change the bundle lifecyle state when a symbol is retrieved

## Detailed design

Bundle::GetSymbol() API will be declared in the file CppMicroServices/framework/include/cppmicroservices/Bundle.h as below
The output of the API will be the function pointer type supplied by the API user.

```

/**
* Retrieves the resolved symbol from bundle shared library and returns a function pointer associated with it
* 
* @param1 Handle to the Bundle's shared library
* @param2 Name of the symbol
*
* @return A function pointer symbols of type T supplied by the user or nullptr if the library is not loaded
* 
* @throws std::runtime_error if the bundle is not started or active.
*
* @pre  Bundle is already started and active
*
* @post The symbol(s) associated with the bundle gets fetched if the library is loaded
* @post If the symbol does not exist, the API returns nullptr 
*/

template<typename T>
T GetSymbol(void * handle, const std::string& symname);

```
This API will first check if the bundle (from which the symbols are to be fetched) is in START + ACTIVE state.
If the state is not START + ACTIVE , it will throw an exception with message as "invalid bundle".

It yes, it will then supply the following incoming params to BundleUtils's GetSymbol function.

1) type of the symbol
2) name of the symbol
3) handle to the bundle's shared library 

If the valid handle is retrieved, it will be returned to the caller
If not, nullptr will be returned.

For ex. in case of DS, it will be the symbols of extern "C" functions containing constructor/destructors calls to services.
Note that, DS will first load the bundle (after waiting on Bundle started event) using SharedLibrary::Load API.

A typical usage workflow will be as below
```
Void DeclarativeServiceImplFunc(const cppmicroservices::Bundle& bd)
{
    try
    {
        SharedLibrary lib (bd.GetLocation());
        lib.Load();
        if(lib.IsLoaded())
        {
            auto handleCon = bd.GetSymbol<ComponentInstance*(*)(void)>(sh.GetHandle(), constructor_name);
            auto handleDes = bd.GetSymbol<void(*)(ComponentInstance*)>(sh.GetHandle(), destructor_name);
            // Do stuff with the handleCon and handleDes
        }
    }
    catch(...)
    {
        some::Log << "Error loading bundle\n" ;
    }    
}
```


## How we teach this

If the proposal is accepted, the CppMicroServices doxygen guide will be modified to reflect the new functionality.
GetSymbol API function will be specific to extenders of Cppmicroservices such as DS and is not catered for every client of CppMicroServices.
Clients except for DS must follow the usual bundle lifetime (install => start => stop => uninstall) and avoid using GetSymbol API.

## Drawbacks

This allows one (framework API user) to bypass a crucial step in the Bundle::Start() operation, the one where it loads the bundle's shared library.

## Unresolved questions
