- Start Date: 2020-12-03
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: (fill me with the URL of the CppMicroServices issue that necessisated this RFC, otherwise leave blank)

# Bundle Manifest Cache Service Interface

## Summary

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
   * @return an optional reference to an AnyMap containing the manifest for the bundleLocation,
   */
  std::optional<std::reference_wrapper<AnyMap>> 
    manifestForBundle(std::string const& bundleLocation) noexcept = 0;
};
```

###  Integration

This interface provdes a mechanism for users of CppMicroServices to manage bundle manifests outside of the bundles themselves. Within the BundleRegistry code, if a ManifestCacheService is found, query the service for a manifest to use for the location being installed and use it if found. This means that there are several ways for a manifest to be specified for installation in the BundleRegistry. These different ways imply a precedence hierarchy for specifying the manifest:

* Highest precedence: BundleContext::InstallBundles() - a manifest provided as an argument here is always used.
* ManifestCacheService::manifestForBundle - if no manifest is provided as an argument, the service is consulted and if a manifest is returned, it is used
* Lowest precedence: manifest embedded within bundle - if a manifest is not provided as an argument, nor returned from the cache service, the manifest is read out of the bundle



```c++
/* From BundleContext */
std::vector<Bundle> BundleContext::InstallBundles(
  const std::string& location,
  const cppmicroservices::AnyMap& bundleManifest)
{
...
  return b->coreCtx->bundleRegistry.Install(location, b, bundleManifest);
}

/* From BundleRegistry */

AnyMap Bundleregistry::GetOverrideManifest(std::string const& location,
                                           AnyMap const& bundleManifest) 
{
    if (!bundleManifest.empty())
      return bundleManifest; // this will make a copy which can be moved into the registry
    
    auto emptyManifest = AnyMap { any_map::UNORDERED_MAP_CASEINSENSITIVE_KEYS) };
    auto manifestCacheService = /* ... fetch from coreCtx */;
    if (manifestCacheService) {
      auto manifest = manifestCacheService->manifestForBundle(location);
      
      /* an empty map forces the manifest to be read from the bundle */
      return manifest.value_or(emptyManifest);
    }
    
    /* an empty map forces the manifest to be read from the bundle */
    return emptyManifest;
}

std::vector<Bundle> BundleRegistry::Install(std::string const& location,
  		                                      BundlePrivate*,
                                            cppmicroservices::AnyMap const& bundleManifest)
{
...
  auto overrideManifest = GetOverrideManifest(location, bundleManifest);
...
  
  // Populate the resultingBundles and alreadyInstalled vectors with the appropriate data
  // based on what bundles are already installed
  auto resCont = GetAlreadyInstalledBundlesAtLocation(bundlesAtLocationRange,
                                                      location,
                                                      overrideManifest,
                                                      resultingBundles,
                                                      alreadyInstalled);

  // Perform the install, which will move the overrideManifest into the registry if it's not empty, 
  // or read the manifest out of the bundle otherwise.
  auto newBundles = Install0(location, resCont, alreadyInstalled, overrideManifest);
...
  
}

```

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