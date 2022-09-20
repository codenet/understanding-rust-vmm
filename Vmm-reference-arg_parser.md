# [Arg Parser](https://github.com/codenet/vmm-reference/blob/main/src/vmm/src/config/arg_parser.rs)

## Usage 
`Arg Parser` is used to read the command line arguments, which will be given for initializing the vmm. These arguments are passed to the vmm_config. The vmm config will check if these arguments are correct or not and accordingly successfully configure the vmm or throw an error. To know that there are no errors in giving the command line arguments itself, we need to make the arg_parser robust for handling such input errors.

## Code explanation:

```rs
use std::collections::HashMap;
use std::fmt;
use std::str::FromStr;
```

Importing the built-in Data Structures Hash maps, fmt for formatting and displaying the string, and FromStr to parse the string.


```rs
pub(super) enum CfgArgParseError {
    ParsingFailed(&'static str, String),
    UnknownArg(String),
}
```
These represent the two types of errors the arg parser will handle.  
1. **ParsingFailed:** This error will be thrown if there is an issue with the parsing of known parameters. The value of the parameter or no value is passed.
2. **UnknownArg:** For unknown arguments given by the user. This will be thrown if some unknown parameters have been passed.

```rs 
impl fmt::Display for CfgArgParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::ParsingFailed(param, err) => {
                write!(f, "Param '{}', parsing failed: {}", param, err)
            }
            Self::UnknownArg(err) => write!(f, "Unknown arguments found: '{}'", err),
        }
    }
}
```

`match` compares which enum type is called. As discussed above, there are two functions for handling errors.  
1. If `CfgArgParseError::ParsingFailed` is called, then the match will switch to the parsing failed function and print the message describing at which parameter we have the error and what the error message is.
2. If `CfgArgParseError::UnknownArg` is called, then match will switch to the Unknown Arg function and simply print the error message. We have probably given some parameters which are not required.

```rs
pub(super) struct CfgArgParser {
    args: HashMap<String, String>,
}
```
This will contain all the arguments. The parameter name is a string key and it's value is a String. They will be stored in a hash map. Next they implemented the CfgArgParser Class, which contains the following functions: new, value_of, all_consumed

```rs 
pub(super) fn new(input: &str) -> Self {
    let args = input
        .split(',')
        .filter(|tok| !tok.is_empty())
        .map(|tok| {
            let mut iter = tok.splitn(2, '=');
            let param_name = iter.next().unwrap();
            let value = iter.next().unwrap_or("").to_string();
            (param_name.to_lowercase(), value)
        })
        .collect();
    Self { args }
}
```
We are splitting the arguments by commas and then filtering out the empty tokens (since they have no information). splitn will separate the values using the equal to sign and return 2 substrings from the given token. The first value is the parameter name, and the second value is the value of the parameter. These key-value pairs are inserted into the hashmaps and returned by the function.

```rs
pub(super) fn value_of<T: FromStr>(
    &mut self,
    param_name: &'static str,
) -> Result<Option<T>, CfgArgParseError>
where
    <T as FromStr>::Err: fmt::Display,
{
    match self.args.remove(param_name) {
        Some(value) if !value.is_empty() => value
            .parse::<T>()
            .map_err(|err| CfgArgParseError::ParsingFailed(param_name, err.to_string()))
            .map(Some),
        _ => Ok(None),
    }
} 
```
This will read the value of the parameter from the Hashmap and also delete it from the Hashmap at the same time. If the value is empty, then the ParsingFailed error is thrown, telling the user that there is some error with accessing this parameter.


```rs
pub(super) fn all_consumed(&self) -> Result<(), CfgArgParseError> {
    if self.args.is_empty() {
        Ok(())
    } else {
        Err(CfgArgParseError::UnknownArg(self.to_string()))
    }
}
```
This checks if all the arguments have been consumed or not. If there are some arguments that are left to use, it means they are extra, or some information is still not used. In both cases, an unknown arg error is thrown.

