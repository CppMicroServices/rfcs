- Start Date: 2020-10-12
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue:

# A Generic Asynchronous Work Service Interface

## Summary

> One paragraph explanation of the feature.

A service interface which allows users to perform and wait on unbounded asynchronous work allowing service interface implementations control over task scheduling and thread management.

This is a RFC for the service interface and not for the implementation of the service interface. There will be a separate RFC for the implementation.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

This is motivated by recent discoveries that using `std::async` does not scale as the number of processors/cores grows. Using `std::async` to perform asynchronous operations at scale is not efficient since each call can create a new thread, which can be an expensive operation. 

The primary motivation is to remove patterns in code that:

- create N threads because there are N cores
- create a thread per object (e.g. Bundle, socket, connection, widget, etc...)

An additional motivating factor is code re-use and not re-inventing the wheel. Considering that the CppMicroServices core framework, Declarative Services and Configuration Admin all use threads to perform asynchronous work, it makes sense to use the same mechanism to do so. Instead of maintaining separate implementations to manage async work, using a generic service to post asynchronous work would reduce duplicate code and facilitate better control over the number of threads used by any software system using CppMicroServices.

A generic service to post and wait for asynchronous tasks enables users of CppMicroServices to use the same mechanism and provides users a means to write their own service implementations to better meet their needs for processing asynchronous work.



### Use Case 1: CppMicroServices core Framework and compendium services

Jeff is a CppMicroServices maintainer responsible for CppMicroServices and it's compendium services. Two compendium services use asynchronous operations. Declarative Services does this by using a boost asio thread pool while Config Admin uses `std::async`.

Jeff wants to make the same improvements to Config Admin that were made to Declarative Services, namely implementing a thread pool and remove the use of `std::async` for asynchronous tasks. To accomplish this Jeff can migrate the same Declarative Services change, which includes a build-time dependency on boost asio, to Config Admin (**PP1**, **PP2**).  



### Use Case 2: Applications integrating with CppMicroServices

Nicole is an application developer responsible for developing a large scientific computing application. This application has many features which use asynchronous tasks and has at least one thread pool implementation used to run these asynchronous tasks. She wants to integrate with CppMicroServices and also be able to manage the threads used by CppMicroServices. Currently it is not possible to control the threads used by CppMicroServices (**PP1**).



### Use Case 3: Unbounded Asynchronous Work

Jeff is a CppMicroServices maintainer responsible for CppMicroServices and it's compendium services. All of the asynchronous tasks done in CppMicroServices core framework, Declarative Services and Config Admin are unbounded and need to be waited upon. Most of the asynchronous tasks involve calling back into user code via callbacks and as such cannot be guaranteed to finish in a bounded amount of time. Currently `std::async` or `boost::asio::post` are being used, both of which use futures for the calling thread to block on. Any solution which replaces `std::async` or `boost::Asio::post` would need to provide a way to wait on the result and a way to receive a result or failure from the async task (**PP3**). 



### Requirements

| ID   | Statement                                                    | Source / Pain Point | Priority  |
| ---- | ------------------------------------------------------------ | ------------------- | --------- |
| 1    | The service should be re-usable by any other CppMicroServices service or application. | PP1                 | Must Have |
| 2    | The service should be decoupled from CppMicroServices and all compendium services. | PP2                 | Must Have |
| 3    | CppMicroServices service based design                        | OSGi compliance     | Must Have |
| 4    | Allow users to block on the asynchronous task until it has finished. | PP3                 | Must Have |
| 5    | Allow users to receive the result of a failure from the asynchronous task. | PP3                 | Must Have |



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

    class AsyncWork {
    public:
      virtual ~AsyncWork() noexcept = default;
      
      /**
       *
       */
      virtual std::future<void> post(std::packaged_task<void()>&&) = 0;
    };
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

This feature is best represented using the existing names and terminology used in CppMicroServices. This service interface is meant to be implemented as a CppMicroServices bundle and accessed using the CppMicroServices Framework, like all the compendium services or any other user-provided service.

Public API doxygen documentation should be sufficient to teach users about this service interface and how to use it. Any questions about integrating this service into another application are covered by existing documentation or documentation that will be created as part of the RFC for this service interface's implementation.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching CppMicroServices,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

There will be additional complexity within compendium services implementations to handle the optionality of this asynchronous work service, i.e. what to do if the service is not available?

Given that compendium services which want to use this service will need to have a fallback if the service is not present means that a default implementation of the service needs to ship with CppMicroServices or each compendium service needs a way to execute work asynchronously that doesn't involve using this service interface.

Limitations of using template classes and functions in service interfaces. 

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

There is a limitation to what can be passed in to and returned from `post`. Templates cannot be used for the API as that requires compile time knowledge and breaks the service based design. Returning future<void> and passing in `std::packaged_task<void()>` puts restrictions on API users. For example, to pass in parameters to a packaged_task users would need to use `std::bind`. <u>Can an API be created to return future<T> and pass in a Callable object with a function signature other than void()?</u>