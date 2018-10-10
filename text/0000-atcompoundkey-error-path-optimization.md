- Start Date: 2018-10-05
- RFC PR: 
- CppMicroServices Issue: https://github.com/CppMicroServices/CppMicroServices/issues/302

# Improving the error path performance for AtCompoundkey API

## Motivation

Using AtCompoundKey with non-existent keys produces an exception. If the API is used extensively, the application performace is degraded as reported by a user. Users prefer not to pay for the cost of exception handling in case a compound key is not found in the AnyMap.

## Description of the Problem

AnyMap provides a convenience API to retrieve objects from the map using compound keys with dot notation. The API is very nifty and saves users from writing redundant code to parse through the nested maps. However, the performance of the error path is significantly higher than for the happy path in this API. This have resulted in performace degradation in some use cases. The performace hit is due to C++ exception handling mechanism. Although AtCompoundKey is similar to the STL map's `at` API, STL maps have other methods such as the `find` which can be used to efficiently access an element in the map without having to deal with the exceptions.

The following graph shows the difference between the performance of happy path and error path in AtCompoundKey API

<div>
    <a href="https://plot.ly/~karthikreddy09/1/?share_key=siExwBF3DybX1HhrW4dHe0" target="_blank" title="Plot 1" style="display: block; text-align: center;"><img src="https://plot.ly/~karthikreddy09/1.png?share_key=siExwBF3DybX1HhrW4dHe0" alt="Plot 1" style="max-width: 100%;width: 600px;"  width="600" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="karthikreddy09:1" sharekey-plotly="siExwBF3DybX1HhrW4dHe0" src="https://plot.ly/embed.js" async></script>
</div>


## Option 1 - Modify existing API to return a empty const Any& if the key is not found
Remove the non-const version of the API and retain just the const version. Return empty `Any` by reference if the key is not found

<strike>Any& AtCompoundkey(const std::string& key);</strike>

`const Any& AtCompoundKey(const std::string& key) const;`

### Pros
* Addresses the performance problem. The happy path and the error path have similar performance. 

### Cons
* Happy path has a slight performance decrease due to the use of 'find' API instead of the 'at' API
* Although the API returns a const reference, the user could add back a non-const version by const_cast'ing the returned ref, which will break every subsequent call. Infact the current non-const API does exactly this, which is ikky.
* The information regarding the error conditions is lost. However, do the users really care about the information is a puzzle.
* If the BundleManifest class is modified to handle "null" values in the manifest, this API has to change.  

## Option 2 - Modify the current API to return an Any object by value
Return an empty `Any` object if the key is not found

```
Any AtCompoundKey(const std::string& key);
```

### Pros
* Addresses the performance problem. The happy path and the error path have similar performance. 

###Cons
* Returning by value results in copies of the returned object which could be costly, specially if this API is used to retrieve intermediate maps of larger size.
* It is not clear how a null value from a manifest file is treated. specifying "null" is legal in JSON. Current implementation of bundle metadata parser ignores the entries with null value. However, we cannot assume this will not change in the future.
* The information regarding he error conditions is lost. However, do the users really care about the information is a puzzle.

## Option 2.1 (Preferred) - Add an overload with a default value parameter 
Add an overload with the following signature:

```
Any AtCompoundKey(const std::string& key, Any defaultvalue) const;
```

Return the user provided default value if the key is not found in the map.

### Pros
* Addresses the performance problem. The happy path and the error path have similar performance. 
* Backwards compatible. Since this API has different number of arguments, this API can be added as an overload set. This allows us to let users choose the right API based on their performance and error handling requirements. Also, lets us deprecate the old one if no use-cases exist.

### Cons
* User has to provide an extra argument for the default value.
* The information regarding the error conditions is lost. However, if the users care about the error conditions, they can use the overload version that throws.

## Option 3 - Expose the bundle's metadata as a third party JSON object.
Remove the `AtCompoundKey` API and expose the bundle's metadata in the form of an object of a JSON parsing library. E.g, rapidjson::Document from the RapidJson library

```
rapidjson::Document Bundle::GetHeaders();
```

### Pros
* Users can use the JSON object to write their custom parsing functions or pass around JSON objects in their code 

