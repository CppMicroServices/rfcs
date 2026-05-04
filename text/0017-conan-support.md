- Start Date: 2026-05-03
- RFC PR:
- CppMicroServices Issue:

# Conan Package Support

## Summary

Add the ability to build CppMicroServices using externally-provided (e.g. Conan-managed) dependencies in place of the vendored copies under `third_party/`, while preserving the existing self-contained build as the default. This is achieved through a set of `US_USE_SYSTEM_*` CMake options that switch each vendored dependency between an internal IMPORTED target and a `find_package()`-resolved external one, plus a Conan recipe that exercises those options to produce a redistributable package.

## Motivation

CppMicroServices currently vendors all of its third-party dependencies under `third_party/`. This makes the initial build experience simple — clone and build with no external setup — but creates friction in several scenarios:

- **Distribution as a pre-built package.** Organizations that manage C++ dependencies through a package manager (Conan, vcpkg, system packages) cannot easily consume CppMicroServices without also pulling in its vendored copies, which may conflict with versions already present in the dependency graph.
- **Version management.** Vendored copies become stale over time. Updating them requires manual intervention and carries the risk of subtle ABI or behavioral mismatches with other copies of the same library in the same process.
- **Reproducible builds.** Package managers provide lockfiles and dependency resolution that make builds reproducible across machines and CI environments. The vendored approach relies on the snapshot checked into the repo.
- **Downstream integration.** When CppMicroServices is one node in a larger Conan dependency graph, its transitive dependencies (miniz, spdlog, etc.) must be visible to the resolver so that version conflicts are detected early rather than at link time.

The goal is to support Conan as a first-class distribution channel without breaking the existing build workflow. Users who do not use Conan should notice no change.

## Detailed design

### CMake option matrix

Seven `US_USE_SYSTEM_*` cache variables control whether each dependency is resolved externally or from `third_party/`. All default to `OFF`.

| Option | Dependency | Used by | `find_package()` call when ON |
|---|---|---|---|
| `US_USE_SYSTEM_BOOST` | Boost (Nowide) | `rc` tool, framework | `Boost 1.74.0 REQUIRED` |
| `US_USE_SYSTEM_SPDLOG` | spdlog | LogServiceImpl | `spdlog REQUIRED` |
| `US_USE_SYSTEM_RAPIDJSON` | RapidJSON | framework, jsonschemavalidator | `rapidjson REQUIRED` |
| `US_USE_SYSTEM_JSONCPP` | jsoncpp | `rc` tool, SCRCodeGen | `jsoncpp REQUIRED` |
| `US_USE_SYSTEM_MINIZ` | miniz | framework, `rc` tool | `miniz REQUIRED` |
| `US_USE_SYSTEM_CLI11` | CLI11 | `rc`, change\_namespace, jsonschemavalidator | `CLI11 REQUIRED` |
| `US_USE_SYSTEM_GTEST` | Google Test | test suite | *(pre-existing, unchanged)* |

