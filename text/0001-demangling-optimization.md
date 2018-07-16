- Start Date: 2018-07-10
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/2
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/284

# Caching Demangling Interfaces for Performance

## Motivation

When getting a service reference, significant amount of time is spent
in the conversion of interface types to names. By caching the name
of the interface every time we encounter a new interface type, we can
significantly reduce the time it takes to get a service reference.

## Description of the Problem

CppMicroServices stores interface names in the ServiceRegistry as the keys. An example of an interface name is `Foo::Bar`. This interface name is found by demangling the typeid of T whenever a service is registered through a `BundleContext::RegisterService<T>` call.

Later, when the service is requested through a `BundleContext::GetServiceReference<T>` call, CppMicroServices converts the T into the interface name and queries the map in the service registry.

However, these demangling calls are costly - benchmark profile for a `GetServiceReference<T>` call suggests that the demangling is done thrice - and takes about 50% of the time. (Refer to the GitHub issue).

## Detailed Design

The proposal introduces a function local static that gets initialized by demangling the typename once for T (during RegisterService<T>) - this has the added advantage of being thread-safe. Even when users provide a custom interface name through `CPPMICROSERVICES_DECLARE_SERVICE_INTERFACE` macro, this solution works seamlessly, because all that macro does is to specialize the `us_service_interface_iid` for T. The following is the proposed code changes:

`File:ServiceInterface.h`

```diff
+template <typename T> struct Demangle
+{
+  std::string const& operator()() const
+  {
+    static const std::string name = cppmicroservices::detail::GetDemangledName(typeid(T));
+    return name;
+  }
+};
+
 /// \cond
 template<class T> std::string us_service_interface_iid()
 {
-  return cppmicroservices::detail::GetDemangledName(typeid(T));
+  return Demangle<T>()();
 }
```

## Drawbacks

Because the demangled names are cached, a new static std::string that represents the interface name gets statically initialized each time the framework encounters a new T (either through RegisterService<T> or GetServiceReference<T>). Corollary: No storage needed for an already encountered T.

However, this slight increase in storage pales in comparison to the benefits of the speed-up. 

## Alternatives

### Approach 1

To eliminate demangling (and store the mangled strings in ServiceRegistry), we can simply remove the calls that take interface names when registering or getting a service, however this approach currently seems infeasible because these variants are used in declarative services (where the references to a service is specified as interface strings in the service description). 

Also, this approach is not backwards compatible.

### Approach 2

We can store the mangled strings in the ServiceRegistry and can maintain backwards compatibility by not removing the calls in Approach #1, but this would necessitate that we have a function that takes the interface string and converts it to mangled string and then stores it in the registry. Implementing this reverse demangled string -> mangled string is arcane and hard.
