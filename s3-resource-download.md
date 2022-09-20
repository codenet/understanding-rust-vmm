# [VMM Resource Downloader](../src/utils/src/resource_download.rs)

The `VMM Resource Downloader` is a utility present in the VMM repository that is used to download resources required for tests and returns the absolute path of the downloaded resource to `stdout`.

This is done using the [s3_downloads.py](../tests/tools/s3_download.py) file and the resource list present in [resource_manifest.json](../tests/tools/s3/resource_manifest.json) which contains the description (resource name and types) for several kernel images and `initrd` files.

## Code Description

The resource_download first imports `PathBuf` and `Command`. 

```
use std::path::PathBuf;
use std::process::Command;
```

`PathBuf` is used for mutating paths in place with functions like `push` and `set_extension`. 

`Command` is a process builder, providing fine-grained control over how a new process should be spawned. Command can be reused to spawn multiple processes. The builder methods change the command without needing to immediately spawn the process.

Next, is defined an enum `Error` that is used for error handling, in case there are errors encountered/some operation fails during fetching a resource from S3.

```
#[derive(Debug, PartialEq)]
pub enum Error {
    DownloadError(String),
}
```
The function `s3_download` does the heavy lifting, which downloads the requested resource and returns its absolute path. It takes two parameters: 

- `r_type: &str` - this is used to describe the resource type to be downloaded, e.g. `kernel`, `disk` etc.
- `r_tags: Option<&str>` - this takes optional tags to filter resources

The return value is of type `Result<PathBuf, Error>`, which is either a path to the resource or an `Error` indicating failure.

`dld_script` is used to store the path to the download script present in `~/tests/tools/`.
Then, `Command` is used to execute the Python download script along with `r_type` and `r_tags` provided as options parameters to `s3_download` and its result is stored in `output`.

If the command fails, i.e. `!output.status.success()` is true, an `Error` is returned.

Else, the command succeeds and the output of the command is taken from stdout, formatted and returned as a `PathBuf`.

```
pub fn s3_download(r_type: &str, r_tags: Option<&str>) -> Result<PathBuf, Error> {
    let dld_script = format!(
        "{}/../../tests/tools/s3_download.py",
        env!("CARGO_MANIFEST_DIR")
    );

    let output = Command::new(dld_script.as_str())
        .arg("-t")
        .arg(r_type)
        .arg("--tags")
        .arg(r_tags.unwrap_or("{}"))
        .arg("-1")
        .output()
        .expect("failed to execute process");

    if !output.status.success() {
        return Err(Error::DownloadError(
            String::from_utf8(output.stderr).unwrap(),
        ));
    }

    let res: String = String::from_utf8(output.stdout)
        .unwrap()
        .split('\n')
        .map(String::from)
        .next()
        .ok_or_else(|| Error::DownloadError(String::from("Not found.")))?;
    Ok(PathBuf::from(res))
}
```
This is also accompanied by a unit test that checks whether the `s3_download` fails succesfully on certain inputs, like a blank string or an invalid resource type.

```

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_error_cases() {
        assert!(matches!(
            s3_download("", None).unwrap_err(),
            Error::DownloadError(e) if e.contains("Missing required parameter")
        ));

        assert!(matches!(
            s3_download("random", None).unwrap_err(),
            Error::DownloadError(e) if e.contains("No resources found")
        ));
    }
}
```