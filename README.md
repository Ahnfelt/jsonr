# JSONR (.jsonr)

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

A JSONR file consists of zero or more parameter definitions followed by either a value or zero or more fields. The EBNF grammar is as follows:

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


## Banner

![JSONR is JSON, but with concise syntax, simpler schemas and efficient binary encoding.](https://repository-images.githubusercontent.com/234411191/97ebc200-3ac5-11ea-9ced-272a44d3bbff)


# Why?

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


# JSONR implementations

JSONR is very new and very much in progress, but there's already an impelementation being fleshed out by `somebody1234`:
https://glitch.com/~jsonr


# Features in the textual format

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


# Schemas

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
| `object(of: ...)` | A JSON object with the given fields. If `required: [...]` is specified, only those fields are required. If `map: ...` is specified, arbitrary other fields may be specified as long as their values adhere to the given type, and `of: ...` is then optional. If `nulls: true` is specified, 'null' is also an accepted value for the optional fields. |
| `variant()` | If `object: {...}` is specified, a JSON object whose sole field is one of the given options. If `array: {...}` is specified, a JSON array whose first element is one of the given options |
| `tuple(of: ...)` | An array with elements of different types in the order specified. If `required: n` is specified, only the first `n` elements are required |
| `data()` | A data URL. If `mediatype: ...` is specified (suffixed with `;base64` if applicable), the value must only include the part after the `,` |
| `any()` | Any JSON value |
| `type(of: ...)` | References a type defined in a schema by name. If `schema: ...` is specified, the type is pulled from the schema imported by the specified name, and `of: ...` becomes optional, defaulting to the primary type of the refernced schema |

Note: 54 bit integers fit accurately in a double precision floating point number, and are thus easily consumable in languages such as JavaScript and Lua.

All the types may specify `nullable: true`, which means that `null` is an accepted value as well. The exception is `any()`, which is always nullable.

Variant types must specify either `object: {tag1: typeA, tag2: typeB, ...}` or `array: {tag1: [typeX, typeY, ...], tag2: ..., ...}`, but not both. 


## Schema top level fields

| Field | Description |
| :------ | :------------ |
| `_: ...` | Specifies the name of the primary type of the schema (required) |
| `_imports: {my_name: "...", ...}` | Imports other schemas, giving them a name that can be used in the `schema` field of `type(...)` |
| `_documentation: {my_type: "...", ...}` | Documentation for each type in the schema |
| `_hints: {my_type: {...}, ...}` | Key/value hints on how to represent each type in programming languages |
| `_strings: ["...", ...]` | An array of up to 2048 strings of at most 64 UTF-8 bytes each for the static dictionary in the binary serialization |

In the `_documentation` and `_hints` fields, the reserved key `_` is a documentation/hint entry for the schema itself.


# Binary encoding

JSONR specifies a binary encoding for JSONR values (and thus also JSON values) that can be optionally used in place of the textual format to reduce space usage.

It may be combined with a schema to reduce space usage further.

The format is **forward compatible**, meaning you can add optional fields to the schema and still be able to decode old files.


## Dictionaries

| Dictionary | Description |
| :------ | :------------ |
| Static dictionary | Populated by the `_strings` field in the schema (if any) |
| Dynamic dictionary | The 128 most recently encountered strings |

The static dictionary consists of up to 2048 strings of at most 64 bytes each.

The dynamic dictionary ignores empty strings and strings that are longer than 128 bytes. It's initialy populated with single character strings for each of the 128 ASCII codes.


## Header

The binary encoding starts with the 32 bit magic number `\211 J R b` for "JSONR (binary encoding)". Then comes a single byte version number, which must currently be the bits `00000001`. Then comes two bytes that is the number of static dictionary entries (max 2048), and then four bytes that is the CRC-32 of those strings. Everything is in network byte order. 

The last thing in the file is the encoded value, described by the table below.


## Value encoding

| Bits | Description |
| :------ | :------------ |
| `0xxx xxxx` | Dynamic dictionary entry `x` |
| `10xx xxxx` | Integer `x-16` |
| `1100 0xxx  xxxx xxxx` | Static dictionary entry `x` |
| `1100 1xxx  xxxx xxxx` | An 11 bit non-negative integer `x+1008` |
| `1101 0000` | `null` |
| `1101 0001` | `false` |
| `1101 0010` | `true` |
| `1101 0100  (then 32 bits of x)` | A 32 bit integer `x-2^31` |
| `1101 0101  (then 32 bits of x)` | A 32 bit floating point number `x` |
| `1101 0110  (then 64 bits of x)` | A 64 bit floating point number `x` |
| `1101 0111  0000 xxxx  (then 48 more bits of x)` | Array of size `x` |
| `1101 0111  0001 xxxx  (then 48 more bits of x)` | Object of size `x` |
| `1101 0111  0010 xxxx  (then 48 more bits of x)` | String of size `x` |
| `1101 0111  0011 xxxx  (then 48 more bits of x)` | Data of size `x` |
| `1101 1000  (then 16 bits of x)` | Array of size `x` |
| `1101 1001  (then 16 bits of x)` | Object of size `x` |
| `1101 1010  (then 16 bits of x)` | String of size `x` |
| `1101 1011  (then 16 bits of x)` | Data of size `x` |
| `1110 0xxx` | Array of size `x` |
| `1110 1xxx` | Object of size `x` |
| `1111 0xxx` | String of size `x` |
| `1111 1xxx` | Data of size `x` |

 * Arrays are followed by `x` values, each an element in the array.
 * Objects are followed by `2*x` values, each pair a key/value in the object. If any static dictionary keys are used, those keys must come in ascending order of static dictionary index.
 * Strings are followed by `x` UTF-8 bytes.
 * Datas are followed by a null or string value with the mediatype (suffixed with `;base64` if applicable), and then `x` bytes. If null, the mediatype must be specified by the schema. The `x` bytes are binary data if the mediatype has the `;base64` suffix, and UTF-8 bytes otherwise. Note the mediatype string participates in the dictionaries on the same terms as all other strings.

The encoding uses network byte order. If no schema is specified, it's assumed to be `_: "dynamic", dynamic: any()`.
