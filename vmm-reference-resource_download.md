
`resource_download` contains methods for interacting with the resources such as kernel, disk and it returns pathbuf structure on successful reading and error when unsucessful by implementing function call s3_download.

```rs
///     - `r_type`: the resource type; e.g. "kernel", "disk".
///     - `r_tags`: optional tags to filter the resources; e.g. "{\"halt-after-boot\": true}"
pub fn s3_download(r_type: &str, r_tags: Option<&str>) -> Result<PathBuf, Error>
```

This function inturn calls s3_download python file implemented for testing. This file calls s3_download function which inturn call s3_resource_fetchers' download method. This methods creates the download path if not exist already and downloads data from s3 to that location.

NOTE : 
* r_tags are passed as arguments while calling the python file to download resources.
* There may be various resources that satify the given type and tags, in that case depending on the tag call 'first' it is decided to copy only first resource or to copy all the resources.


This file also contains some testcases to check the error messages.
