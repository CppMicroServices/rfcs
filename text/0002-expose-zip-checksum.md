- Start Date: 2018-10-16
- RFC PR: https://github.com/CppMicroServices/CppMicroServices/pull/308
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/307

# Expose Zip Checksum

## Summary

Allow clients of CppMicroServices to query the crc32 checksum for each bundle resource file.

## Motivation

My team is using CppMicroServices in deployment where DLLs may be loaded, unloaded, and reloaded through the process lifetime. 

We would like to utilize a checksum to know if a bundle has changed from a prior load.  
Other attributes of the bundle resources such as compressed and uncompressed size are already exposted to clients.

Given this and that there is already a crc32 checksum stored in the zip-data, it seems natural to expose the checksum to clients.

## Detailed design

```miniz.c``` provides ```struct mz_zip_archive_file_stat```, which has a ```mz_uint32 m_crc32;``` member and is always set in ```mz_zip_reader_file_stat```.
- Add a new private member to 'BundleResourceContainer::Stat'; ```uint32_t crc32```.
- The member would be assigned by ```BundleResourceContainer::GetStat```, which already has the value inside ```struct mz_zip_archive_file_stat```.
- Add a new getter to 'BundleResource', ```uint32_t GetCrc32() const;```.

## How we teach this

The API documentation should suffice to educate clients on how to use this feature.

## Drawbacks

Exposing an implementation detail such as the checksum does pin the implementation down to using a technology that uses crc32 checksums. If CppMicroServices ever wants to use another technology to encapsulate bundle resources, that may conflict with the information that has been made public to clients. For example, what if the technology does not use crc32 checksums?

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

None.
