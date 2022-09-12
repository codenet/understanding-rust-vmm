# [Resource Download](/vmm-reference/src/utils/src/resource_download.rs)

`resource_download.rs` calls [s3_download.py](/vmm-reference/tests/tools/s3_download.py) to download a resource from resource_manifest.json, which can be found in /vmm-reference/tests/tools/s3 folder.

```rs
pub enum Error {
    DownloadError(String),
}
```
Defining an enum for DownloadError that would be used to return error value in case resource download fails. 

<br>

``` rs
pub fn s3_download(r_type: &str, r_tags: Option<&str>) -> Result<PathBuf, Error>
```

1. `r_type`: argument for the type of the resource {"kernel, "core"}

2. `r_tags`: Optional tags to filter the resources for example: "halt-after-boot: true",  "image format: elf" etc.

3. `Result<PathBuf, Error>`: Result<T,E> is a type that returns type T value on success, type E error otherwise. 

<br>

```rs
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

1. `dld_script`: variable to store the relative path to s3_download.py

2. `output`: store the output of the command run to download resources of the given type and according to the given tags, `Command` is an rust api for a process builder.

3. if the download fails for some reason the function returns the error received

4. if the download is a success the function returns the path of the resultant image, the path can be used to run the image. `PathBuf` is a rust struct to store mutable paths which can be dereferenced.

<br>

```rs
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

Unit test for checking cases when s3_download returns errors

The above function checks for error in case of missing resource type parameter(which is a required parameter) and it also checks for error in case of no resource is found to download.

<br>

## Usage

1. Multiple usages in integration tests: vmm-reference/src/vmm/tests/integration_tests.rs

```rs
fn test_dummy_vmm_elf() {
    let tags = r#"
    {
        "halt_after_boot": true,
        "image_format": "elf"
    }
    "#;

    let elf_halt = s3_download("kernel", Some(tags)).unwrap();
    run_vmm(elf_halt);
}
```
2. Multiple Usages in vmm-reference/src/vmm/src/lib.rs

```rs
fn default_bzimage_path() -> PathBuf {
    let tags = r#"
    {
        "halt_after_boot": true,
        "image_format": "bzimage"
    }
    "#;
    s3_download("kernel", Some(tags)).unwrap()
}
```