```rs 
impl fmt::Display for CfgArgParser {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "{}",
            self.args
                .keys()
                .map(|val| Val.as_str())
                .collect::<Vec<&str>>()
                .join(", ")
        )
    }
}
```
This will display the arguments that have been collected in the `args` Hashmap.


```rs 
mod tests {
    use super::*;
    use std::num::NonZeroU8;
    use std::path::PathBuf;

    #[test]
    fn test_cfg_arg_parse() -> Result<(), CfgArgParseError> {
        let input_params = "path=/path,string=HelloWorld,int=123,u8=1";
        let mut arg_parser = CfgArgParser::new(input_params);

        // No parameter was consumed yet
        assert!(arg_parser.all_consumed().is_err());

        assert_eq!(
            arg_parser.value_of::<PathBuf>("path")?.unwrap(),
            PathBuf::from("/path")
        );
        assert_eq!(
            arg_parser.value_of::<String>("string")?.unwrap(),
            "HelloWorld".to_string()
        );
        assert_eq!(
            arg_parser.value_of::<NonZeroU8>("u8")?.unwrap(),
            NonZeroU8::new(1).unwrap()
        );
        assert_eq!(arg_parser.value_of::<u64>("int")?.unwrap(), 123);

        // Params now is empty, use the default instead.
        let default = 12;
        assert_eq!(arg_parser.value_of("int")?.unwrap_or(default), default);

        // Params is empty and no Default provided:
        assert!(arg_parser.value_of::<u64>("int")?.is_none());

        // All params were consumed:
        assert!(arg_parser.all_consumed().is_ok());

        let input_params = "path=";
        assert!(CfgArgParser::new(input_params)
            .value_of::<String>("path")?
            .is_none());

        Ok(())
    }
}
```
This is a sample code to test our arg parser. We have given an input that contains different parameters and is comma-separate from each other. The values of the parameters are written ahead of them after the equal to sign. Then we checked out args data using assert statements. The assert statements work as follows:  

1. 
```rs
assert!(arg_parser.all_consumed().is_err());
``` 
First we will check if all the parameters have been consumed or not. We are asserting that some error should be thrown in this case. This is true, an error will be thrown because we have not used any parameters yet and they are present in the args. Hence this assertion is true.

2. 
```rs 
assert_eq!(
    arg_parser.value_of::<PathBuf>("path")?.unwrap(),
    PathBuf::from("/path")
);
assert_eq!(
    arg_parser.value_of::<String>("string")?.unwrap(),
    "HelloWorld".to_string()
);
assert_eq!(
    arg_parser.value_of::<NonZeroU8>("u8")?.unwrap(),
    NonZeroU8::new(1).unwrap()
);
assert_eq!(arg_parser.value_of::<u64>("int")?.unwrap(), 123);
```
assert_eq checks whether the 2 values written are equal or not. All of these statements are true. The value of the "path" argument is "/path" as passed in the input string. Similarly, the value of "string" is "HelloWorld", "u8" is 1 and the integer value of "int" is 123. Hence, all of these assertions are true. Also, by this time, all the parameters in the args have been consumed.

3. 
```rs 
let default = 12;
assert_eq!(arg_parser.value_of("int")?.unwrap_or(default), default);
```
Now, since we have consumed the "int" parameter, this means it is not present in the args. So a default value can be provided. This assertion checks if the default works correctly or not.

4.
```rs 
assert!(arg_parser.value_of::<u64>("int")?.is_none());
```
If no default value is provided, it should return none, and we are checking if it is none using "is_none". Hence, this assertion is also true.

5.
```rs 
assert!(arg_parser.all_consumed().is_ok());
```
Since all the params have been consumed by now, this will return true. Hence the is_ok() will return true, and our assertion is correct.

6. 
```rs
let input_params = "path=";
assert!(CfgArgParser::new(input_params)
    .value_of::<String>("path")?
    .is_none());
```
Now we are adding a new parameter into the args map. Notice that we have not given any value to the parameter. So the value should be none. We are asserting that none is returned whether it is true or not. Hence, our assertion is true.

All our assertions are true, and the test arg parser successfully quits and returns OK(()).
