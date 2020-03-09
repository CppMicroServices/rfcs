- Start Date: 2020-03-03
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/9
- CppMicroServices Issue: TBD

# Shared library loading error handling in Declarative Services

## Summary

> One paragraph explanation of the feature.

Provide a means for clients to know when a service implemented using Declarative Services could not be retrieved due to a failure to load the bundle's shared library. Such exceptional conditions are caught internally by Declarative Services and logged. Such exceptional behavior should be visible to clients similar to how starting a bundle using `Bundle::Start` when using Bundle Activators will throw an exception if the bundle's shared library cannot be loaded.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

When moving bundle implementations from using Bundle Activators to Declarative Services with lazy loading of the bundle's shared library, failures from loading the shared library will not occur until the service is retrieved, i.e. when `BundleContext::GetService()` is called. When the shared library fails to load the result is that `BundleContext::GetService()` returns a `nullptr`. It's not clear whether the `nullptr` is a result of the shared library failing to load or by some other means.

Declarative Services does log more detailed information about the source of the failure however this is disconnected from the point at which the failure actually occurred. 

The problem is that it becomes more difficult to add error handling mechanisms which make it easy for developers to pinpoint the source of shared library loading errors when using Declarative Services.



#### Use Case 1 

setup_system.cpp - code to setup the system, allowing GetService to be called by other clients

```c++
// get the framework

// install and start a bundle which will be used by other clients later in the lifetime of the system

// for the purposes of this use case, this bundle's shared library has missing link-time dependencies which will cause dlopen/LoadLibrary to fail.
auto bundles = framework.GetBundleContext().InstallBundles("bundlefoo.dll");

try {
  // Start() will not throw, it will appear that there are no failures.
  bundles.front().Start();
} catch(...) {
  // display meaningful message to users.
  // Nothing will be caught if the bundle is using DS.
}
```

client.cpp

```c++
// get the framework

// DS will provide a valid ServiceReference before it even tries to load the shared library. At this point, it isn't known whether the shared library will fail to load or not.
auto svcRef = framework.GetBundleContext().GetServiceReference<Foo>();

// This will return a nullptr if the shared library failed to load.
auto svc = framework.GetBundleContext().GetService(svcRef);

if(nullptr == svc) {
  // ...  error handling if svc == nullptr? ... don't know the source nor context of the failure...
  // how does the client code report a meaningful error message?
}
```



## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

#### Overview

- Create a new exception type for shared library load failures - `SharedLibraryException`
- Throw this exception from `Bundle::Start` and have DS throw it from its `ServiceFactory::GetService` implementation.
- Modify CppMicroServices to catch a `SharedLibraryException` exception, log it per the OSGi spec, and rethrow it.
- Modify the DS implementation to not catch exceptions thrown from a failure to load the shared library.
- `SharedLibraryException` will be constructed with a valid `Bundle` object representing the shared library which failed to load.



#### Functional Design

```c++
namespace cppmicroservices {
class SharedLibraryException final : public std::system_error
{
public:
  explicit SharedLibraryException(const std::error_code ec, 
                                  const std::string& what,
                                  const cppmicroservices::Bundle& origin);
  ~SharedLibraryException() override;
  Bundle GetBundle() const;
    
private:
  Bundle origin;  ///< The bundle of the shared library which failed to load.
}
}
```



client.cpp

```c++
// get the framework

// DS will provide a valid ServiceReference before it even tries to load the shared library. At this point, it isn't known whether the shared library will fail to load or not.
auto svcRef = framework.GetBundleContext().GetServiceReference<Foo>();

// This will throw a SharedLibraryException if the shared library failed to load.
// either allow the throw to propagate or try/catch and do something
try {
  auto svc = framework.GetBundleContext().GetService(svcRef);
} catch(const cppmicroservices::SharedLibraryException& ex) {
  // do something with ex...
}

if(nullptr == svc) {
  // do something knowing that the reason this is nullptr is due to a recoverable failure.
}
```