### Cons
* Users have to parse through the JSON tree to retrieve the values. 
* The team currently using AtCompoundKey does not like this option. They do not want to deal with the any_cast at each level of the map

## Option 4

Add a "KeyExists" API which returns a bool to signify the existence of the entry.

```
if(map.KeyExists("com.foo.bar"))
{
  auto val = map.AtCompoundKey("com.foo.bar");
}
```

### Pros
* Similar to the STL map's "count" API
* Backwards compatible

### Cons
* Users have to call this API before calling the existing AtCompoundKey API. The resulting code would require parsing the dot notation key twice which may not be desirable.


## Performance Comparison of Options

Here's a graph showing the performance of various options proposed above

<div>
    <a href="https://plot.ly/~karthikreddy09/3/?share_key=ZLPmKjBwUYfJ4TxCPUemBW" target="_blank" title="Plot 3" style="display: block; text-align: center;"><img src="https://plot.ly/~karthikreddy09/3.png?share_key=ZLPmKjBwUYfJ4TxCPUemBW" alt="Plot 3" style="max-width: 100%;width: 600px;"  width="600" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="karthikreddy09:3" sharekey-plotly="ZLPmKjBwUYfJ4TxCPUemBW" src="https://plot.ly/embed.js" async></script>
</div>
---
<div>
    <a href="https://plot.ly/~karthikreddy09/5/?share_key=k4CBKqmVpgN46clMnusUv2" target="_blank" title="Plot 5" style="display: block; text-align: center;"><img src="https://plot.ly/~karthikreddy09/5.png?share_key=k4CBKqmVpgN46clMnusUv2" alt="Plot 5" style="max-width: 100%;width: 600px;"  width="600" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="karthikreddy09:5" sharekey-plotly="k4CBKqmVpgN46clMnusUv2" src="https://plot.ly/embed.js" async></script>
</div>
### Manifest file used for gathering the above metrics.
```
{
		"relativelylongkeyname_element": true,
		"relativelylongkeyname_map": {
			"relativelylongkeyname_element": true,
			"relativelylongkeyname_map": {
				"relativelylongkeyname_element": true,
				"relativelylongkeyname_map": {
					"relativelylongkeyname_element": true,
					"relativelylongkeyname_map": {
						"relativelylongkeyname_element": true,
						"relativelylongkeyname_map": {
							"relativelylongkeyname_element": true,
							"relativelylongkeyname_map": {
								"relativelylongkeyname_element": true,
								"relativelylongkeyname_map": {
									"relativelylongkeyname_element": true,
									"relativelylongkeyname_map": {
										"relativelylongkeyname_element": true,
										"relativelylongkeyname_map": {
											"relativelylongkeyname_element": true,
											"relativelylongkeyname_map": {
												"relativelylongkeyname_element": true,
												"relativelylongkeyname_map": {
													"relativelylongkeyname_element": true,
													"relativelylongkeyname_map": {
														"relativelylongkeyname_element": true,
														"relativelylongkeyname_map": {
															"relativelylongkeyname_element": true,
															"relativelylongkeyname_map": {
																"relativelylongkeyname_element": true,
																"relativelylongkeyname_map": {
																	"relativelylongkeyname_element": true,
																	"relativelylongkeyname_map": {
																		"relativelylongkeyname_element": true,
																		"relativelylongkeyname_map": {
																			"relativelylongkeyname_element": true,
																			"relativelylongkeyname_map": {
																				"relativelylongkeyname_element": true,
																				"relativelylongkeyname_map": {
																					"relativelylongkeyname_element": true,
																					"relativelylongkeyname_map": {
																						"relativelylongkeyname_element": true,
																						"relativelylongkeyname_map": {
																							"relativelylongkeyname_element": true,
																							"relativelylongkeyname_map": {
																								"relativelylongkeyname_element": true,
																								"relativelylongkeyname_map": {
																									"relativelylongkeyname_element": true,
																									"relativelylongkeyname_map": {}
																								}
																							}
																						}
																					}
																				}
																			}
																		}
																	}
																}
															}
														}
													}
												}
											}
										}
									}
								}
							}
						}
					}
				}
			}
		}
	}
}
```

## Unresolved questions

> Will we ever support null values in bundle's manifest.json file?