`US_USE_SYSTEM_BOOST` and `US_USE_SYSTEM_GTEST` existed before this change. `US_USE_SYSTEM_BOOST` is updated to handle the standalone-to-Boost.Nowide transition (see [Platform-specific concerns](#platform-specific-concerns-boostnowide)).

### The IMPORTED target pattern

Every consumer in the build tree links against a canonical namespaced CMake target (e.g. `miniz::miniz`). The root `CMakeLists.txt` ensures that target exists regardless of which path is active:

```cmake
if(US_USE_SYSTEM_MINIZ)
    find_package(miniz REQUIRED)
else()
    add_library(miniz::miniz INTERFACE IMPORTED)
    target_include_directories(miniz::miniz INTERFACE
        ${CMAKE_CURRENT_SOURCE_DIR}/third_party)
endif()
```

When the option is **OFF**, a header-only IMPORTED INTERFACE target is created pointing at the vendored source. When **ON**, `find_package()` locates the external installation, which already provides the same target name. Some dependencies (jsoncpp, rapidjson) need a fallback alias when the upstream config file uses a different target name:

```cmake
if(US_USE_SYSTEM_JSONCPP)
    find_package(jsoncpp REQUIRED)
    if(NOT TARGET jsoncpp::jsoncpp AND TARGET JsonCpp::JsonCpp)
        add_library(jsoncpp::jsoncpp ALIAS JsonCpp::JsonCpp)
    endif()
else()
    # ... vendored path
endif()
```

Downstream `CMakeLists.txt` files are updated to use `target_link_libraries()` against these targets instead of raw `include_directories()` calls. For dependencies that were previously compiled directly into their consumer (miniz compiled into the framework, jsoncpp compiled into `rc` and SCRCodeGen), the source file is conditionally excluded when the system version is used:

```cmake
if(NOT US_USE_SYSTEM_MINIZ)
    list(APPEND _srcs ../../third_party/miniz.c)
endif()
```

### Compatibility shims

**Boost.Nowide namespace alias.** The standalone `nowide` library (vendored) and Boost.Nowide use different header paths and namespaces. A compile definition `US_HAVE_BOOST_NOWIDE` is set when the system Boost path is active. `ResourceCompiler.cpp` uses this to select the correct headers and create a namespace alias:

```cpp
#ifdef US_HAVE_BOOST_NOWIDE
#    include <boost/nowide/args.hpp>
#    include <boost/nowide/fstream.hpp>
namespace us_nowide = boost::nowide;
#else
#    include <nowide/args.hpp>
#    include <nowide/fstream.hpp>
namespace us_nowide = nowide;
#endif
```

All call sites use `us_nowide::` instead of a hardcoded namespace.

**CLI11 forwarding header.** Conan's CLI11 package provides `<CLI/CLI.hpp>`, while the vendored copy ships as `<CLI/CLI11.hpp>`. A thin forwarding header at `third_party/CLI/CLI.hpp` bridges the gap:

```cpp
// Forwarding header — works whether CLI11 comes from Conan or third_party/.
#include "CLI11.hpp"
```

Source files are updated to `#include "CLI/CLI.hpp"` uniformly.

### `CppMicroServicesHelpers.cmake`

When CppMicroServices is consumed as an installed package (whether via Conan or a plain `cmake --install`), consumers need access to the bundle-authoring API: `usFunctionEmbedResources`, `usFunctionAddResources`, the resource compiler target, and code-generation template paths. The upstream `CppMicroServicesConfig.cmake.in` provides this for direct CMake installs.

For Conan, a new `CppMicroServicesHelpers.cmake` module is installed alongside the existing CMake scripts and included via Conan's `cmake_build_modules` mechanism. It:

1. Resolves template paths (`US_BUNDLE_INIT_TEMPLATE`, `US_CMAKE_RESOURCE_DEPENDENCIES_CPP`, `US_RESOURCE_RC_TEMPLATE`) relative to its own install location.
2. Creates an imported `usResourceCompiler` executable target pointing at the installed `usResourceCompiler3` binary.
3. Includes all `usFunction*.cmake` helper scripts.
4. Sets the `US_LIBRARIES` convenience variable.

All paths are derived from `CMAKE_CURRENT_LIST_DIR`, making the module fully relocatable across Conan cache layouts.

### Platform-specific concerns: Boost.Nowide

Boost.Nowide is header-only on Unix but has compiled sources on Windows (it wraps Win32 console I/O). The `US_USE_SYSTEM_BOOST` path handles this:

```cmake
if(US_USE_SYSTEM_BOOST)
    if(WIN32)
        find_package(Boost 1.74.0 REQUIRED)
        add_library(nowide::nowide INTERFACE IMPORTED GLOBAL)
        target_include_directories(nowide::nowide INTERFACE ${Boost_INCLUDE_DIRS})
        target_compile_definitions(nowide::nowide INTERFACE US_HAVE_BOOST_NOWIDE)
        target_link_libraries(nowide::nowide INTERFACE boost::boost)
    else()
        find_package(Boost 1.74.0 REQUIRED)
        add_library(nowide::nowide INTERFACE IMPORTED GLOBAL)
        target_include_directories(nowide::nowide INTERFACE ${Boost_INCLUDE_DIRS})
        target_compile_definitions(nowide::nowide INTERFACE US_HAVE_BOOST_NOWIDE)
    endif()
endif()
```

On Windows the wrapper links `boost::boost` to pull in the compiled Nowide library. On Unix only the include path is needed.

### Conan recipe overview

The Conan recipe (`conanfile.py`) lives in the conan-center-index repository and is the primary consumer of the `US_USE_SYSTEM_*` options. Key design decisions:

**Dependencies and visibility.** Each dependency's Conan `visible` flag controls whether it appears in the consumer's dependency graph:

| Dependency | Version | `visible` (shared) | `visible` (static) | Rationale |
|---|---|---|---|---|
| Boost | 1.86.0 | `False` | `False` | Build-tool only (`rc`, codegen) |
| CLI11 | 2.4.1 | `False` | `False` | Build-tool only |
| miniz | 3.0.2 | `False` | `True` | Baked into shared lib; must link for static |
| spdlog | 1.14.1 | `False` | `True` | Same |
| jsoncpp | 1.9.5 | `False` | `True` | Same |
| RapidJSON | cci.20220822 | `False` | `True` | Same |

For shared builds, these implementation dependencies are statically linked into the CppMicroServices shared libraries, so consumers don't need them. For static builds, consumers must link them directly.

**Target name alignment.** The recipe uses `CMakeDeps.set_property()` to ensure Conan-generated config files produce the same target names the upstream CMake expects:

```python
deps.set_property("cli11",     "cmake_target_name", "CLI11::CLI11")
deps.set_property("miniz",     "cmake_target_name", "miniz::miniz")
# ... etc.
```

**Component mapping.** `package_info()` defines three component groups:

- **`framework`** — always present. Maps to the `CppMicroServices` CMake target.
- **`logservice`** — always present (header-only). Maps to `usLogService`.
- **Compendium bundles** — only when `shared=True` and `with_threading=True`. Maps DeclarativeServices, ConfigurationAdmin, AsyncWorkService, etc. to their respective CMake targets.

**Boost configuration.** On Unix, Boost is configured as header-only. On Windows, only the `nowide`, `filesystem`, `atomic`, and `system` libraries are built; all others are explicitly disabled to minimize build time.

### Constraints and caveats

**Deterministic builds.** `US_USE_DETERMINISTIC_BUNDLE_BUILDS` requires the bundled miniz because it depends on the `MINIZ_NO_TIME` compile definition being set during miniz compilation. When `US_USE_SYSTEM_MINIZ=ON`, the build emits a warning:

```
US_USE_DETERMINISTIC_BUNDLE_BUILDS requires bundled miniz (MINIZ_NO_TIME).
Set US_USE_SYSTEM_MINIZ=OFF or disable deterministic builds.
```

**Options are independent.** Each `US_USE_SYSTEM_*` option can be toggled individually. A user could use system spdlog but vendored miniz. The Conan recipe sets them all to ON, but other integration scenarios may mix.

## How we teach this

**For existing users: nothing changes.** All `US_USE_SYSTEM_*` options default to OFF. The vendored `third_party/` build continues to work exactly as before. No documentation, tutorial, or build instruction needs updating for the default path.

**For Conan users:** The primary entry point is `conan install cppmicroservices/3.8.10` (or adding it to a `conanfile.txt`/`conanfile.py`). Consumption is standard Conan:

```cmake
find_package(CppMicroServices REQUIRED)

add_executable(myapp main.cpp)
target_link_libraries(myapp PRIVATE CppMicroServices)

# Bundle authoring works out of the box:
usFunctionEmbedResources(TARGET myapp BUNDLE_NAME myapp FILES manifest.json)
```

The `CppMicroServicesHelpers.cmake` module is automatically included by Conan's generated config, so `usFunctionEmbedResources` and the resource compiler are available without additional setup.

**For integrators using system packages without Conan:** The `US_USE_SYSTEM_*` options can be set manually. This is an advanced use case and should be documented in the project's build instructions with a note that the options exist and what they control.

**Terminology:** No new concepts are introduced. "System dependency" and "vendored dependency" are well-understood terms in the C++ ecosystem.

## Drawbacks

- **Added CMake complexity.** The root `CMakeLists.txt` grows by ~60 lines of if/else blocks. Each dependency has two code paths that must be kept consistent. This is the cost of supporting both vendored and external dependencies simultaneously.
- **Version drift risk.** The vendored copies and the versions specified in the Conan recipe may diverge over time. A bug fixed in vendored miniz might not be present in the Conan-specified version, or vice versa. The mitigation is to keep the Conan recipe's version pins reasonably close to what is vendored.
- **Testing surface.** The matrix of ON/OFF combinations across seven options is large (128 combinations). In practice, only two configurations matter: all OFF (legacy) and all ON (Conan). CI should cover both.
- **Conan-specific shims.** `CppMicroServicesHelpers.cmake` and the target name alignment in the recipe are Conan-specific concerns that live partly in the upstream repo and partly in conan-center-index. Changes to the CMake install layout require coordinated updates.

## Alternatives

- **vcpkg.** vcpkg is another widely-used C++ package manager. The `US_USE_SYSTEM_*` mechanism is package-manager-agnostic — it would work with vcpkg's `find_package()` integration as well. However, the Conan recipe and `CppMicroServicesHelpers.cmake` are Conan-specific. A vcpkg port could be added later using the same CMake options. Conan was chosen first due to existing organizational familiarity.
- **CMake FetchContent.** FetchContent could download and build dependencies at configure time, avoiding vendoring. However, it doesn't solve the distribution problem (consumers still need to fetch sources), doesn't provide binary caching, and introduces configure-time network dependencies.
- **Status quo (vendored only).** Continuing to vendor everything is the simplest option but blocks distribution through package managers and forces all consumers to use the specific vendored versions.
- **Remove vendored copies entirely.** This would simplify the build system (no if/else) but would break the zero-dependency build experience that is a key feature of the project.

## Unresolved questions

- **Minimum versions for system dependencies.** The Conan recipe pins specific versions (e.g. miniz 3.0.2, spdlog 1.14.1). The CMake `find_package()` calls do not currently enforce minimum versions (except Boost >= 1.74.0). Should minimum version constraints be added to the `find_package()` calls for non-Conan users?
- **Static build testing.** The Conan recipe supports `shared=False`, and the `package_info()` correctly propagates transitive dependencies for static builds. However, the static build path through Conan has received less testing than the shared path. This should be validated before release.
- **`US_USE_DETERMINISTIC_BUNDLE_BUILDS` and system miniz.** Currently this combination emits a warning. Should it be a hard error instead?
- **Conan Center Index submission.** The recipe currently uses a Git clone from the `conan-support` branch for development. Before submission to conan-center-index, it needs to switch to tarball-based source retrieval from a tagged release.
