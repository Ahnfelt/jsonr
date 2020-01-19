#  JSONR (.jsonr)

JSONR is JSON, but with concise syntax, simpler schemas and efficient binary encoding.


## Example

    # This is a .jsonr file
    
    $database_server: "localhost"

    path: "/tmp"

    database: {
        server: $database_server
        port: 2345
        max_connections: 5000
        enabled: true
    }

    hosts: [
      "tiger"
      "bobcat"
    ]

    query: term("name": "Mike")


## Syntax

A JSONR file consists of zero or more parameter definitions followed by either a value or zero or more fields. 

```
file = {'$' string ':' value [',']} (value | fields)
parameter = '$' string ['(' fields ')']
value = parameter | json_string | json_number | object | array | 'true' | 'false' | 'null'
string = /[A-Za-z_][A-Za-z0-9_]*/ | json_string
object = '{' fields '}' | string '(' (value | fields) ')'
array = string? '[' {value [',']} ']'
fields = {string ':' value [',']}
```

The `json_number` and `json_string` rules are exactly as JSON numbers and JSON strings respectively. Whitespace is as in JSON and comments begin with `#` and last to the end of the file or the end of the line `\n`, whichever comes first.


## Implementations

JSONR is very new and very much in progress, but there's already an impelementation being fleshed out by `somebody1234`:
https://glitch.com/~jsonr


## Why?

JSON is a great format, but it has a number of pain points. 

It's somewhat cumbersome to write JSON by hand due to the required quotes around field names and the inability to move things around in the file without editing commas. 

The lack of comments means that you can't specify why something is like it is in a JSON configuration file.

Projects that use JSON for configuration often have vague documentation, with example snippets of JSON you don't know where to put in your own configuration file, and no way to check if it's valid. 

For APIs, JSON files are quite wasteful, since field names are constantly repeated.

In addition, it's common to end up with a bunch of configuration files that are tiny variations of each other for slightly different use cases.

JSONR aims to remove all of these pain points, while strictly adhering to the JSON value model.


## Why not YAML, TOML, XML?

These formats have complex syntaxes that are hard to parse, and they each deviate from the simple JSON value model. 

In contrast, JSONR values and JSON values are one and the same, and the syntax is a conservative extension of JSON.


## Parameters

Parameters allow a JSONR file to be a template for e.g. configurations. They are declared in the top of the file with the syntax `$my_parameter: default_value`, where `default_value` is the default value of the parameter. A parameter must not be declared twice. 

Immediately after its declaration, a parameter can be used in place of a value. The value is then the value of the parameter.

It's possible to branch on parameters, using the syntax `$my_parameter(value1: 10, value2: 20)`. This means that if `$my_parameter` is the string `"value1"`, return `10`. Otherwise, if it's `"value2"`, return `20`. Otherwise, fail.

When parsing a JSONR file with parameter declarations, a map of parameter values may be supplied that is then used instead of the default value of the parameter declaration.


## Variants

Variants, also known as tagged unions or sum types, are commonplace in data format specifications. It's a way to distinguish between multiple types by using a tag. There are two dominant ways to encode this in JSON: `{"tag": value}` and `["tag", value1, value2, ...]`. Unfortunately, it can be quite hard to read the resulting JSON. JSONR has syntactic sugar to make it readable.

| Variant shorthand | Equivalent JSON |
| :------ | :------------ |
| `foo()` | `{"foo": {}}` |
| `foo(x: 1, y: 2)` | `{"foo": {"x": 1, "y": 2}}` | 
| `foo(42)` | `{"foo": 42}` | 
| `foo["hello", true]` | `["foo", "hello", true]` | 


## Schemas

Schemas are written in JSONR files, and consist of zero or more fields. Fields that start with `_` are reserved, and the rest of the fields each define a type in the schema. The `_` field specifies the primary type of the schema.

Example:

    _: "configuration"

    configuration: object(of: {
        title: string()
        owner: object(of: {
            name: string()
            dob: object(of: {
                year: int()
                month: int()
                day: int()
            })
        }, required: ["name"])
        query: type(of: "query")
        body: string()
    })

    query: variant(object: {
        term: map(of: string())
        bool: type(of: "bool_query")
        match_all: object(of: {})
    })
    
    bool_query: object(of: {
        must: array(of: type(of: "query"))
        must_not: array(of: type(of: "query"))
        should: array(of: type(of: "query"))
        minimum_should_match: int()
    }) 

Types:

| Type | Description |
| :------ | :------------ |
| `int()` | A 54 bit signed integer (see note). The value can be constrained by specifying `minimum: ...` and/or `maximum: ...` |
| `float()` | A double precision floating point number. The value can be constrained by specifying `minimum: ...` and/or `maximum: ...` |
| `bool()` | Either `true` or `false` |
| `string()` | A JSON string. If `of: ...` is specified, it must be one of the given strings. If `pattern: ...` is specified, the string must match that regular expression |
| `array(of: ...)` | A JSON array with elements of the given type. If `omit: true` is specified, a non-array value is also accepted as if it was a single element array of that value. |
| `object(of: ...)` | A JSON object with the given fields. If `required: [...]` is specified, only those fields are required. If `other: ...` is specified, arbitrary other fields may be specified as long as their values adhere to the given type. If `nulls: true` is specified, 'null' is also an accepted value for the optional fields |
| `variant()` | If `object: {...}` is specified, a JSON object whose sole field is one of the given options. If `array: {...}` is specified, a JSON array whose first element is one of the given options |
| `tuple(of: ...)` | An array with elements of different types in the order specified. If `required: n` is specified, only the first `n` elements are required |
| `binary()` | A data URL. If `mediatype: ...` is specified (suffixed with `;base64` if applicable), the value must only include the part after the ',' |
| `any()` | Any JSON value. If `of: ...` is specified, the value must adhere to one of the given types |
| `type(of: ...)` | References a type defined in a schema by name. If `schema: ...` is specified, the type is pulled from the schema imported by the specified name, and `of: ...` becomes optional, defaulting to the primary type of the refernced schema |

Note: 54 bit integers fit accurately in a double precision floating point number, and are thus easily consumable in languages such as JavaScript and Lua.

All the types may specify `nullable: true`, which means that `null` is an accepted value as well.

Variant types must specify either `object: {tag1: typeA, tag2: typeB, ...}` or `array: {tag1: [typeX, typeY, ...], tag2: ..., ...}`, but not both. 

If `of: ...` is specified in `any()`, no two options may accept the same kind of JSON value (number/string/bool/null/object/array). If nullable, none of the types may accept `null`.


## Schema top level fields

| Field | Description |
| :------ | :------------ |
| `_: ...` | Specifies the name of the primary type of the schema (required). |
| `_imports: {my_name: "...", ...}` | Imports other schemas, giving them a name that can be used in the `schema` field of `type(...)`. |
| `_documentation: {my_type: "...", ...}` | Documentation for each type in the schema. |
| `_hints: {my_type: {...}, ...}` | Key/value hints on how to represent each type in programming languages. |
| `_strings: ["...", ...]` | An array of up to 128 strings that are reserved and prepopulated in the binary encoding dictionary. |

In the `_documentation` and `_hints` fields, the reserved key `_` is a documentation/hint entry for the schema itself.


## Binary encoding

JSONR specifies a binary encoding for JSONR values (and thus also JSON values) that can be optionally used in place of the textual format to reduce space usage.

It may be combined with a schema to reduce space usage further.

A dictionary of the last 256 seen non-empty strings of length 255 or less is maintained by the encoder and decoder, to avoid repeating common strings such as field names in the encoding. Up to 128 of the lower entries in the dictionary may be reserved by the `_strings` field in the schema. The unreserved entries are sorted so that the most recently strings come first. Strings leave the dictionary when they get an index >= 256 due to more recently seen strings.

The format is **forward compatible**, meaning you can add optional fields to the schema and still be able to decode old files.

The binary encoding starts with the 32 bit magic number `\211 J R b` for "JSONR (binary encoding)". Then comes a single byte version number, which must be the bits `00000001`. Then comes a byte that is the number of reserved strings (max 128), and if non-zero, then the CRC-32 of those strings, in network byte order. The last thing in the file is the encoded value, described by the table below.

| Bits | Description |
| :------ | :------------ |
| `1xxx xxxx` | Dictionary entry `x` |
| `0111 0000 1xxx xxxx` | Dictionary entry `128+x` |
| `0111 0001` | `null` |
| `0111 0010` | `false` |
| `0111 0011` | `true` |
| `0111 0100 xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx` | Array of size `x` |
| `0111 0101 xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx` | Object of size `x` |
| `0111 0110 xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx` | String of size `x` |
| `0111 0111 xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx` | Binary of size `x` |
| `0111 1100 xxxx xxxx xxxx xxxx` | Array of size `x` |
| `0111 1101 xxxx xxxx xxxx xxxx` | Object of size `x` |
| `0111 1110 xxxx xxxx xxxx xxxx` | String of size `x` |
| `0111 1111 xxxx xxxx xxxx xxxx` | Binary of size `x` |
| `0100 sxxx` | An integer `x` with sign bit `s` |
| `0101 0000 xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx` | A single precision floating point number `x` |
| `0101 0001 xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx` | A double precision floating point number `x` |
| `0000 xxxx` | Array of size `x` |
| `0001 xxxx` | Object of size `x` |
| `0010 xxxx` | String of size `x` |
| `0011 xxxx` | Binary of size `x` |

 * Arrays are followed by `x` values, each an element in the array.
 * Objects are followed by `x*2` values, each pair a key/value in the object. The keys must be strings.
 * Strings are followed by `x` UTF-8 bytes.
 * Binaries are followed by string value with the mediatype (suffixed with `;base64` if applicable), and then `x` bytes. If the string is empty, the mediatype must be specified by the schema. Note the mediatype string participates in the dictionary on the same terms as all other strings.

The encoding uses network byte order. 