#### CppMicroServices implementation changes

- `ServiceReferenceBasePrivate::GetServiceFromFactory` needs to explicitly catch `SharedLibraryException` before catching `std::exception`, log a `FrameworkEvent` and rethrow.
- `Bundle::Start()` needs to throw `SharedLibraryException` if `dlopen/LoadLibrary` failed.



#### DS implementation changes

- Declarative Services needs to throw a `SharedLibraryException` in two cases; 
  - When the service component has `immediate` set to true.
    - `Bundle::Start()` will throw a `SharedLibraryException`. 
  - When the service component has `immediate` set to false. 
    -  `BundleContext::GetService()` will throw a `SharedLibraryException`.
    - Declarative Services has try/catch blocks around the operations to enable a service component. These try/catch blocks will have to be modified to catch a `SharedLibraryException` and rethrow it. The try/catch block starts in `SCRActivator::CreateExtension` with more try/catch blocks contained within the call chain of this method, ultimately ending in `BundleLoader::GetComponentCreatorDeletors` where the `dlopen/LoadLibrary` happens. 



## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing CppMicroServices patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the CppMicroServices guides must be
re-organized or altered? Does it change how CppMicroServices is taught to new users
at any level?

> How should this feature be introduced and taught to existing CppMicroServices
users?

- Requires updates to the doxygen documentation for  `BundleContext::GetService` and `Bundle::Start()`describing the new exception type which can be thrown. 



## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching CppMicroServices,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

- Not OSGi compliant. The relevant OSGi text can be found under ServiceFactory::GetService where it describes how errors coming from service factories (which DS uses to do its job) should be handled:

> The Framework must check that the returned service object is valid. If the returned service object is `null` or is not an `instanceof` all the classes named when the service was registered, a framework event of type [FrameworkEvent.ERROR](https://osgi.org/specification/osgi.core/7.0.0/framework.api.html#org.osgi.framework.FrameworkEvent.ERROR) is fired containing a service exception of type [ServiceException.FACTORY_ERROR](https://osgi.org/specification/osgi.core/7.0.0/framework.api.html#org.osgi.framework.ServiceException.FACTORY_ERROR) and `null` is returned to the bundle. **If this method throws an exception, a framework event of type [FrameworkEvent.ERROR](https://osgi.org/specification/osgi.core/7.0.0/framework.api.html#org.osgi.framework.FrameworkEvent.ERROR) is fired containing a service exception of type [ServiceException.FACTORY_EXCEPTION](https://osgi.org/specification/osgi.core/7.0.0/framework.api.html#org.osgi.framework.ServiceException.FACTORY_EXCEPTION) with the thrown exception as the cause and `null` is returned to the bundle.** If this method is recursively called for the specified bundle, a framework event of type [FrameworkEvent.ERROR](https://osgi.org/specification/osgi.core/7.0.0/framework.api.html#org.osgi.framework.FrameworkEvent.ERROR) is fired containing a service exception of type [ServiceException.FACTORY_RECURSION](https://osgi.org/specification/osgi.core/7.0.0/framework.api.html#org.osgi.framework.ServiceException.FACTORY_RECURSION) and `null` is returned to the bundle.



## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

#### Alternative 1 

Implement the LogService callback mechanism and have user code use it to receive the exception object when a shared library fails to load.

#### Alternative 2 

Create a `BundleContext::GetService` overload - ` GetService(ServiceReference, std::exception_ptr& out)` - for callers who want to know about exceptional, unexpected behavior. The second parameter is an out parameter containing the exception thrown from the user implemented `ServiceFactory::GetService` method.

#### Alternative 3

Modify user code to register a framework listener and do appropriate error handling when a framework event of type `Error` with an exception type of `FactoryException`. **NOTE**: event listeners cannot rethrow from their callback and expect it to propagate. It will be caught by CppMicroServices.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

The name of the exception class, `SharedLibraryException`, is a work in progress...

How does OSGi report back failures to load bundle JARs? Can that be applied in C++?
