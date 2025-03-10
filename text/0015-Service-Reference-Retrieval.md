- Start Date: 2024-10-10
- CppMicroServices PR: https://github.com/CppMicroServices/CppMicroServices/pull/1044

# Add an free API in that can retrieve a <code>ServiceReference</code> from a <code>std::shared_ptr\<T\></code> given to the client by the framework

## Summary

Currently there is no API in the <code>cppmicroservices</code> namespace that can retrieve a <code>ServiceReference</code> from the <code>std::shared_ptr\<T\></code> itself.
This document discusses need for such API in the particular use case of declarative services (DS) and its possible design/solution.

## Motivation

Developers using DS rely on DS's SCR (service component runtime) to inject services statically at construction or dynamically using the <code>bind</code> and <code>unbind</code> methods. These <code>std::shared_ptr\<T\></code>s can then be used by the client. However, clients often use properties in the Service to store metadata relevant to that service. Those properties, however, are not available to the <code>std::shared_ptr\<T\></code> itself, only via its <code>ServiceReference</code>.

Given this it seems appropriate to have a new method that the client can use to translate a <code>std::shared_ptr\<T\></code> into its <code>ServiceReference</code>.

## Requirements

This API must:

1. take in a <code>std::shared_ptr\<T\></code> and return the <code>ServiceReference\<T\></code> for that service where <code>T</code> is any class interface
2. not expose internal implementation details to the user

## Detailed design

<code>ServiceReferenceFromService</code> API will be declared in the file <i>framework/include/cppmicroservices/ServiceReference.h</i> in the namespace <code><bold>cppmicroservices::</bold></code> as below:

```c++

/**
* \ingroup MicroServices
* \ingroup gr_servicereference
*
* A method to retrieve a service object's original <code>ServiceReference<void></code>
*
*/
US_Framework_EXPORT ServiceReferenceU ServiceReferenceFromService(std::shared_ptr<void> const& s);

/**
* \ingroup MicroServices
* \ingroup gr_servicereference
*
* A method to retrieve a service object's original <code>ServiceReference<U></code>
*
* @tparam T The class type of the service object
* @tparam U The class type of the <code>ServiceReference</code>. Defaults to <code>T</code> if not specified.
*/
template <typename T, typename U = T>
ServiceReference<U>
ServiceReferenceFromService(std::shared_ptr<T> const& s)
{
    return ServiceReference<U>(ServiceReferenceFromService(std::static_pointer_cast<void>(s)));
}

```

This API will take the <code>std::shared_ptr\<T\></code> and first verify that it was a <code>std::shared_ptr\<T\></code> created by the CppMicroServices framework.

If it is valid, it will return the original <code>ServiceReference</code> for that <code>std::shared_ptr\<T\></code>.

A typical usage workflow could be as below, in the static constructor of a service <code>ServiceImpl</code> which depends on a service of type <code>ServiceInterface2</code>.

```c++
ServiceImpl::ServiceImpl(std::shared_ptr<ServiceInterface2> const& dep) {
    // get the reference
    ServiceReference<ServiceInterface2> depRef = cppmicroservices::ServiceReferenceFromService(dep);
    // get a property from the ServiceReference's metadata
    auto someProp = cppmicroservices::any_cast<std::string>(retSRef.GetProperty("someProp"));
    // use that property
    doSomethingWith(someProp);
}

```

## Implementation

In order to solve this problem, we have to somehow embed metadata into the <code>std::shared_ptr\<T\></code> about its <code>ServiceReference</code> in a way that is inaccessible directly by clients.

One solution that we have found is to embed a custom deleter into the <code>std::shared_ptr\<ServiceHolder\></code> and retrieve that using the <code>std::get_deleter</code> functionality.

The objects that would allow this can be seen below:

