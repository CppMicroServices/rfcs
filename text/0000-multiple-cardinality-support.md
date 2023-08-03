- Start Date: 2023-07-19
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue:

# Adding support for multiple cardinality for service references

## Summary

> One paragraph explanation of the feature.

This RFC documents enabling support for multiple cardinality for service references in Declarative Services. Multiple cardinality implies that service components can now use multiple instances of a service reference by specifying multiplicity in the manifest.json description file.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

Service components specify dependencies in the component's manifest.json file under the 'references' element along with various properties. Here they also mention the reference cardinality which is the number of instances of the service reference type required/supported by the service component. We specify it as follows: "<minimum_cardinality>..<maximum_cardinality>".

Currently, CPPMS supports specifying minimum cardinality as 0 or 1 and maximum cardinality as 1 only . That is, at a time a service component can have maximum of 1 reference to a dependency.

With this feature service components can specify multiplicity for service references and thus use multiple instances of it in their implementation.


### Use Case 1: Service authors using service references in their service components

Service authors while defining a new service component specify service dependencies of the component and various properties to define the service reference usage like policy, policy-option, cardinality, etc. They specify the service reference cardinality to mention the number of reference instances they can use.

Currently, users can provide following options for cardinality of a service reference:

0..1 - Optional and unary - component can have minimum 0 service references (optionality) and maximum 1 service reference (multiplicity) in their implementation.

1..1 - Mandatory and unary (Default setting) - component should always have 1 service reference (optionality) and maximum 1 service reference (multiplicity) in their implementation.

Users thus can use maximum of 1 instance of the service reference in their service component. There is no method to write services which use multiple service reference instances in Declarative Services. Only workaround for this workflow could be to not use Declarative Services which invalidates its benefits.

**Pain Point 1**:
Service authors cannot create service components that can bind to multiple instances of service dependencies dynamically or statically. They can be bound to maximum of 1 service reference.


### Requirements

| ID   | Statement                                                    | Source / Pain Point | Priority  |
| ---- | ------------------------------------------------------------ | ------------------- | --------- |
| 1    | Support for multiple cardinality for service references. Add support for specifying multiplicity in the 'cardinality' property in the manifest.json component description file. | PP1                 | Must Have |
| 2    | For a service reference with 'dynamic' policy, support for binding to multiple service reference instances by invoking user defined bind/unbind methods accordingly. | PP1                 | Must Have |
| 3    | For a service reference with 'static' policy, support for injecting multiple service reference instances in the constructor of the component and trigger reactivation accordingly. | PP1                 | Must Have |
| 4    | LocateService / LocateServices API provided by Declarative Services should return correct result for multiple cardinality case. | PP1                 | Must Have |
| 5    | For a service component with inject-references property set to true, support injecting multiple service reference instances in the constructor of the component. | PP1                 | Must Have |
| 6    | When inject-references property set to true, throw compilation error if the user component's implementation class does not define a constructor taking in multiple service reference instances. | PP1                 | Must Have |
| 7    | Any implementation should adhere to the specification OSGi Declarative Services Specification version 1.4, as it makes sense for C++. | The OSGi specification is an industry standard for creating modular applications. It saves us from re-inventing the wheel. | Must Have |
| 8    | The implementation must be efficient, even when stressed with many bundles and service references. | Multiplicity support should work efficiently when the bound service references are high in number, maximum limit should be set. | Must Have |

<br>

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

### Specifying multiple cardinality for a service reference:
Multiplicity property in cardinality of a service reference only specifies if the component implementation is written to handle being bound to multiple services (n) or requires SCR to select and bind to a single service (1).

Cardinality property is specified in the manifest.json component description file as a property of the service reference. For multiple cardinality, following values are introduced:

- 0..n: Optional and multiple; component can have minimum of 0 reference instances and maximum 'n' i.e. multiple instances (more than 1) <br>
- 1..n: Mandatory and multiple; component should have minimum 1 reference instance and maximum 'n' i.e. multiple instances (more than 1) <br>
**NOTE**: 'n' here is the alphabet and not any specific number that can be defined by user. It just denotes the multiplicity for the service reference.

Example snippet for how cardinality is specified in 'references' element of manifest.json file:

````json
"references":[{
    "name" : "foo",
    "cardinality" : "0..n",
    "policy" : "static",
    "policy-option" : "greedy",
    "interface" : "test::Interface1"
}]
````
Maximum number of instances allowed is specified as the max limit of size_t data type. Definition for 'n' is added in ReferenceMetadata.cpp file:

````c++
maxCardinality = std::numeric_limits<std::size_t>::max();
````
### Workflow for service references using 'dynamic' policy property:

To work with multiple cardinality, user will have to define a member(s) in its implementation class to store multiple instances of that reference. For example, class in below snippet uses a container - vector to accommodate the multiple references.

User still needs to define the bind/unbind method with 1 service reference as input argument (as is currently followed). These methods will be invoked multiple times by the framework to bind/unbind multiple references. Binding/unbinding logic is provided by the user.

Example class with dynamic policy and multiple cardinality:

````c++
class ServiceComponentDynamicExample final : public test::Interface2
    {
      public:
        ServiceComponentDynamicExample() = default;
        ~ServiceComponentDynamicExample() = default;
        virtual std::string ExtendedDescription() override;

        void Activate(std::shared_ptr<ComponentContext> const&);
        void Deactivate(std::shared_ptr<ComponentContext> const&);

        void Bindfoo(std::shared_ptr<test::Interface1> const&); // Bind method for foo-Interface1 reference required
        void Unbindfoo(std::shared_ptr<test::Interface1> const&); // Unbind method for foo-Interface1 reference required

      private:
        std::mutex fooMutex;
        std::vector< std::shared_ptr<test::Interface1>> fooPtrs; // member vector to store multiple instances of Interface1
    };
