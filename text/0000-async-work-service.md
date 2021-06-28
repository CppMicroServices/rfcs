- Start Date: 2020-10-12
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue:

# A Generic Asynchronous Work Service Interface

## Summary

> One paragraph explanation of the feature.

A service interface which allows users to execute and wait on  asynchronous work, allowing service interface implementations control over task scheduling and thread management.

This is a RFC for the service interface and not for the implementation of the service interface. There will be a separate RFC for the implementation.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

This is motivated by recent discoveries that using `std::async` does not scale as the number of processors/cores grows. Using `std::async` to perform asynchronous operations at scale is not efficient since each call can create a new thread, which can be an expensive operation. 

The primary motivation is to remove patterns in code that:

- create N threads because there are N cores
- create a thread per object (e.g. Bundle, socket, connection, widget, etc...)

An additional motivating factor is code re-use and not re-inventing the wheel. Considering that the CppMicroServices core framework, Declarative Services and Configuration Admin all use threads to perform asynchronous work, it makes sense to use the same mechanism to do so. Instead of maintaining separate implementations to manage async work, using a generic service to post asynchronous work (to a thread, thread pool or inline) would reduce duplicate code and facilitate control over the number of threads used by any software system using CppMicroServices.

A generic service to post and wait for asynchronous tasks enables CppMicroServices clients to use the same mechanism and provides a means to write their own service implementations to better meet their needs for executing asynchronous work.



### Use Case 1: CppMicroServices core Framework and compendium services

Jeff is a CppMicroServices maintainer responsible for CppMicroServices and it's compendium services. Two compendium services use multiple threads to run asynchronous work. Declarative Services does this by using a `boost::asio::thread_pool` while Config Admin uses `std::async`.

Jeff wants to use the same mechanism to execute asynchronous work in the core Framework, Config Admin and Declarative Services as not to maintain three separate asynchronous work mechanisms and to manage the number of threads used across the core Framework and it's compendium services.  There is no way to extend the asynchronous mechanisms used within these modules and make them accessible without deviating from the OSGi spec (**PP1**, **PP2**). 



### Use Case 2: Application integration with CppMicroServices

Nicole is an application developer responsible for developing a large scientific computing application. This application has many features which execute tasks asynchronously and has at least one thread pool implementation used to run these tasks. She uses CppMicroServices within the application and wants to have CppMicroServices use the same thread pool implementation used throughout the application (**PP1**). Currently it is not possible to control the threads used internally by CppMicroServices and it's compendium services.



### Use Case 3: Executing Asynchronous Tasks

Jeff is a CppMicroServices maintainer responsible for CppMicroServices and it's compendium services. All of the asynchronous tasks run in the CppMicroServices core framework, Declarative Services and Config Admin need to be waited upon. Currently, `std::async` and `boost::asio::post` are being used to execute asynchronous tasks, both of which use futures for the calling thread to block on. Any solution which replaces `std::async` or `boost::asio::post` needs to provide a way to wait on the result and a way to receive a failure from the asynchronous task (**PP3**). 



### Requirements

| ID   | Statement                                                    | Source / Pain Point | Priority  |
| ---- | ------------------------------------------------------------ | ------------------- | --------- |
| 1    | The service interface should be re-usable by any other CppMicroServices service or application. | PP1                 | Must Have |
| 2    | The service interface should be decoupled from CppMicroServices and all compendium services. | PP2                 | Must Have |
| 3    | CppMicroServices service based design                        | OSGi compliance     | Must Have |
| 4    | Allow users to block on the asynchronous task until it has finished. | PP3                 | Must Have |
| 5    | Allow users to receive a failure from the asynchronous task. | PP3                 | Must Have |



## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.



### API

```c++
namespace cppmicroservices { 
  namespace async {
      namespace detail {
    /**
     *
     */
    class AsyncWorkService {
    public:
      virtual ~AsyncWorkService() noexcept = default;
      
      /**
       * Run a std::packaged_task<void()> asynchronously on another thread.
       * The std::future<void> associated with the std::packaaged_task<void()>
       * task object will contain the result from the task object.
       *
       * @param task A std::packaged_task<void()> wrapping a Callable target
       * to execute asynchronously.
       *
       * @note The caller is required to manage the std::future<void> associated
       * with the std::packaged_task<void()> in order to wait on the async task.
       * 
       */
      virtual void post(std::packaged_task<void()>&& task) = 0;
    };
  }
  }
}
```



### Design Cases

This is what the proposed workflows would look like in contrast to any existing workflows.

#### Design Case #1: CppMicroServices and compendium services

##### Current Workflow for Configuration Admin:

Configuration Admin has wrapped a call to `std::async` in a templated function which is called throughout the code.

```c++
// This template function currently wraps a call to std::async and manages
// the std::future<void> objects returned.
template <typename Functor>
void ConfigurationAdminImpl::PerformAsync(Functor&& f)
{
    std::lock_guard<std::mutex> lk{futuresMutex};
    decltype(completeFutures){}.swap(completeFutures);
    auto id = ++futuresID;
    incompleteFutures.emplace(id, std::async(std::launch::async, [this, func = std::forward<Functor>(f), id]
                                                      {
                                                          func();
                                                          std::lock_guard<std::mutex> lk{futuresMutex};
                                                          auto it = incompleteFutures.find(id);
                                                          assert(it != std::end(incompleteFutures) &&
                                                                 "Invalid future iterator");
                                                          completeFutures.push_back(std::move(it->second));
                                                          incompleteFutures.erase(it);
                                                          if (incompleteFutures.empty())
                                                          {
                                                              futuresCV.notify_one();
                                                          }
                                                      }));
}
```