```c++

/* @brief Private helper struct used to facilitate the shared_ptr aliasing constructor
    *        in BundleContext::GetService method. The aliasing constructor helps automate
    *        the call to UngetService method.
    *
    *        Service consumers can simply call GetService to obtain a shared_ptr to the
    *        service object and not worry about calling UngetService when they are done.
    *        The UngetService is called when all instances of the returned shared_ptr object
    *        go out of scope.
    */
template <class S>
struct ServiceHolder
{
    bool singletonService;
    std::weak_ptr<BundlePrivate> const b;
    ServiceReferenceBase const sref;
    std::shared_ptr<S> const service;
    InterfaceMapConstPtr const interfaceMap;

    ServiceHolder(ServiceHolder&) = default;
    ServiceHolder(ServiceHolder&&) noexcept = default;
    ServiceHolder& operator=(ServiceHolder&) = delete;
    ServiceHolder& operator=(ServiceHolder&&) noexcept = delete;

    ServiceHolder(std::shared_ptr<BundlePrivate> const& b,
                    ServiceReferenceBase const& sr,
                    std::shared_ptr<S> s,
                    InterfaceMapConstPtr im)
        : singletonService(s ? true : false)
        , b(b)
        , sref(sr)
        , service(std::move(s))
        , interfaceMap(std::move(im))
    {
    }

    ~ServiceHolder()
    {
        try
        {
            singletonService ? destroySingleton() : destroyPrototype();
        }
        catch (...)
        {
            // Make sure that we don't crash if the shared_ptr service object outlives
            // the BundlePrivate or CoreBundleContext objects.
            if (!b.expired())
            {
                DIAG_LOG(*b.lock()->coreCtx->sink)
                    << "UngetService threw an exception. " << util::GetLastExceptionStr();
            }
            // don't throw exceptions from the destructor. For an explanation, see:
            // https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md
            // Following this rule means that a FrameworkEvent isn't an option here
            // since it contains an exception object which clients could throw.
        }
    }

    private:
    void
    destroySingleton()
    {
        sref.d.Load()->UngetService(b.lock(), true);
    }

    void
    destroyPrototype()
    {
        auto bundle = b.lock();
        if (sref)
        {
            bool isPrototypeScope
                = sref.GetProperty(Constants::SERVICE_SCOPE).ToString() == Constants::SCOPE_PROTOTYPE;

            if (isPrototypeScope)
            {
                sref.d.Load()->UngetPrototypeService(bundle, interfaceMap);
            }
            else
            {
                sref.d.Load()->UngetService(bundle, true);
            }
        }
    }
};

/* @brief Private helper struct used to facilitate the retrieval of a serviceReference from
    *        a serviceObject.
    *
    *        Service consumers can pass a service to the public API ServiceReferenceFromService.
    *        This method can use the std::get_deleter method to retrieve this object and through
    *        it the original serviceReference.
    */
class CustomServiceDeleter
{
    public:
    CustomServiceDeleter(ServiceHolder<void>* sh) : sHolder(sh) {}

    void
    operator()(ServiceHolder<void>* sh)
    {
        delete sh;
    }

    [[nodiscard]] ServiceReferenceBase
    getServiceRef() const
    {
        return sHolder->sref;
    }

    private:
    ServiceHolder<void> const* const sHolder;
};

ServiceReferenceU
ServiceReferenceFromService(std::shared_ptr<void> const& s)
{
    auto deleter = std::get_deleter<CustomServiceDeleter>(s);
    if (!deleter)
    {
        throw std::runtime_error("The input is not a CppMicroServices managed ServiceObject");
    }
    return deleter->getServiceRef();
}
```

The new way that <code>ServiceHolder</code> objects would be constructed can be seen below:

```c++
// For a Singleton object (from framework/src/Bundle/BundleContext.cpp)
auto serviceHolder = new ServiceHolder<void>(b, reference, reference.d.Load()->GetService(b.get()), nullptr);
std::shared_ptr<ServiceHolder<void>> h(serviceHolder, CustomServiceDeleter { serviceHolder });
return std::shared_ptr<void>(h, h->service.get());

// For a prototype object (from framework/src/service/ServiceObjects.cpp)
auto sh = new ServiceHolder<void> { bundle_, d->m_reference, nullptr, result };
std::shared_ptr<ServiceHolder<void>> h(sh, CustomServiceDeleter { sh });
return InterfaceMapConstPtr(h, h->interfaceMap.get());
```

## How we teach this

If the proposal is accepted, the CppMicroServices doxygen guide will be modified to reflect the new functionality.
Most clients will not need this functionality. Configurations can be injected into DS services using ConfigAdmin.

## Drawbacks

- We are using custom deleters in a way that they were not intended to be used.
