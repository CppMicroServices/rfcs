- Start Date: 2025-04-09
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue:

# Tool for replacing default namespace in CppMicroServices

## Summary

> This RFC documents a tool for replacing the default namespace in CppMicroServices sources. It will allow developers to easily change the default `cppmicroservices` namespace used throughout their CppMicroServices source code. This enables them to use multiple versions of CppMicroServices libraries without symbol conflicts.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

Currently, if customer environment has multiple versions of CppMicroServices libraries, there can be symbol conflicts due to the default `cppmicroservices` namespace used throughout the source code. The proposed tool will allow developers to easily change the default namespace to a custom one, enabling them to use multiple versions of CppMicroServices libraries without such conflicts.

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

### How to invoke the tool?

The namespace replacement tool can be invoked via command line:

```bash
change_namespace --cppms_src /path/to/cppmicroservices/source --namespace new_namespace --namespace-alias /path/to/destination/directory
```

The tool accepts the following arguments:

- `--cppms_src`: Directory containing CppMicroServices source files to process [REQUIRED]
- `--new-namespace`: New namespace to use as replacement for default: cppmicroservices [REQUIRED]
- `--namespace-alias`: Optional argument: makes namespace cppmicroservices an alias of the namespace set with --namespace [OPTIONAL]
- `<output_dir>`: Directory where output files with replaced namespace will be written [REQUIRED]

Tool will throw error if any of the required arguments are missing.

Example usage replacing the default namespace with a custom one:

```cpp:example.cpp
// Before
namespace cppmicroservices {
  class Bundle { /*...*/ };
}

// After running the tool
namespace customNamespace {
  class Bundle { /*...*/ };
}
```

The tool will:
1. Recursively scan all C++ files (.hpp/.h/.cpp), test config files (.h.in) and manifest.json files in the input directory
2. Replace namespace declarations, using-directives, namespace used in function/class calls, template arguments, etc. 
3. include paths are not updated as we are not changing the directory structure
4. Preserve code formatting and comments

Resulting files after replacement are copied to the user specified destination directory. This output directory will thus contain new CppMicroServices source with new custom namespace.

If user provides the --namespace-alias argument, the tool will also defines an alias for the cppmicroservices namespace:
```cpp
namespace customNamespace {} namespace cppmicroservices = customNamespace; namespace customNamespace
{
....namespace elements.....
}
```

#### Error Cases

Following conditions can lead to errors:
- If the `--cppms_src` directory does not exist or is not accessible
- If the `--namespace` argument is not provided
- If the `<output_dir>` argument is not provided
- If invalid arguments are provided (e.g. empty strings, non-existent directories, etc.)
- If the copy operations fail while copying the modified files to the output directory



## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing CppMicroServices patterns, or as a
wholly new one?

The namespace replacement tool can be introduced as a new utility that complements the existing CppMicroServices framework. It should be presented as a way to manage namespace conflicts when using multiple versions of the CppMicroServices library, without requiring changes to the core CppMicroServices API or usage patterns from user end. The tool does the part of replacing namespaces.

> Would the acceptance of this proposal mean the CppMicroServices guides must be
re-organized or altered? Does it change how CppMicroServices is taught to new users
at any level?

We will have to add guidance on how to use this namespace replacement tool in the CppMicroServices documentation and guides. 

It does not change how CppMicroServices is taught to new users, but provides an additional tool to help manage namespace conflicts when using multiple versions of the CppMicroServices library.

> How should this feature be introduced and taught to existing CppMicroServices
users?

This is a simple command line tool that users can try out to replace the default `cppmicroservices` namespace in their CppMicroServices source code with a custom namespace. It can be taught through how to invoke the tool and explain the result expected.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching CppMicroServices,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

No such drawbacks have been identified.

