- Start Date: 2026-11-20
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/30
- CppMicroServices Issue: (none; supersedes the approach in https://github.com/CppMicroServices/CppMicroServices/pull/1028)

# White-box unit tests for the core framework via a shared object library

## Summary

Compile the core framework sources once, into a CMake `OBJECT` library
(`CppMicroServicesObjs`). Link the shipping `CppMicroServices` library from
those objects, and link the same objects directly into a new test executable,
`usFrameworkUnitTests`. Tests in that executable can construct, inspect, and
mock framework-private classes (`CoreBundleContext`, `BundleRegistry`,
`BundleManifest`, `BundleArchive`, `Properties`, ...) without exporting any of
them from the shipping library. This is the pattern Declarative Services
(`DeclarativeServicesObjs`) and Configuration Admin (`ConfigurationAdminObjs`)
already use, applied to the framework. The existing `usFrameworkTests` suite
is unchanged and remains the black-box suite that runs against the shipping
library and real test bundles.

## Motivation

The framework's test suite (`usFrameworkTests`) can only reach the framework
through its exported, public API. That has two costs:

1. **Every internal scenario needs a real bundle.** To test, for example, how
   the manifest parser handles an invalid version or an empty bundle name, the
   project builds and installs a dedicated shared-library test bundle per case
   (`libMWithInvalidVersion`, `libMWithEmptyBundleName`,
   `libMWithoutBundleName`, `libWithDeepManifest`, ...). These are slow to
   build, awkward to write, and test the parser only indirectly.

2. **Internals cannot be mocked or observed.** Classes such as
   `CoreBundleContext`, `BundleRegistry`, `ServiceRegistry`, `BundleStorage`
   and the free function `GetPrivate(Bundle const&)` are not exported. A test
   cannot inject a fake `BundleStorage`, assert on the contents of the service
   registry, or drive `BundleManifest::Parse` with an in-memory string.

[PR #1028](https://github.com/CppMicroServices/CppMicroServices/pull/1028)
addressed this by introducing a `US_ABI_TEST` macro that, when tests are
enabled, expands to `US_ABI_EXPORT` on selected private classes, so that
`BundlePrivate`, `BundleRegistry`, `CoreBundleContext`, etc. are exported
from the `CppMicroServices` library itself. That works, but:

- the shipping binary's ABI depends on whether `US_BUILD_TESTING` is on, so
  the library that is tested is not the library that is shipped;
- every class a test wants to touch must be individually annotated in
  production headers;
- private implementation details become part of the library's exported
  surface in test builds, inviting accidental dependence on them.

The expected outcome of this RFC is a test executable with unrestricted
access to framework internals, while the shipping library exports exactly
the same symbols with or without testing enabled.

## Detailed design

### Build structure

```
framework/src/*.cpp ──► CppMicroServicesObjs (OBJECT library, compiled once)
                              │
               ┌──────────────┴───────────────┐
               ▼                              ▼
   CppMicroServices (shipping)        usFrameworkUnitTests (new)
   = $<TARGET_OBJECTS:...>            links CppMicroServicesObjs + util
     + util objs + resources          does NOT link CppMicroServices
               │
               ▼
   usFrameworkTests (existing, unchanged) + shared test bundles
```

**`framework/CMakeLists.txt`**

- `add_library(CppMicroServicesObjs OBJECT ${_framework_srcs} ...)` with
  `POSITION_INDEPENDENT_CODE ON`.
- Usage requirements on `CppMicroServicesObjs`:
  - `PUBLIC` include directories: the public include dirs plus the private
    `src/util`, `src/service` and `src/bundle` directories. Exposing private
    headers is intentional: the only consumers of the object library are the
    shipping library and white-box tests.
  - `PUBLIC` compile definition `CppMicroServices_EXPORTS` (see below).
  - `PRIVATE` definitions previously set on the shipping target:
    `US_BUNDLE_NAME=system_bundle`, `MINIZ_NO_ARCHIVE_WRITING_API`,
    `MINIZ_NO_ZLIB_COMPATIBLE_NAMES`.
  - `PUBLIC` link of the platform link libraries and `miniz::miniz` (a private
    header, `BundleResourceContainer.h`, includes `miniz.h`); `PRIVATE` link of
    `rapidjson::rapidjson`.
- `usMacroCreateBundle(Framework ... SOURCES $<TARGET_OBJECTS:CppMicroServicesObjs>)`.
  The shipping target keeps its existing private `miniz`/`rapidjson` link and
  the `--exclude-libs` handling, so nothing propagates to framework consumers.

There is no extra compilation: each framework source is compiled exactly once.

### The export macro

`Export.h` resolves `US_Framework_EXPORT` to `US_ABI_EXPORT` only when
`CppMicroServices_EXPORTS` is defined, and to `US_ABI_IMPORT` otherwise (in
shared builds). CMake defines `<target>_EXPORTS` automatically only for
sources of a `SHARED` target; object libraries do not get it. Therefore:

- the objects are compiled with `CppMicroServices_EXPORTS` (as they would have
  been as sources of the shared target), and
- the definition is a `PUBLIC` usage requirement, so the unit test executable,
  which includes the same public headers, sees the framework's symbols as
  local definitions rather than `dllimport`s.

In static builds `Export.h` ignores the macro (`US_Framework_EXPORT` is
always `US_ABI_EXPORT`), and no other code reads it.

### `usFrameworkUnitTests`

A new directory, `framework/test/unit/`, holds the white-box tests. Its
executable links `CppMicroServicesObjs`, `util`, gtest and gmock, and is
registered with `add_test` (label `regular`) plus the existing memcheck and
coverage variants, so it runs in CI wherever `usFrameworkTests` runs.

Ground rules, documented in the directory's `CMakeLists.txt`:

1. **Never link `CppMicroServices`.** The executable contains its own copy of
   the framework.
2. **Never install shared test bundles.** Shared test bundles link
   `CppMicroServices` themselves. Loading one would put a second copy of the
   framework into the process, with the bundle calling into the DLL copy while
   operating on objects created by the executable's copy. This only works by
   accident of the two copies being identical, and is an ODR violation. Tests
   use mocks or in-memory fakes instead; anything that genuinely needs real
   bundles belongs in `usFrameworkTests`.
3. **Do not embed the framework's resource zip.** The system bundle never has
   a `BundleArchive`; its identity and headers are constructed in code. In
   shared builds, `BundleRegistry::Install` only skips a `system_bundle` zip
   entry when `US_BUILD_SHARED_LIBS` is *not* defined, so embedding the zip
   causes `Framework::Start()` to throw ("a bundle with same symbolic name and
   version is already installed"). This was verified experimentally, and
   `SystemBundleUnitTest` guards against it.

Example: tests that are impossible against the shipping library today.

```cpp
#include "BundlePrivate.h"      // private header
#include "CoreBundleContext.h"  // private header
#include "mocks/MockBundleStorage.h"

TEST(CoreBundleContextUnitTest, PublicRegistrationIsVisibleInInternalServiceRegistry)
{
    auto framework = FrameworkFactory().NewFramework();
    framework.Start();
    auto* coreCtx = GetPrivate(framework)->coreCtx;   // not exported

    auto reg = framework.GetBundleContext().RegisterService<IUnitTestService>(
        std::make_shared<UnitTestService>());

    std::vector<ServiceRegistrationBase> regs;
    coreCtx->services.Get(us_service_interface_iid<IUnitTestService>(), regs);
    ASSERT_EQ(1u, regs.size());
    ...
}

TEST(BundleArchiveUnitTest, PurgeRemovesArchiveFromOwningStorage)
{
    StrictMock<test::MockBundleStorage> storage;      // mocks a private interface
    auto archive = MakeArchive(&storage, 7);
    EXPECT_CALL(storage, RemoveArchive(archive.get())).WillOnce(Return(true));
    archive->Purge();
}
```

### Exports from the test executable

Because the objects carry `dllexport` directives on Windows (they are the
objects of the shared library), linking them into the executable makes the
executable export the same symbols, and the linker emits an import
library (`usFrameworkUnitTests.lib`/`.exp`). This is harmless:

- the executable is not installed and nothing links against it;
- Windows binds imports by DLL name, so no DLL loaded later can bind to the
  executable's exports by accident;
- the executable does not define the `_us_*_main` bundle entry points, so the
  framework never treats it as a bundle;
- the existing static-build `usFrameworkTests.exe` already does the same
  (604 exports, plus a `.lib`/`.exp` pair).

### Mocking seams

Linking objects directly removes the *export* barrier; it does not by itself
create seams for mocking. Two techniques are available, and this RFC
recommends both, each where it fits:

- **Existing interfaces and injection.** Several internals are already
  abstract (`BundleStorage`) or can be driven directly (`BundleManifest`,
  `Properties`, `BundleArchive`). Where a class owns a concrete collaborator
  (for example `CoreBundleContext::bundleRegistry`), a minimal production
  change such as holding it by `std::unique_ptr` or adding a virtual may be
  justified case by case, as PR #1028 did.
- **Link-time substitution** for OS-facing code (`BundleUtils`,
  `SharedLibrary`, file-system helpers): a unit test target can be built from
  the object list *minus* one translation unit, with a fake implementation
  linked in its place. This requires no production changes, but costs a
  second compilation of the remaining sources for that target, so it should
  be reserved for the few places where virtual seams would be intrusive.

### Verification performed

Implementation:
[CppMicroServices#1297](https://github.com/CppMicroServices/CppMicroServices/pull/1297)
(branch `fwk-test-infra-refactor`, based on `development` at `2360b208`).

Local verification, Windows, MSVC 14.36, x64, Debug (against the earlier
base, `65fd6472`; the rebase only brought in CI configuration changes):

| Check | Shared (`BUILD_SHARED_LIBS=ON`) | Static (`BUILD_SHARED_LIBS=OFF`) |
|---|---|---|
| `usFrameworkUnitTests` (34 tests) | pass | pass |
| `usFrameworkTests` | 410/410 pass | 380/380 pass |
| Unit exe imports `CppMicroServices*.dll` | no (`dumpbin /DEPENDENTS`) | n/a |
| Shipping DLL exports vs. pristine `development` | identical (553 symbols) | n/a |

Upstream CI on the implementation PR (head `57dcf750`):

| Platform / job | Result |
|---|---|
| Linux: ubuntu-22.04 x64 and arm64 (Configurations 0-3), minimum GCC, supported Clang | pass |
| macOS: macos-15 and macos-latest (Configurations 0-3, plus deterministic bundles) | pass |
| Windows: windows-2022/2025 (all configurations, incl. coverage), windows-11-arm (Configurations 0-3), MinGW-w64 | pass |
| AddressSanitizer, UndefinedBehaviorSanitizer, CodeQL, MSVC Code Analysis | pass |
| codecov (patch and project) | pass |
| clang-tidy `review` (test sources analysed with the test-directory configs) | pass |
| ThreadSanitizer | `usFrameworkUnitTests` passes; the job fails on pre-existing races in `usFrameworkTests` and `usDeclarativeServicesTests`, the same failures as on `development` |

Three CI problems on `development` that had been failing this PR were fixed
separately before the rebase:

- [#1298](https://github.com/CppMicroServices/CppMicroServices/pull/1298)
  replaced the retired `macos-14` runner with `macos-latest`;
- [#1299](https://github.com/CppMicroServices/CppMicroServices/pull/1299)
  disabled a false-positive C4789 in MSVC 19.52 for ARM64, which `/WX` had
  turned into an error on `windows-11-arm`;
- [#1300](https://github.com/CppMicroServices/CppMicroServices/pull/1300)
  added test-directory `.clang-tidy` configs and test targets to the
  clang-tidy review's compile database, which had failed on
  `gtest/gtest.h` not found.

## How we teach this

This is a contributor-facing convention; users of CppMicroServices see no
change in API, ABI, packaging, or install layout.

- **Terminology:** "framework unit tests" (`framework/test/unit`,
  `usFrameworkUnitTests`) are white-box tests of framework internals;
  "framework tests" (`framework/test/gtest`, `usFrameworkTests`) are black-box
  tests of the shipping library through its public API and real bundles.
- **Placement rule for new tests:** if a test needs real bundles, a second
  framework instance loaded from disk, or must exercise exported-symbol
  behaviour, it goes in `usFrameworkTests`; if it tests the behaviour of an
  internal class or needs a mock, it goes in `usFrameworkUnitTests`.
- It is presented as a continuation of the existing compendium pattern
  (`DeclarativeServicesObjs`, `ConfigurationAdminObjs`), so contributors
  familiar with those suites already know it.
- The contributor documentation (testing section) should describe the two
  suites, the placement rule, and the three ground rules above.

## Drawbacks

- **Two framework copies are possible if the ground rules are broken.** A
  contributor who installs a shared test bundle from `usFrameworkUnitTests`
  gets subtly wrong behaviour rather than a build error. The ground rules are
  documented, and the absence of a `CppMicroServices` link dependency makes
  such bundles hard to wire up accidentally, but there is no hard guard.
- **Private headers become a de-facto test API.** White-box tests couple to
  implementation details and will need updating when internals are
  refactored. This is the usual trade-off of unit tests at this level.
- **Process-wide state is shared between framework instances in one test
  process.** For example `CoreBundleContext::globalId` and the
  `ServiceReferenceBasePrivate` thread-marks guard. This is already true of
  `usFrameworkTests`.
- **Test executable exports symbols on Windows**, with an import library
  side effect (see above). Harmless, but visible in the build tree.
- **`usMacroCreateBundle` for the framework now receives pre-compiled
  objects.** Compile settings for framework sources live on
  `CppMicroServicesObjs` rather than on the `CppMicroServices` target, and
  anyone adding framework compile definitions must put them there.

## Alternatives

1. **Export internals under `US_ABI_TEST` (PR #1028).** Discussed under
   Motivation. Rejected because it makes the shipping ABI depend on the test
   configuration and requires annotating production headers per class.

2. **A second, test-only framework library** (`CppMicroServicesTesting`,
   built from the same sources with internals exported). Keeps the shipping
   library clean, but compiles the framework twice and still requires per-class
   export annotations. Test bundles would also still link the real library,
   giving two framework copies whenever both are loaded.

3. **Run the whole framework suite against a static framework.** The project
   already supports `BUILD_SHARED_LIBS=OFF`, but that loses coverage of the
   shared-library behaviour (dynamic loading, symbol lookup, resource
   embedding in DLLs) that most users depend on.

4. **`friend` test classes or `#define private public`.** Neither solves the
   export problem on Windows; the latter is undefined behaviour and breaks
   with MSVC name mangling of access specifiers.

5. **Do nothing.** Keep adding a dedicated shared test bundle per internal
   scenario. Test-bundle count and build time keep growing, and internals
   remain unmockable.

Prior art: the DS and Configuration Admin suites in this repository use the
same object-library approach for their bundles' internals.

## Unresolved questions

- **ELF/Mach-O symbol tables.** CI shows the unit tests build and pass on
  Linux and macOS (see Verification performed), so
  `-fvisibility=hidden` plus `CppMicroServices_EXPORTS` in the test
  executable works there. The executable's dynamic symbol table has not
  been inspected directly; it is not linked with `-rdynamic`/
  `ENABLE_EXPORTS`, so it is not expected to interpose symbols for any
  library it loads.
- **Migration of PR #1028's tests.** Which of its mock-based tests
  (`BundleManifestTest`, `BundleTest`, `MockedEnvironment`) move into
  `usFrameworkUnitTests`, and which of its production changes (`BundleUtils`
  as a class, `unique_ptr<BundleRegistry>`, `BundleRegistry::Install1`) are
  still wanted once the export barrier is gone.
- **Retiring test bundles.** Once equivalent unit tests exist, should the
  manifest-only test bundles (`libMWithInvalidVersion`, etc.) be deleted, or
  kept as end-to-end coverage?
- **Guarding the ground rules.** Is a CMake check worth adding (for example,
  failing configuration if `usFrameworkUnitTests` acquires a dependency on
  `CppMicroServices`)?
- **Link-time substitution.** Whether to add a helper (for example a CMake
  function producing an object library minus named sources) now, or wait until
  the first test needs it.
