- Start Date: 2021-04-20
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue:

# Any::FromJSON

## Summary

CppMicroServices provides a container class called **Any** which can hold any kind of data. We provide a way to output the current value held in an Any object as a string in JSON format. 

## Motivation

* No provided way to convert a JSON string into a value usable by CppMicroServices
* Lack of symmetry with provided **Any::ToJSON** method

## Detailed design

```c++
  /**
   * Parse a JSON string and return the results in an Any object. Hierarchical data is stored in
   * maps, and lists are stored in vectors.
   *
   * @throws std::runtime_error with invalid JSON input
   * @param  json a string containing JSON Data
   * @param  use_ci_map_keys a bool which indicates whether or not the keys in any object maps should
   *         be case insensitive. We use case insensitive keys in CppMicroServices, but there are
   *         other uses of this static method that may not need them to be case insensitive.
   * @return an Any object containing the parsed data. 
   */
  static Any FromJSON(std::string const& json, bool use_ci_map_keys = true);
  static Any FromJSON(std::istream& json, bool use_ci_map_keys = true);

```

* Refactor existing parsing logic out of BundleManifest.cpp into utilities to make callable from multiple locations
* Use existing parsing logic to implement Any::FromJSON as a public API.

## How we teach this

* Doxygen comments provide usage information

## Drawbacks

* None.

## Alternatives

* None.

## Unresolved questions

* Should case sensitivity be handled with a default argument?