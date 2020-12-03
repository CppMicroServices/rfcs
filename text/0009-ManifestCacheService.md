- Start Date: 2020-12-03
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: (fill me with the URL of the CppMicroServices issue that necessisated this RFC, otherwise leave blank)

# Bundle Manifest Cache Service Interface

## Summary

A service interface for retrieving bundle manifests

## Motivation

In applications with services numbering in the hundreds or thousands, the act of reading manifests from bundles can add significant overhead. By providing an implementation for this interface, an application may adopt a strategy for more quickly retrieving these manifests during the install operation.

## Detailed design

### Interface

```c++
/**
 * Service interface for allowing users of cppmicroservices to inject manifests for bundles
 * instead of having them read from the bundle files directly. 
 *
 * If an implementation is not provided, or if the manifest returned is empty, the manifest
 * is read from the bundle.
 */
class ManifestCacheService
{
  public:
  
  /**
   * @param  bundleLocation the location of the bundle for which to retrieve the manifest.
   *         the value of location is the same as that passed in to
   *         BundleContext::InstallBundles().
   * @return an r-value reference to an AnyMap suitable for moving into the 
   *         BundleRegistry, either populated, or if not present in the cache,
   *         empty.
   */
  AnyMap&& manifestForBundle(std::string const& bundleLocation) noexcept = 0;
};
```

###  Integration

This interface provdes a mechanism for users of CppMicroServices to manage bundle manifests outside of the bundles themselves.

* Remove manifest argument from the call stack:

  * Remove optional manifest argument from BundleContext::InstallBundles
    * BundleRegistry::Install
      * BundlRegistry::Install0

* Update BundleRegistry::Install0

  * Find implementation of ManifestCacheService

    * If one exists, retrieve manifest to use
    * otherwise use empty manifest

  * Pass manifest to CreateAndInsertArchive() as before.

    

```c++
AnyMap&& BundleRegistry::FetchBundleManifest(std::string const& location)
{
  auto cacheService = /* ... fetch from coreCtx */;
  if (cacheService) {
    return cacheService.ManifestForBundle(location);
  }
  return AnyMap { any_map::UNORDERED_MAP_CASEINSENSITIVE_KEYS };
}

std::vector<Bundle> BundleRegistry::Install0(
  const std::string& location,
  const std::shared_ptr<BundleResourceContainer>& resCont,
  const std::vector<std::string>& alreadyInstalled)
{
...
        auto manifest = FetchBundleManifest(location);

        // Now, create a BundleArchive with the given manifest at 'entry' in the
        // BundleResourceContainer, and remember the created BundleArchive here for later
        // processing, including purging any items created if an exception is thrown.
        barchives.push_back(coreCtx->storage->CreateAndInsertArchive(resCont, 
                                                                     symbolicName, 
                                                                     manifest));
...
  return installedBundles;
}

```



A manifest from the cache is used only if a non-empty manifest is returned. Otherwise, the manifest is read from the bundle.

## How we teach this

Add documentation and an example implementation.

## Drawbacks

It is up to the implementor to manage the consistency of manifests available in the service with those in the bundles themselves. The behavior is undefined should a manifest provided for a bundle not meet the expectations of the bundle implementation.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?