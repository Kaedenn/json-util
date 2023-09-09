# JSON Utility Script

This repository provides `json`, a self-contained Python script intended to simplify command-line JSON activities. This tool's primary intent is to parse and interrogate JSON streams in a way convenient for scripting.

Key features include:
 * Formatting unformatted JSON with indentation, sorting
 * Detailed information about JSON parse failures
 * Data interrogation and access
 * Value formatting for downstream use

## Usage

```
json
  [-h]
  [-l,--lookup PATTERN]
  [-e,--escape | -p,--print]
  [-1,--oneline | -i,--indent N]
  [-s,--sort-keys]
  [-w,--width N]
  [-C,--no-color]
  [-v,--verbose]
  [path [path ...]]
```

 * `-h` - print the help message and exit
 * `-l PATTERN` or `--lookup PATTERN` - extract data from the json (see <strong>Value Lookup</strong> below)
 * `-e` or `--escape` - escape the output for later JSON parsing
 * `-p` or `--print` - print the output with no escaping
 * `-1` or `--oneline` - output on a single line without indentation
 * `-i N` or `--indent N` - indent output with this many spaces (default: 2)
 * `-s` or `--sort-keys` - sort JSON keys
 * `-w N` or `--width N` - amount of context to include in error messages (default: 8)
 * `-C` or `--no-color` - disable color formatting in error messages
 * `-v` or `--verbose` - output diagnostic information
 * `[path [path ...]]` - file(s) to process; defaults to stdin if omitted

The following argument combinations are mutually-exclusive:
 * `-e` or `--escape` with `-p` or `--print`
 * `-1` or `--oneline` with `-i N` or `--indent N`

The following arguments apply only to error messages:
 * `-w N` or `--width N`
 * `-C` or `--no-color`

The following arguments are ignored if `-p` or `--print` are given:
 * `-1` or `--oneline`
 * `-i N` or `--indent N`
 * `-s` or `--sort-keys`
 * `-e` or `--escape`

## Value Lookup

The `-l,--lookup PATTERN` argument permits extracting data from a JSON stream without needing to write extra code. `PATTERN` is an XPath-inspired sequence of substrings separated by slashes `/`.

This tool is intelligent enough to interpret numeric substrings as indexes when examining an array and keys when examining an object.

 * `-l data` is interpreted as `json["data"]`
 * `-l data/0` is interpreted as `json["data"]["0"]` if `json["data"]` is an object.
 * `-l data/0` is interpreted as `json["data"][0]` if `json["data"]` is an array.

Value extraction logic is roughly as follows:
* If the data is `String` or `Array`:
  * If the substring is numeric, obtain the element at that index.
    * If the index is negative, then get that index relative to the end of the string or array.
  * If the substring is a slice (`<start>:<stop>`), obtain either the substring or sub-array between those two indexes.
    * The slice `<start>:` is interpreted as "all elements at or after index `<start>`". The index can be negative.
    * The slice `:<stop>` is interpreted as "all elements before `<stop>`". The index can be negative.
    * The slice `:` resolves to the entire sequence and is effectively a no-op.
  * Otherwise, issue a warning and return the data as-is.
* If the data is an `Object`:
  * If the data contains the substring as a key, return that key's value.
  * Otherwise, issue a warning and return the data as-is.
* If the data is an `Integer` or `Boolean`, issue a warning and return the data as-is.

A substring is considered numeric if it contains only digits between `0` and `9`, optionally with a leading hyphen `-`.

### Error Behavior

Failure applying a pattern is not a fatal error. If a pattern cannot be applied, either because it's invalid or the object doesn't have anything matching the pattern, this program issues a warning and leaves the object unchanged.

### Examples

Assume the following JSON input:
```json
{
  "data": [
    {
      "name": "value",
      "0": "https://example.com/value/0",
      "1": "https://example.com/value/1"
    },
    {
      "name": "value-2",
      "0": "https://example.com/value-2/0",
      "1": "https://example.com/value-2/1"
    }
  ]
}
```

| Lookup Pattern | Python Equivalent | Final Value |
| -------------- | ----------------- | ----------- |
| `data/0` | `json["data"][0]` | `{"name": "value", "0": "https://example.com/value/0", "1": "https://example.com/value/1"}` |
| `data/1` | `json["data"][1]` | `{"name": "value-2", "1": "https://example.com/value-2/0", "1": "https://example.com/value-2/1"}` |
| `data/-1` | `json["data"][-1]` | `{"name": "value-2", "1": "https://example.com/value-2/0", "1": "https://example.com/value-2/1"}` |
| `data/0/name` | `json["data"][0]["name"]` | `"value"` |
| `data/0/0` | `json["data"][0]["0"]` | `"https://example.com/value/0"` |
| `data/1/0` | `json["data"][1]["0"]` | `"https://example.com/value-2/0"` |
| `data/-1/name` | `json["data"][-1]["name"]` | `"value-2"` |
| `data/0/name/0` | `json["data"][0]["name"][0]` | `"v"` |
| `data/0/name/1:` | `json["data"][0]["name"][0]` | `"alue"` |

## Return Codes

 * `0` on success
 * `1` if parsing any input stream raised a `JSONDecodeError`

## Future Work

It would be convenient to allow wildcards or multi-matching in `-l,--lookup`. For instance, given the following data:
```json
{
  "data": [
    {
      "name": "first",
      "value": "example.com"
    },
    {
      "name": "second",
      "value": "example.net"
    },
    {
      "name": "third",
      "value": "example.org"
    }
  ]
}
```
the argument `-l data/*/name` could give `["first", "second", "third"]`, the argument `-l data/1:/name` could give `["second", "third"]`, the argument `-l data/1:/name/0` could give `["f", "s", "t"]`, and so on. Note that the current implementation forbids this, for two reasons:
 1. `data/1:/name/0` attempts to get the index `"name"` of an array, which is invalid.
 2. Assuming we allow a more relaxed pattern-matching approach, `data/1:/name` could give `[entry["name"] for entry in data[1:]]`, but `data/1:/name/0` would only extract the first element of that array.

There would need to be a distinction between "get this value for this item" versus "get this value for all items in this collection".