```c++
// PerformAsync call site example...
PerformAsync([this, pid, managedServiceFactory]
             {
                 std::vector<std::pair<std::string, AnyMap>> pidsAndProperties;
                 {
                     std::lock_guard<std::mutex> lk{configurationsMutex};
                     const auto it = factoryInstances.find(pid);
                     if (it != std::end(factoryInstances))
                     {
                         for (const auto &instance : it->second)
                         {
                             const auto configurationIt = configurations.find(instance);
                             assert(configurationIt != std::end(configurations) && 
                                    "Invalid Configuration iterator");
                             try
                             {
                                 auto properties = configurationIt->second->GetProperties();
                                 pidsAndProperties.emplace_back(instance, std::move(properties));
                             }
                             catch (const std::runtime_error&)
                             {
                                 // Configuration is being removed
                             }
                         }
                     }
                 }
                 for (const auto &pidAndProperties : pidsAndProperties)
                 {
                     notifyServiceUpdated(pidAndProperties.first, *managedServiceFactory, 
                                          pidAndProperties.second, *logger);
                 }
             });
```

##### Proposed Workflow for Configuration Admin:

The use of `std::async` can be replaced with a call to `post`. 

```c++
// This template function currently wraps a call to std::async and manages
// the std::future<void> objects returned.
template <typename Functor>
void ConfigurationAdminImpl::PerformAsync(Functor&& f)
{
    auto asyncTaskService = GetAsyncService();	/// imagine this returns std::shared_ptr<AsyncWorkService> 
    uint64_t id{};
    {
        std::lock_guard<std::mutex> lk{futuresMutex};
        decltype(completeFutures){}.swap(completeFutures);
        id = ++futuresID;
    }

    std::packaged_task<void()> task([id, func = std::forward<Functor>(f), id]() mutable {
        func();
        std::lock_guard<std::mutex> lk{futuresMutex};
        auto it = incompleteFutures.find(id);
        assert(it != std::end(incompleteFutures) &&
               "Invalid future iterator");
        completeFutures.push_back(std::move(it->second));
        incompleteFutures.erase(it);
        if (incompleteFutures.empty())
        {
            futuresCV.notify_one();
        }
    });

    std::future<void> fut = task.get_future();
    {
        std::lock_guard<std::mutex> lk{ futuresMutex };
        incompleteFutures.emplace(id, std::move(fut));
    }
    asyncTaskService->post(std::move(task));
}
```

<u>The PerformAsync call sites do not change.</u>



#### Design Case #2: Applications integrating with CppMicroServices

##### Proposed Workflow:

1. Implement the  service interface as a CppMicroServices bundle.
2. Install and start the bundle in the application code
3. Use the service, for example:

```c++
auto asyncTaskService = GetAsyncService();	/// imagine this returns std::shared_ptr<>

// fictional class types, 'foo' and 'bar'.
foo f;
bar b;

// use a std::packaged_task<void()> and wait on the std::future<void>
std::packaged_task<void()> task([f, b]() {/* do something */});
auto pkgtask_future = task.get_future();
asyncTaskService.post(std::move(task));
pkgtask_future.get();

// use a std::packaged_task<void(double)> wrapped in a std::packaged_task<void()>
// for use with the service and wait on the std::future<void>.
//
// this demonstrates that you can still create arbitrary packaged_task objects which
// require parameters or have return types and still be able to use the AsyncWorkService
// to perform the work on another thread.
using ActualTask = std::packaged_task<void(double)>;
using PostTask = std::packaged_task<void()>;
double my_val = 2.0;
ActualTask task2([](double a) {/* do something */});
auto task2_future = task2.get_future();
PostTask task2runner([my_val, task2Ptr = std::make_shared<ActualTask>(std::move(task2))]() mutable {
    (*task2Ptr)(my_val);
});
asyncTaskService->post(std::move(task2runner));
```

4. CppMicroServices can use this service implementation instead of whatever default async mechanism is used internally.





## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing CppMicroServices patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the CppMicroServices guides must be
re-organized or altered? Does it change how CppMicroServices is taught to new users
at any level?

> How should this feature be introduced and taught to existing CppMicroServices
users?

This feature is best represented using the existing names and terminology used in CppMicroServices. This service interface is meant to be implemented as a CppMicroServices bundle and accessed using the CppMicroServices Framework, like all the compendium services or any other user-provided CppMicroServices service.

Public API doxygen documentation should be sufficient to teach users about this service interface and how to use it. Any questions about integrating this service into another application are covered by existing documentation or documentation that will be created as part of the RFC for this service interface's implementation.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching CppMicroServices,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

There will be additional complexity within compendium services implementations to handle the optionality of this asynchronous work service, i.e. what to do if the service is not available?

Given that compendium services which want to use this service will need to have a fallback if the service is not present means that a default implementation of the service needs to ship with CppMicroServices **OR** each compendium service needs a way to execute work asynchronously that doesn't involve using this service interface.

Limitations of using template classes and functions in service interfaces. The service interface cannot leverage the full expressiveness of C++ templates. Templates cannot be used for the API as that requires compile time knowledge. Returning a value type other than `void` in the `std::future` requires a template wrapper class defined in the service interface header.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

#### Alternative 1

Do nothing, keep separate implementations in any CppMicroServices or compendium service that wants to execute async work. As the number of duplicate implementations grows the maintenance cost will grow. If a bug is found in one copy of the implementation, developers will have to remember to make that fix in all duplicate implementations.

Application developers integrating with CppMicroServices will not be able to leverage the same async task mechanism nor control it.

#### Alternative 2

Create a static library within the CppMicroServices project which implements a generic asynchronous task function and is linked into CppMicroServices, the compendium services and any other code within the CppMicroServices project which wants to use it.

Application developers integrating with CppMicroServices will not be able to leverage the same async task mechanism nor control it.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