````
### Workflow for service references using 'static' policy property or/and using 'inject-references' property as true:
For service references specified with 'static' policy and/or 'inject-references' property as true, constructor injection is used to initialize references in the implementation class.

User's implementation class should include:

- Member variable(s) to store multiple references
- Constructor that takes in the multiple instances of references stored in a std::vector container

Example constructor declaration for 'static' policy or 'inject-references' property set as 'true':
````c++
class ServiceComponentStaticExample final : public test::Interface2
    {
      public:
        ServiceComponentStaticExample() = default;
        ServiceComponentStaticExample(std::vector<std::shared_ptr<test::Interface1>> const&); // constructor taking in multiple arguments
        ~ServiceComponentStaticExample() = default;

        ... other methods ...

      private:

        std::mutex fooMutex;
        std::vector<std::shared_ptr<test::Interface1>> fooPtrs; // member variable storing multiple references
    };
````

If such a constructor is not specified, framework will throw compilation error:
````
>> error C2338: An appropriate constructor was not found and/or the service implementation does not implement all of the service interface's methods. A constructor with service reference input parameters or a constructor with an AnyMap input parameter for configuration properties and service reference input parameters is required when inject-references is true.
````

**NOTE**: Service authors/consumers should note OSGi specification states that static policy is usually not applicable if the cardinality specifies multiple bound services. It is not well suited for multiplicity as the service will be re-created each time new service references appear in the system.

### Behavior defined for multiple cardinality options with the corresponding policy and policy-option values:

OSGi defines following as the requirements for when a higher ranked service or new service (in case no service references are bound yet) becomes available, i.e. the expected behavior for multiple cardinality depending on the policies applied:

| Cardinality   | static reluctant | static greedy | dynamic reluctant | dynamic greedy |
| ---- | --------- | ---------- | --------- | ---------- |
| 0..n	| Ignore | Reactivate to bind the better target service | Bind new target service | Bind new target service
| 1..n | Ignore | Reactivate to bind the better target service | Bind new target service | Bind new target service |
<br>

**EDGE CASE** : Maximum number of bound service references allowed

At a time, any service component can bind to a maximum of size_t type's max value supported (as shown above). If there are more potential higher ranked services available after this, the framework will not bind those references to the service component. A message will logged by Declarative Services informing the user that the limit of multiple references has reached and will continue the code flow ahead.

### LocateService API behavior:

If multiple cardinality is specified and multiple services are bound, LocateService API returns the service with the highest ranking (as specified in its Constants::SERVICE_RANKING property). If there is a tie in ranking, the service with the lowest service id (as specified in its Constants::SERVICE_ID property); that is, the service that was registered first is returned.

For example, if  services are registered with different rankings:
````c++
auto lowerRankedSvc = bc.RegisterService<test::Interface1>(std::make_shared<test::InterfaceImpl>("lower ranked Interface1"),
                    {
                        {Constants::SERVICE_RANKING, Any(1)}
                    });

auto higherRankedSvc = bc.RegisterService<test::Interface1>(std::make_shared<test::InterfaceImpl>("higher ranked Interface1"),
                {
                    {Constants::SERVICE_RANKING, Any(10000)}
                });

auto midRankedSvc = bc.RegisterService<test::Interface1>(std::make_shared<test::InterfaceImpl>("mid ranked Interface1"),
                    {
                        {Constants::SERVICE_RANKING, Any(100)}
                    });
````

LocateService usage and result:

````c++
//LocateService API call
std::shared_ptr<test::Interface1> serviceObj = ctxt->LocateService<test::Interface1>("foo");

>> serviceObj will contain highest ranked service registered i.e. service with ranking 10000
````

### LocateServices API behavior:
For multiple cardinality, this returns the service objects for the specified service reference name and type. A vector of service objects for the referenced service or empty vector if the reference's minimum cardinality is 0 and no bound service is available.

LocateServices usage and result:
````c++
//Registering multiple services
auto lowerRankedSvc = bc.RegisterService<test::Interface1>(std::make_shared<test::InterfaceImpl>("lower ranked Interface1"),
                    {
                        {Constants::SERVICE_RANKING, Any(1)}
                    });

auto higherRankedSvc = bc.RegisterService<test::Interface1>(std::make_shared<test::InterfaceImpl>("higher ranked Interface1"),
                {
                    {Constants::SERVICE_RANKING, Any(10000)}
                });

auto midRankedSvc = bc.RegisterService<test::Interface1>(std::make_shared<test::InterfaceImpl>("mid ranked Interface1"),
                    {
                        {Constants::SERVICE_RANKING, Any(100)}
                    });

//LocateServices API usage
std::vector<std::shared_ptr<test::Interface1>> services = ctxt->LocateServices<test::Interface1>("foo");

>> services will contain vector of all the test::Interface1 references registered above -  lowerRankedSvc, higherRankedSvc and midRankedSvc
````
<br>

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing CppMicroServices patterns, or as a
wholly new one?

This is a continuation of existing CppMicroServices patterns/concepts. This introduces the concept of multiplicity to service reference cardinality in Declarative Services.<br><br>

> Would the acceptance of this proposal mean the CppMicroServices guides must be
re-organized or altered? Does it change how CppMicroServices is taught to new users
at any level?

The guides will need to be altered to add guidance on when and how to use multiple cardinality.<br><br>

> How should this feature be introduced and taught to existing CppMicroServices
users?

This should be introduced through the existing OSGi DS spec and Declarative Services spec: https://github.com/CppMicroServices/rfcs/blob/master/text/0003-declarative-services.md. It can be taught through user guides on using service references in Declarative Services.<br><br>

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching CppMicroServices,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

No such drawbacks have been identified with this feature.