- Start Date: 2025-04-09
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/27
- CppMicroServices Issue: (none; implemented in https://github.com/CppMicroServices/CppMicroServices/pull/1050)

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
change_namespace --cppms_src /path/to/cppmicroservices/source --namespace new_namespace [--namespace-alias] /path/to/destination/directory
```

The tool accepts the following arguments:

- `--cppms_src <DIR>`: Directory containing the CppMicroServices source tree to process [REQUIRED]
- `--namespace <NAME>`: New namespace to use as replacement for the default `cppmicroservices` namespace [REQUIRED]
- `--namespace-alias`: Optional boolean flag (it takes no value). When present, `cppmicroservices` is additionally declared as an alias of the new namespace (see below) [OPTIONAL]
- `<destination>`: Positional argument; directory where the output files with the replaced namespace are written. The directory must already exist; the tool does not create it [REQUIRED]
- `-h`, `--help`: Print usage and exit

If a required argument is missing, is empty, or its value begins with `-`, the tool prints an error and the usage text and exits with a non-zero status.

If the tool is invoked with no arguments at all, it prints a `Just one argument` line followed by the usage text, and exits with status 0.

> **Note:** printing a debug line and exiting with status 0 when no arguments are given appears to be a bug; see [CppMicroServices#1302](https://github.com/CppMicroServices/CppMicroServices/issues/1302).

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
1. Recursively walk the entire source tree given by `--cppms_src`, excluding the tool's own directory (`tools/change_namespace`), and copy every file into the same relative path under the destination directory, creating subdirectories as needed. If a destination file already exists, it is overwritten.
2. Rewrite the contents of files with one of the following kinds of name:
   - Source files with extensions `.c`, `.cxx`, `.h`, `.hxx`, `.inc`, `.inl`, `.y`, `.yy`, `.pp`, and any `.?pp` extension (one character followed by `pp`, for example `.cpp`, `.hpp`, `.ipp`, `.tpp`); matching is case-insensitive.
   - Test config files (`*.h.in`).

   In these files the tool replaces:
   - namespace declarations, including `using namespace cppmicroservices`;
   - namespace alias definitions (`namespace X = cppmicroservices`);
   - qualified uses (`cppmicroservices::...`), including as template arguments (`<cppmicroservices::...`);
   - macro definitions (`#define X cppmicroservices`);
   - parenthesized uses (`(cppmicroservices)`).

   Suffixed namespaces such as `cppmicroservices_foo` become `<new>_foo`. Unqualified, bare `cppmicroservices` tokens in other positions are not replaced.
3. Rewrite all `*.json` files (not only `manifest.json`): only string values of the form `"cppmicroservices::..."` are changed.
4. Copy all other files (including CMake files) unchanged.
5. Leave include paths, file and directory names, and CMake target names unchanged, since the directory structure is not changed.
6. Preserve code formatting. Occurrences of `cppmicroservices::` inside comments and string literals are also rewritten.

The replacement is regex-based, not parser-based. Because the patterns have no leading word boundary, `cppmicroservices` is also replaced when it is part of a longer identifier: for example `mycppmicroservices::Bar` becomes `mycustomNamespace::Bar`, and `namespace cppmicroservicesExtra` becomes `namespace customNamespaceExtra`.

> **Note:** replacing `cppmicroservices` inside longer identifiers appears to be a bug; see [CppMicroServices#1303](https://github.com/CppMicroServices/CppMicroServices/issues/1303).

Resulting files after replacement are copied to the user specified destination directory. This output directory will thus contain new CppMicroServices source with new custom namespace.

If user provides the `--namespace-alias` flag, the tool will also define an alias for the cppmicroservices namespace. This is done for every `namespace customNamespace {` opening in the rewritten source files:
```cpp
namespace customNamespace {} namespace cppmicroservices = customNamespace; namespace customNamespace
{
....namespace elements.....
}
```

#### Error Cases

Following conditions lead to an error message and a non-zero exit status:
- If the `--cppms_src` argument is missing or empty, or the directory does not exist
- If the `--namespace` argument is missing or empty
- If the destination argument is missing or empty, or the directory does not exist
- If an argument value begins with `-` (for example a missing value followed by another option)

The "does not exist" errors for the source and destination directories are printed to standard output.

Copy and write failures are not currently handled:
- A filesystem error while removing, creating or copying a file throws an exception that is not caught, which terminates the tool.
- Errors while writing a rewritten file are not detected, and the tool still exits with status 0.

> **Note:** the missing handling of copy and write failures appears to be a bug; see [CppMicroServices#1302](https://github.com/CppMicroServices/CppMicroServices/issues/1302).

### Build, install and tests

- The tool is built as the executable target `change_namespace` whenever `tools/change_namespace` exists in the source tree; there is no option to disable it. It depends on CLI11.
- It is installed into the tools install directory (`TOOLS_INSTALL_DIR`) unless `US_NO_INSTALL` is set.
- When `US_BUILD_TESTING` is enabled, the GTest suite `ChangeNamespaceTests` (`tools/change_namespace/test`) is built and registered with CTest.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing CppMicroServices patterns, or as a
wholly new one?

The namespace replacement tool can be introduced as a new utility that complements the existing CppMicroServices framework. It should be presented as a way to manage namespace conflicts when using multiple versions of the CppMicroServices library, without requiring changes to the core CppMicroServices API or usage patterns from user end. The tool does the part of replacing namespaces.

> Would the acceptance of this proposal mean the CppMicroServices guides must be
re-organized or altered? Does it change how CppMicroServices is taught to new users
at any level?

We will have to add guidance on how to use this namespace replacement tool in the CppMicroServices documentation and guides. This documentation has not been written yet; the tool is currently only mentioned in the changelog.

It does not change how CppMicroServices is taught to new users, but provides an additional tool to help manage namespace conflicts when using multiple versions of the CppMicroServices library.

> How should this feature be introduced and taught to existing CppMicroServices
users?

This is a simple command line tool that users can try out to replace the default `cppmicroservices` namespace in their CppMicroServices source code with a custom namespace. It can be taught through how to invoke the tool and explain the result expected.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching CppMicroServices,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

The replacement is regex-based rather than parser-based. As a result it:
- can replace `cppmicroservices` inside longer identifiers (see [CppMicroServices#1303](https://github.com/CppMicroServices/CppMicroServices/issues/1303));
- rewrites `cppmicroservices::` inside comments and string literals;
- does not replace unqualified, bare `cppmicroservices` tokens outside the supported patterns.
