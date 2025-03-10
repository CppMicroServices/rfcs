- Start Date: 2024-07-01
- RFC PR: https://github.com/CppMicroServices/CppMicroServices/commit/24

# Dynamic Targeting

## Summary

In Declarative Services, we support two methods of dynamic targeting using Configuration Admin for factory component instances:
 - Method 1: manual injection into the config using the key value pairs: [refName.target, (someLDAP=filter)]
 - Method 2: reference to properties within the configuration using string replace in the target within manifest:
    - references [{
        name: "someName",
        interface: "someInterface",
        target: "(someKey={{someValueFromConfig}})
    }]

## Motivation

Users want to be able to inject properties into their targets at the time of configuration injection. These targets can be determined at run time. By allowing dynamic construction of targets using only what is in the configuration injected to create the factory instance, users are able to dynamically configure services without needing an intermediary.

## Detailed design
### Functional Design

Prior to creation of a factory instance when the required config is injected, Declarative Services parses the configuration properties and the targets.

If refName.target is a property in the config, that reference's target property is replaced with the value from the config. 

If the reference target (from the manifest) uses the "{{keyValue}}" syntax, the value in the injected config at that key replaces the {{keyValue}} within that reference's target. 

If both methods are provided by the reference (ie the original target from the manifest uses {{keyValue}} syntax AND refName.target is in the config) then the refName.target from the config is used. This behavior prioritizes the OSGI spec, which documents the refName.target mechanism. 

If any of the following conditions are met, the creation of the factory instance will throw:
 - the resulting LDAP filter from either Method 1 or 2 is an invalid filter
 - the key specified in Method 2 does not exist in the injected config

### Architecturally Significant Design Case
#### Method 1: direct injection of refName.target into Configuration

##### Manifest.json
```
{
    "implementation-class": "sample::someService",
    "configuration-policy" : "require",
    "configuration-pid" : ["servicePid"],
    "factory" : "someService Factory",
    "references":[{
        "name": "ServiceB",
        "interface" : "test::ServiceBInt",
        "target": "(ServiceBId=serviceBIDValue)"
    }],
    "service": {
        "interfaces": ["test::ServiceAInt"]
    }
},
```

##### Injected Configuration with configuration-pid: servicePid

The configuration contains "serviceB.target" as a property.
```
{
    "serviceB.target": "(totallyOtherTarget=SomeNewIDValueFromConfig)"
}
```

##### Resulting Factory Instance Configuration

As a result, the reference with name "ServiceB" is directly replaced with the value from the config.
```
{
    "implementation-class": "sample::someService",
    "references":[{
        "name": "ServiceB",
        "interface" : "test::ServiceBInt",
        "target": "(totallyOtherTarget=SomeNewIDValueFromConfig)"
    }],
    "service": {
        "interfaces": ["test::ServiceAInt"]
    }
},
```
#### Method 2: Replacement of values in target expression from configuration

##### Manifest.json

The target for reference "ServiceB" uses "{{someKeyFromConfig}}" syntax in its target expression.
```
{
    "implementation-class": "sample::someService",
    "configuration-policy" : "require",
    "configuration-pid" : ["servicePid"],
    "factory" : "someService Factory",
    "references":[{
        "name": "ServiceB",
        "interface" : "test::ServiceBInt",
        "target": "(ServiceBId={{someKeyFromConfig}})"
    }],
    "service": {
        "interfaces": ["test::ServiceAInt"]
    }
},
```

##### Injected Configuration with configuration-pid: servicePid

The same key value, "someKeyFromConfig", exists as a property in the configuration
```
{
    "someKeyFromConfig": "someValueFromConfig"
}
```

##### Resulting Factory Instance Configuration

As a result, that part of the target expression is replaced with the value from the config at that key
```
{
    "implementation-class": "sample::someService",
    "references":[{
        "name": "ServiceB",
        "interface" : "test::ServiceBInt",
        "target": "(ServiceBId=someValueFromConfig)"
    }],
    "service": {
        "interfaces": ["test::ServiceAInt"]
    }
},
```

#### Both methods used

If both of the above methods are used, <b>Method</b> 1 is prioritized
##### Manifest.json
```
{
    "implementation-class": "sample::someService",
    "configuration-policy" : "require",
    "configuration-pid" : ["servicePid"],
    "factory" : "someService Factory",
    "references":[{
        "name": "ServiceB",
        "interface" : "test::ServiceBInt",
        "target": "(ServiceBId={{someKeyFromConfig}})"
    }],
    "service": {
        "interfaces": ["test::ServiceAInt"]
    }
},
```

##### Injected Configuration with configuration-pid: servicePid
```
{
    "someKeyFromConfig": "someValueFromConfig",
    "serviceB.target": "(totallyOtherTarget=SomeNewIDValueFromConfig)"
}
```

##### Resulting Factory Instance Configuration
In this case we prioritize the refName.target in the configuration to align with the OSGI spec
```
{
    "implementation-class": "sample::someService",
    "references":[{
        "name": "ServiceB",
        "interface" : "test::ServiceBInt",
        "target": "(totallyOtherTarget=SomeNewIDValueFromConfig)"
    }],
    "service": {
        "interfaces": ["test::ServiceAInt"]
    }
},
```